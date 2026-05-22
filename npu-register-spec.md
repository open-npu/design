# Open-NPU CSR 寄存器规格

> 版本：V1.0 Draft
> 日期：2025-05-19
> 总线：Wishbone Slave，32-bit数据宽度，字节寻址
> 地址空间：4KB (0x000 - 0xFFF)

---

## 1. 概述

| 参数 | 值 |
|------|-----|
| 数据宽度 | 32-bit |
| 寻址方式 | 字节地址，4字节对齐（bit[1:0]忽略） |
| 地址空间 | 4KB (1024个寄存器位置) |
| 字节序 | 小端 (Little-Endian) |
| 默认值 | 复位后所有寄存器为0（除VERSION/CONFIG只读寄存器） |

### 访问类型

| 缩写 | 含义 |
|------|------|
| RW | 读写 |
| RO | 只读（硬件写入，软件只能读） |
| W1C | 写1清零 |
| WO | 只写（读返回0） |

---

## 2. 寄存器地址映射总览

| 地址范围 | 组 | 用途 | 寄存器数量 |
|---------|-----|------|-----------|
| 0x000 - 0x03F | Group 0 | 控制与状态 | 16 |
| 0x040 - 0x0FF | Group 1 | 层参数配置 | 48 |
| 0x100 - 0x17F | Group 2 | DMA配置 | 32 |
| 0x180 - 0x1FF | Group 3 | 后处理配置 | 32 |
| 0x200 - 0x3FF | Group 4 | LUT数据 | 128 |
| 0x400 - 0xFFF | Group 5 | 保留(V2.0扩展) | 768 |

---

## 3. Group 0：控制与状态 (0x000 - 0x03F)

### 0x000 NPU_CTRL — 控制寄存器 [RW]

| Bit | 名称 | 复位值 | 描述 |
|-----|------|--------|------|
| [0] | START | 0 | 写1启动当前层计算（自动清零） |
| [1] | ABORT | 0 | 写1中止当前计算（自动清零） |
| [2] | SOFT_RST | 0 | 写1软复位NPU所有状态（自动清零） |
| [3] | AUTO_NEXT | 0 | 1=层完成后自动加载下一层描述符并启动 |
| [31:4] | — | 0 | 保留 |

### 0x004 NPU_STATUS — 状态寄存器 [RO]

| Bit | 名称 | 描述 |
|-----|------|------|
| [0] | BUSY | 1=NPU正在执行计算 |
| [1] | DMA_BUSY | 1=DMA正在搬运 |
| [2] | ERROR | 1=发生错误（详见ERROR_CODE） |
| [3] | DONE | 1=最近一次计算已完成 |
| [15:8] | CURR_LAYER | 当前正在执行的层号(0-255) |
| [31:16] | — | 保留 |

### 0x008 NPU_IRQ_EN — 中断使能寄存器 [RW]

| Bit | 名称 | 复位值 | 描述 |
|-----|------|--------|------|
| [0] | DONE_EN | 0 | 层计算完成中断使能 |
| [1] | ERROR_EN | 0 | 错误中断使能 |
| [2] | DMA_DONE_EN | 0 | DMA传输完成中断使能 |
| [31:3] | — | 0 | 保留 |

### 0x00C NPU_IRQ_STATUS — 中断状态寄存器 [W1C]

| Bit | 名称 | 描述 |
|-----|------|------|
| [0] | DONE_IRQ | 层计算完成（写1清除） |
| [1] | ERROR_IRQ | 错误发生（写1清除） |
| [2] | DMA_DONE_IRQ | DMA完成（写1清除） |
| [31:3] | — | 保留 |

### 0x010 NPU_ERROR_CODE — 错误码寄存器 [RO]

| Bit | 名称 | 描述 |
|-----|------|------|
| [3:0] | ERR_CODE | 0=无错误, 1=DMA地址越界, 2=配置非法, 3=scratchpad溢出, 4=总线超时 |
| [31:4] | — | 保留 |

### 0x014 NPU_VERSION — 版本寄存器 [RO]

| Bit | 名称 | 描述 |
|-----|------|------|
| [7:0] | PATCH | 补丁版本 |
| [15:8] | MINOR | 次版本 |
| [23:16] | MAJOR | 主版本 |
| [31:24] | — | 保留 |

复位值：0x00_01_00_00 (V1.0.0)

### 0x018 NPU_HW_CONFIG — 硬件配置寄存器 [RO]

