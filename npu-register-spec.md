# Open-NPU CSR 寄存器规格

> 版本：V1.3
> 更新：2026-11-26
> 总线：Wishbone-B4 Slave，32-bit 数据宽度，字节寻址（单周期应答）
> 地址空间：4 KB (`0x000 - 0xFFF`)，与 `rtl/src/npu_csr.v` 严格一致

---

## 1. 概述

| 参数 | 值 |
|------|-----|
| 数据宽度 | 32-bit |
| 寻址方式 | 字节地址，4 字节对齐（`addr[1:0]` 忽略） |
| 地址空间 | 4 KB |
| 字节序 | 小端（Little-Endian） |
| 复位默认值 | 所有可写寄存器复位为 `0`；只读标识寄存器返回固定硬件常量 |

### 访问类型

| 缩写 | 含义 |
|------|------|
| `RW` | 读写寄存器 |
| `RO` | 只读寄存器（写操作忽略） |
| `W1C` | 写 `1` 清零 |
| `SC` | 自清零控制位（写 `1` 产生单拍脉冲，读回 `0`） |

---

## 2. 寄存器地址映射总览

| 地址范围 | 分组 | 用途 |
|---------|------|------|
| `0x000 - 0x03F` | Group 0 | 全局控制、状态、中断与顶层性能计数 |
| `0x040 - 0x0FF` | Group 1 | 层几何、算子模式、切块与片上 SRAM 基址 |
| `0x100 - 0x17F` | Group 2 | DMA 地址/步长、K-major 权重布局、参数预取与 Super-Block 控制 |
| `0x180 - 0x19F` | Group 3A | 后处理（PPU）与双分支 Add 配置 |
| `0x1A0 - 0x1FF` | Group 3B | 计算状态机细粒度周期直方图计数器（RO） |
| `0x200 - 0x3FF` | Group 4 | 保留（历史 LUT 激活表已裁撤，`HAS_LUT = 0`，读回 `0`） |
| `0x400 - 0x4AF` | Group 5 | Super-Block 描述符窗口（最多 4 个槽位 x 44 个 32-bit 字） |
| `0x4B0 - 0xFFF` | — | 保留 |

---

## 3. Group 0：控制、状态与全局计数器 (`0x000 - 0x03F`)

### `0x000 NPU_CTRL` — 控制寄存器 `[RW/SC]`

| Bit | 名称 | 类型 | 描述 |
|-----|------|------|------|
| `[0]` | `START` | SC | 写 `1` 启动当前层推理（单拍脉冲后自动清零） |
| `[1]` | `ABORT` | SC | 写 `1` 中止在飞计算与 DMA，并阻止 `AUTO_NEXT` 续跑 |
| `[2]` | `SOFT_RST` | SC | 写 `1` 软复位控制器、DMA 与计算状态机（保留 CSR 与 SRAM 内容） |
| `[3]` | `AUTO_NEXT` | RW | `1` = 当前层完成后若 `CURR_LAYER + 1 < LAYER_COUNT` 则自动启动下一层 |
| `[31:4]` | — | RO | 保留（`0`） |

### `0x004 NPU_STATUS` — 运行状态寄存器 `[RO]`

| Bit | 名称 | 描述 |
|-----|------|------|
| `[0]` | `BUSY` | `1` = NPU 控制器或计算核忙 |
| `[1]` | `DMA_BUSY` | `1` = DMA 引擎正在传输 |
| `[2]` | `ERROR` | `1` = 硬件守卫或总线报错（详见 `NPU_ERROR_CODE`） |
| `[3]` | `DONE` | 最近一次启动的层已完成脉冲/状态 |
| `[15:8]` | `CURR_LAYER` | 当前层序号（`0 - 255`） |
| `[31:16]` | — | 保留（`0`） |

### `0x008 NPU_IRQ_EN` / `0x00C NPU_IRQ_STATUS` — 中断使能 `[RW]` 与状态 `[W1C]`

| Bit | 名称 | 描述 |
|-----|------|------|
| `[0]` | `DONE_IRQ` | 层推理完成中断 |
| `[1]` | `ERROR_IRQ` | 错误中断 |
| `[2]` | `DMA_DONE_IRQ` | 单次 DMA 传输完成中断 |
| `[31:3]` | — | 保留（`0`） |

