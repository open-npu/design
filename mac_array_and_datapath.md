# 64-Lane MAC 行引擎与数据通路详解

本文档描述 **当前可综合 RTL（`rtl/src/`）实际如何从 512-bit SRAM 取数、如何在 64-lane 行引擎中完成稠密与深度卷积乘加、以及如何经 8-lane PPU 写回片上存储**。

对应源文件：

| 模块 | 文件 | 职责 |
|---|---|---|
| 顶层与四库 SRAM 互联 | `rtl/src/npu_top.v` | 例化 `IFM / OFM / WGT / PARAM` 四块 512-bit 宽口 SRAM，管理双缓冲与跨层 bank 互换 |
| 计算微定序器与行引擎 | `rtl/src/npu_compute.v` | 激活 gather、64-bank 单写口缓冲、64-lane MAC 行、DW 折叠复用、Pool / Add / Resize |
| 后处理单元组 | `rtl/src/npu_ppu_bank.v` + `npu_ppu.v` | 8 条并行 4 级后处理流水线（`PPU_LANES = 8`），分 8 拍排空 64 路累加结果 |
| 宽口双端口 SRAM | `rtl/src/npu_sram_wide.v` | A 口 32-bit（DMA 侧），B 口 512-bit 读 / 按字节掩码写（计算侧） |

---

## 1. 计算核在整颗 NPU 中的位置

```
                    Wishbone 32-bit
         CPU ──slave──▶ npu_csr ──▶ npu_ctrl ──start/cfg──▶ npu_compute
         npu_dma ──master─▶ DDR
          │
          │ Port A (32-bit)
          ▼
    ┌──────────┬──────────┬──────────┬──────────┐
    │ IFM SRAM │ OFM SRAM │ WGT SRAM │ PRM SRAM │
    │  32 KiB  │  48 KiB  │  80 KiB  │  12 KiB  │
    └────┬─────┴────┬─────┴────┬─────┴────┬─────┘
         │ Port B   │ Port B   │ Port B   │ Port B
         │ 512-bit  │ 512-bit  │ 512-bit  │ 512-bit
         └──────────┴────┬─────┴──────────┘
                         ▼
                   npu_compute
      ┌──────────────────┼──────────────────┐
      ▼                  ▼                  ▼
  abuf / avec        wrow_b[0..31]      Pool / Add /
 (64 单写口 bank)   (64 单写口 bank)     Resize 通路
      │                  │
      └────────┬─────────┘
               ▼
     64-Lane MAC Row Engine
   (16x16 有符号乘法 x 64 路)
               │
               ▼
       racc[0..63] (44-bit)
               │  每拍选 8 路送入 PPU
               ▼
       npu_ppu_bank (8 Lanes)
               │
               ▼
       OFM SRAM (512-bit 掩码写)
```

核心架构特征：
- **无二维脉动阵列与斜切延迟**：早期 2D 脉动阵列（`npu_systolic.v` / `npu_pe.v`）及独立深度卷积阵列（`npu_dw_conv.v` / `npu_dw_bank.v`）均已退役，全部乘加算子统一收敛到单拍无气泡的 **64-lane 一维行引擎**。
- **峰值算力**：每拍完成 `64` 次 `16x16 -> 32` 有符号乘加并累加进 44-bit 寄存器 `racc[0..63]`（INT8 操作数符号扩展至 16-bit 共用同一乘法器）。

---

## 2. 四块 512-bit 片上 SRAM 与单写口分库缓冲

### 2.1 四库 Scratchpad（`SPAD_KB = 172`）

| 物理库 | 实例名 | 深度（32-bit word） | 容量 | A 口（DMA） | B 口（Compute） |
|---|---|---|---|---|---|
| IFM | `u_sram_act` | `8,192` | `32 KiB` | 32-bit 写 | 512-bit 读（每拍 64 字节） |
| OFM | `u_sram_ofm` | `12,288` | `48 KiB` | 32-bit 读 | 512-bit 读 + 64-byte 掩码写 |
| WGT | `u_sram_wgt` | `20,480` | `80 KiB` | 32-bit 写 | 512-bit 读（每拍 64 个 INT8 或 32 个 INT16 权重） |
| PARAM | `u_sram_param` | `3,072` | `12 KiB` | 32-bit 写 | 512-bit 读（支持 ping/pong 预取） |

SRAM 为同步读（发出读地址后下一拍锁存输出，计算侧按标准两拍流水消费），综合时直接映射为目标工艺 SRAM / BRAM（Yosys 下为 4 个 `$mem_v2`）。

### 2.2 消除多写口爆炸：`wrow / abuf / avec` 64-Bank 单写口结构

为了支持零黑盒硬件综合并彻底消除 `1024` 路动态多写口 MUX，`npu_compute.v` 内部的三组核心暂存区均拆分为 **64 个独立的深度 32、位宽 16-bit 单写口 Bank**（总计 `3 x 64 x 32 x 16 = 98,304` bit，连同辅助缓冲共 `102,400` bit `$mem_v2`）：

1. `wrow_b[0..31][0..63]`：缓存 K-major 权重步或 Depthwise 卷积核权重（最多 `32 x 64` 个 16-bit 槽位）。
2. `abuf_b[0..31][0..63]`：缓存当前像素块从 IFM 预取的激活窗口数据。
3. `avec_b[0..31][0..63]`：缓存重排对齐后的待发射激活向量；边界 padding 元素按单拍顺序填 `cfg_in_zp`，无动态移位写端口。