| Bit | 名称 | 描述 |
|-----|------|------|
| [7:0] | ARRAY_SIZE | 脉动阵列边长（16/32/64） |
| [11:8] | NUM_ARRAYS | 阵列数量（1/2/4，用于多阵列并行扩展） |
| [15:12] | DW_CHANNELS_LOG2 | DW模块并行通道数 = 2^N（4=16ch, 5=32ch, 6=64ch） |
| [23:16] | SPAD_SIZE_4KB | Scratchpad总大小，单位4KB（如32=128KB, 128=512KB, max=1020KB） |
| [24] | HAS_INT16 | 1=支持INT16 |
| [25] | HAS_LUT | 1=支持LUT激活 |
| [26] | HAS_IPU | 1=集成了IPU模块 |
| [31:27] | — | 保留 |

### 0x01C NPU_PERF_CNT — 性能计数器 [RO]

| Bit | 名称 | 描述 |
|-----|------|------|
| [31:0] | CYCLE_COUNT | 自上次START以来经过的时钟周期数 |

### 0x020 NPU_MAC_CNT — MAC计数器 [RO]

| Bit | 名称 | 描述 |
|-----|------|------|
| [31:0] | MAC_COUNT | 实际执行的MAC操作数（用于计算利用率） |

### 0x024 NPU_SERIAL_LO — 序列号低32位 [RO]

| Bit | 名称 | 描述 |
|-----|------|------|
| [31:0] | SERIAL_LO | 芯片唯一序列号低32位（出厂烧入eFuse/OTP） |

### 0x028 NPU_SERIAL_HI — 序列号高32位 [RO]

| Bit | 名称 | 描述 |
|-----|------|------|
| [31:0] | SERIAL_HI | 芯片唯一序列号高32位 |

合计64-bit唯一ID，可用于：
- 软件License授权绑定（编译器/SDK绑定特定芯片）
- 生产追溯
- 防止仿冒（客户可验证IP来源）

### 0x02C NPU_VENDOR_ID — 厂商/产品标识 [RO]

| Bit | 名称 | 描述 |
|-----|------|------|
| [15:0] | VENDOR_ID | 厂商ID（开源版=0x4F4E "ON" for Open-NPU） |
| [31:16] | PRODUCT_ID | 产品型号ID（客户定制时可改） |

### 0x030 - 0x03F 保留

---

## 4. Group 1：层参数配置 (0x040 - 0x0FF)

### 0x040 LAYER_MODE — 层类型与模式 [RW]

| Bit | 名称 | 复位值 | 描述 |
|-----|------|--------|------|
| [3:0] | OP_TYPE | 0 | 算子类型（见下表） |
| [4] | DATA_TYPE | 0 | 0=INT8, 1=INT16 |
| [5] | PINGPONG_SEL | 0 | 当前使用的activation bank (0/1) |
| [31:6] | — | 0 | 保留 |

**OP_TYPE 编码：**

| 值 | 算子 | 说明 |
|----|------|------|
| 0 | CONV2D | 标准2D卷积 |
| 1 | DW_CONV | Depthwise卷积 |
| 2 | FC | 全连接（等价于1×1 conv, in_h=in_w=1） |
| 3 | POOLING | 池化 (Max/Avg) |
| 4 | ELTWISE_ADD | 逐元素加法 |
| 5 | RESIZE | 上采样/下采样 |
| 6 | DECONV | 转置卷积 |
| 7 | CONCAT | 通道拼接 |
| 8-15 | 保留 | V2.0扩展 |

### 0x044 IN_DIM_H_W — 输入高度与宽度 [RW]

| Bit | 名称 | 描述 |
|-----|------|------|
| [15:0] | IN_H | 输入特征图高度 |
| [31:16] | IN_W | 输入特征图宽度 |

### 0x048 IN_DIM_C — 输入通道数 [RW]

| Bit | 名称 | 描述 |
|-----|------|------|
| [15:0] | IN_C | 输入通道数 |
| [31:16] | — | 保留 |

### 0x04C OUT_DIM_H_W — 输出高度与宽度 [RW]

| Bit | 名称 | 描述 |
|-----|------|------|
| [15:0] | OUT_H | 输出特征图高度 |
| [31:16] | OUT_W | 输出特征图宽度 |

### 0x050 OUT_DIM_C — 输出通道数 [RW]

| Bit | 名称 | 描述 |
|-----|------|------|
| [15:0] | OUT_C | 输出通道数 |
| [31:16] | — | 保留 |

### 0x054 KERNEL_SIZE — 卷积核尺寸 [RW]

