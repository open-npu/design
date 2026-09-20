# MAC 阵列与数据进入路径

本文描述 **当前 RTL 实际怎么把数送进脉动阵列、阵列里怎么乘加、结果怎么出来**。以 `rtl/src/` 为准，默认 `ARRAY_SIZE=8`、`ACC_WIDTH=44`、计算口 256-bit。

相关文件：

| 模块 | 文件 | 职责 |
|---|---|---|
| 顶层 + SRAM 互联 | `rtl/src/npu_top.v` | IFM / OFM / WGT / PARAM，DMA 与计算分口 |
| 计算序列器 | `rtl/src/npu_compute.v` | 寻址、拼向量、灌权、收 psum、PPU |
| 脉动阵列 | `rtl/src/npu_systolic.v` | 斜切 + PE 网格 + 列向归约 |
| 单个 PE | `rtl/src/npu_pe.v` | 驻留一个权，一拍一次 MAC |
| 宽口 SRAM | `rtl/src/npu_sram_wide.v` | A 口 32-bit（DMA），B 口读 256-bit / 写 32-bit |

层调度、DDR DMA、融合换 bank 在 `npu_ctrl.v` / `npu_dma.v`。本文只在需要时提到它们，不展开控制器。

---

## 1. 它在整颗 NPU 里的位置

```
                    Wishbone 32-bit
         CPU ──slave──▶ CSR ──▶ npu_ctrl ──start/cfg──▶ npu_compute
         DMA ──master─▶ DDR
          │
          │ Port A 32-bit
          ▼
    ┌──────────┬──────────┬──────────┬──────────┐
    │ IFM SRAM │ OFM SRAM │ WGT SRAM │ PARAM    │
    │ u_sram   │ u_sram   │          │ SRAM     │
    │ _act     │ _ofm     │          │          │
    └────┬─────┴────┬─────┴────┬─────┴────┬─────┘
         │ Port B   │ Port B   │ Port B   │ 32-bit
         │ 256-bit  │ 256/32   │ 256-bit  │
         │          │          │          │
         └──────────┴────┬─────┴──────────┘
                         ▼
                   npu_compute
                    │         │
            sa_wgt  │         │ sa_act  (每拍最多 ARRAY 个 DATA_W)
            sa_cmd  │         │
                    ▼         ▼
                 ┌─────────────────┐
                 │  npu_systolic   │  ARRAY × ARRAY PE
                 │  weight-stat.   │
                 └────────┬────────┘
                          │ psum_out  (ROWS 拍后，COLS 路 ACC_W)
                          ▼
                    px_acc_buf (双 bank)
                          ▼
                       npu_ppu × COLS
                          ▼
                      OFM SRAM (32-bit 写)
```

约定：

- **行 r**：展平后的 K 维切片（`k = kh × kw × IC` 里的第 r 个）。
- **列 c**：当前 `oc_group` 里的第 c 个输出通道。
- **`PE[r][c]`** 驻留 `W[k=r][oc=c]`，计算期间权不动（weight-stationary）。
- 峰值：每拍最多 `ARRAY × ARRAY` 次 MAC。默认 8×8 = **64 MAC/拍**。

INT8 / INT16 共用同一套 PE 乘法器（`DATA_WIDTH=16`）。没有「INT8 拆成两路 MAC」的双倍吞吐。

---

## 2. 片上四块 SRAM（数据从哪来）

`SPAD_KB=192` 时深度（32-bit word）：

| Bank | 例化 | 深度 | 约合 | A 口（DMA） | B 口（计算） |
|---|---|---|---|---|---|
| IFM | `u_sram_act` | `SPAD×64` | 48 KB | 32-bit 写入（load act） | **256-bit 读** |
| OFM | `u_sram_ofm` | `SPAD×64` | 48 KB | 32-bit 读出（store） | 256-bit RMW 读 + **32-bit 写** |
| WGT | `u_sram_wgt` | `SPAD×128` | 96 KB | 32-bit 写入 | **256-bit 读** |
| PARAM | `u_sram_param` | `SPAD×16` | 12 KB | 32-bit | 32-bit 读（bias / scale / shift / zp） |

融合层可用 `act_role_swap` 把两块物理 IFM/OFM 对调，下一层直接读上一层写过的 bank，不必 memcpy。

**256-bit beat**：从 `b_addr` 起连续 8 个 32-bit word。INT8 一拍最多 32 个数，INT16 最多 16 个。SRAM 同步读，**发出地址后第 2 拍数据稳定**（issue → wait → 可用）。跳过 wait 会读到上一拍。