---

## 3. 稠密卷积与全连接（Conv2D / FC）数据流

### 3.1 K-Major 广播乘加

设展平归约维度 `K = kernel_h * kernel_w * in_c`：
- **满宽输出通道（`G = 64`）**：
  - 对于每一个 `k in [0, K - 1]`，WGT SRAM 的一个 512-bit beat 恰好提供当前 OC 组 64 个输出通道在下标 `k` 处的 INT8 权重 `W[k][0..63]`（INT16 模式下分两拍读入 64 个 16-bit 权重）。
  - 同一拍取当前输出像素在下标 `k` 处的激活标量 `a[k]`，向全部 64 条 lane 广播：
    `racc[c] <= racc[c] + a[k] * W[k][c]`（`c = 0..63`）。
- **窄输出通道多像素打包（`G < 64`）**：
  - 当层输出通道较少（例如 `G = 16` 或 `G = 32`）时，若只算 1 个像素，64 条 lane 会闲置大半。
  - 行引擎自动开启像素打包（`P = 64 / G`）：同一拍将 `P` 个相邻输出像素各自的 `a_p[k]` 分别广播到对应的 `G` 条通道子组中，与同一组 `G` 个通道权重并行相乘，使窄通道层同样打满 `64 MACs/cycle`。

### 3.2 循环层次

```
tile_y -> tile_x -> oc_group (每组最多 64 通道) -> 像素微块 -> k = 0 .. K-1 -> 8-lane PPU 排空写回
```

当 `out_c` 对应的总权重超出 `80 KiB` WGT SRAM 时，`npu_ctrl` 利用 `oc_group_done` / `wgt_reload_done` 握手按 `cfg_wgt_per_oc` 自动从主存重载下一组输出通道权重。

---

## 4. Depthwise Conv2D 的折叠复用路径

Depthwise 卷积每个通道独立在 `kernel_h * kernel_w` 窗口上乘加，不再设置专用 `npu_dw_conv` 乘法器或触发器权重阵列，而是完全折叠到行引擎既有资源上：

1. **权重驻留复用 `wrow_b`**：当前 8 通道组的卷积核权重（最大支持 `256` 个核元素，覆盖 `7x7` 乃至更大核）直接加载进 `wrow_b[0..31][0..63]`。
2. **乘加累加复用 `racc[0..7]`**：滑窗扫描每个核位置 `(fh, fw)` 时，同时取出当前 8 个通道的激活与 `wrow_b` 中对应的 8 个权重做乘加，累加在 `racc[0..7]` 中。
3. **后处理直通 8-Lane PPU**：窗口结束后，`racc[0..7]` 恰好对应 `npu_ppu_bank` 的 8 条并行流水线，单拍送入 PPU 完成偏置、乘移重量化与写回。
4. **全局 DW 流式切片**：当 `kernel_h == in_h` 且 `kernel_w == in_w`（输出 `1x1xC`）且整张输入特征图大于 IFM SRAM 时，`npu_ctrl` 自动按 16 通道切片流式加载激活与权重，输出暂存在 OFM 高地址区 `DW_STREAM_OUT_BASE`，无需片上容纳整张特征图。

---

## 5. 8-Lane 分时复用 PPU 与算子协同

### 5.1 为什么 PPU 采用 8-Lane 而非 64-Lane

64-lane 行引擎在 `K` 维归约期间需要连续累加数十至数千拍才产出一组 64 个输出累加值（例如 `3x3x64` 卷积需 `K = 576` 拍）。因此写回带宽需求远低于乘加带宽：
- 例化 8 条 4 级流水 `npu_ppu`（`PPU_LANES = 8`），仅需 **8 拍** 即可排空 `racc[0..63]`，占典型卷积块周期不到 `2%`；
- 直接省去 `56` 条 60-bit 乘法-移位-钳位流水线，显著压缩组合逻辑与寄存器面积。

### 5.2 Elementwise Add 与 Pooling 精简通路

- **8-Lane 流水 Add**：双分支残差相加每拍处理 8 个通道（共 16 个缩放乘法器），直接与 8-lane PPU 截断写回流水重叠，无需单独的 `add_vec_out` 整行暂存寄存器。
- **Pooling + 40 拍串行倒数器**：AvgPool 在进入空间循环前，用 40 拍恢复余数状态机算出 `round(2^16 / pool_area)`（17-bit 定点倒数），窗口求和后做单次乘法右移即可得到均值，整芯片无任何组合除法器。

---

## 6. 利用率与性能观测口径

`npu_csr.v` 内置细粒度硬件周期直方图（`0x01C-0x03C` 与 `0x1A0-0x1D0`），可直接从寄存器读出单层或整网的周期去向：

- `0x01C PERF_CNT`：总墙钟周期（含主存 DMA 与计算）
- `0x020 MAC_CNT`：有效 MAC 次数
- `0x034 / 0x038 / 0x03C`：DMA 周期 / Compute 周期 / DMA-Compute 重叠周期
- `0x1A0-0x1D0`：计算微状态机在 `MAC 发射 / 等待激活 / 等待权重 / 等待双缓冲 / PPU 写回 / Pool / Add / Resize` 各阶段的精确拍数统计

计算阶段阵列利用率实测入口见 `rtl/tb/integration/test_array_util.py`（公开基准模型 `model_c / model_e / model_s`）。