| Bit | 名称 | 描述 |
|-----|------|------|
| [7:0] | KERNEL_H | 卷积核高度 (1-15) |
| [15:8] | KERNEL_W | 卷积核宽度 (1-15) |
| [23:16] | DILATION_H | 膨胀系数高度 (1-15, 默认1) |
| [31:24] | DILATION_W | 膨胀系数宽度 (1-15, 默认1) |

### 0x058 STRIDE — 步长 [RW]

| Bit | 名称 | 描述 |
|-----|------|------|
| [7:0] | STRIDE_H | 垂直步长 (1-15) |
| [15:8] | STRIDE_W | 水平步长 (1-15) |
| [31:16] | — | 保留 |

### 0x05C PADDING — 填充 [RW]

| Bit | 名称 | 描述 |
|-----|------|------|
| [7:0] | PAD_TOP | 顶部padding |
| [15:8] | PAD_BOTTOM | 底部padding |
| [23:16] | PAD_LEFT | 左侧padding |
| [31:24] | PAD_RIGHT | 右侧padding |

### 0x060 POOL_CFG — 池化配置 [RW]

| Bit | 名称 | 描述 |
|-----|------|------|
| [0] | POOL_MODE | 0=MaxPool, 1=AvgPool |
| [7:4] | POOL_H | 池化窗口高度 |
| [11:8] | POOL_W | 池化窗口宽度 |
| [15:12] | POOL_STRIDE_H | 池化步长高度 |
| [19:16] | POOL_STRIDE_W | 池化步长宽度 |
| [20] | GLOBAL_POOL | 1=全局池化(忽略POOL_H/W) |
| [31:21] | — | 保留 |

### 0x064 RESIZE_CFG — Resize配置 [RW]

| Bit | 名称 | 描述 |
|-----|------|------|
| [0] | RESIZE_MODE | 0=最近邻, 1=双线性 |
| [15:8] | SCALE_H | 垂直缩放因子 (Q4.4定点: 如2.0=0x20) |
| [23:16] | SCALE_W | 水平缩放因子 (Q4.4定点) |
| [31:24] | — | 保留 |

### 0x068 DECONV_CFG — 转置卷积配置 [RW]

| Bit | 名称 | 描述 |
|-----|------|------|
| [7:0] | INSERT_H | 垂直方向插零数 |
| [15:8] | INSERT_W | 水平方向插零数 |
| [31:16] | — | 保留 |

### 0x06C CONCAT_CFG — Concat配置 [RW]

| Bit | 名称 | 描述 |
|-----|------|------|
| [15:0] | CONCAT_OFFSET | 拼接位置偏移(通道维度) |
| [31:16] | CONCAT_TOTAL_C | 拼接后总通道数 |

### 0x070 TILE_CFG — Tiling配置 [RW]

| Bit | 名称 | 描述 |
|-----|------|------|
| [15:0] | TILE_H | 当前tile的高度 |
| [31:16] | TILE_W | 当前tile的宽度 |

### 0x074 TILE_COUNT — Tile计数 [RW]

| Bit | 名称 | 描述 |
|-----|------|------|
| [15:0] | TILE_NUM_H | 垂直方向tile数量 |
| [31:16] | TILE_NUM_W | 水平方向tile数量 |

### 0x078 - 0x0FF 保留

---

## 5. Group 2：DMA配置 (0x100 - 0x17F)

### 0x100 DMA_IN_ADDR — 输入数据基地址 [RW]

| Bit | 描述 |
|-----|------|
| [31:0] | 输入特征图在外部SRAM中的字节地址 |

### 0x104 DMA_OUT_ADDR — 输出数据基地址 [RW]

| Bit | 描述 |
|-----|------|
| [31:0] | 输出特征图在外部SRAM中的字节地址 |

### 0x108 DMA_WEIGHT_ADDR — 权重数据基地址 [RW]

| Bit | 描述 |
|-----|------|
| [31:0] | 权重数据在外部SRAM/Flash中的字节地址 |

### 0x10C DMA_PARAM_ADDR — Per-channel参数基地址 [RW]

| Bit | 描述 |
|-----|------|
| [31:0] | Per-channel requantize参数（M/S/zp/bias）在外部SRAM的字节地址（4字节对齐） |

等价于 POST_PARAM_ADDR (0x184)，两者写任一个均可（硬件共用同一寄存器，双映射便于编程）。

### 0x110 DMA_IN_STRIDE — 输入数据步长 [RW]

| Bit | 名称 | 描述 |
|-----|------|------|
| [15:0] | IN_STRIDE_ROW | 输入相邻行之间的字节间距 |
| [31:16] | IN_STRIDE_CH | 输入相邻通道之间的字节间距（NCHW模式用） |