布局按 **NHWC**（或 tile 内同等打包）：同一像素的通道连续，下一像素紧挨着。这是 1×1 / 单 tap 能「一拍喂满一行」的前提。

---

## 3. 阵列内部：权驻留 + 行广播 + 列向 psum

### 3.1 一个 PE

```
                    weight_reg  ◀── WGT_LOAD 时锁存
                         │
    act_in ──▶ × ────────┤
                         ▼
              psum_in ──▶ + ──▶ psum_out   (ACC_W=44)
                         │
                    psum_valid_out
```

`npu_pe.v`：COMPUTE 且 `valid_in` 时

```
psum_out <= psum_in + act_in * weight_reg
```

PE **不跨像素自累加**。K 维归约完全靠列上的 psum 链。多 pass 的部分和累加在阵列外的 `px_acc_buf`。

### 3.2 网格连线（8×8 示意）

```
              列0 (oc0)     列1 (oc1)     …    列7 (oc7)
            ┌──────────┐ ┌──────────┐      ┌──────────┐
  行0  act0 │ PE00     │ │ PE01     │      │ PE07     │
  (k0) ═════╡ ×W00     │ │ ×W01     │      │ ×W07     │   ← 同行广播同一 act
            │ +0       │ │ +0       │      │ +0       │
            └───┬──────┘ └───┬──────┘      └───┬──────┘
                ▼ psum       ▼                 ▼
            ┌──────────┐ ┌──────────┐      ┌──────────┐
  行1  act1 │ PE10     │ │ PE11     │      │ PE17     │
  (k1) ═════╡ ×W10     │ │ ×W11     │      │ ×W17     │
            │ +psum    │ │ +psum    │      │ +psum    │
            └───┬──────┘ └───┬──────┘      └───┬──────┘
                ▼            ▼                 ▼
               …            …                 …
                ▼            ▼                 ▼
            ┌──────────┐ ┌──────────┐      ┌──────────┐
  行7  act7 │ PE70     │ │ PE71     │      │ PE77     │
  (k7) ═════╡          │ │          │      │          │
            └───┬──────┘ └───┬──────┘      └───┬──────┘
                ▼            ▼                 ▼
           psum[oc0]    psum[oc1]         psum[oc7]     同一拍 COLS 路点积
```

- 激活：**行内广播**，列与列之间没有横向链路。
- 部分和：**自上而下**。行 0 的 `psum_in = 0`，底行吐出

  `((((0+a0·w0)+a1·w1)+…)+a7·w7)`

  与旧的阵列外加法树同序、同位宽，bit-exact。

### 3.3 为什么要斜切（skew）

列上的 psum 每过一行花 1 拍。行 r 的 `a[r]·W[r]` 必须和上面传下来的部分和在同一拍相遇。因此：

- `npu_compute` 一拍送出**未斜切**的整向量 `sa_act_data[0..ROWS-1]`，`sa_act_valid` 一拍。
- `npu_systolic` 里行 r 把 `act[r]` 推迟 **r** 拍（三角寄存器，共 `ROWS×(ROWS-1)/2` 个 `DATA_W`）。
- 行 0 直通。结果在 **`sa_act_valid` 之后第 ROWS 拍** 从底行出现（`psum_out_valid`）。

默认 8 行：喂向量与收点积隔 **8 拍**。飞行中最多 `ARRAY_SIZE` 个向量（`IF_DEPTH = ARRAY_SIZE`）。

```
拍   送入阵列的向量          底行吐出
 t    V0 (像素0, pass p)
 t+1  V1
 ...
 t+7  V7
 t+8  V8                     V0 的 COLS 路 psum
 t+9                         V1
```

阵列一旦进入 COMPUTE，每拍都能再收一个向量（`ready` 在 COMPUTE 也为 1）。不必每个像素重新 `sa_cmd`。

### 3.4 灌权重

一列一拍：

1. `npu_compute` 从 WGT SRAM 读 256-bit，解出该列的 `ROWS` 个权，放到 `sa_wgt_data[0..ROWS-1]`。
2. `sa_wgt_valid` 脉冲，阵列按 `wgt_col_cnt` 只让 **一列** PE 锁存。
3. 重复 COLS 拍，整阵驻留一组 `ROWS × COLS` 权，对应当前 `k_pass`。