### `0x010 NPU_ERROR_CODE` — 硬件守卫错误码 `[RO]`

`[3:0]` 返回 `0`（无错误）或硬件守卫码 `E1 - E9`（详见 `rtl_design_overview.md` §3）。

### `0x014 NPU_VERSION` — 版本号 `[RO]`

默认返回 `0x00010000`（`V1.0.0`：`[23:16]` Major，`[15:8]` Minor，`[7:0]` Patch）。

### `0x018 NPU_HW_CONFIG` — 硬件能力只读寄存器 `[RO]`

| Bit | 名称 | 默认值 | 描述 |
|-----|------|--------|------|
| `[7:0]` | `MAC_LANES` | `64` | 稠密卷积/FC 一维行引擎并行 MAC 通道数 |
| `[11:8]` | `NUM_ARRAYS` | `1` | 行引擎个数 |
| `[15:12]` | `DW_CH_LOG2` | `3` | DW 每组并行通道数对数（`2^3 = 8` 通道） |
| `[23:16]` | `SPAD_KB` | `172` | 片上四库 Scratchpad 总容量（单位：`1 KB`，`32+48+80+12 = 172`） |
| `[24]` | `HAS_INT16` | `1` | `1` = 支持有符号 INT16 模式 |
| `[25]` | `HAS_LUT` | `0` | `0` = LUT 非线性查表已裁撤 |
| `[26]` | `HAS_IPU` | `0` | 图像前端扩展标志 |
| `[28:27]` | `DMA_BUS_WIDTH` | `0` | 外部 DMA 数据位宽编码：`0` = `32-bit`，`1` = `64-bit`，`2` = `128-bit` |
| `[29]` | `DMA_BUS_PROTO` | `0` | 外部 DMA 主接口协议：`0` = Wishbone-B4 Master，`1` = AXI4 Burst Master |
| `[31:30]` | — | `0` | 保留 |

### `0x01C - 0x03C` 标识、层数与全局性能寄存器

| 偏移 | 名称 | 类型 | 描述 |
|------|------|------|------|
| `0x01C` | `NPU_PERF_CNT` | RO | 自上次 `START` 起累计的总时钟周期数 |
| `0x020` | `NPU_MAC_CNT` | RO | 累计有效 MAC 操作计数 |
| `0x024` | `NPU_SERIAL_LO` | RO | 序列号低 32 位（开源默认 `0x00000001`） |
| `0x028` | `NPU_SERIAL_HI` | RO | 序列号高 32 位（开源默认 `0x00000000`） |
| `0x02C` | `NPU_VENDOR_ID` | RO | `{Product=0x0001, Vendor=0x4F4E ("ON")}` |
| `0x030` | `NPU_LAYER_COUNT` | RW | `AUTO_NEXT` 模式下的总层数 `[7:0]` |
| `0x034` | `NPU_PERF_DMA_CNT` | RO | DMA 活跃周期计数 |
| `0x038` | `NPU_PERF_COMPUTE_CNT` | RO | 计算核活跃周期计数 |
| `0x03C` | `NPU_PERF_OVERLAP_CNT` | RO | DMA 与计算核并行重叠周期计数 |

---

## 4. Group 1：层参数配置 (`0x040 - 0x0FF`)

### `0x040 LAYER_MODE` — 算子类型、精度与输入零点 `[RW]`

| Bit | 名称 | 描述 |
|-----|------|------|
| `[3:0]` | `OP_TYPE` | `0`=CONV2D, `1`=DW_CONV, `2`=FC, `3`=POOLING, `4`=ELTWISE_ADD, `5`=RESIZE, `7`=CONCAT（`6` 已停用） |
| `[4]` | `DATA_TYPE` | `0` = INT8，`1` = INT16 |
| `[5]` | `PINGPONG_SEL` | 激活双缓冲初始 bank 选择 |
| `[15:8]` | `IN_ZP` | 有符号 8-bit 输入零点（用于边界 padding 填充） |
| `[31:16]` | — | 保留 |