### 0x114 DMA_OUT_STRIDE — 输出数据步长 [RW]

| Bit | 名称 | 描述 |
|-----|------|------|
| [15:0] | OUT_STRIDE_ROW | 输出相邻行之间的字节间距 |
| [31:16] | OUT_STRIDE_CH | 输出相邻通道之间的字节间距 |

### 0x118 DMA_CTRL — DMA控制寄存器 [RW]

> **注意**：此寄存器是 RTL 硬件 DMA 引擎的配置 CSR，与离线模型 descriptor 中的 `sched_ctrl` 字段无关。
> 离线模型格式的调度控制位（DB_EN、FUSE_*）定义见 `design/interface-spec.md` §4。

| Bit | 名称 | 复位值 | 描述 |
|-----|------|--------|------|
| [0] | TRANSPOSE_EN | 0 | 1=搬运时做NCHW→NHWC转置 |
| [1] | OUT_TRANSPOSE_EN | 0 | 1=输出时做NHWC→NCHW转置 |
| [5:4] | BURST_LEN | 0 | Burst长度: 0=4, 1=8, 2=16, 3=32 (words) |
| [7:6] | PRIORITY | 0 | DMA优先级: 0=低, 3=高 |
| [8] | CIRCULAR_EN | 0 | 1=循环缓冲模式(ping-pong自动) |
| [31:9] | — | 0 | 保留 |

### 0x11C DMA_STATUS — DMA状态寄存器 [RO]

| Bit | 名称 | 描述 |
|-----|------|------|
| [0] | IN_BUSY | 输入DMA忙 |
| [1] | OUT_BUSY | 输出DMA忙 |
| [2] | WEIGHT_BUSY | 权重DMA忙 |
| [15:8] | XFER_COUNT | 已完成的传输块数 |
| [31:16] | — | 保留 |

### 0x120 DMA_ADD_B_ADDR — Add节点第二输入地址 [RW]

| Bit | 描述 |
|-----|------|
| [31:0] | Eltwise Add操作的第二输入（分支B）数据地址 |

等价于 POST_ADD_INPUT_ADDR (0x198)，双映射。

### 0x124 DMA_ADD_PARAM_ADDR — Add节点rescale参数地址 [RW]

| Bit | 描述 |
|-----|------|
| [31:0] | Add节点的rescale参数 (M_A/S_A/M_B/S_B) 在外部SRAM的地址 |

等价于 POST_ADD_PARAM_ADDR (0x194)，双映射。

### 0x128 DMA_IN_SIZE — 输入数据总大小 [RW]

| Bit | 描述 |
|-----|------|
| [31:0] | 输入数据总字节数 |

### 0x12C DMA_WEIGHT_SIZE — 权重数据总大小 [RW]

| Bit | 描述 |
|-----|------|
| [31:0] | 当前层权重总字节数 |

### 0x130 - 0x17F 保留

---

## 6. Group 3：后处理配置 (0x180 - 0x1FF)

Per-channel requantize 参数（M/S/bias/zp）存储在外部 SRAM 中，由 DMA 在每层计算前加载到片上 Parameter SRAM。此组寄存器配置加载地址、参数格式和后处理行为。

### 0x180 POST_CTRL — 后处理控制寄存器 [RW]

| Bit | 名称 | 复位值 | 描述 |
|-----|------|--------|------|
| [1:0] | PPU_MODE | 0 | 后处理模式: 0=CONV_REQ, 1=ADD, 2=RELU_ONLY, 3=PASS_THROUGH |
| [2] | RELU_EN | 0 | 1=在requantize后启用ReLU (max(0,x)) |
| [3] | RELU6_EN | 0 | 1=ReLU6模式 (clamp to [0, 6×scale]) |
| [4] | LUT_EN | 0 | 1=启用LUT激活函数（替代ReLU） |
| [5] | ZP_EN | 0 | 1=启用输出零点加法（INT8/INT16通用） |
| [6] | BIAS_EN | 0 | 1=启用per-channel bias加法 |
| [7] | INT16_OUT | 0 | 1=输出为INT16, 0=输出为INT8 |
| [31:8] | — | 0 | 保留 |

后处理管道执行顺序（固定，由硬件pipeline实现）：
```
acc(40-bit) → +bias_q[ch] → ×M[ch] → >>S[ch](+round) → +zp[ch] → clamp → ReLU/LUT → out
```

### 0x184 POST_PARAM_ADDR — Per-channel参数基地址 [RW]

