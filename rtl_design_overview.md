# Open-NPU RTL 设计总览（信号级）

本文档描述 **RTL 实际实现的样子**，逐信号说明。

> 与 `architecture-spec.md` 的关系：那份是**设计意图**文档，第 8 章（脉动阵列）描述的
> 是一个目标架构，与 `rtl/src/` 里跑通的实现仍有实质差异（行语义、INT8 双倍吞吐
> 都对不上）。阵列本身已改成 weight-stationary + 列向 psum 链，drain / 外部加法树
> 已去掉。差异清单见 [§9](#9-与-architecture-specmd-的差异)。
> 做 RTL 修改、写驱动、估性能，以本文档为准。

代码基准：`rtl/src/`，`ARRAY_SIZE=16`，`SPAD_KB=192`，`ACC_WIDTH=44`。
RTL 回归（`phase1_regression.py --section rtl`，含 42 条 DMA E2E）在这次改写后
**全绿**。

---

## 1. 模块清单

| 模块 | 行数 | 职责 |
|------|------|------|
| `npu_top.v` | 760 | 顶层；例化 3 块 SRAM，做 DMA↔SRAM 端口复用 |
| `npu_csr.v` | 493 | Wishbone slave 寄存器堆 |
| `npu_ctrl.v` | 1320 | 层级序列器：DMA 阶段调度、tiling 外循环、双缓冲、融合 |
| `npu_dma.v` | 346 | Wishbone master，1D / 2D strided 搬运 |
| `npu_compute.v` | 3427 | 计算微序列器（**全部算子的主状态机**，77 个状态） |
| `npu_systolic.v` | 264 | 16×16 PE 阵列 |
| `npu_pe.v` | 91 | 单个 PE |
| `npu_ppu.v` | 317 | 后处理单流水线（例化 ARRAY_SIZE 条） |
| `npu_dw_conv.v` | 104 | Depthwise 卷积单元 |
| `npu_sram.v` | 57 | 通用双端口同步 SRAM（param，A/B 都是 32 位） |
| `npu_sram_wide.v` | ~70 | act/wgt：A 口 32 位 DMA，B 口 256 位计算读 / 32 位写 |

层次：

```
npu_top
├── npu_csr            (Wishbone slave  ← CPU)
├── npu_dma            (Wishbone master → DDR)
├── npu_sram_wide × 2  (act / wgt，A32 / B256)
├── npu_sram           (param，32 位)
├── npu_ctrl           (层序列器)
└── npu_compute        (计算序列器)
    ├── npu_systolic → npu_pe × 256
    ├── npu_dw_conv  × 16
    └── npu_ppu      × 16
```

---

## 2. 顶层信号

```verilog
module npu_top #(
    parameter ARRAY_SIZE  = `ARRAY_SIZE,   // 16
    parameter SPAD_KB     = `SPAD_KB,      // 192
    parameter ACT_DEPTH   = SPAD_KB * 64,  // 12288 words = 48KB
    parameter WGT_DEPTH   = SPAD_KB * 128, // 24576 words = 96KB
    parameter PARAM_DEPTH = SPAD_KB * 16   //  3072 words = 12KB
)
```

| 信号组 | 方向 | 说明 |
|--------|------|------|
| `clk`, `rst_n` | in | 单时钟域，异步低有效复位。全设计**只有一个时钟域** |
| `wb_*_i / wb_*_o`（slave） | — | CPU 访问 CSR。32 位地址/数据，Wishbone B4 classic |
| `wbm_*_o / wbm_*_i`（master） | — | DMA 访问 DDR |
| `irq` | out | 电平中断，由 `IRQ_STATUS & IRQ_EN` 产生 |

**注意 `SPAD_KB` 的命名有误导性**：三块 SRAM 加起来是 `SPAD_KB × 832` 字节，不是
`× 1024`。`SPAD_KB=192` 实际例化 48+96+12 = **156KB**，剩下的 1/16 没有例化。
做面积预算时按 832 B/KB 算。

---

## 3. 寄存器 → 控制 → 计算 的配置流

CPU 侧只写寄存器，不参与逐层时序：

```
CPU ──wb write──▶ npu_csr ──reg_* wires──▶ npu_ctrl ──cfg_* wires──▶ npu_compute
                     ▲                          │
                     └──── hw_busy/done/error ──┘
```

### 3.1 npu_csr 关键寄存器

| 偏移 | 名称 | 关键位 |
|------|------|--------|
| `0x000` | `CTRL` | `[0]` START（自清）`[1]` ABORT `[2]` SOFT_RST `[3]` AUTO_NEXT |
| `0x004` | `STATUS` | `[0]` BUSY `[1]` DMA_BUSY `[2]` ERROR `[3]` DONE |
| `0x008` | `IRQ_EN` | 中断使能 |
| `0x00C` | `IRQ_STATUS` | 写 1 清除（W1C） |
| `0x010` | `ERROR` | 错误码 E1–E9，见 §3.3 |
| `0x018` | `HW_CONFIG` | `[7:0]` ARRAY_SIZE，`[23:16]` SPAD_KB |
| `0x01C` | `PERF_CNT` | 周期计数 |
| `0x040+` | 层描述符 | 形状、kernel、stride、pad、tiling、post_ctrl |
| `0x100+` | DMA 配置 | 源/目的地址、长度、2D 参数、`DMA_CTRL` |
| `0x180+` | PPU 配置 | mode、relu、bias、zp、clamp_max |

> `HW_CONFIG[23:16]` 打包的是 **KB 数**（192），不是寄存器规格书里写的 4KB 单位。
> 固件 `soc/firmware/main.c` 的 `npu_spad_kb()` 按 KB 解读，与 RTL 一致，
> 但与 `design/npu-register-spec.md` 的文字描述不一致 —— 规格书需要改。

### 3.2 ctrl → compute 握手信号

这是整个设计里最需要看懂的一组信号：

| 信号 | 方向（compute 视角） | 语义 |
|------|---------------------|------|
| `start` | in | 电平，拉高启动一个 tile 的计算 |
| `done` | out | 电平，整层（所有 tile、所有 oc_group）完成 |
| `tile_done` | out | **1 拍脉冲**，非最后一个 tile 的边界 |
| `oc_group_done` | out | **1 拍脉冲**，请求 ctrl 重新装载下一个 oc_group 的权重 |
| `oc_group_out[15:0]` | out | 当前 oc_group 序号，ctrl 据此算权重 DDR 偏移 |
| `wgt_reload_done` | in | ctrl 通知：权重已就位，可以继续 |
| `cfg_wgt_per_oc[31:0]` | in | 每个 oc 的权重字数；**为 0 表示权重全部装得下，不需要重载** |
| `db_prefetch_done` | in | 双缓冲：下一 tile 的输入已预取完，可以切 bank |
| `tile_out_h_actual` / `_w_actual` | out | 当前 tile 的**实际**输出尺寸（边界 tile 会被裁剪），ctrl 用它算 per-tile store 的 DDR 写偏移 |

`tile_done` / `oc_group_done` 是单拍脉冲，ctrl 侧必须在对应状态里**恰好**采到。
这是一类典型的易错点（见 §10 风险）。

### 3.3 硬件守卫（错误码）

`npu_ctrl.v` 在 `S_IDLE` 里对层描述符做一轮合法性检查，不合法就进 `S_ERROR` 并
在 `ERROR` 寄存器里给码：

| 码 | 条件 |
|----|------|
| E1 | slice_stream 模式下 `out_c × 4 > SPAD_KB×16`（param SRAM 放不下） |
| E2 | grp8 模式与 per-oc param 重载不兼容 |
| E3 | DW 通道切片超过 `DW_STREAM_OUT_BASE` |
| E4 | 输入激活搬运量超过 act SRAM（DB_EN 下是半个 bank） |
| E5 | 输出地址范围超过 act SRAM，会发生地址截断 |
| E6 | Conv/FC 的 `k_depth > 65535`（超出 16 位寄存器） |
| E7 | 全局 AvgPool 的除数不在倒数 LUT 里 |
| E8 / E9 | 初始权重 / 参数 DMA 范围超过各自 SRAM 深度 |

**缺一条**：没有任何守卫检查 44 位累加器溢出。见 §10。

---

## 4. SRAM 组织

act / wgt 用 `npu_sram_wide`（容量仍按 32-bit word 计），param 仍是 32 位 `npu_sram`：

| 块 | 深度（SPAD_KB=192） | 字节 | Port A | Port B |
|----|--------------------|------|--------|--------|
| act | 12288 | 48KB | DMA 32 位读写 | compute 256 位读 + 32 位写回 |
| wgt | 24576 | 96KB | DMA 32 位写 | compute 256 位读 |
| param | 3072 | 12KB | DMA 32 位写 | compute 32 位读 |

B 口一拍读出从 `b_addr` 起连续 8 个 word，`b_rdata[31:0]` 仍是被寻址的那个 word，所以 DW / pool / add / resize / RMW 不用改地址。DMA 仍走 32 位 A 口。

**读时序**：同步读，**1 拍延迟**。同地址同拍读写返回**旧数据**（read-first）。

> `npu_sram.v` 的文件头注释写的是 "Write-first: ... read returns new data"，
> 但代码是 `mem[a_addr] <= a_wdata; a_rdata <= mem[a_addr];`，非阻塞赋值下
> `a_rdata` 拿到的是旧值。**注释是错的**，代码行为是 read-first。
> FPGA 上 Vivado 会推断成 READ_FIRST BRAM，与仿真一致；但依赖注释写逻辑会出错。

激活 SRAM 的地址布局由 `SRAM_BASE` 寄存器给出：`[15:0]` = 输入基址，
`[31:16]` = 输出基址（都是**字**地址）。DB_EN 时输出 bank 偏移 `ACT_DEPTH/2`：

```verilog
// npu_top.v:644
wire [ACT_ADDR_W-1:0] effective_act_base = reg_sram_base[ACT_ADDR_W-1:0]
    + (db_en_is_active && ping_pong_flag ? (ACT_DEPTH >> 1) : {ACT_ADDR_W{1'b0}});
```

---

## 5. PE 阵列 —— 这是怎么回事

### 5.1 一句话

16×16 **weight-stationary** 阵列。`PE[r][c]` 驻留 `W[k=r][oc=c]`；激活按行广播，
部分和沿列向下流，底行每拍吐出一整组 16 路点积。阵列内部完成 K 维归约，
**没有 drain 相位，也没有阵列外加法树**。

这是 TPU 系常用的 psum 链。跟本设计体量更近的 NVDLA / Ethos-U 用的是
"点积单元 + 加法树 + 累加 SRAM"；旧 RTL 结构上属于后者，但喂数是 one-hot、
drain 按列串行，实测利用率 0.82%。改成 psum 链后，同一条 8×8×16→16 k=3
路径从 58,884 拍降到 **30,660 拍（1.92×）**，Yosys 综合每个 PE 触发器从 122
降到 61。roadmap 仍可在加 IFM staging / 分 bank SRAM 之后再评估
output-stationary。

### 5.2 行和列分别是什么

| 维度 | 映射到 | 大小 |
|------|--------|------|
| **行 r** | K 维度的一个切片（`k = kernel_h × kernel_w × in_c` 展平后的第 r 个元素） | 16 |
| **列 c** | 输出通道（当前 oc_group 里的第 c 个） | 16 |

所以 `PE[r][c]` 存的权重是 `W[k=r][oc=c]`，算的是 `act[k=r] × W[r][c]`。

- K 超过 16：拆成 `k_pass = ceil(k_depth/16)` 趟，每趟重新装权重，跨趟部分和
  仍攒在 `npu_compute` 的 `px_acc_buf` 里。
- 输出通道超过 16：拆成 `oc_group`，由 ctrl 重新 DMA 权重。

`architecture-spec.md` §8.1 写的"行对应不同的输出像素"，**和 RTL 不符**。

### 5.3 PE 内部（`npu_pe.v`）

```verilog
input  [1:0] mode;                 // 00=IDLE 01=WGT_LOAD 10=COMPUTE
input        valid_in;
input  signed [15:0] act_in;       // 同行广播（经过行斜切）
input  signed [15:0] weight_in;    // 列选通广播
input  signed [43:0] psum_in;      // 上一行送来；第 0 行接 0
output signed [43:0] psum_out;     // 本拍 MAC 之后送给下一行
output               psum_valid_out;
```

三种模式（`MODE_DRAIN` 已删除）：

| mode | 行为 |
|------|------|
| `WGT_LOAD` | `valid_in` 时 `weight_reg <= weight_in` |
| `COMPUTE` | `valid_in` 时 `psum_out <= psum_in + act_in * weight_reg`，`psum_valid_out <= 1` |
| `IDLE` | 不动（`psum_out` 保持） |

PE **不跨拍累加**。一个输出像素在一列上的点积是
`((((0 + a0*w0) + a1*w1) + …) + a15*w15)`，宽度始终 `ACC_WIDTH`，
和它替换掉的外部加法树同序同宽，所以 bit-exact。

乘法器仍是**一个** 16×16 有符号乘法器。INT8 只是符号扩展到 16 位送进来，
**没有** `architecture-spec.md` §8.5 说的 INT8 双倍吞吐。

每个 PE 综合后 **61 个触发器**（16 位权重 + 44 位 `psum_out` + 1 位 valid）。
旧 PE 有 `acc_reg` 和 `acc_out` 两份 44 位寄存器，外加激活右移流水，122 个触发器。

### 5.4 权重装载：广播 + 列选通，不是移位链

```verilog
assign pe_wgt_in[r][c] = wgt_data[r];
assign pe_valid[r][c]  = (state == S_WGT_LOAD) ? (wgt_valid && wgt_col_cnt == c)
                       : (state == S_COMPUTE)  ? skew_valid[r]
                       : 1'b0;
```

`wgt_col_cnt` 每拍 +1，第 c 拍第 c 列锁存 `wgt_data[0..15]`。
16 拍装满整个 16×16 权重阵。这是**广播总线 + 列使能**，不是脉动移位链。

阵列 FSM：`S_IDLE → S_WGT_LOAD → S_READY ⇄ S_COMPUTE`。
`ready` 在 `S_READY` **和** `S_COMPUTE` 都为 1，像素之间不必弹回 `S_READY`，
可以连续喂向量。

### 5.5 激活喂入：256 位 beat 一拍拼向量

act / wgt SRAM 的计算口是 256 位（从字地址起连续 8 个 32-bit word）。
`S_ACT_EMIT` 从这一拍解出最多 16 个 INT8（或 16 个 INT16），写进
`sa_act_data[0..npack-1]`；第 0 个元素时把其余 lane 清零。`sa_act_valid`
只在向量拼齐的那一拍拉高。in_c≥16 且对齐时，一个 (fh,fw) 一拍就能填满
整列；INT16 非对齐时可能再吃一拍。

```verilog
// npu_compute.v S_ACT_EMIT
shifted = act_buf >> (act_byte_sel * 8);
npack   = min(remain_word, remain_k, remain_ch, ARRAY_SIZE);
sa_act_data[act_cnt + ei] <= lanes[ei];          // 最多 ARRAY_SIZE 路
sa_act_valid <= (act_cnt + npack >= k_pass_remain);
```

`npu_systolic` 收到这个未斜切的向量后，用三角缓冲把第 r 行延迟 r 拍，
再按行广播给 16 列。下降的 psum 正好撞上对应的激活：

```
cycle t+0:  row0 吃 a0，psum = a0*w0
cycle t+1:  row1 吃 a1，psum = a0*w0 + a1*w1
…
cycle t+15: row15 吃 a15，底行吐出完整点积，psum_out_valid=1
```

斜切缓冲代价：`ROWS*(ROWS-1)/2` 个 16 位寄存器 + 同等数量的 valid，
16×16 时是 120 个 16 位寄存器（实测阵列本地约 1,944 个触发器，含 valid）。

### 5.6 结果出口：底行一拍 16 路，没有 drain

```verilog
assign psum_out_flat[ACC_W*c +: ACC_W] = pe_psum_out[ROWS-1][c];
assign psum_out_valid = pe_psum_val[ROWS-1][0];
```

`npu_compute` 在 `sa_act_valid` 之后把该像素标成 in-flight，结果到达时
（与当前 FSM 状态无关）写入 `px_acc_buf`。同一 block 的下一个像素立刻开始
拼向量；新向量要等上一拍 psum 落地才脉冲（1 级重叠）。`S_PSUM_COLLECT`
只等 block 里最后那个向量。旧的 drain / 外部加法树已删除。

这同时修掉了旧版硬编码 `4'd15` 的参数化 bug：`ARRAY_SIZE` 为 8/32 时不再
静默丢掉高列。

### 5.7 完整时序：一个 (像素, k_pass) 迭代

以 8×8×16 → 16、k=3×3 为例。`k_depth = 144`，`k_pass_max = 9`，
64 个输出像素按 16 个一"块"分 4 块。

```
每块、每趟（block × k_pass = 4 × 9 = 36 次）：
  S_WGT_CMD/LOAD/EMIT   ~48 拍    装 16 列（每列 1 次 256 位读，3 拍发行/等待/解包）
  └─ 之后 16 个像素复用这批权重：

     每个像素（与上一像素的 psum 飞行重叠）：
       S_ACT_LOAD           2 拍   256 位 beat
       S_ACT_EMIT         ~11 拍   一拍填满向量，其余在等阵列 16 拍延迟
       S_ACT_FLUSH          1 拍   标 in-flight，立刻开下一像素
```

### 5.8 实测：口宽上去之后，瓶颈改到阵列延迟

同一条路径（8×8×16→16，k=3）：

| 版本 | 周期 | 相对 |
|------|------|------|
| 旧：one-hot + 串行 drain + 外部加法树 | 58,884 | 1.00× |
| 向量脉冲 + psum 链 | 30,660 | 1.92× |
| 多 lane（32 位口）+ 1 级 psum 重叠 | 20,046 | 2.94× |
| 现：A32/B256 计算口 | **14,052** | **4.19×** |

`test_array_util.py`：`lane_slots/feed_cycles = 13.44`（满宽 16；padding 会少）。
周期去向：`ACT_EMIT` 47%、`WGT_LOAD` 12%、`ACT_LOAD` 11%、`PPU_STREAM` 7%。
装权和 SRAM 读已经不是大头；`ACT_EMIT` 里大半是 `psum_pending` 空等阵列
16 拍延迟（拼向量现在 1 拍就完成）。

墙钟利用率约 **3.4%**（8.8 MAC/拍 vs 峰值 256）。

> `tools/internal/perf_model.py` 在 `in_c≥16 且 tile_oc≥16` 时仍假设
> `util=1.0`（256 MAC/拍）。它算的是架构上限，不是这份 RTL 的实测值。

### 5.9 还剩的提速方向

| # | 改动 | 预计收益 | 风险 |
|---|------|---------|------|
| 1 | **更深的 psum 重叠 / 双缓冲向量**（现在 1-deep，拼完就堵） | 砍掉现在 47% 的 `ACT_EMIT` 空等 | 中。和阵列延迟对齐 |
| 2 | **跨 k_pass 驻留、psum 接着往下加** | 少一次权重重装 | 低。阵列已经按列吐完整点积 |
| 3 | 权重跨列打包，或把列装载和计算重叠 | 再削 12% 的 `WGT_LOAD` | 中 |

**这些都不影响 bit-exact**：改的是调度，不是算术。现有 CSIM 对拍回归可以直接
当回归网。

---

## 6. PPU 流水线（`npu_ppu.v`）

例化 16 条，每条 4 级流水、1 拍 1 个结果。

```
acc_in(44b) → [S1] +bias → [S2] ×M + 舍入位 → [S3] >>>S 并饱和到 17b
            → [S4] +zp → clamp → ReLU/ReLU6 → out_data(16b)
```

| 信号 | 位宽 | 说明 |
|------|------|------|
| `mode` | 2 | `00`=CONV_REQ `01`=ADD `10`=RELU_ONLY `11`=PASSTHROUGH |
| `acc_in` | 44 | 来自 `px_acc_buf`（阵列底行点积，跨 k_pass 累加后） |
| `bias` | 44 | 每通道 bias_q（描述符里是 64 位，**截断**成 44 位送进来） |
| `mult_m` | 15 | 无符号乘数 M |
| `shift_s` | 6 | 右移位数 S |
| `zero_point` | 16 | 有符号 zp |
| `clamp_max` | 16 | ReLU6 上界；转换器保证 INT8 时 ≤127 |
| `int16_mode` | 1 | 1=钳到 ±32767，0=钳到 ±127 |
| `out_data` | 16 | INT8 模式下符号扩展到 16 位 |

**舍入方式**：在 S2 就把舍入位加进去，S3 再算术右移：

```verilog
product_v = product_v + (1 << (s1_shift_s - 1));   // S2
shifted_full = s2_product >>> shift_amt;            // S3
```

这是 **round-half-up（.5 向 +∞）**，不是 round-half-away-from-zero。
负数的 .5 会向上取。CSIM 必须用同样的规则，否则对不上。

**注意**：源码里大量注释还写着 "40-bit"（`ACC_WIDTH` 早就是 44），
S3 里还有 `56'sd65535`、`-56'sd1` 这类字面量（`PROD_W` 实际是 60）。
位宽够用所以功能对，但注释和字面量都是陈旧的。

---

## 7. DMA（`npu_dma.v`）

Wishbone master，支持 1D 连续和 2D strided。

| 信号 | 说明 |
|------|------|
| `dma_start` | 启动脉冲 |
| `dma_busy` / `dma_done` | 状态 |
| `dma_src_addr` / `dma_dst_addr` | DDR 地址（字节） |
| `dma_len` | 长度 |
| `dma_2d_en` | 2D 模式使能 |
| `row_len` / `row_count` / `src_stride` / `dst_stride` | 2D 参数 |
| `dma_sram_sel` | 目标 SRAM（act / wgt / param），在 `dma_start` 时**锁存** |
| `dma_sram_addr` / `dma_sram_wdata` / `dma_sram_we` | SRAM 侧 |

2D 模式用来搬 tile：从 DDR 里一个 NHWC 张量中抠出 `row_count` 行、每行
`row_len` 字节，行间跨 `src_stride`。halo 由 ctrl 在算 `row_count` 时加进去。

---

## 8. DW Conv（`npu_dw_conv.v`）

16 个独立单元，每个负责一个通道。权重存在 `weights[0:255]` 数组里，
`in_valid` 时按 `compute_idx` 逐个取出做 MAC，攒满 `kernel_h × kernel_w` 个
就拉 `out_valid`。

**面积警告**：复位分支里有

```verilog
for (wi = 0; wi < MAX_KSZ*MAX_KSZ; wi = wi + 1)
    weights[wi] <= {DATA_W{1'b0}};
```

异步复位遍历 256 个条目，综合工具**没法推成 RAM**，只能推成 256×16 = 4096 个
触发器；16 个通道就是 **65536 个触发器**。这在 ASIC 上是实打实的面积，在 FPGA 上
会吃掉大量 LUT-FF。去掉这个复位循环（或改成同步、按需清）就能推成分布式 RAM。

---

## 9. 与 architecture-spec.md 的差异

| 项 | architecture-spec.md §8 | RTL 实际 |
|----|------------------------|---------|
| 行的语义 | 不同的输出像素 | K 维切片（`kh×kw×in_c` 展平） |
| 列的语义 | 输出通道 | 输出通道 ✓ |
| 激活喂入 | 每拍一组 16 个**像素** | 每拍从 SRAM 拼 **1 个 K 元素**，拼齐后脉冲一次 16-lane 向量（行=K，不是像素） |
| 计算周期 | `C_in + N - 1 = 19` | 每 (像素,趟) ~33 拍（旧路径 81 拍）；阵列本身固定 `ARRAY_SIZE` 拍出结果 |
| Drain | "1 cycle，所有行并行" | **已删除**。底行每拍吐 16 路点积 |
| 跨行求和 | 阵列内 partial sum | 阵列内列向 psum 链 ✓（替换了外部加法树） |
| 累加器 | 40 位 | 44 位（`ACC_WIDTH`）；PE 不再跨拍累加 |
| INT8 | 乘法器 2× 并行 | 无，8 位符号扩展到 16 位走同一个乘法器 |
| 权重装载 | 逐列加载 | 逐列加载 ✓（广播+列选通） |

结论：**§8 是目标架构，不是实现**。要么把 §8 改成本文档的内容，要么明确标注它是
roadmap。现在这样，任何按 §8 做性能估算或写驱动的人都会被误导。

---

## 10. 已知风险与缺口

| # | 位置 | 问题 | 级别 |
|---|------|------|------|
| 1 | `npu_compute.v` 喂数/装权 | A32/B256 后利用率约 3.4%（8.8 MAC/拍）。新瓶颈是 `ACT_EMIT` 等阵列 16 拍延迟 | 性能 |
| 2 | `npu_ctrl.v` E1–E9 | **没有累加器溢出守卫**。44 位有符号范围 ±8.8e12；INT16 下 `3×3×1024` 卷积最坏 `2^30 × 9216 ≈ 9.9e12` 会**静默回绕** | 潜在 BUG |
| 3 | ~~`npu_systolic` drain MUX 硬编码 `4'd15`~~ | **已随 psum 链删除**。`ARRAY_SIZE` 8/16/32 走 generate | 已修 |
| 4 | `npu_sram.v:9` | 头注释说 write-first，代码是 read-first | 文档 BUG |
| 5 | `npu_dw_conv.v:66` | 复位遍历 256 项权重数组 → 推成 6.5 万个触发器 | 面积 |
| 6 | `npu_dw_conv.v:82` | `acc_clear` 在窗口中途到达时，当拍的 `product` 仍用旧的 `compute_idx` 取权重，整个新窗口错位。正常流式下 `compute_idx` 回到 0 所以碰不到 | 潜在 BUG |
| 7 | `npu_ppu.v:187` | INT8 + ReLU6 时 `out_data <= {{8{clamp_hi[7]}}, clamp_hi[7:0]}`，只取低 8 位。转换器保证了 `clamp_max ≤ 127` 所以现在是对的，但**硬件不自我保护**：驱动写个 200 进去会输出 −56 | 潜在 BUG |
| 8 | `npu_ppu.v:262` | 非 CONV_REQ 模式 `shifted_v = s2_product[16:0]`，先截断到 17 位再钳位。INT16 Add 最大 65534 刚好卡在边界内，再大就回绕 | 潜在 BUG |
| 9 | `npu_ppu.v` 全文 | 注释写 40 位、字面量写 56 位，实际 44/60 位 | 陈旧 |
| 10 | `perf_model.py` | 假设 256 MAC/拍，比 RTL 乐观约 120× | 文档 |
| 11 | `npu_compute.v:26,28` | `ACT_ADDR_W=13` / `PARAM_ADDR_W=11` 的默认值是按 `SPAD_KB=128` 写死的。`npu_top` 会覆盖，但独立例化 `npu_compute` 时会静默错 | 参数化 |

---

## 11. 换个 SRAM 大小要动什么

`SPAD_KB` 是唯一的旋钮，`npu_top` 会自动推导三块 SRAM 深度和地址宽度。改小它：

- `npu_ctrl` 的 E4/E5/E8/E9 守卫会自动跟着变（它们用的是 `ACT_DEPTH` 等派生量）。
- 固件 `main.c` 通过 `HW_CONFIG[23:16]` 运行时读出来，不需要重编译常量。
- `tools/hw_config.py` 的 `HWConfig` 要同步改，否则转换器算出的 tiling 与硬件不符。
- `DW_STREAM_OUT_BASE = SPAD_KB*64 - 1024` 会跟着变；`SPAD_KB < 16` 时它会变成
  负数/回绕，**这是 SPAD_KB 的下限**。

用 `tools/internal/spad_sizing.py` 可以跑真实的 tiling 计算器，看某个网络在不同
`SPAD_KB` 下要分多少块、多搬多少 DMA。

---

## 12. FPGA 小网络：猫狗二分类

ABCDE 那组模型是用来压适配性的，SRAM 装不下也没关系。真正上 FPGA 时 SPAD
不能大（成本），所以给了一条专门的小网络：

```
examples/catdog/
    train_catdog.py   # CIFAR-10 的 cat/dog，32×32×3
    run_demo.sh       # ONNX → NPU1 → CSIM → SoC sim
```

结构（通道全是 16 的倍数，GAP 窗口 64，正好落在 reciprocal LUT 表上）：

```
Conv 3→16 → Conv 16→16 → MaxPool 2
Conv 16→32 → Conv 32→32 → MaxPool 2
Conv 32→64 → GAP 8×8 → FC 64→2
```

`SPAD_KB=64`（实际约 52KB scratchpad）下每一层都能驻留，0 个 tiled layer。
训练：

```
python3 examples/catdog/train_catdog.py --data /tmp/cifar --epochs 30
examples/catdog/run_demo.sh --bits 8 --spad 64
```

SoC 仿真前要把 `rtl/include/npu_defines.vh` 和 `tools/hw_config.py` 的
`SPAD_KB` 改成 64，否则 tiling 和硬件对不上。