### `0x044 - 0x078` 几何尺寸、卷积参数与切块配置 `[RW]`

| 偏移 | 名称 | 位域定义 |
|------|------|----------|
| `0x044` | `IN_DIM_H_W` | `[15:0]` `IN_H`，`[31:16]` `IN_W` |
| `0x048` | `IN_DIM_C` | `[15:0]` `IN_C` |
| `0x04C` | `OUT_DIM_H_W` | `[15:0]` `OUT_H`，`[31:16]` `OUT_W` |
| `0x050` | `OUT_DIM_C` | `[15:0]` `OUT_C` |
| `0x054` | `KERNEL_SIZE` | `[7:0]` `KH`，`[15:8]` `KW`，`[23:16]` `DILATION_H`，`[31:24]` `DILATION_W` |
| `0x058` | `STRIDE` | `[7:0]` `STRIDE_H`，`[15:8]` `STRIDE_W` |
| `0x05C` | `PADDING` | `[7:0]` `PAD_TOP`，`[15:8]` `PAD_BOTTOM`，`[23:16]` `PAD_LEFT`，`[31:24]` `PAD_RIGHT` |
| `0x060` | `POOL_CFG` | `[0]` `0=Max/1=Avg`，`[7:4]` `POOL_H`，`[11:8]` `POOL_W`，`[15:12]` `STRIDE_H`，`[19:16]` `STRIDE_W`，`[20]` `GLOBAL_POOL` |
| `0x064` | `RESIZE_CFG` | `[0]` `0=Nearest/1=Bilinear`，`[15:8]` `SCALE_H` (Q4.4)，`[23:16]` `SCALE_W` (Q4.4) |
| `0x068` | — | 保留（历史 `DECONV_CFG` 已裁撤） |
| `0x06C` | `CONCAT_CFG` | `[15:0]` `CONCAT_OFFSET`，`[31:16]` `CONCAT_TOTAL_C` |
| `0x070` | `TILE_CFG` | `[15:0]` `TILE_H`（`0` = 不切块），`[31:16]` `TILE_W` |
| `0x074` | `TILE_COUNT` | `[15:0]` `TILE_NUM_H`，`[31:16]` `TILE_NUM_W` |
| `0x078` | `SRAM_BASE` | `[12:0]` `ACT_BASE`（IFM 字偏移），`[28:16]` `OUT_BASE`（OFM 字偏移） |

---

## 5. Group 2：DMA、权重布局、参数预取与 Super-Block (`0x100 - 0x17F`)