| Bit | 描述 |
|-----|------|
| [31:0] | Per-channel requantize参数在外部SRAM的字节地址（4字节对齐） |

DMA 在计算启动前自动将参数加载到片上 Parameter SRAM。

### 0x188 POST_PARAM_COUNT — 参数通道数 [RW]

| Bit | 名称 | 描述 |
|-----|------|------|
| [15:0] | PARAM_CH_COUNT | 需加载的输出通道数（= OUT_C） |
| [31:16] | — | 保留 |

DMA 加载字节数 = PARAM_CH_COUNT × 14 bytes。

### 0x18C POST_CLAMP — Clamp范围 [RW]

| Bit | 名称 | 描述 |
|-----|------|------|
| [15:0] | CLAMP_MIN | 最小值 (INT16 signed: -32768~32767; INT8: -128~127) |
| [31:16] | CLAMP_MAX | 最大值 (INT16: 32767; INT8: 127; ReLU: min=0) |

### 0x190 POST_ACT_CFG — 激活函数配置 [RW]

| Bit | 名称 | 复位值 | 描述 |
|-----|------|--------|------|
| [3:0] | ACT_TYPE | 0 | 激活类型: 0=None, 1=ReLU, 2=ReLU6, 3=LUT, 4-15=保留 |
| [7:4] | LUT_INPUT_SHIFT | 0 | LUT输入预移位（将INT16缩放到LUT索引范围） |
| [31:8] | — | 0 | 保留 |

### 0x194 POST_ADD_PARAM_ADDR — Add节点参数地址 [RW]

| Bit | 描述 |
|-----|------|
| [31:0] | Add节点rescale参数在外部SRAM的字节地址 |

Add 参数格式（每节点 8 bytes）：
```
┌──────────────────────────────────────────────────────────────────┐
│ byte[0:1] │ M_A[14:0] + reserved[15]                             │
│ byte[2]   │ S_A[5:0] + reserved[7:6]                             │
│ byte[3]   │ reserved                                             │
│ byte[4:5] │ M_B[14:0] + reserved[15]                             │
│ byte[6]   │ S_B[5:0] + reserved[7:6]                             │
│ byte[7]   │ reserved                                             │
└──────────────────────────────────────────────────────────────────┘
```

### 0x198 POST_ADD_INPUT_ADDR — Add节点第二输入地址 [RW]

| Bit | 描述 |
|-----|------|
| [31:0] | Add操作的第二输入（分支B）在外部SRAM的字节地址 |

### 0x19C POST_ADD_STRIDE — Add节点第二输入步长 [RW]

| Bit | 名称 | 描述 |
|-----|------|------|
| [15:0] | ADD_IN_STRIDE_ROW | 第二输入相邻行字节间距 |
| [31:16] | — | 保留 |

### 0x1A0 - 0x1FF 保留

---

### Per-Channel 参数内存布局（软件规约）

模型转换工具为每层生成 per-channel 参数数组，存放于外部 SRAM：

```
基地址: POST_PARAM_ADDR

偏移 (per channel, 14 bytes each):
┌─────────────────────────────────────────────────────────────────┐
│  byte offset │ 内容                                              │
├──────────────┼──────────────────────────────────────────────────┤
│  [0:1]       │ M[14:0] (15-bit unsigned multiplier, bit15=0)    │
│  [2]         │ S[5:0]  (6-bit shift amount, bit7:6=0)           │
│  [3]         │ reserved (0x00)                                   │
│  [4:5]       │ zp[15:0] (16-bit signed output zero point)       │
│  [6:13]      │ bias_q[63:0] (64-bit signed, little-endian)      │
└──────────────┴──────────────────────────────────────────────────┘

总大小: OUT_C × 14 bytes
示例: 512通道 = 7,168 bytes
```

硬件加载后存入 PPU Parameter SRAM，计算时按 ch_idx 索引查询。

---

## 7. Group 4：LUT数据 (0x200 - 0x3FF)

256-entry查找表，用于任意非线性激活函数。

### INT8模式：64个寄存器存256个entries

地址 0x200 - 0x2FC (64个寄存器)

| 地址 | 内容 |
|------|------|
| 0x200 | LUT[0], LUT[1], LUT[2], LUT[3] (各8-bit, 从LSB开始) |
| 0x204 | LUT[4], LUT[5], LUT[6], LUT[7] |
| ... | ... |
| 0x2FC | LUT[252], LUT[253], LUT[254], LUT[255] |

使用方式：输入值x (INT8, 0-255范围转为无符号索引) → 输出 LUT[x]

### INT16模式：128个寄存器存256个entries