之后同一 `k_pass` 上，一块空间像素（最多 16 个）复用这组权。下一块若 `wgt_held` 且第一趟仍是这组权，可跳过再灌（驻留 skip）。影子缓冲 `wgt_sh` 可在算的时候预取下一趟权。

---

## 4. 计算循环（谁决定送哪个向量）

`npu_compute` 外→内：

```
tile_y → tile_x → oc_group → 空间块(最多 16 像素) → k_pass → 块内像素
```

- `k_depth = kh × kw × IC`
- `k_pass_max = ⌈k_depth / ARRAY⌉ − 1`
- 一趟 `k_pass` 喂 `k_pass_remain`（通常 = ARRAY）个 K 槽，对应阵列的一行宽。
- `oc_group` 一次最多 ARRAY 个输出通道；`OC > ARRAY` 再切一组，重装权。
- 空间块：`blk_px_cnt ≤ 16`。`px_acc_buf[0:511]` 按 `{bank, px[3:0], col[3:0]}`，双 bank：一块在累加，上一块 PPU 可读。

K 下标展平：

```
flat = fh × (kw × IC) + fw × IC + ch
pass 起点 = k_pass × ARRAY
→ (pass_fh, pass_fw, pass_ch)
```

1×1：`fh=fw=0`，`ch = k_pass × ARRAY`。  
3×3：通道走完再动 `fw`，再动 `fh`。

---

## 5. 激活怎么进阵列（核心）

### 5.1 从地址到 `sa_act_valid`

```
  输出像素 (oh, ow) 当前 tap (fh, fw, ch)
           │
           ▼
  ih = oh×stride_h − pad_top + fh
  iw = ow×stride_w − pad_left + fw
           │
     ┌─────┴──────┐
     │ 越界？      │ 是 → act_buf 填 in_zp，不读 SRAM
     └─────┬──────┘
           │ 否
           ▼
  elem = ih×W×IC + iw×IC + ch     (非 tile)
       或 tile 内等价公式
  byte = INT16 ? elem×2 : elem
  word = act_base + byte[17:2]
           │
           ▼
  ACT_LOAD：act_rd_en / act_rd_addr
  等 2 拍 ──▶ 256-bit beat
           │
           ▼
  ACT_EMIT：按 act_byte_sel 对齐，从 beat 里取出连续通道
            填 sa_act_data[0 .. npack-1]，其余 lane 清 0
            拼满 k_pass_remain → sa_act_valid=1
```

`npack` 受三件事裁剪：beat 里剩下的元素、本 tap 还剩的通道、本 pass 还剩的 K 槽。对齐且 `IC` 够时，**一拍就能填满 ARRAY 路**。

### 5.2 单 tap 快路径（1×1，以及 IC 盖得住的 3×3）

若一整趟 `k_pass` 都在同一个 `(fh,fw)`（`pass_ch + k_pass_remain ≤ IC`）：

1. **少 1 拍采样**：`ACT_LOAD` 在 wait 那拍就进 `EMIT`，用 `act_use_rd` 直接吃 `act_rd_data`（issue 后第 2 拍）。
2. **2-ahead**：wait 拍已发下一像素同一 tap 的读；`EMIT` 再发下下个。稳态可以 **1 向量/拍** 留在 `S_ACT_EMIT`。
3. **跳过 `SPATIAL_SETUP` + `ACT_CMD`**：块内下一像素地址用 `conv_tap_byte(oh,ow,pass_fh,pass_fw,pass_ch)` 当场算。
4. 最后像素可直接去 `PSUM_COLLECT`，不必再 `FLUSH`。

3×3 的 pad 按 **这个 tap** 的 `(ih,iw)` 判断，不是输出中心。下一像素若该 tap 是 pad，`ahead_pad=1`，不误用上一拍数据。

跨 tap 的 pass（例如 `IC=1, k=3`，一趟要扫多个窗位）**不走**快路径，仍是：

`SETUP → CMD → LOAD(3 拍) → EMIT → 下一 tap 或 FLUSH → SETUP`

### 5.3 慢路径（跨 tap / 不对齐 / deconv）

每个向量：

| 状态 | 典型拍数 | 做什么 |
|---|---|---|
| `S_SPATIAL_SETUP` | 1 | 窗原点 + 分解 `(fh,fw,ch)` |
| `S_ACT_CMD` | 1+ | 算 word 地址；首像素发 `MODE_COMPUTE` |
| `S_ACT_LOAD` | 3（有效）或 1（pad） | SRAM issue / wait / 采样进 `act_buf` |
| `S_ACT_EMIT` | ≥1 | 拼 lane，满则 `sa_act_valid` |
| `S_ACT_FLUSH` | 1 | 记 in-flight，下像素回 SETUP |