| 偏移 | 名称 | 类型 | 描述 |
|------|------|------|------|
| `0x100` | `DMA_IN_ADDR` | RW | 输入激活主存字节基址 |
| `0x104` | `DMA_OUT_ADDR` | RW | 输出激活主存字节基址 |
| `0x108` | `DMA_WEIGHT_ADDR` | RW | 权重数据主存字节基址 |
| `0x10C` | `DMA_PARAM_ADDR` | RW | 逐通道参数主存字节基址（与 `0x184` 双映射） |
| `0x110` | `DMA_IN_STRIDE` | RW | `[15:0]` 输入行字节跨度 |
| `0x114` | `DMA_OUT_STRIDE` | RW | `[15:0]` 输出行字节跨度 |
| `0x118` | `DMA_CTRL` | RW | 调度与双缓冲/融合控制位（`DB_EN`、`FUSE_*` 等） |
| `0x11C` | `DMA_STATUS` | RO | `[2:0]` DMA 忙状态标志 |
| `0x120` | `DMA_ADD_B_ADDR` | RW | Add 第二输入分支主存地址（与 `0x198` 双映射） |
| `0x124` | `DMA_ADD_PARAM_ADDR` | RW | Add 双路缩放参数主存地址（与 `0x194` 双映射） |
| `0x128` | `DMA_IN_SIZE` | RW | 输入激活总字节数 |
| `0x12C` | `DMA_WEIGHT_SIZE` | RW | 当前层（或单组）权重字节数 |
| `0x130` | `DMA_OUT_SIZE` | RW | 输出激活总字节数 |
| `0x134` | `DMA_TILE_IN_SIZE` | RW | 单空间 tile 输入字节数（`DB_EN` 切块使用） |
| `0x138` | `DMA_TILE_OUT_SIZE` | RW | 单空间 tile 输出字节数 |
| `0x13C` | `TILE_IN_HW` | RW | `[31:16]` `TILE_IN_H`，`[15:0]` `TILE_IN_W` |
| `0x140` | `DMA_STORE_MODE` | RW | `[0]` `PER_TILE_STORE_EN`（切块按 2D 行距写回主存大图） |
| `0x144` | `DMA_ROW_CFG` | RW | `[31:16]` `ROW_COUNT`，`[15:0]` `ROW_LEN`（2D DMA 配置） |
| `0x148` | `DMA_WGT_PER_OC` | RW | `[29:0]` 每组权重 32-bit 字数（`0` = 常驻不重载）；`[30]` 启用 `2 x 40 KB` 权重 Ping-Pong；`[31]` K-slice 流式模式 |
| `0x14C` | `WGT_LAYOUT` | RW | `[0]` `1` = K-major 64-lane 打包布局（现行标准）；`0` = 旧版兼容 |
| `0x150` | `NEXT_PARAM_ADDR` | RW | 下一层逐通道参数预取主存地址 |
| `0x154` | `NEXT_PARAM_CFG` | RW | `[31]` 预取使能，`[15:0]` 预取 32-bit 字数 |
| `0x158` | `PARAM_BANK_CTRL` | RW | 参数 SRAM 前后半区（ping/pong）读取与预取目标选择 |
| `0x15C` | `PARAM_BANK_STATUS` | RO | `[31:16]` 半库深度（words），`[0]` `NEXT_PARAM_READY` |
| `0x160` | `SUPER_CTRL` | RW | `[0]` Super-Block 请求，`[2:1]` 启动槽位，`[3]` 启用第 4 槽残差 `Add`，`[4]` 按阶段流式加载参数，`[9:8]` 窗口映射槽位（`0..3`） |
| `0x164` | `SUPER_STATUS` | RO | `[3:2]` 当前激活槽位，`[1]` 冻结标志，`[0]` 硬件多槽状态机使能标志 |

---

## 6. Group 3：后处理配置与计算周期直方图 (`0x180 - 0x1FF`)

### `0x180 - 0x19C` 后处理配置 `[RW]`

| 偏移 | 名称 | 位域定义 |
|------|------|----------|
| `0x180` | `POST_CTRL` | `[1:0]` `PPU_MODE` (`0`=CONV_REQ, `1`=ADD, `2`=RELU_ONLY, `3`=PASSTHROUGH)，`[2]` `RELU_EN`，`[3]` `RELU6_EN`，`[5]` `ZP_EN`，`[6]` `BIAS_EN`，`[7]` `INT16_OUT`（`[4]` 保留） |
| `0x184` | `POST_PARAM_ADDR` | 同 `0x10C DMA_PARAM_ADDR` |
| `0x188` | `POST_PARAM_COUNT` | `[15:0]` 输出通道参数个数（每通道 14 字节：`M[14:0]`, `S[5:0]`, `zp[15:0]`, `bias_q[63:0]`） |
| `0x18C` | `POST_CLAMP` | `[15:0]` `CLAMP_MIN`（有符号 16-bit），`[31:16]` `CLAMP_MAX`（有符号 16-bit） |
| `0x190` | `POST_ACT_CFG` | `[3:0]` 激活函数辅助编码（`0`=None，`1`=ReLU，`2`=ReLU6） |
| `0x194` | `POST_ADD_PARAM_ADDR` | 同 `0x124 DMA_ADD_PARAM_ADDR`（8 字节：`M_A, S_A, M_B, S_B`） |
| `0x198` | `POST_ADD_INPUT_ADDR` | 同 `0x120 DMA_ADD_B_ADDR` |
| `0x19C` | `POST_ADD_STRIDE` | `[15:0]` Add 分支 B 行字节跨度 |

PPU 4 级流水执行顺序（44-bit 累加器输入）：
```
acc(44-bit) -> + bias_q[ch] -> * M[ch] + (1 << (S[ch]-1)) -> >>> S[ch] -> + zp[ch] -> clamp -> ReLU/ReLU6 -> out
```