地址 0x200 - 0x3FC (128个寄存器)

| 地址 | 内容 |
|------|------|
| 0x200 | LUT[0](低16bit), LUT[1](高16bit) |
| 0x204 | LUT[2](低16bit), LUT[3](高16bit) |
| ... | ... |
| 0x3FC | LUT[254](低16bit), LUT[255](高16bit) |

---

## 8. 寄存器地址汇总表

| 偏移 | 名称 | 类型 | 描述 |
|------|------|------|------|
| 0x000 | NPU_CTRL | RW | 启动/中止/复位控制 |
| 0x004 | NPU_STATUS | RO | 运行状态 |
| 0x008 | NPU_IRQ_EN | RW | 中断使能 |
| 0x00C | NPU_IRQ_STATUS | W1C | 中断状态 |
| 0x010 | NPU_ERROR_CODE | RO | 错误码 |
| 0x014 | NPU_VERSION | RO | IP版本号 |
| 0x018 | NPU_HW_CONFIG | RO | 硬件配置信息(阵列大小/数量/scratchpad) |
| 0x01C | NPU_PERF_CNT | RO | 性能计数器(周期数) |
| 0x020 | NPU_MAC_CNT | RO | MAC计数器 |
| 0x024 | NPU_SERIAL_LO | RO | 序列号低32位(授权/追溯) |
| 0x028 | NPU_SERIAL_HI | RO | 序列号高32位 |
| 0x02C | NPU_VENDOR_ID | RO | 厂商ID + 产品ID |
| 0x040 | LAYER_MODE | RW | 层类型与数据类型 |
| 0x044 | IN_DIM_H_W | RW | 输入高度×宽度 |
| 0x048 | IN_DIM_C | RW | 输入通道数 |
| 0x04C | OUT_DIM_H_W | RW | 输出高度×宽度 |
| 0x050 | OUT_DIM_C | RW | 输出通道数 |
| 0x054 | KERNEL_SIZE | RW | 卷积核尺寸+膨胀系数 |
| 0x058 | STRIDE | RW | 步长 |
| 0x05C | PADDING | RW | 四边padding |
| 0x060 | POOL_CFG | RW | 池化配置 |
| 0x064 | RESIZE_CFG | RW | Resize配置 |
| 0x068 | DECONV_CFG | RW | 转置卷积配置 |
| 0x06C | CONCAT_CFG | RW | Concat配置 |
| 0x070 | TILE_CFG | RW | 当前Tile尺寸 |
| 0x074 | TILE_COUNT | RW | Tile数量 |
| 0x100 | DMA_IN_ADDR | RW | 输入数据地址 |
| 0x104 | DMA_OUT_ADDR | RW | 输出数据地址 |
| 0x108 | DMA_WEIGHT_ADDR | RW | 权重地址 |
| 0x10C | DMA_PARAM_ADDR | RW | Per-channel参数地址 (M/S/zp/bias) |
| 0x110 | DMA_IN_STRIDE | RW | 输入步长(行/通道) |
| 0x114 | DMA_OUT_STRIDE | RW | 输出步长 |
| 0x118 | DMA_CTRL | RW | DMA控制(转置/burst/优先级) |
| 0x11C | DMA_STATUS | RO | DMA状态 |
| 0x120 | DMA_ADD_B_ADDR | RW | Add第二输入地址 |
| 0x124 | DMA_ADD_PARAM_ADDR | RW | Add rescale参数地址 |
| 0x128 | DMA_IN_SIZE | RW | 输入数据总大小 |
| 0x12C | DMA_WEIGHT_SIZE | RW | 权重数据总大小 |
| 0x180 | POST_CTRL | RW | 后处理模式/使能控制 |
| 0x184 | POST_PARAM_ADDR | RW | Per-channel参数基地址 |
| 0x188 | POST_PARAM_COUNT | RW | 参数通道数 |
| 0x18C | POST_CLAMP | RW | Clamp范围 (INT8/INT16) |
| 0x190 | POST_ACT_CFG | RW | 激活函数配置 |
| 0x194 | POST_ADD_PARAM_ADDR | RW | Add节点rescale参数地址 |
| 0x198 | POST_ADD_INPUT_ADDR | RW | Add第二输入地址 |
| 0x19C | POST_ADD_STRIDE | RW | Add第二输入步长 |
| 0x200-0x3FF | LUT_DATA | RW | 256-entry激活函数查找表 |

---

## 9. 典型使用流程

### 9.1 CPU配置一层Conv2D + ReLU的示例 (INT16 per-channel)