这就是 3×3 在快路径之前 `ACT_LOAD` 能占到 34% 周期的原因：每个 (像素, pass) 都重新握手。

### 5.4 飞行 FIFO

`sa_act_valid` 时把 `(px_in_blk, k_pass)` 推进深度 `ARRAY` 的 FIFO。底行 `psum_out_valid` 时弹出，按 `if_kp == k_first` **覆盖**或**累加**进 `px_acc_buf`。

- 第一趟（或块内约定的 `k_first`）：写入。
- 其后各趟：加到已有部分和。

FIFO 满则 `EMIT` 停（`if_full`），避免超过阵列延迟。这是 1×1 `ACT_EMIT` 里多出来那十几拍的来源。

---

## 6. 一条完整「灌权 → 喂数 → 收数」时间线

默认 8×8，1×1，`IC=16`（2 个 pass），一块 16 像素，单 tap 快路径：

```
WGT_CMD / WGT_LOAD / WGT_EMIT × 8     灌 pass 0 的 8 列权
ACT_CMD                               首像素，阵列进 COMPUTE
ACT_LOAD (2 拍，并发下一像素)
ACT_EMIT ── 16 个像素，约 1 拍/向量 ──▶ 阵列每拍 64 MAC
          8 拍后 psum 回流，累加进 bank
PSUM_COLLECT                          等 FIFO 排空
（下一 pass：驻留或再灌权，再扫 16 像素）
最后一趟齐 → PPU 读 bank、量化、写 OFM
  非最后一块：ppu_bg=1，与下一块灌权/计算重叠
```

3×3、`IC=16`：每 pass 仍是「一个 tap 上的 8 个通道」，快路径与 1×1 相同，只是地址带 `(fh,fw)`。`IC=1` 的 3×3 无法单 tap 填满一行，仍走慢 gather。

---

## 7. 结果离开阵列之后

```
psum_out[c]  ──▶  px_acc_buf[bank][px][c]     (多 pass 累加)
                      │
                      │ 块结束，切 bank
                      ▼
              PPU × COLS（bias / requant / ReLU / zp …）
                      │
                      ▼
              OFM SRAM 32-bit 打包写
                      │
                      ▼
              DMA store → DDR   （或 fusion：对调成下一层 IFM）
```

PARAM 可在灌权时预填。最后一块的 PPU 不能和下一块计算重叠，会暴露在 `S_PPU_STREAM`。

Depthwise **不进**这张阵列，走 `npu_dw_conv`。

---

## 8. 和「利用率」怎么对上

口径（`rtl/tb/integration/test_array_util.py`）：

```
有用 MAC = Σ (sa_act_valid 时非零 lane 数) × COLS
峰值     = 总周期 × ROWS × COLS
利用率   = 有用 / 峰值
```

只统计 **compute 段**（数据已在 SRAM），不含 DDR DMA。

`8×8×16→16`、阵列 8，当前实测大约：

| 层 | 周期 | 利用率 | 喂数时 lane |
|---|---|---|---|
| 1×1 | 742 | **34.5%** | 8.00 / 8 |
| 3×3 | 6510 | **29.7%** | 6.72 / 8（pad 的零 lane） |

喂数拍本身是满的（或仅 pad 空）。空拍在块头灌权、`PSUM_COLLECT`、最后 PPU、以及未走快路径的 gather。这是序列器 + 16 像素切块的税，不是 PE 乘错。

整网 A/D 再叠上 32-bit 片外总线和层间 DMA 后，墙钟还会被 **字节** 卡住，和这篇里的阵列忙闲不是同一层。

---

## 9. 对照：阵列「看到」的是什么

阵列接口非常窄。它不理解卷积窗：

| 信号 | 含义 |
|---|---|
| `cmd` / `cmd_valid` | IDLE / WGT_LOAD / COMPUTE |
| `wgt_data[r]` + `wgt_valid` | 当前列的一行权 |
| `act_data[r]` + `act_valid` | 当前像素当前 pass 的 K 向量 |
| `psum_out[c]` + `psum_out_valid` | 该向量对 COLS 个 OC 的点积 |

窗、pad、NHWC、k_pass、16 点块、双 bank，全部在 `npu_compute`。阵列只做：

**驻留一块 `ROWS×COLS` 权，每拍吃一个 K 向量，ROWS 拍后吐 COLS 路点积。**