### `0x1A0 - 0x1D0` 计算微状态机周期直方图 `[RO]`

| 偏移 | 名称 | 统计含义 |
|------|------|----------|
| `0x1A0` | `PERF_ST_MAC` | Row / DW 乘加发射状态周期数 |
| `0x1A4` | `PERF_ST_WAIT` | 等待激活 gather / 权重 / 双缓冲握手总周期数 |
| `0x1A8` | `PERF_ST_WB` | 8-lane PPU 排空与 OFM 写回周期数 |
| `0x1AC` | `PERF_ST_POOL` | Pooling 通路周期数 |
| `0x1B0` | `PERF_ST_ADD` | Elementwise Add 通路周期数 |
| `0x1B4` | `PERF_ST_RSZ` | Resize 通路周期数 |
| `0x1B8` | `PERF_ST_CFG` | 层/块初始化与边界设置周期数 |
| `0x1BC` | `PERF_ST_OTH` | 其余控制过渡周期数 |
| `0x1C0` | `PERF_WAIT_ACT` | 等待激活数据就绪细分周期数 |
| `0x1C4` | `PERF_WAIT_WGT` | 等待权重加载细分周期数 |
| `0x1C8` | `PERF_WAIT_DB` | 等待双缓冲 `db_prefetch_done` 细分周期数 |
| `0x1CC` | `PERF_MAC_IDLE` | 处于 MAC 状态但未发射有效乘加的空拍数 |
| `0x1D0` | `PERF_WAIT_STORE` | 等待主存写回完成周期数 |

---

## 7. Group 5：Super-Block 描述符窗口 (`0x400 - 0x4AF`)

当 `SUPER_CTRL[9:8]` 选择槽位 `s`（默认 `0 <= s <= 2`，当 `SUPER_CTRL[3] == 1` 时开放第 4 槽 `s == 3`）且未处于运行冻结态时，向 `0x400 + 4*w`（`0 <= w < 44`）读写即访问第 `s` 槽的第 `w` 个层描述符寄存器镜像。写入 `NPU_CTRL.START` 的同一总线周期若 `SUPER_CTRL[0] == 1`，硬件原子冻结当前启动槽位，防止运行期描述符被意外改写。

---

## 8. 典型单层编程示例（K-Major Conv2D + ReLU）

```c
// 1. 层几何与精度: INT8 Conv2D 3x3, 56x56x64 -> 56x56x128, pad=1, stride=1
REG(0x040) = 0x00;                              // OP_CONV2D, INT8, in_zp=0
REG(0x044) = (56u << 16) | 56u;
REG(0x048) = 64u;
REG(0x04C) = (56u << 16) | 56u;
REG(0x050) = 128u;
REG(0x054) = (1u << 24) | (1u << 16) | (3u << 8) | 3u;
REG(0x058) = (1u << 8) | 1u;
REG(0x05C) = 0x01010101u;
REG(0x070) = 0u;                                // 不切块

// 2. DMA 与 K-major 权重布局
REG(0x100) = act_in_addr;
REG(0x104) = act_out_addr;
REG(0x108) = wgt_kmajor_addr;
REG(0x10C) = param_addr;
REG(0x128) = 56u * 56u * 64u;
REG(0x12C) = 128u * 3u * 3u * 64u;
REG(0x130) = 56u * 56u * 128u;
REG(0x14C) = 1u;                                // WGT_LAYOUT = K-major

// 3. PPU 后处理: INT64 bias + per-channel requant + ReLU
REG(0x180) = (1u << 6) | (1u << 2) | 0x00u;     // BIAS_EN | RELU_EN | CONV_REQ
REG(0x188) = 128u;
REG(0x18C) = (127u << 16) | (uint16_t)(-128);
REG(0x190) = 1u;

// 4. 清中断、启动并等待完成
REG(0x00C) = 0x07u;
REG(0x000) = 0x01u;                             // CTRL_START
while (!(REG(0x00C) & 0x01u)) {}
REG(0x00C) = 0x07u;
```