```c
// 假设模型转换工具已生成:
//   - 权重数据 @ 0x08000000 (INT16, NHWC layout)
//   - per-channel参数 @ 0x08020000 (M/S/zp/bias, 8 bytes/ch)
//   - 输入激活 @ 0x20000000 (INT16, 前一层输出)
//   - 输出目标 @ 0x20010000

// 1. 配置层参数
REG(LAYER_MODE)    = (1 << 4) | 0x00;  // DATA_TYPE=INT16, OP_TYPE=CONV2D
REG(IN_DIM_H_W)   = (56 << 16) | 56;  // 56×56输入
REG(IN_DIM_C)     = 64;               // 64通道输入
REG(OUT_DIM_H_W)  = (56 << 16) | 56;  // 56×56输出
REG(OUT_DIM_C)    = 128;              // 128通道输出
REG(KERNEL_SIZE)  = (1 << 24) | (1 << 16) | (3 << 8) | 3; // 3×3, dilation=1
REG(STRIDE)       = (1 << 8) | 1;     // stride=1
REG(PADDING)      = (1 << 24) | (1 << 16) | (1 << 8) | 1; // pad=1 all sides

// 2. 配置DMA地址
REG(DMA_IN_ADDR)      = 0x20000000;   // 输入激活地址
REG(DMA_OUT_ADDR)     = 0x20010000;   // 输出目标地址
REG(DMA_WEIGHT_ADDR)  = 0x08000000;   // 权重地址
REG(DMA_PARAM_ADDR)   = 0x08020000;   // per-channel参数地址 (M/S/zp/bias)
REG(DMA_IN_STRIDE)    = (56 * 64 * 2); // row stride = W × C_in × 2B
REG(DMA_CTRL)         = 0x00;         // 无转置, burst=4

// 3. 配置后处理 (per-channel requantize + ReLU)
REG(POST_CTRL)        = (1 << 6)      // BIAS_EN = 1
                      | (0 << 5)      // ZP_EN = 0 (symmetric, zp=0)
                      | (1 << 2)      // RELU_EN = 1
                      | (1 << 7)      // INT16_OUT = 1
                      | 0x00;         // PPU_MODE = CONV_REQ
REG(POST_PARAM_ADDR)  = 0x08020000;   // 同 DMA_PARAM_ADDR
REG(POST_PARAM_COUNT) = 128;          // 128通道参数 = 128×10 = 1280 bytes
REG(POST_CLAMP)       = (32767 << 16) | (uint16_t)(-32768); // INT16 full range
REG(POST_ACT_CFG)     = 1;            // ACT_TYPE = ReLU

// 4. 启动
REG(NPU_CTRL) = 0x01;  // START

// 5. 等待完成
while (!(REG(NPU_IRQ_STATUS) & 0x01));  // 等DONE_IRQ
REG(NPU_IRQ_STATUS) = 0x01;            // 写1清中断
```

### 9.2 Eltwise Add (残差连接) 示例

```c
// Add: out = rescale_A(branch_A) + rescale_B(branch_B)
// 两路输入scale不同, 需各自rescale到输出scale

// 层参数 (Add是element-wise, 输入输出尺寸相同)
REG(LAYER_MODE)    = (1 << 4) | 0x04;  // INT16, OP_TYPE=ELTWISE_ADD
REG(IN_DIM_H_W)   = (14 << 16) | 14;
REG(IN_DIM_C)     = 256;
REG(OUT_DIM_H_W)  = (14 << 16) | 14;
REG(OUT_DIM_C)    = 256;

// DMA: 分支A的输入地址 (主输入)
REG(DMA_IN_ADDR)      = 0x20000000;   // branch A 数据
REG(DMA_OUT_ADDR)     = 0x20008000;   // 输出目标

// Add 专用配置
REG(POST_ADD_INPUT_ADDR) = 0x20004000;   // branch B 数据地址
REG(POST_ADD_PARAM_ADDR) = 0x08030000;   // Add rescale参数 (M_A/S_A/M_B/S_B)
REG(POST_ADD_STRIDE)     = 14 * 256 * 2; // branch B row stride

// 后处理: Add mode + ReLU
REG(POST_CTRL)  = (1 << 2)   // RELU_EN
                | (1 << 7)   // INT16_OUT
                | 0x01;      // PPU_MODE = ADD
REG(POST_CLAMP) = (32767 << 16) | (uint16_t)(-32768);
REG(POST_ACT_CFG) = 1;      // ReLU

// 启动
REG(NPU_CTRL) = 0x01;
while (!(REG(NPU_IRQ_STATUS) & 0x01));
REG(NPU_IRQ_STATUS) = 0x01;
```

### 9.3 INT8 模式 Conv2D (per-channel scale+zp)

```c
// INT8 模式: 非对称量化, 每通道有独立 zp
REG(LAYER_MODE)    = 0x00;            // DATA_TYPE=INT8, OP_TYPE=CONV2D
REG(OUT_DIM_C)    = 128;

// 后处理: per-channel requantize with zero-point
REG(POST_CTRL)    = (1 << 6)          // BIAS_EN
                  | (1 << 5)          // ZP_EN = 1 (INT8 needs zp)
                  | (1 << 2)          // RELU_EN
                  | (0 << 7)          // INT16_OUT = 0 (INT8 output)
                  | 0x00;             // CONV_REQ mode
REG(POST_PARAM_ADDR)  = 0x08040000;  // INT8 per-channel参数
REG(POST_PARAM_COUNT) = 128;
REG(POST_CLAMP) = (127 << 16) | (uint16_t)(-128); // INT8 range

// ... 其余同 9.1
REG(NPU_CTRL) = 0x01;
```

### 9.4 Sigmoid激活（通过LUT）

```c
// 预计算sigmoid LUT: 256 entries, INT16 mode
// LUT[i] = quantize(sigmoid(dequantize(i)))
for (int i = 0; i < 128; i++) {
    uint32_t packed = (uint16_t)lut_values[i*2]
                    | ((uint32_t)(uint16_t)lut_values[i*2+1] << 16);
    REG(0x200 + i*4) = packed;
}

// 后处理: requantize → LUT (替代ReLU)
REG(POST_CTRL)    = (1 << 6)  // BIAS_EN
                  | (1 << 4)  // LUT_EN
                  | (1 << 7)  // INT16_OUT
                  | 0x00;     // CONV_REQ
REG(POST_ACT_CFG) = (4 << 4) // LUT_INPUT_SHIFT=4 (INT16→8-bit index)
                   | 3;       // ACT_TYPE = LUT
```

---

## 10. 设计备注

1. **Per-channel参数加载流程**：CPU配置 POST_PARAM_ADDR 和 POST_PARAM_COUNT 后，NPU 在 START 后自动通过 DMA 将参数加载到片上 Parameter SRAM。参数 SRAM 大小固定为 8KB（支持最多 512 通道 × 14 bytes = 7KB，余量用于对齐），大于 512 通道时需分 tile 处理（Control Unit 自动管理）。

2. **参数 SRAM 生命周期**：每层计算前加载，计算完成后内容失效。下一层开始时重新加载新参数。由于参数量小（128ch = 1.25KB），加载时间远小于计算时间，不影响性能。

3. **Bias 合并到参数包**：传统设计中 bias 独立存储，本设计将 bias_q 合并到 per-channel 参数包中（14 bytes/ch 内含 64-bit bias + 16-bit zp），减少一次 DMA 事务，简化控制逻辑。使用 int64 bias 是因为 INT8 量化大通道卷积的 bias 累加精度需要。

4. **Add 节点参数**：Add rescale 参数很少（每个 Add 节点 8 bytes），对于整个模型（12 个 Add 节点 = 96 bytes），可一次性加载或跟随对应层的参数一起加载。

5. **Tiling由离线编译器计算**：编译器（`tools/tiling.py`）根据SRAM预算计算最优tile尺寸，结果写入layer descriptor的tile_h/tile_w/tile_num_h/tile_num_w字段。Control Unit按descriptor指定的tile参数执行循环，不自行推导tile尺寸。CPU每层配置一次完整参数，ping-pong切换由Control Unit管理。

6. **AUTO_NEXT模式**：如果配置了层描述符链表（存在外部SRAM），NPU可以自动加载下一层配置，无需CPU每层干预。V1.0可先不实现，每层手动配置。

7. **性能计数器**：NPU_PERF_CNT和NPU_MAC_CNT用于性能分析和利用率计算：
   - 利用率 = MAC_COUNT / (CYCLE_COUNT × ARRAY_SIZE²)

8. **LUT填充时机**：LUT数据在NPU空闲时写入，不能在计算过程中修改。如果相邻层激活函数不同，需在层间切换时更新LUT。

9. **双映射寄存器**：DMA_PARAM_ADDR 与 POST_PARAM_ADDR、DMA_ADD_B_ADDR 与 POST_ADD_INPUT_ADDR 等为同一物理寄存器的双地址映射，方便驱动在配置 DMA 组或后处理组时就近访问。
