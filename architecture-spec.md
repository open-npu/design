# Open-NPU 整体架构规格

> 版本：V1.0 Draft
> 日期：2025-05-18
> 状态：阶段1架构定义

---

## 1. 设计概述

| 参数 | 规格 |
|------|------|
| 定位 | MCU挂载NPU IP核，端侧CNN推理加速 |
| 峰值算力 | 0.2 TOPS (200 GOPS) @ 200MHz |
| 数据类型 | INT16 activation × INT16 weight（主力），兼容INT8模式 |
| 累加器 | 40-bit signed，支持1×1 Conv最大C_in=512 |
| 工作负载 | CNN：Conv2D, DW Conv, FC, Pooling, ReLU, Add |
| 控制方式 | 层级寄存器配置 + 可编程后处理流水线 |
| 存储架构 | 分布式Scratchpad（64-128KB） |
| 总线 | Wishbone（LiteX原生），商业化时加AXI Wrapper |
| Demo SoC | VexRiscv + LiteX |
| 许可证 | Apache 2.0 |

---

## 2. 顶层架构框图

```
                         Wishbone Bus (LiteX SoC)
                    ┌──────────┴──────────────────────┐
                    │                                  │
               CSR Slave                         DMA Master
            (层级寄存器配置)                    (Wishbone B4 Pipeline)
                    │                                  │
┌───────────────────┼──────────────────────────────────┼──────────────┐
│                   │          NPU Top                 │              │
│                   ▼                                  ▼              │
│  ┌─────────────────────┐              ┌──────────────────────┐     │
│  │    Control Unit      │              │      DMA Engine      │     │
│  │  (层调度/状态机)      │              │  (外部SRAM↔Scratchpad)│     │
│  └──────────┬──────────┘              └───────────┬──────────┘     │
│             │ 控制信号                             │ 数据搬运       │
│             ▼                                     ▼              │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    Scratchpad Memory                         │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │   │
│  │  │Weight Buffer │  │Activation Buf│  │  DW Buffer       │  │   │
│  │  │  32-64 KB    │  │32-64KB(PingP)│  │  (小型专用)       │  │   │
│  │  └──────┬───────┘  └──────┬───────┘  └────────┬─────────┘  │   │
│  └─────────┼─────────────────┼───────────────────┼─────────────┘   │
│            │                 │                   │                  │
│            ▼                 ▼                   ▼                  │
│  ┌──────────────────┐  ┌─────────┐    ┌─────────────────────┐     │
│  │  Systolic Array   │  │(共享输入)│    │   DW Conv Module    │     │
│  │  16×16 WS INT8   │  │         │    │   16ch Parallel     │     │
│  │  (支持INT16模式)   │  │         │    │                     │     │
│  └────────┬─────────┘  └─────────┘    └──────────┬──────────┘     │
│           │                                       │                │
│           └──────────────┬────────────────────────┘                │
│                          ▼                                         │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │              Post-Processing Unit (PPU) ×16                  │   │
│  │  [+Bias] → [×M] → [>>S] → [+ZP] → [Clamp] → [ReLU]       │   │
│  │                                                              │   │
│  │  (~8800 LUT, 详见§8 PPU微架构)                               │   │
│  └──────────────────────────┬──────────────────────────────────┘   │
│                             │                                      │
│                             ▼ 结果写回Activation Buffer            │
└────────────────────────────────────────────────────────────────────┘
```

---

## 3. 模块清单与职责

| 模块 | 功能 | 主要接口 | 估计面积 |
|------|------|----------|----------|
| **CSR Register File** | 接收CPU层参数配置、启动/状态信号 | Wishbone Slave | ~500 LUT |
| **Control Unit** | 层调度状态机，tiling控制，驱动计算流程 | 读CSR → 控制各模块 | ~1000-2000 LUT |
| **DMA Engine** | 外部存储↔Scratchpad数据搬运，ping-pong调度 | Wishbone B4 Master + Scratchpad端口 | ~1500-3000 LUT |
| **Weight Scratchpad** | 存储当前tile权重数据 | DMA写入 → Systolic Array读出 | 32-64 KB SRAM |
| **Activation Scratchpad** | ping-pong双缓冲，存输入/输出激活 | DMA读写 + 计算模块读写 | 32-64 KB SRAM |
| **DW Buffer** | DW Conv模块专用小缓冲 | DMA写入 → DW模块读出 | 2-4 KB SRAM |
| **Systolic Array 16×16** | Weight Stationary脉动阵列，主力矩阵乘法 | 读Weight+Activation → 输出PSUMs | ~512 DSP / ~16K LUT |
| **DW Conv Module** | 16通道并行Depthwise卷积 | 读DW Buffer → 输出结果 | ~500-1000 LUT |
| **Post-Processing Unit ×16** | Requantize: bias/mul/shift/zp/clamp/relu，支持Conv与Add两种模式 | 接计算输出 → 写回Activation Buffer | ~8800 LUT (16组并行) |

---

## 4. 数据流（单层推理）

```
1. CPU通过Wishbone写CSR：配置当前层参数
   - 输入尺寸 (H, W, C_in)
   - 输出尺寸 (H_out, W_out, C_out)
   - 卷积核参数 (K_h, K_w, stride, padding)
   - 计算模式 (Conv2D / DW Conv / FC / Pooling)
   - 后处理配置 (shift值, scale, LUT选择)
   - 数据地址 (权重地址, 输入地址, 输出地址)

2. CPU写start寄存器，NPU开始执行

3. Control Unit调度（tiling循环）：
   a. DMA加载第一个tile的权重 → Weight Scratchpad
   b. DMA加载Tile[0]输入 → Activation Scratchpad Bank[0]（ping）
   c. 计算Tile[0]的同时：
      - DMA预取Tile[1]输入 → Activation Scratchpad Bank[1]（pong）
   d. Systolic Array / DW Module计算
   e. 结果经Post-Processing Pipeline处理
   f. 处理后的结果写回Activation Scratchpad（或直接DMA搬出）
   g. DMA将完成的tile结果搬回外部SRAM
   h. ping/pong切换，重复c-g直到所有tile完成

4. 所有tile完成 → NPU置完成中断，通知CPU

5. CPU配置下一层参数，重复1-4
```

---

## 5. Ping-Pong双缓冲设计

### 5.1 核心思想

计算和搬运并行进行，互不等待：

```
时间 →  ┃ T0        ┃ T1        ┃ T2        ┃ T3        ┃
━━━━━━━━╋━━━━━━━━━━━╋━━━━━━━━━━━╋━━━━━━━━━━━╋━━━━━━━━━━━╋
DMA     ┃ 加载Tile0  ┃ 加载Tile1  ┃ 加载Tile2  ┃ 加载Tile3  ┃
        ┃ →Bank[0]  ┃ →Bank[1]  ┃ →Bank[0]  ┃ →Bank[1]  ┃
━━━━━━━━╋━━━━━━━━━━━╋━━━━━━━━━━━╋━━━━━━━━━━━╋━━━━━━━━━━━╋
计算    ┃ (idle)    ┃ 算Tile0    ┃ 算Tile1    ┃ 算Tile2    ┃
        ┃           ┃ ←Bank[0]  ┃ ←Bank[1]  ┃ ←Bank[0]  ┃
━━━━━━━━╋━━━━━━━━━━━╋━━━━━━━━━━━╋━━━━━━━━━━━╋━━━━━━━━━━━╋
写回    ┃           ┃           ┃ 回写Tile0  ┃ 回写Tile1  ┃
```

只有第一个tile有启动延迟（cold start），之后DMA、计算、写回三级流水完全重叠。

### 5.2 硬件结构

```
              ┌─────────────────────────────────┐
              │     Activation Scratchpad        │
              │                                  │
              │  ┌───────────┐  ┌───────────┐   │
              │  │  Bank[0]  │  │  Bank[1]  │   │
              │  │ 16-32 KB  │  │ 16-32 KB  │   │
              │  └─────┬─────┘  └─────┬─────┘   │
              │        │              │          │
              └────────┼──────────────┼──────────┘
                       │              │
                 ┌─────┴──────────────┴─────┐
                 │         2:1 MUX           │
                 │   sel = ping_pong_flag    │
                 └────┬─────────────┬───────┘
                      │             │
              ┌───────┴───┐  ┌─────┴──────┐
              │ To Compute│  │  To DMA    │
              │ (读输入)   │  │ (写入/读出) │
              └───────────┘  └────────────┘
```

### 5.3 控制逻辑

```verilog
// Ping-pong控制
reg ping_pong_flag;  // 0=Bank[0]给计算, Bank[1]给DMA
                     // 1=Bank[1]给计算, Bank[0]给DMA

// 每个tile完成后翻转
always @(posedge clk) begin
    if (tile_compute_done && tile_dma_done)
        ping_pong_flag <= ~ping_pong_flag;
end

// 端口分配（物理隔离，无冲突）
assign compute_read_port = ping_pong_flag ? bank1_rdata : bank0_rdata;
assign dma_write_port    = ping_pong_flag ? bank0_wdata : bank1_wdata;
```

### 5.4 时序分析

以MobileNet-v2某层为例验证搬运可被完全隐藏：

```
一个tile的计算时间：
  tile大小: 8×8×64 = 4096 Bytes输入
  计算量: 8×8×3×3×64×128 = 37,748,736 MACs
  @ 256 MACs/cycle (16×16阵列): 147,456 cycles
  @ 200MHz: ~0.74 ms

一个tile的DMA搬运时间：
  搬运量: 4096 Bytes输入
  @ Wishbone 32-bit pipeline: 4096/4 = 1024 cycles
  @ 200MHz: ~0.005 ms

搬运/计算比: 0.005/0.74 = 0.7%  ✓ 搬运完全被隐藏
```

### 5.5 设计参数

| 设计项 | 选择 |
|--------|------|
| Buffer数量 | 2（ping/pong） |
| 每个Buffer大小 | 16-32 KB |
| 切换粒度 | 每个tile完成后切换 |
| 切换条件 | compute_done AND dma_done 同时满足 |
| 端口隔离 | Bank[0]和Bank[1]物理独立SRAM，无端口冲突 |
| DMA方向 | 加载输入 + 回写输出（可在同一DMA时间片串行完成） |

### 5.6 失效场景与应对

| 情况 | 原因 | 解决方案 |
|------|------|----------|
| Tile过小 | 计算太快，DMA来不及预取 | 编译器增大tile尺寸 |
| 外部总线慢 | DMA带宽不足 | 权重预加载 / 总线加宽 |
| FC层 | 权重量大但计算相对少 | Weight Buffer也做ping-pong分半 |
| 层融合 | DW→PW结果不写回外部 | 输出直接留在本地作为下一层输入，跳过回写 |

---

## 6. 可配置参数化设计

NPU IP核采用参数化设计，综合时通过配置生成不同规格的硬件，适配不同面积/精度需求。

### 6.1 配置参数表

```verilog
// Open-NPU 顶层可配置参数
parameter DATA_WIDTH    = 16;       // 激活/权重位宽: 8 or 16
parameter ACC_WIDTH     = 40;       // 累加器位宽: DATA_WIDTH=8时32即可;
                                    //            DATA_WIDTH=16时可选32~48
parameter ARRAY_M       = 16;       // Systolic Array 行数: 8, 16
parameter ARRAY_N       = 16;       // Systolic Array 列数: 8, 16
parameter ACT_BUF_KB    = 32;       // Activation Buffer大小(KB): 16, 32, 64
parameter WGT_BUF_KB    = 32;       // Weight Buffer大小(KB): 16, 32, 64
parameter SUPPORT_INT16 = 1;        // 是否支持INT16模式: 0=纯INT8, 1=INT8+INT16
parameter SUPPORT_DW    = 1;        // 是否包含DW Conv模块: 0, 1
parameter DW_CHANNELS   = 16;       // DW并行通道数: 8, 16
parameter POST_LUT_EN   = 1;        // 是否包含LUT激活: 0, 1
parameter POST_LUT_DEPTH= 256;      // LUT深度: 64, 256
```

**ACC_WIDTH 取值说明**：ACC_WIDTH 是独立可调参数，不绑定DATA_WIDTH。用户可根据目标模型的最大通道数自由选择32~48范围内的任意值。

### 6.2 典型配置方案

| 配置 | DATA_WIDTH | ACC_WIDTH | ARRAY | SRAM | 面积估算 | 适用场景 |
|------|-----------|-----------|-------|------|---------|---------|
| **Tiny** | 8 | 32 | 8×8 | 16+16KB | ~0.08mm² | 极低面积，简单分类 |
| **Base** | 8 | 32 | 16×16 | 32+32KB | ~0.20mm² | INT8主力，面积敏感 |
| **Pro** | 16 | 40 | 16×16 | 32+32KB | ~0.23mm² | INT16，1×1 Conv≤512ch |
| **Pro+** | 16 | 44 | 16×16 | 32+32KB | ~0.24mm² | INT16，1×1 Conv≤8192ch |
| **Max** | 16 | 48 | 16×16 | 64+64KB | ~0.40mm² | INT16，通道数无限制 |

### 6.3 参数联动约束

```verilog
// 自动推导的约束关系
generate
    if (SUPPORT_INT16 == 0) begin : gen_int8_only
        // 纯INT8模式：乘法器8×8，可拆分为2路并行INT8 MAC（吞吐翻倍）
        localparam MULT_WIDTH = 8;
        // ACC_WIDTH仍可由用户自由配置(32~48)，32已足够
    end else begin : gen_int16
        // INT16模式：乘法器16×16
        localparam MULT_WIDTH = 16;
        // ACC_WIDTH由用户根据目标模型通道数自由配置(32~48)
        // 推荐：40(C_in≤512) 或 44(C_in≤8192)
    end
endgenerate

// 编译时校验：ACC_WIDTH必须≥2*DATA_WIDTH（容纳单次乘法结果）
initial begin
    if (ACC_WIDTH < 2 * DATA_WIDTH)
        $error("ACC_WIDTH(%0d) must be >= 2*DATA_WIDTH(%0d)", ACC_WIDTH, 2*DATA_WIDTH);
end

// 1×1 Conv最大C_in由ACC_WIDTH自动推导（供工具链/驱动使用）
localparam MAX_CIN_1x1 = (2**(ACC_WIDTH-1) - 1) / ((2**(DATA_WIDTH-1)-1) ** 2);
```

### 6.4 各配置最大通道支持

INT16×INT16模式下，ACC_WIDTH决定1×1 Conv的最大C_in：

| ACC_WIDTH | 1×1 Conv max C_in | 3×3 Conv max C_in | 面积增量(vs 32-bit) |
|-----------|-------------------|-------------------|-------------------|
| 32-bit | 2 | 不可用 | baseline |
| 34-bit | 8 | 不可用 | +512 LUT/FF |
| 36-bit | 32 | 3 | +1,024 |
| **40-bit** | **512** | **56** | **+2,048** |
| **44-bit** | **8,192** | **910** | **+3,072** |
| 48-bit | 131,072 | 14,563 | +4,096 |

INT8×INT8模式下，32-bit累加器即可支持C_in > 100K，无需加宽。

---

## 7. 量化精度与累加器设计

### 7.1 量化方案选择

| 方案 | Cosine vs FP32 | L2误差 | 32-bit溢出 | 评估结论 |
|------|---------------|--------|-----------|---------|
| INT8act × INT8wgt | 0.882 | 47.5% | 无 | 精度不足 |
| INT16act × INT8wgt | 0.948 | 32.2% | 无 | 权重精度成瓶颈 |
| INT16act × INT16wgt | 0.999999 | 0.15% | 5层 | **选定方案** |
| FP32 (参考) | 1.0 | 0 | — | — |

**决策：采用 INT16×INT16 作为主力数据类型。**

理由：
1. 目标模型（掌纹识别等）训练数据量小，权重/激活分布不均匀，INT8量化损失大
2. 512维embedding对每一维精度敏感，INT16几乎无损（cosine=0.999999）
3. MCU场景无法做量化感知训练（QAT），PTQ下INT16容错空间大
4. 硬件面积代价可控（累加器加宽8-bit，整芯片面积增加<4%）

### 7.2 累加器规格

| 参数 | 规格 |
|------|------|
| 累加器位宽 | **40-bit signed** |
| 单次乘法结果 | 16×16 → 32-bit (signed) |
| 最大安全累加次数 | 2^39 / (32767×32767) = **512次** |
| Partial sum存储 | 40-bit (tiling时不截断) |

### 7.3 各卷积类型最大通道支持

累加次数 = K_h × K_w × C_in / group，40-bit累加器安全上限512次：

| 卷积类型 | 累加次数公式 | 最大安全C_in |
|---------|------------|-------------|
| 1×1 Conv (Pointwise) | C_in | **512** |
| 3×3 Conv | 9 × C_in | **56** |
| 3×3 DWConv (group=C_in) | 9 | **永远安全** |
| 5×5 Conv | 25 × C_in | **20** |
| 7×7 Conv | 49 × C_in | **10** |

注：当前目标模型最大层为 1×1 Conv C_in=512，刚好在安全边界内。

### 7.4 INT8兼容模式

Systolic Array同时支持INT8模式（高吞吐低精度场景）：
- INT8×INT8 → 16-bit乘积，32-bit累加器子集即可
- 同一个40-bit累加器，INT8模式下可安全累加 2^39 / (127×127) = **34,078,720次**
- 每个16-bit乘法器可拆分为2个并行INT8乘法（吞吐翻倍）

### 7.4.1 量化策略选择：INT16作为精度兜底

**设计原则：INT16是兜底量化推理策略，用来保证模型精度的正确性。**

| 场景 | 推荐策略 | 理由 |
|------|---------|------|
| 训练数据充足、权重分布均匀 | INT8 | 吞吐翻倍，精度损失可接受 |
| 训练数据不足、权重/激活分布离散 | **INT16** | 避免量化截断导致精度崩塌 |
| 对输出精度极度敏感（embedding等） | **INT16** | 保证cosine≥0.999 |
| 无法做QAT、只能PTQ部署 | **INT16** | PTQ下INT8误差累积快 |

典型案例：
- 掌纹/人脸识别模型，训练集<10K张，权重方差大 → **必须INT16**
- ImageNet预训练的分类模型，权重分布均匀 → INT8即可（cosine≥0.998）
- 检测模型多分支融合（Resize+Concat），requantize累积误差 → 关键层用INT16

**验证要求：每次E2E回归测试必须同时覆盖INT8和INT16两种模式**，确认：
1. INT16 cosine vs FP32 >= 0.999（near-lossless基线）
2. INT8 cosine vs FP32 >= 0.95（可部署下限）
3. INT16/INT8模型大小比约2x（权重加倍，描述符不变）

### 7.5 面积代价估算

| 组件 | 32-bit方案 | 40-bit方案 | 增量 |
|------|-----------|-----------|------|
| 累加器加法器 (×256 PE) | 8,192 LUT | 10,240 LUT | +2,048 |
| 累加器寄存器 (×256 PE) | 8,192 FF | 10,240 FF | +2,048 |
| PE间互连 (systolic) | 32-bit线宽 | 40-bit线宽 | +25% 布线 |
| Partial sum SRAM | 32-bit/word | 40-bit/word | +25% 容量 |
| 整体PE阵列面积增加 | — | — | **~12%** |
| 整芯片面积增加 | — | — | **~3.6%** |

---

## 8. 脉动阵列 (Systolic Array) 微架构

16×16 Weight Stationary 脉动阵列是 NPU 的主力计算单元，负责 Conv2D / FC 的矩阵乘法。

### 8.1 整体结构

```
              Weight Load (from Weight Scratchpad)
              ↓ ↓ ↓ ↓ ↓ ↓ ↓ ↓ ↓ ↓ ↓ ↓ ↓ ↓ ↓ ↓   (16 columns)
          ┌───┬───┬───┬───┬───┬───────────────┬───┐
          │W  │W  │W  │W  │W  │     ...       │W  │  ← weight preload
          │L  │L  │L  │L  │L  │               │L  │     (per-column)
          ├───┼───┼───┼───┼───┼───────────────┼───┤
 act[0]→  │PE │PE │PE │PE │PE │     ...       │PE │  → drain[0]
          │0,0│0,1│0,2│0,3│0,4│               │0,F│
          ├───┼───┼───┼───┼───┼───────────────┼───┤
 act[1]→  │PE │PE │PE │PE │PE │     ...       │PE │  → drain[1]
          │1,0│1,1│1,2│1,3│1,4│               │1,F│
          ├───┼───┼───┼───┼───┼───────────────┼───┤
 act[2]→  │PE │PE │PE │PE │PE │     ...       │PE │  → drain[2]
          │2,0│2,1│2,2│2,3│2,4│               │2,F│
          ├───┼───┼───┼───┼───┼───────────────┼───┤
          │   │   │   │   │   │               │   │
          │        ... (rows 3~14)             │   │
          │   │   │   │   │   │               │   │
          ├───┼───┼───┼───┼───┼───────────────┼───┤
 act[F]→  │PE │PE │PE │PE │PE │     ...       │PE │  → drain[F]
          │F,0│F,1│F,2│F,3│F,4│               │F,F│
          └───┴───┴───┴───┴───┴───────────────┴───┘
                                                │
                                    drain (psum out)
                                    → 16 个 PPU 并行处理
```

- **行 (Row)**：对应不同的输出像素（或 batch 内不同位置）
- **列 (Column)**：对应不同的输出通道 (C_out)
- **Activation** 从左侧逐行广播，向右逐拍传递
- **Weight** 预先加载到每个 PE 内部寄存器，计算期间不动 (Weight Stationary)
- **Partial Sum** 在每个 PE 内本地累加，计算结束后从右侧 drain 排出

### 8.2 PE (Processing Element) 内部结构

```
┌─────────────────────────────────────────────────────┐
│                    PE[row][col]                       │
│                                                      │
│  ┌─────────────┐                                     │
│  │ weight_reg  │  ← Weight Load 阶段写入              │
│  │ (16-bit)    │     计算期间保持不变 (stationary)     │
│  └──────┬──────┘                                     │
│         │ w                                          │
│         ▼                                            │
│  ┌─────────────────┐     ┌──────────────┐            │
│  │   MULTIPLIER    │◄────│  act_in      │←── 来自左邻 │
│  │  16×16 → 32-bit │     │  (16-bit)    │    PE/输入  │
│  │  (INT8: 2×并行)  │     └──────┬───────┘            │
│  └────────┬────────┘            │                    │
│           │ product (32-bit)    │ act_out            │
│           ▼                     ▼                    │
│  ┌─────────────────┐     ┌──────────────┐            │
│  │   ACCUMULATOR   │     │  act_reg     │──→ 传递给   │
│  │   40-bit signed │     │  (16-bit)    │    右邻PE   │
│  │   acc += product│     │  1-cycle延迟  │            │
│  └────────┬────────┘     └──────────────┘            │
│           │                                          │
│           │ acc (40-bit)                             │
│           ▼                                          │
│  ┌─────────────────┐                                 │
│  │   acc_reg       │  计算完成后 drain → PPU          │
│  │  (40-bit)       │  或继续累加 (partial sum)        │
│  └─────────────────┘                                 │
│                                                      │
└──────────────────────────────────────────────────────┘

PE 接口：
  输入: clk, rst_n, mode(WGT_LOAD/COMPUTE/DRAIN)
        act_in[15:0], wgt_in[15:0]
  输出: act_out[15:0], acc_out[39:0]
```

### 8.3 数据流时序 (Weight Stationary)

以 1×1 Conv 为例：C_in=4, C_out=16, 输出像素=16

```
Phase 1: Weight Load (预加载权重到 PE)
═══════════════════════════════════════
  - Control Unit 从 Weight Scratchpad 逐列加载
  - 每 cycle 加载 1 列 (16 个权重)
  - 16 列共需 16 cycles（可流水与前一层 drain 重叠）

  cycle 0: col0 的 16 个 PE 各加载 w[0][0..15]
  cycle 1: col1 的 16 个 PE 各加载 w[1][0..15]
  ...
  cycle 15: col15 的 16 个 PE 各加载 w[15][0..15]

Phase 2: Compute (激活数据流过阵列)
═══════════════════════════════════════
  - 每 cycle 输入一组 C_in 中的 1 个通道的 16 个像素
  - 激活从左向右传播，每 PE 延迟 1 cycle

  cycle 0: act[:,cin=0] 进入 col0, PE[r,0].acc += act[r,0] × w[0,cin=0]
  cycle 1: act[:,cin=0] 传到 col1, PE[r,1].acc += act[r,0] × w[1,cin=0]
           act[:,cin=1] 进入 col0, PE[r,0].acc += act[r,1] × w[0,cin=1]
  ...
  cycle 3: act[:,cin=0] 传到 col3 (最后有效列对cin=0)
           同时 cin=1,2,3 分别在 col2,1,0

  计算总 cycles = C_in + ARRAY_N - 1 = 4 + 16 - 1 = 19 cycles

Phase 3: Drain (排出累加结果)
═══════════════════════════════════════
  - 所有 PE 完成累加后，每行的 acc 同时向右输出
  - 16 行同时 drain → 16 个 PPU 并行处理
  - drain 耗时 = 1 cycle（所有行并行）
```

### 8.4 通用矩阵映射

Conv2D 通过 im2col 展开映射到矩阵乘法：

```
标准 Conv2D: Output[n, oh, ow, co] = Σ Input[n, oh+kh, ow+kw, ci] × Weight[co, kh, kw, ci]

展开为矩阵乘:
  A[M×K] × B[K×N] = C[M×N]

  M = batch × OH × OW     (输出像素总数)    → 映射到 rows
  K = KH × KW × C_in      (reduction dim)  → 逐 cycle 流入
  N = C_out                (输出通道)       → 映射到 columns

Tiling:
  M_tile = ARRAY_M = 16   (每次处理16个输出像素)
  N_tile = ARRAY_N = 16   (每次处理16个输出通道)
  K 方向逐步流入，无需tiling（除非partial sum场景）
```

### 8.5 INT8 双倍吞吐模式

当 `SUPPORT_INT16=1` 时，每个 PE 的 16-bit 乘法器在 INT8 模式下可拆分：

```
INT16 模式:                    INT8 模式:
┌──────────────────┐           ┌──────────────────┐
│  16×16 → 32-bit │           │ 上半: 8×8 → 16-bit│  ← MAC pair A
│  1 MAC/cycle     │           │ 下半: 8×8 → 16-bit│  ← MAC pair B
└──────────────────┘           │ 2 MACs/cycle      │
                               └──────────────────┘

INT8 吞吐 = 2 × INT16 吞吐
  INT16: 16×16×1 = 256 MACs/cycle
  INT8:  16×16×2 = 512 MACs/cycle
```

### 8.6 Weight Load 与 Compute 重叠

为避免 weight load 的空闲时间，采用**双缓冲权重寄存器**：

```
┌────────────────────────────────────────────────────┐
│  PE 内部双权重寄存器                                 │
│                                                    │
│  ┌────────────┐    ┌────────────┐                   │
│  │ weight_A   │    │ weight_B   │                   │
│  │ (active)   │    │ (loading)  │                   │
│  └─────┬──────┘    └─────┬──────┘                   │
│        │                 │                          │
│        └────┐   ┌───────┘                           │
│             ▼   ▼                                   │
│         ┌──────────┐                                │
│         │ 2:1 MUX  │  sel = buf_sel                 │
│         └────┬─────┘                                │
│              │ → 送入乘法器                           │
│              ▼                                      │
└────────────────────────────────────────────────────┘

时序:
  Layer N 计算中 (用 weight_A)，同时 DMA 加载 Layer N+1 权重到 weight_B
  Layer N 完成后，buf_sel 翻转，Layer N+1 立即开算，零等待
```

### 8.7 Partial Sum 处理

当 K (reduction dimension) 超过累加器安全范围时，需分段累加：

```
C_in = 1024, 40-bit acc 安全上限 = 512 (1×1 Conv)

分段策略:
  Pass 1: 累加 cin[0:511]   → psum_1 (40-bit) → 写回 SRAM
  Pass 2: 累加 cin[512:1023] → psum_2 (40-bit)
  Final:  psum_1 + psum_2 → 完整 acc → PPU requantize

Partial sum 存储:
  - 存回 Activation Scratchpad (40-bit/word)
  - 每个输出元素需 5 bytes (40-bit 对齐到 5B)
  - 16×16 tile: 16×16×5 = 1280 bytes
```

### 8.8 控制状态机

```
                    ┌──────────┐
         reset ───→│   IDLE   │
                    └────┬─────┘
                         │ start
                         ▼
                    ┌──────────┐    weight load done
              ┌───→│ WGT_LOAD │────────────────┐
              │    └──────────┘                │
              │                                ▼
              │    ┌──────────┐           ┌──────────┐
              │    │  DRAIN   │◄──────────│ COMPUTE  │
              │    └────┬─────┘ K exhausted└──────────┘
              │         │                       │
              │         │ drain done            │ K > max_safe
              │         ▼                       ▼
              │    ┌──────────┐           ┌──────────┐
              │    │   DONE   │           │PSUM_STORE│
              │    └────┬─────┘           └────┬─────┘
              │         │ more tiles           │ store done
              │         ▼                      │
              │    (next tile)                  │
              └────────────────────────────────┘
                         │ all tiles done
                         ▼
                    IRQ → CPU
```

### 8.9 阵列接口信号

```verilog
module systolic_array #(
    parameter ARRAY_M    = 16,    // rows
    parameter ARRAY_N    = 16,    // columns
    parameter DATA_WIDTH = 16,    // 8 or 16
    parameter ACC_WIDTH  = 40     // 32~48
)(
    input  wire                    clk,
    input  wire                    rst_n,

    // Control
    input  wire [1:0]              mode,          // IDLE/WGT_LOAD/COMPUTE/DRAIN
    input  wire                    buf_sel,       // weight double-buffer select

    // Weight Load (from Weight Scratchpad)
    input  wire [ARRAY_M-1:0]     wgt_load_en,   // per-row load enable
    input  wire [DATA_WIDTH-1:0]  wgt_load_data [ARRAY_M-1:0],  // broadcast per column

    // Activation Input (from Activation Scratchpad)
    input  wire [DATA_WIDTH-1:0]  act_in [ARRAY_M-1:0],  // 16 rows parallel

    // Accumulator Output (to PPU)
    output wire [ACC_WIDTH-1:0]   acc_out [ARRAY_M-1:0],  // 16 rows parallel (drain)
    output wire                   acc_valid,

    // Partial Sum I/O (to/from SRAM)
    input  wire [ACC_WIDTH-1:0]   psum_in [ARRAY_M-1:0],
    input  wire                   psum_load,
    output wire [ACC_WIDTH-1:0]   psum_out [ARRAY_M-1:0],
    output wire                   psum_valid
);
```

### 8.10 资源估算

| 组件 | 每 PE | 16×16 阵列 (256 PE) |
|------|-------|---------------------|
| 16×16 乘法器 | 1 DSP48 或 ~60 LUT | 256 DSP / ~15,360 LUT |
| 40-bit 累加器 | ~60 LUT + 40 FF | ~15,360 LUT + 10,240 FF |
| weight_reg ×2 | 32 FF | 8,192 FF |
| act_reg | 16 FF | 4,096 FF |
| 控制/MUX | ~10 LUT | ~2,560 LUT |
| **PE 小计** | ~130 LUT + 88 FF (+ 1 DSP) | |
| **阵列总计** | | **~33K LUT + 22K FF + 256 DSP** |

加上阵列外围控制逻辑 (~2K LUT)，整个 Systolic Array 模块约 **35K LUT + 256 DSP**。

---

## 9. DMA Engine 微架构

DMA Engine 负责外部存储（主 SRAM）与内部 Scratchpad 之间的数据搬运，支持多维张量地址生成、ping-pong 调度和总线突发传输。

### 9.1 整体结构

```
┌──────────────────────────────────────────────────────────────────────┐
│                         DMA Engine                                    │
│                                                                      │
│  ┌──────────────┐     ┌──────────────────┐     ┌──────────────────┐  │
│  │  Descriptor  │     │  Address         │     │  Wishbone B4     │  │
│  │  Queue       │────→│  Generator       │────→│  Master Port     │  │
│  │  (4-entry)   │     │  (2D/3D stride)  │     │  (pipeline)      │  │
│  └──────┬───────┘     └────────┬─────────┘     └────────┬─────────┘  │
│         │                      │                        │            │
│         │ desc                 │ ext_addr               │ bus req/   │
│         │                      │                        │ resp       │
│  ┌──────┴───────┐     ┌────────┴─────────┐     ┌───────┴─────────┐  │
│  │  DMA         │     │  SRAM            │     │  Data            │  │
│  │  Controller  │     │  Address Gen     │     │  Buffer          │  │
│  │  (FSM)       │     │  (local side)    │     │  (FIFO 8-entry)  │  │
│  └──────┬───────┘     └────────┬─────────┘     └───────┬─────────┘  │
│         │ ctrl                 │ sram_addr              │ data       │
│         │                      │                        │            │
│         ▼                      ▼                        ▼            │
│  ┌───────────────────────────────────────────────────────────────┐   │
│  │                  Scratchpad Write/Read Port                    │   │
│  │  (Weight Buf / Activation Buf / DW Buf / PSum)                │   │
│  └───────────────────────────────────────────────────────────────┘   │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
                    │                              ▲
                    ▼                              │
        ┌─────────────────────┐        ┌─────────────────────┐
        │  Wishbone Bus       │        │  Control Unit        │
        │  (to ext SRAM)      │        │  (descriptor kick)   │
        └─────────────────────┘        └─────────────────────┘
```

### 9.2 Descriptor 描述符格式

每次 DMA 传输由一个描述符驱动，描述符由 Control Unit 填充后 kick 到队列：

```
┌─────────────────────────────────────────────────────────────────┐
│                    DMA Descriptor (128 bits)                      │
├──────────────┬──────────────────────────────────────────────────┤
│ [31:0]       │ ext_base_addr    外部SRAM起始地址                  │
│ [63:32]      │ sram_base_addr   内部Scratchpad起始地址            │
│ [79:64]      │ xfer_len         单行传输长度 (bytes)              │
│ [91:80]      │ row_count        行数 (2D传输)                    │
│ [111:92]     │ ext_row_stride   外部行步长 (bytes)                │
│ [119:112]    │ sram_row_stride  内部行步长 (bytes, 0=连续)        │
│ [121:120]    │ direction        0=EXT→SRAM(load), 1=SRAM→EXT(store)│
│ [123:122]    │ target           00=Act, 01=Wgt, 10=DW, 11=PSum   │
│ [124]        │ irq_en           传输完成后是否触发中断             │
│ [125]        │ chain            是否自动启动队列中下一条描述符      │
│ [127:126]    │ reserved                                          │
└──────────────┴──────────────────────────────────────────────────┘
```

### 9.3 多维地址生成器

支持 2D 张量搬运（带 stride），用于 im2col-free 的输入 tile 加载：

```
                ┌───────────────────────────────────────┐
                │       Address Generator               │
                │                                       │
                │  for (row = 0; row < row_count; row++)│
                │    for (col = 0; col < xfer_len; col += BUS_WIDTH) │
                │      ext_addr  = ext_base  + row × ext_row_stride  + col │
                │      sram_addr = sram_base + row × sram_row_stride + col │
                │                                       │
                └───────────────────────────────────────┘

典型用例:
┌─────────────────────────────────────────────────────────────┐
│ 加载 3×3 Conv 输入 patch (with padding):                      │
│                                                              │
│ 外部 Feature Map (H×W×C layout):                             │
│   ┌───┬───┬───┬───┬───┬───┐                                  │
│   │   │   │   │   │   │   │  row_stride = W × C × sizeof    │
│   ├───┼───┼───┼───┼───┼───┤                                  │
│   │   │ x │ x │ x │   │   │  ← row 0 of patch              │
│   ├───┼───┼───┼───┼───┼───┤                                  │
│   │   │ x │ x │ x │   │   │  ← row 1 of patch              │
│   ├───┼───┼───┼───┼───┼───┤                                  │
│   │   │ x │ x │ x │   │   │  ← row 2 of patch              │
│   ├───┼───┼───┼───┼───┼───┤                                  │
│   │   │   │   │   │   │   │                                  │
│   └───┴───┴───┴───┴───┴───┘                                  │
│                                                              │
│ DMA descriptor:                                              │
│   ext_base = &feature[oh][ow][0]                             │
│   xfer_len = tile_W × C_in × 2 bytes                        │
│   row_count = tile_H (包含kernel扩展)                         │
│   ext_row_stride = W × C_in × 2                             │
│   sram_row_stride = tile_W × C_in × 2 (连续存放)            │
└─────────────────────────────────────────────────────────────┘
```

### 9.4 Data Buffer (FIFO)

解耦总线时序与 Scratchpad 写入时序：

```
         Wishbone              FIFO (8-entry × 32-bit)         Scratchpad
        ─────────            ─────────────────────────         ──────────
                              ┌───┬───┬───┬───┬───┐
  bus_dat_i ──→ push ──→      │ 0 │ 1 │ 2 │ 3 │...│  ──→ pop ──→ sram_wdata
                              └───┴───┴───┴───┴───┘
  bus_ack   ──→ push_en       wr_ptr        rd_ptr     ──→ sram_wen

  FIFO 作用:
  1. 吸收 Wishbone pipeline burst 的突发数据
  2. Scratchpad 端口被占用时（计算读取）暂存数据
  3. Store方向: Scratchpad读出 → FIFO → Wishbone写出
```

### 9.5 Ping-Pong 调度与 Chain 模式

```
Control Unit 发射描述符序列 (chain=1 自动连续执行):

  Desc[0]: LOAD activation tile[0] → Act_Buf Bank[0]     chain=1
  Desc[1]: LOAD weight[0]          → Wgt_Buf Bank[0]     chain=0 (等计算)
           ── 计算启动 tile[0] ──
  Desc[2]: LOAD activation tile[1] → Act_Buf Bank[1]     chain=1 (与计算并行)
  Desc[3]: LOAD weight[1]          → Wgt_Buf Bank[1]     chain=0
           ── ping-pong 切换 ──
  Desc[4]: STORE result tile[0]    → 外部SRAM             chain=1
  Desc[5]: LOAD activation tile[2] → Act_Buf Bank[0]     chain=0
           ...

描述符队列深度=4，流水线式填充，Control Unit 提前2个tile准备描述符。
```

### 9.6 Wishbone B4 Pipeline 接口

```
Burst 传输时序 (pipeline mode):
                ___     ___     ___     ___     ___     ___
  clk       ___|   |___|   |___|   |___|   |___|   |___|   |___
             ___ ___ ___ ___ ___
  cyc       |   |   |   |   |   |
  stb       |___|___|___|___|___|_______
  addr       A0   A1   A2   A3   A4
                 ___ ___ ___ ___ ___
  ack           |   |   |   |   |   |
  dat_i          D0   D1   D2   D3   D4

  - Pipeline mode: 不等 ack 即可发下一个 stb
  - 连续地址自增, 最大 burst = xfer_len / 4
  - 支持 incrementing burst (wrap burst 不需要)
  - 最大突发长度: 受限于 FIFO 深度 (8 words = 32 bytes/burst)

带宽:
  @ 200MHz, 32-bit bus, pipeline mode:
  峰值: 200M × 4B = 800 MB/s
  实际 (考虑turnaround): ~600 MB/s
```

### 9.7 DMA Controller 状态机

```
                    ┌──────────┐
         reset ───→│   IDLE   │
                    └────┬─────┘
                         │ desc_valid (队列非空)
                         ▼
                    ┌──────────┐
                    │  SETUP   │  解析描述符, 配置地址生成器
                    └────┬─────┘
                         │
              ┌──────────┴──────────┐
              │ direction?          │
              ▼                     ▼
        ┌──────────┐          ┌──────────┐
        │  LOAD    │          │  STORE   │
        │ ext→sram │          │ sram→ext │
        └────┬─────┘          └────┬─────┘
             │                     │
             │ row loop            │ row loop
             │ xfer_len done       │ xfer_len done
             │ row_count done      │ row_count done
             ▼                     ▼
        ┌──────────────────────────────┐
        │           DONE               │
        └────┬──────────────────┬──────┘
             │ chain=1           │ chain=0
             │ 且队列非空         │ 或队列空
             ▼                   ▼
        (取下一条desc)       ┌──────────┐
        → 回到 SETUP         │  IDLE    │
                             └──────────┘
                              (if irq_en → 触发中断)
```

### 9.8 Zero Padding 支持

Conv 的 padding 在 DMA 加载时处理，避免计算单元特殊逻辑：

```
┌─────────────────────────────────────────────────────────┐
│  SRAM Activation Buffer (tile with padding):             │
│                                                          │
│  ┌───┬───────────────────┬───┐                           │
│  │ 0 │        0          │ 0 │  ← top padding (DMA填0)  │
│  ├───┼───────────────────┼───┤                           │
│  │ 0 │  real data (DMA)  │ 0 │  ← left/right pad = 0   │
│  │ 0 │                   │ 0 │                           │
│  │ 0 │                   │ 0 │                           │
│  ├───┼───────────────────┼───┤                           │
│  │ 0 │        0          │ 0 │  ← bottom padding        │
│  └───┴───────────────────┴───┘                           │
│                                                          │
│  实现方式:                                                │
│  1. DMA 先将 tile 区域清零 (一次burst写0)                 │
│  2. 再用带stride的2D搬运写入真实数据到中间区域             │
│  或:                                                     │
│  1. Address Gen 对 pad 区域跳过总线请求, 直接写0到SRAM    │
│     (节省总线带宽, 需硬件判断 row/col 是否在 pad 区域)    │
└─────────────────────────────────────────────────────────┘
```

### 9.9 接口信号

```verilog
module dma_engine #(
    parameter FIFO_DEPTH   = 8,     // Data FIFO entries
    parameter DESC_DEPTH   = 4,     // Descriptor queue depth
    parameter BUS_WIDTH    = 32,    // Wishbone data width (bits)
    parameter ADDR_WIDTH   = 32,    // Address width
    parameter SRAM_ADDR_W  = 17     // Internal SRAM address (128KB)
)(
    input  wire                     clk,
    input  wire                     rst_n,

    // Wishbone B4 Master Port
    output wire                     wb_cyc_o,
    output wire                     wb_stb_o,
    output wire                     wb_we_o,
    output wire [ADDR_WIDTH-1:0]    wb_adr_o,
    output wire [BUS_WIDTH-1:0]     wb_dat_o,
    output wire [BUS_WIDTH/8-1:0]   wb_sel_o,
    input  wire                     wb_ack_i,
    input  wire [BUS_WIDTH-1:0]     wb_dat_i,
    input  wire                     wb_stall_i,   // pipeline flow control

    // Scratchpad Ports
    output wire                     sram_wen,
    output wire [SRAM_ADDR_W-1:0]   sram_addr,
    output wire [BUS_WIDTH-1:0]     sram_wdata,
    input  wire [BUS_WIDTH-1:0]     sram_rdata,
    output wire [1:0]               sram_target,  // 00=Act,01=Wgt,10=DW,11=PSum

    // Control Interface (from Control Unit)
    input  wire                     desc_push,
    input  wire [127:0]             desc_data,
    output wire                     desc_full,

    // Status
    output wire                     busy,
    output wire                     irq,           // transfer complete interrupt
    output wire [15:0]              bytes_done     // progress counter
);
```

### 9.10 资源估算

| 组件 | 规格 | 面积 |
|------|------|------|
| Descriptor Queue | 4×128-bit register file | ~80 LUT + 512 FF |
| Address Generator (ext) | 32-bit adder + stride mul | ~100 LUT |
| Address Generator (sram) | 17-bit adder + stride | ~50 LUT |
| DMA Controller FSM | 状态机 + 行/列计数器 | ~150 LUT |
| Data FIFO | 8×32-bit | ~60 LUT + 256 FF |
| Wishbone Master | pipeline burst 逻辑 | ~120 LUT |
| Zero-pad 判断 | row/col 边界比较器 | ~40 LUT |
| **DMA Engine 总计** | | **~600 LUT + ~800 FF** |

DMA Engine 面积很小（~600 LUT），但对整体性能影响关键——搬运能否被计算隐藏决定了有效利用率。

### 9.11 带宽需求分析

```
以模型最大层为例 (1×1 Conv, C_in=512, C_out=512, 14×14 output):

输入  tile: 16 pixels × 512 ch × 2B = 16,384 bytes
权重  tile: 16 C_out × 512 C_in × 2B = 16,384 bytes
输出  tile: 16 pixels × 16 ch × 2B   = 512 bytes (PPU产出后写回)

计算时间: 16 × 512 × 16 / 256 MACs_per_cycle = 512 cycles

需搬运: 16,384 + 16,384 = 32,768 bytes (input + weight of next tile)
搬运时间: 32,768 / 4 = 8,192 cycles (32-bit bus, 1 word/cycle)

搬运/计算比: 8192 / 512 = 16 ← 搬运来不及!

解决方案:
  1. 权重复用: 同一权重 tile 用于多个输入 tile, 不需每次重加载
     → 权重加载摊薄到 tile_count 次
  2. 实际: 14×14 = 196 pixels / 16 = ~12 个输入tile共用同一权重
     → 权重搬运摊到 16384/12 ≈ 1365 bytes/tile
     → 总搬运 = 16384 + 1365 = 17,749 bytes = 4,437 cycles
     → 搬运/计算 = 4437/512 = 8.7 ← 仍需优化

  3. 总线加宽到 64-bit: 搬运时间减半 → 4437/2 = 2,218 cycles → 比值=4.3
  4. 或将 bus clock 与 compute clock 解耦 (bus 2:1 faster)

  V1.0 选择: 32-bit bus, 接受该层利用率 ~12%
  权重已驻留 Scratchpad 时 (小模型): 利用率可达 ~80%+
```

---

## 10. Control Unit 微架构

Control Unit 是 NPU 的"大脑"，负责解析 CSR 层参数、生成 tiling 循环、协调 DMA / Systolic Array / PPU 的执行顺序。

### 10.1 整体结构

```
┌──────────────────────────────────────────────────────────────────────┐
│                         Control Unit                                  │
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐   │
│  │                    CSR Register Shadow                         │   │
│  │  (从CSR slave复制的层参数: 尺寸/stride/pad/mode/地址等)        │   │
│  └──────────────────────────────┬────────────────────────────────┘   │
│                                 │ layer params                       │
│                                 ▼                                    │
│  ┌───────────────────────────────────────────────────────────────┐   │
│  │                  Tiling Calculator                             │   │
│  │                                                               │   │
│  │  输入: OH, OW, C_in, C_out, K, ARRAY_M, ARRAY_N, BUF_SIZE    │   │
│  │  输出: tile_H, tile_W, n_tiles_h, n_tiles_w,                 │   │
│  │        n_tiles_cout, n_passes_cin                             │   │
│  └──────────────────────────────┬────────────────────────────────┘   │
│                                 │ tiling params                      │
│                                 ▼                                    │
│  ┌───────────────────────────────────────────────────────────────┐   │
│  │                  Main Sequencer (FSM)                          │   │
│  │                                                               │   │
│  │  三层嵌套循环:                                                  │   │
│  │    for tile_cout  (输出通道 tiling)                             │   │
│  │      for tile_row (空间 H tiling)                              │   │
│  │        for tile_col (空间 W tiling)                            │   │
│  │          for cin_pass (C_in 分段, partial sum)                 │   │
│  │            → 调度 DMA + Compute + Drain                       │   │
│  └───────────┬────────────────┬────────────────┬─────────────────┘   │
│              │                │                │                     │
│              ▼                ▼                ▼                     │
│  ┌────────────────┐ ┌──────────────┐ ┌──────────────────┐           │
│  │ DMA Descriptor │ │  SA Control  │ │  PPU Control     │           │
│  │ Generator      │ │  Signals     │ │  Signals         │           │
│  │                │ │              │ │                  │           │
│  │ 填充desc并push │ │ mode/buf_sel │ │ ch_idx/mode     │           │
│  │ 到DMA Engine   │ │ 到Systolic   │ │ 到PPU           │           │
│  └────────┬───────┘ └──────┬───────┘ └────────┬─────────┘           │
│           │                │                   │                     │
│           ▼                ▼                   ▼                     │
│  ┌───────────────────────────────────────────────────────────────┐   │
│  │              Handshake / Sync Logic                            │   │
│  │                                                               │   │
│  │  - dma_done[load_act] / dma_done[load_wgt] / dma_done[store] │   │
│  │  - sa_compute_done                                            │   │
│  │  - ppu_drain_done                                             │   │
│  │  - ping_pong_flag                                             │   │
│  └───────────────────────────────────────────────────────────────┘   │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
        │              │              │               │
        ▼              ▼              ▼               ▼
   DMA Engine    Systolic Array     PPU ×16        CSR (status/IRQ)
```

### 10.2 主状态机 (Main Sequencer)

```
                        ┌──────────┐
             reset ───→ │   IDLE   │ ← 等待CPU写start
                        └────┬─────┘
                             │ csr.start = 1
                             ▼
                        ┌──────────┐
                        │  PARSE   │  读取CSR shadow, 计算tiling参数
                        └────┬─────┘
                             │ tiling ready
                             ▼
                        ┌──────────┐
              ┌────────→│ TILE_INIT│  设置当前tile的地址偏移
              │         └────┬─────┘
              │              │
              │              ▼
              │    ┌─────────────────┐
              │    │   LOAD_WEIGHT   │  push weight DMA desc
              │    └────────┬────────┘
              │             │ (若权重已驻留则跳过)
              │             ▼
              │    ┌─────────────────┐
              │    │   LOAD_ACT      │  push activation DMA desc
              │    └────────┬────────┘
              │             │ dma_done[load]
              │             ▼
              │    ┌─────────────────┐
              │    │   COMPUTE       │  SA mode=COMPUTE, 同时prefetch下一tile
              │    └────────┬────────┘
              │             │ sa_compute_done
              │             ▼
              │    ┌─────────────────┐
              │    │   DRAIN_PPU     │  SA mode=DRAIN → PPU处理 → 写回SRAM
              │    └────────┬────────┘
              │             │ ppu_drain_done
              │             ▼
              │    ┌─────────────────┐       ┌─────────────────┐
              │    │  PSUM_STORE?    │──yes──→│  STORE_PSUM     │
              │    │ (cin_pass<last) │       └────────┬────────┘
              │    └────────┬────────┘                │ done
              │             │ no (last cin_pass)      │
              │             ▼                         │
              │    ┌─────────────────┐               │
              │    │  STORE_OUTPUT   │  push store DMA desc
              │    └────────┬────────┘               │
              │             │ dma_done[store]         │
              │             ▼                         │
              │    ┌─────────────────┐               │
              │    │  NEXT_TILE      │◄──────────────┘
              │    └────────┬────────┘
              │             │
              │    ┌────────┴────────────────┐
              │    │ more tiles?             │
              │    ├─── yes ────→ flip p/p ──┘
              │    │
              │    └─── no ─────┐
              │                 ▼
              │         ┌──────────┐
              │         │   DONE   │ → set csr.status=complete, IRQ
              │         └──────────┘
              │                 │ CPU写start (下一层)
              └─────────────────┘
```

### 10.3 Tiling Calculator

> **V1.0 实现决策**：Tiling 由离线编译器 (`tools/tiling.py`) 预先计算并写入 layer descriptor。
> RTL Control Unit 不自行推导 tile 尺寸，仅按 descriptor 指定的 tile_h/tile_w/tile_num_h/tile_num_w 执行循环。
> 以下描述的是概念模型（计算逻辑），实际由编译器离线完成。

根据层参数和硬件资源自动计算 tile 尺寸：

```
输入:
  OH, OW          输出空间尺寸
  C_in, C_out     输入/输出通道数
  KH, KW          卷积核大小
  stride, pad     步长与padding
  ARRAY_M, ARRAY_N   阵列行/列数
  ACT_BUF_KB      Activation Buffer 可用大小 (单bank)
  WGT_BUF_KB      Weight Buffer 可用大小

计算过程 (硬件实现为组合逻辑+少量除法器):

  // 输出通道 tiling
  tile_cout = ARRAY_N                       // 16 (一次处理16个输出通道)
  n_tiles_cout = ceil(C_out / tile_cout)

  // 空间 tiling (受限于 Activation Buffer)
  // 输入tile所需 = (tile_OH × stride + KH - 1) × (tile_OW × stride + KW - 1) × C_in × 2B
  // 目标: 不超过 ACT_BUF_KB × 1024
  tile_OH = min(OH, max_rows_fitting_buffer)
  tile_OW = min(OW, max_cols_fitting_buffer)
  n_tiles_h = ceil(OH / tile_OH)
  n_tiles_w = ceil(OW / tile_OW)

  // C_in 分段 (partial sum, 受累加器安全限制)
  max_cin_per_pass = MAX_CIN_1x1 / (KH × KW)  // e.g. 512/9=56 for 3×3
  n_passes_cin = ceil(C_in / max_cin_per_pass)

  // 权重 tiling (受限于 Weight Buffer)
  // 一个tile权重 = tile_cout × KH × KW × cin_per_pass × 2B
  // 若超 WGT_BUF_KB → 进一步切分 tile_cout 或 cin_per_pass
```

### 10.4 DMA Descriptor 生成逻辑

Control Unit 根据当前 tile 坐标计算外部地址并组装描述符：

```
当前 tile 坐标: (th, tw, tc, cp) = (tile_row, tile_col, tile_cout, cin_pass)

Activation Load Descriptor:
  ext_base = act_base_addr
           + (th × tile_OH × stride - pad) × W × C_in × 2
           + (tw × tile_OW × stride - pad) × C_in × 2
           + cp × max_cin_per_pass × 2
  sram_base = 0 (当前 bank 起始)
  xfer_len = input_tile_W × cin_per_pass × 2
  row_count = input_tile_H
  ext_row_stride = W × C_in × 2
  direction = LOAD
  target = ACT

Weight Load Descriptor:
  ext_base = wgt_base_addr
           + tc × tile_cout × KH × KW × C_in × 2
           + cp × max_cin_per_pass × KH × KW × 2
  sram_base = 0
  xfer_len = tile_cout × KH × KW × cin_per_pass × 2
  row_count = 1 (权重一维连续)
  direction = LOAD
  target = WGT

Output Store Descriptor:
  ext_base = out_base_addr
           + th × tile_OH × OW × C_out × 2
           + tw × tile_OW × C_out × 2
           + tc × tile_cout × 2
  sram_base = output area offset
  xfer_len = tile_OW × tile_cout × 2
  row_count = tile_OH
  ext_row_stride = OW × C_out × 2
  direction = STORE
  target = ACT
```

### 10.5 Prefetch 流水线控制

为最大化计算利用率，Control Unit 在计算当前 tile 的同时预取下一 tile：

```
时间线:
─────────────────────────────────────────────────────────────────────
  TILE[0]          TILE[1]          TILE[2]          TILE[3]
─────────────────────────────────────────────────────────────────────
  LOAD_W[0]
  LOAD_A[0]→Bank0
  COMPUTE[0]       LOAD_A[1]→Bank1
                   LOAD_W[1](若需)
  DRAIN[0]         
  STORE[0]         COMPUTE[1]       LOAD_A[2]→Bank0
                                    LOAD_W[2](若需)
                   DRAIN[1]
                   STORE[1]         COMPUTE[2]       LOAD_A[3]→Bank1
─────────────────────────────────────────────────────────────────────

Prefetch 控制信号:
  - prefetch_start: COMPUTE阶段开始时同时发射下一tile的LOAD描述符
  - prefetch_bank:  ping_pong_flag 取反 (写入非计算bank)
  - prefetch_wait:  若LOAD未完成且COMPUTE已完成 → 阻塞在NEXT_TILE
```

### 10.6 算子模式派发

```
┌────────────────────────────────────────────────────────────┐
│  mode (from CSR)        Dispatch                           │
├────────────────────────────────────────────────────────────┤
│  CONV2D / FC            → Systolic Array (im2col映射)      │
│  DW_CONV                → DW Conv Module (bypass SA)       │
│  POOLING (avg/max)      → Pooling Logic (复用PPU datapath)  │
│  ADD (element-wise)     → PPU (ADD mode, bypass SA)        │
│  RELU_ONLY              → PPU (RELU_ONLY mode)             │
│  RESIZE (nearest/bilinear) → Resize FSM + PPU + RMW WB    │
│  CONCAT                 → DMA only (重排写地址)             │
└────────────────────────────────────────────────────────────┘

不同算子走不同硬件路径, Control Unit 根据 mode 决定:
  - 哪些模块激活
  - DMA 描述符怎么组装
  - tiling 策略怎么选择
```

### 10.7 错误处理与看门狗

```
┌───────────────────────────────────────────────────┐
│  Watchdog & Error Logic                           │
│                                                   │
│  ┌─────────────────┐                              │
│  │ watchdog_counter│  每次状态转换清零             │
│  │ (24-bit)        │  连续 2^24 cycle无转换 → 超时 │
│  └────────┬────────┘                              │
│           │ timeout                               │
│           ▼                                       │
│  csr.status.error = 1                             │
│  csr.error_code = TIMEOUT / BUS_ERR / OVERFLOW    │
│  → IRQ (error)                                    │
│                                                   │
│  其他错误源:                                       │
│  - DMA bus error (wb_err_i)                       │
│  - Descriptor queue overflow                      │
│  - Invalid layer params (C_out=0, etc.)           │
└───────────────────────────────────────────────────┘
```

### 10.8 接口信号

```verilog
module control_unit #(
    parameter ARRAY_M      = 16,
    parameter ARRAY_N      = 16,
    parameter ACC_WIDTH    = 40,
    parameter DATA_WIDTH   = 16,
    parameter ACT_BUF_KB   = 32,
    parameter WGT_BUF_KB   = 32
)(
    input  wire                clk,
    input  wire                rst_n,

    // CSR Interface (registered layer params)
    input  wire                csr_start,         // CPU kick
    input  wire [3:0]          csr_op_mode,       // CONV2D/DW/FC/POOL/ADD/...
    input  wire [11:0]         csr_ih, csr_iw,    // input H, W
    input  wire [11:0]         csr_oh, csr_ow,    // output H, W
    input  wire [11:0]         csr_cin, csr_cout, // channels
    input  wire [3:0]          csr_kh, csr_kw,    // kernel size
    input  wire [3:0]          csr_stride,
    input  wire [3:0]          csr_pad,
    input  wire [31:0]         csr_act_addr,      // external SRAM addresses
    input  wire [31:0]         csr_wgt_addr,
    input  wire [31:0]         csr_out_addr,
    output reg                 csr_busy,
    output reg                 csr_done,
    output reg  [3:0]          csr_error_code,

    // DMA Engine Interface
    output reg                 dma_desc_push,
    output reg  [127:0]        dma_desc_data,
    input  wire                dma_desc_full,
    input  wire                dma_busy,
    input  wire                dma_irq,           // transfer done

    // Systolic Array Interface
    output reg  [1:0]          sa_mode,           // IDLE/WGT_LOAD/COMPUTE/DRAIN
    output reg                 sa_buf_sel,
    input  wire                sa_compute_done,

    // PPU Interface
    output reg  [1:0]          ppu_mode,          // CONV_REQ/ADD/RELU_ONLY/PASS
    output reg  [11:0]         ppu_ch_idx,        // current output channel
    input  wire                ppu_done,

    // Ping-Pong Control
    output reg                 ping_pong_flag,

    // Interrupt
    output wire                irq
);
```

### 10.9 资源估算

| 组件 | 规格 | 面积 |
|------|------|------|
| CSR Shadow Registers | ~20 regs × 12~32 bit | ~300 FF |
| Tiling Calculator | 除法器(迭代式) + 比较器 | ~400 LUT |
| Main Sequencer FSM | ~12 states + 转换逻辑 | ~250 LUT |
| Tile Loop Counters | 4层嵌套 × 12-bit | ~100 LUT + 48 FF |
| DMA Desc Generator | 地址计算(乘加器) | ~300 LUT |
| Prefetch Logic | 状态跟踪 + 同步 | ~80 LUT |
| Handshake/Sync | done信号采样 + flag | ~50 LUT |
| Watchdog | 24-bit counter + 比较 | ~40 LUT |
| **Control Unit 总计** | | **~1,220 LUT + ~500 FF** |

Control Unit 面积约 1.2K LUT，占整芯片 < 2%，但其正确性直接决定整个 NPU 能否正常工作。

---

## 11. DW Conv Module 微架构

Depthwise Convolution (DW Conv) 模块是 NPU 的专用加速单元，处理 MobileNet 系列中大量存在的 Depthwise 层。DW Conv 的特点是每个输入通道独立卷积（group = C_in），不存在通道间累加，因此不适合脉动阵列（利用率仅 1/C_in），需专用硬件。

### 11.1 整体结构

```
┌──────────────────────────────────────────────────────────────────────┐
│                      DW Conv Module                                   │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │                   DW Control FSM                              │    │
│  │  - 从 Control Unit 接收 DW 层参数                              │    │
│  │  - 驱动滑窗移动 + 通道切换                                      │    │
│  └──────────────┬───────────────────────────┬───────────────────┘    │
│                 │ ctrl                       │ ctrl                   │
│                 ▼                            ▼                        │
│  ┌──────────────────────────┐  ┌─────────────────────────────────┐   │
│  │    Line Buffer           │  │      Weight Register File       │   │
│  │    (KH行 × tile_W × 16ch)│  │      (KH × KW × 16ch)          │   │
│  │                          │  │                                 │   │
│  │  ┌─────────────────────┐ │  │  ┌───┬───┬───┐                 │   │
│  │  │ Line 0  (tile_W×16) │ │  │  │w00│w01│w02│  ch0             │   │
│  │  ├─────────────────────┤ │  │  ├───┼───┼───┤                 │   │
│  │  │ Line 1  (tile_W×16) │ │  │  │w10│w11│w12│  ch0             │   │
│  │  ├─────────────────────┤ │  │  ├───┼───┼───┤                 │   │
│  │  │ Line 2  (tile_W×16) │ │  │  │w20│w21│w22│  ch0             │   │
│  │  └─────────────────────┘ │  │  └───┴───┴───┘                 │   │
│  │  (3×3 kernel需3行缓存)    │  │  ... × 16 channels             │   │
│  └──────────────┬───────────┘  └──────────────┬──────────────────┘   │
│                 │ act window                   │ weights              │
│                 ▼                              ▼                      │
│  ┌───────────────────────────────────────────────────────────────┐   │
│  │              16-Channel Parallel MAC Array                     │   │
│  │                                                               │   │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐       ┌─────────┐      │   │
│  │  │ DW_PE_0 │ │ DW_PE_1 │ │ DW_PE_2 │  ...  │ DW_PE_15│      │   │
│  │  │ ch = 0  │ │ ch = 1  │ │ ch = 2  │       │ ch = 15 │      │   │
│  │  └────┬────┘ └────┬────┘ └────┬────┘       └────┬────┘      │   │
│  │       │            │            │                 │           │   │
│  └───────┼────────────┼────────────┼─────────────────┼───────────┘   │
│          │            │            │                 │               │
│          ▼            ▼            ▼                 ▼               │
│  ┌───────────────────────────────────────────────────────────────┐   │
│  │                    PPU ×16 (共用)                              │   │
│  │         requantize → clamp → relu → 写回 SRAM                 │   │
│  └───────────────────────────────────────────────────────────────┘   │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### 11.2 DW PE (Processing Element) 内部结构

每个 DW_PE 处理 1 个通道的完整卷积窗口：

```
┌───────────────────────────────────────────────────────────────┐
│                      DW_PE[ch]                                 │
│                                                               │
│  输入: KH×KW 个激活值 (来自Line Buffer滑窗)                     │
│        KH×KW 个权重值 (来自Weight Register)                     │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │              乘累加树 (Multiply-Accumulate Tree)          │  │
│  │                                                         │  │
│  │  a[0,0]×w[0,0] ──┐                                     │  │
│  │  a[0,1]×w[0,1] ──┼──→ Adder                            │  │
│  │  a[0,2]×w[0,2] ──┘      │                              │  │
│  │                          ├──→ Adder                     │  │
│  │  a[1,0]×w[1,0] ──┐      │       │                      │  │
│  │  a[1,1]×w[1,1] ──┼──→ Adder     │                      │  │
│  │  a[1,2]×w[1,2] ──┘              ├──→ Final Adder ──→ acc│  │
│  │                                  │                      │  │
│  │  a[2,0]×w[2,0] ──┐              │                      │  │
│  │  a[2,1]×w[2,1] ──┼──→ Adder ────┘                      │  │
│  │  a[2,2]×w[2,2] ──┘                                     │  │
│  │                                                         │  │
│  │  3×3 = 9个乘法 + 3级加法树                                │  │
│  │  流水线: 1 cycle throughput (乘法+加法树各1级)             │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                               │
│  输出: acc[39:0] → PPU                                        │
│                                                               │
└───────────────────────────────────────────────────────────────┘

乘法器规格:
  INT16: 9 × (16×16 → 32-bit) 乘法器/PE
  INT8:  9 × (8×8 → 16-bit) 乘法器/PE，可复用INT16乘法器低半部
  
累加: 9个32-bit积 → 加法树 → 40-bit结果
  3×3: max = 9 × 32767² = 9,663,676,041 (34-bit) ← 40-bit acc绰绰有余
```

### 11.3 Line Buffer 设计

Line Buffer 缓存 KH 行输入数据，支持滑窗操作避免重复加载：

```
┌─────────────────────────────────────────────────────────────────┐
│  Line Buffer (3×3 kernel 示例, 16 channels parallel)             │
│                                                                  │
│  DMA 输入 (逐行写入):                                             │
│  ──────────────→ ┌───────────────────────────────┐               │
│                  │ Line[wr_ptr] (tile_W × 16ch)  │               │
│                  └───────────────────────────────┘               │
│                                                                  │
│  滑窗读取 (3行×3列×16ch 同时输出):                                 │
│                                                                  │
│  ┌────────────────────────────────────────────┐                  │
│  │              tile_W columns                 │                  │
│  │  ┌───┬───┬───┬───┬───┬───┬───┬───┬───┐    │                  │
│  │  │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │...│   │ Line[rd+0]      │
│  │  ├───┼───┼───┼───┼───┼───┼───┼───┼───┤    │                  │
│  │  │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │...│   │ Line[rd+1]      │
│  │  ├───┼───┼───┼───┼───┼───┼───┼───┼───┤    │                  │
│  │  │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │...│   │ Line[rd+2]      │
│  │  └───┴───┴───┴───┴───┴───┴───┴───┴───┘    │                  │
│  │        ▲                                    │                  │
│  │        │ window_col (滑动指针)               │                  │
│  │        └─ 每 cycle 右移1列                   │                  │
│  └────────────────────────────────────────────┘                  │
│                                                                  │
│  循环缓冲:                                                        │
│    - 3行用 circular buffer 管理 (wr_ptr mod 3)                   │
│    - 新行写入时覆盖最老的行                                        │
│    - 滑窗扫完一行后, DMA写入下一行, rd_ptr++                       │
│                                                                  │
│  容量: KH × tile_W × 16ch × 2B                                   │
│    3×3 kernel, tile_W=16: 3 × 16 × 16 × 2 = 1,536 bytes        │
│    3×3 kernel, tile_W=32: 3 × 32 × 16 × 2 = 3,072 bytes        │
└─────────────────────────────────────────────────────────────────┘
```

### 11.4 数据流时序

```
以 3×3 DW Conv, 16ch parallel, tile_W=8 为例:

Phase 1: Weight Load (1次, 整层共用)
══════════════════════════════════════
  16ch × 3×3 = 144 个权重, 从Weight Buffer加载到Weight Reg
  耗时: 144 / (bus_width/2) cycles ≈ 36 cycles (32-bit bus)

Phase 2: Line Buffer 预填充 (KH-1 行)
══════════════════════════════════════
  DMA加载前2行到Line Buffer
  耗时: 2 × tile_W × 16 × 2 = 512 bytes → 128 cycles

Phase 3: 滑窗计算 (逐行推进)
══════════════════════════════════════
  for each output_row:
    1. DMA加载新一行 → Line Buffer (与计算并行)
    2. 滑窗从左到右:
       cycle 0: window_col=0, 16个PE同时输出16ch的acc → PPU
       cycle 1: window_col=1, ...
       ...
       cycle 7: window_col=7 (tile_W-1)
    3. PPU 处理 + 写回
    4. Line Buffer rd_ptr 推进

  每行输出 = tile_W cycles = 8 cycles (16ch/cycle)
  总输出像素: tile_H × tile_W × 16ch
  总 cycle: tile_H × tile_W = OH × 8 (per 16ch group)

通道分组:
  若 C_in > 16: 分 ceil(C_in/16) 组依次处理
  每组独立权重, 独立 Line Buffer 数据
```

### 11.5 Stride 支持

```
stride=1:  window_col 每 cycle +1
stride=2:  window_col 每 cycle +2, Line Buffer 读取跳1行

┌─────────────────────────────────────────┐
│  stride=2 示例 (5×5 input → 2×2 output)  │
│                                          │
│  Input:        读取模式:                  │
│  ┌───┬───┬───┬───┬───┐                   │
│  │ * │ * │ * │   │   │ ← window pos 0   │
│  ├───┼───┼───┼───┼───┤                   │
│  │ * │ * │ * │   │   │                   │
│  ├───┼───┼───┼───┼───┤                   │
│  │ * │ * │ * │   │   │                   │
│  ├───┼───┼───┼───┼───┤                   │
│  │   │   │ * │ * │ * │ ← window pos 1   │
│  ├───┼───┼───┼───┼───┤   (col += stride) │
│  │   │   │ * │ * │ * │                   │
│  └───┴───┴───┴───┴───┘                   │
│                                          │
│  硬件: window_col += stride (可配置)       │
│  Line Buffer rd_ptr += stride (行方向)    │
└─────────────────────────────────────────┘
```

### 11.6 与 Systolic Array 的资源共享

DW Conv 和 Systolic Array 不会同时工作（由 Control Unit 调度互斥），可共享部分资源：

```
┌──────────────────────────────────────────────────────────────┐
│  共享资源                                                      │
├──────────────────────────────────────────────────────────────┤
│  PPU ×16:        SA输出 / DW输出 → MUX → PPU (已有)           │
│  Activation SRAM: DW读输入 / SA读输入 → 时分复用              │
│  DMA Engine:      DW权重加载 / SA权重加载 → 同一DMA通道       │
├──────────────────────────────────────────────────────────────┤
│  独占资源                                                      │
├──────────────────────────────────────────────────────────────┤
│  Line Buffer:     DW专用 (SA不需要)                            │
│  Weight Reg File: DW专用 (SA用Scratchpad存权重)               │
│  9×MAC Tree ×16:  DW专用乘累加树 (SA有自己的PE阵列)           │
└──────────────────────────────────────────────────────────────┘

PPU 入口 MUX:
  ┌──────────────┐
  │ Systolic Arr │──→┐
  │ acc_out[15:0]│   │    ┌─────┐
  └──────────────┘   ├───→│ MUX │───→ PPU[0..15]
  ┌──────────────┐   │    │sel= │
  │ DW Conv      │──→┘    │mode │
  │ acc_out[15:0]│        └─────┘
  └──────────────┘
```

### 11.7 支持的 Kernel 尺寸

| Kernel | 乘法数/PE | 加法树深度 | 说明 |
|--------|----------|-----------|------|
| 1×1 | 1 | 0 | 极少用于DW，但硬件可支持 |
| 3×3 | 9 | 4级(ceil(log2(9))) | **主力**，MobileNet标配 |
| 5×5 | 25 | 5级 | V1.0暂不支持，需扩展MAC树 |
| 7×7 | 49 | 6级 | V1.0暂不支持 |

**V1.0 仅支持 3×3 DW Conv**。5×5/7×7 可通过多次 3×3 近似或后续版本扩展 MAC 树。

### 11.8 控制状态机

```
                    ┌──────────┐
         reset ───→│   IDLE   │ ← 等待 Control Unit 派发 DW 任务
                    └────┬─────┘
                         │ dw_start
                         ▼
                    ┌──────────┐
                    │ LOAD_WGT │  加载 16ch × 3×3 权重到 Weight Reg
                    └────┬─────┘
                         │ wgt_done
                         ▼
                    ┌──────────┐
                    │ FILL_BUF │  预填充 Line Buffer (KH-1=2 行)
                    └────┬─────┘
                         │ fill_done
                         ▼
              ┌─────────────────────┐
              │      COMPUTE        │  滑窗扫描, 逐列产出 16ch acc
              │  (window_col loop)  │
              └──────────┬──────────┘
                         │ row done
                         ▼
              ┌─────────────────────┐
              │    NEXT_ROW         │  DMA加载新行, rd_ptr++
              └──────────┬──────────┘
                         │
                ┌────────┴───────┐
                │ more rows?     │
                ├── yes → COMPUTE│
                │                │
                └── no ──────────┤
                                 ▼
              ┌─────────────────────┐
              │    NEXT_GROUP       │  C_in > 16 时切换到下一组16ch
              └──────────┬──────────┘
                         │
                ┌────────┴───────┐
                │ more groups?   │
                ├── yes → LOAD_WGT│ (加载下一组权重)
                │                │
                └── no ──────────┤
                                 ▼
                         ┌──────────┐
                         │   DONE   │ → 通知 Control Unit
                         └──────────┘
```

### 11.9 接口信号

```verilog
module dw_conv_module #(
    parameter DATA_WIDTH  = 16,
    parameter ACC_WIDTH   = 40,
    parameter DW_CHANNELS = 16,    // 并行通道数
    parameter MAX_KH      = 3,     // 最大kernel高
    parameter MAX_KW      = 3,     // 最大kernel宽
    parameter MAX_TILE_W  = 32     // 最大tile宽度
)(
    input  wire                     clk,
    input  wire                     rst_n,

    // Control Interface (from Control Unit)
    input  wire                     dw_start,
    input  wire [11:0]              dw_oh, dw_ow,      // output spatial size
    input  wire [11:0]              dw_cin,            // input channels (= output ch)
    input  wire [3:0]              dw_kh, dw_kw,       // kernel size
    input  wire [3:0]              dw_stride,
    input  wire [3:0]              dw_pad,
    output wire                     dw_done,
    output wire                     dw_busy,

    // Weight Load (from DMA via Weight Scratchpad)
    input  wire                     wgt_wen,
    input  wire [7:0]              wgt_addr,           // 0..143 (16ch × 9)
    input  wire [DATA_WIDTH-1:0]   wgt_wdata,

    // Activation Input (from Activation Scratchpad / DMA)
    input  wire                     act_line_valid,
    input  wire [DATA_WIDTH-1:0]   act_line_data [DW_CHANNELS-1:0],  // 16ch parallel

    // Output (to PPU)
    output wire [ACC_WIDTH-1:0]    acc_out [DW_CHANNELS-1:0],  // 16ch parallel
    output wire                     acc_valid,

    // Line Buffer read request (to Scratchpad/DMA)
    output wire                     line_req,
    output wire [11:0]              line_col            // current column request
);
```

### 11.10 资源估算

| 组件 | 每 PE | 16 PE 总计 |
|------|-------|-----------|
| 9 × 16-bit 乘法器 | 9 DSP 或 ~540 LUT | 144 DSP 或 ~8,640 LUT |
| 加法树 (9→1) | ~120 LUT | ~1,920 LUT |
| Weight Reg (9 × 16-bit) | 144 FF | 2,304 FF |
| **MAC 小计** | | ~10,560 LUT + 2,304 FF |

| 组件 | 规格 | 面积 |
|------|------|------|
| Line Buffer | 3 × 32 × 16 × 2B = 3KB SRAM | 3 KB SRAM (或 ~1,500 LUT as dist RAM) |
| DW Control FSM | 状态机 + 计数器 | ~200 LUT |
| Stride/Pad 逻辑 | 地址偏移计算 | ~80 LUT |
| **DW Module 总计** | | **~12,340 LUT + ~2,500 FF + 3KB SRAM** |
| **(若用 DSP)** | | **~3,700 LUT + 144 DSP + 3KB SRAM** |

**V1.0 选择：用 DSP 实现乘法**（DSP 总预算 256 个中用 144 个给 DW，余 112 个给 SA 内不够的部分，或 SA 全用 DSP 则需额外 FPGA 选型）。

实际取舍：ASIC 目标下 DSP 不存在，全用组合逻辑乘法器，面积 ~12K LUT 可接受。

---

## 12. 后处理单元 (PPU) 微架构

Post-Processing Unit 负责累加器域→输出量化域的转换，支持 Conv requantize 和 Add rescale 两种工作模式，复用同一条 datapath。

### 12.1 PPU 数据通路

```
┌──────────────────────────────────────────────────────────────────────┐
│                   Post-Processing Unit (PPU)                          │
│                                                                      │
│  ┌──────────┐                                                        │
│  │ Control  │  mode: CONV_REQ / ADD / RELU_ONLY / PASS_THROUGH       │
│  │   FSM    │  ch_idx → 查表 M[ch], S[ch], bias_q[ch], zp[ch]       │
│  └────┬─────┘                                                        │
│       │ ctrl signals                                                 │
│       ▼                                                              │
│  ════════════════════════ Datapath ═══════════════════════════════    │
│                                                                      │
│  ┌───────────┐    ┌──────────┐                                       │
│  │ Systolic  │    │  SRAM    │                                       │
│  │  Array    │    │(Add输入) │                                       │
│  │  Output   │    │          │                                       │
│  └─────┬─────┘    └────┬─────┘                                       │
│        │ 40-bit        │ 16-bit                                      │
│        ▼               ▼                                             │
│  ┌─────────────────────────────┐                                     │
│  │          MUX_IN             │  CONV: acc (40-bit)                 │
│  │    sel = mode               │  ADD cycle1: a_q (sign-ext 40-bit) │
│  │                             │  ADD cycle2: b_q (sign-ext 40-bit) │
│  └──────────────┬──────────────┘                                     │
│                 │ 40-bit                                             │
│                 ▼                                                    │
│  ┌─────────────────────────────┐    ┌─────────────┐                  │
│  │         ADDER_BIAS          │◄───│  bias_q[ch] │                  │
│  │      40-bit + 40-bit        │    │  (40-bit)   │                  │
│  │  CONV: +bias_q   ADD: +0    │    └─────────────┘                  │
│  └──────────────┬──────────────┘                                     │
│                 │ 40-bit                                             │
│                 ▼                                                    │
│  ┌─────────────────────────────┐    ┌─────────────┐                  │
│  │        MULTIPLIER           │◄───│   M[ch]     │                  │
│  │     40-bit × 15-bit         │    │  (15-bit)   │                  │
│  │     → 55-bit product        │    └─────────────┘                  │
│  └──────────────┬──────────────┘                                     │
│                 │ 55-bit                                             │
│                 ▼                                                    │
│  ┌─────────────────────────────┐    ┌─────────────┐                  │
│  │       BARREL SHIFTER        │◄───│   S[ch]     │                  │
│  │    >> S (with rounding)     │    │  (6-bit)    │                  │
│  │    round = +(1 << (S-1))    │    └─────────────┘                  │
│  └──────────────┬──────────────┘                                     │
│                 │ 16-bit (truncated)                                 │
│                 ▼                                                    │
│  ┌─────────────────────────────┐    ┌─────────────┐                  │
│  │         ADDER_ADD           │◄───│  temp_reg   │                  │
│  │      16-bit + 16-bit        │    │  (16-bit)   │                  │
│  │  ADD: +a_rescaled           │    └─────────────┘                  │
│  │  CONV: +0 (bypass)          │          ▲                          │
│  └──────────────┬──────────────┘          │ ADD cycle1 写入          │
│                 │ 17-bit                  │                          │
│                 ▼                         │                          │
│  ┌─────────────────────────────┐    ┌─────────────┐                  │
│  │         ADDER_ZP            │◄───│   zp[ch]    │                  │
│  │  +zp (16-bit signed)       │    │  (16-bit)   │                  │
│  │  ZP_EN=0时 bypass          │    └─────────────┘                  │
│  └──────────────┬──────────────┘                                     │
│                 │ 16-bit                                             │
│                 ▼                                                    │
│  ┌─────────────────────────────┐                                     │
│  │           CLAMP             │  INT16: [-32768, 32767]             │
│  │     saturate to range       │  INT8:  [-128, 127]                 │
│  └──────────────┬──────────────┘                                     │
│                 │                                                    │
│                 ▼                                                    │
│  ┌─────────────────────────────┐                                     │
│  │           ReLU              │  if enabled: max(0, x)              │
│  │     (fused, optional)       │  else: pass-through                 │
│  └──────────────┬──────────────┘                                     │
│                 │                                                    │
│                 ▼                                                    │
│            out_q [16-bit] → 写回 Activation SRAM                     │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### 12.2 工作模式与时序

#### CONV_REQ 模式（1 cycle/element，流水线化）

Conv/FC/DW 后处理链路，每个输出元素 1 个流水拍产出：

```
Stage 1: MUX_IN = acc (from systolic array)
Stage 2: ADDER_BIAS = acc + bias_q[ch]
Stage 3: MULTIPLIER = biased_acc × M[ch]    → 55-bit
Stage 4: BARREL_SHIFT = product >> S[ch]     → 16-bit (with rounding)
Stage 5: ADDER_ZP = shifted + zp[ch]        (ZP_EN=1时生效, 否则bypass)
Stage 6: CLAMP + ReLU → out_q
```

流水线启动后每周期产出 1 个结果。

#### ADD 模式（2 cycles/element）

```
Cycle 1: 处理输入 A
    MUX_IN = a_q (sign-ext to 40-bit)
    ADDER_BIAS = +0 (bypass)
    MULTIPLIER = a_q × M_A
    BARREL_SHIFT = >> S_A
    结果 → temp_reg (暂存 a_rescaled)

Cycle 2: 处理输入 B + 累加
    MUX_IN = b_q (sign-ext to 40-bit)
    ADDER_BIAS = +0 (bypass)
    MULTIPLIER = b_q × M_B
    BARREL_SHIFT = >> S_B
    ADDER_ADD = b_rescaled + temp_reg
    CLAMP + ReLU → out_q
```

#### 其他模式

| 模式 | 描述 | 时序 |
|------|------|------|
| RELU_ONLY | 仅做 ReLU，其余旁路 | 1 cycle |
| PASS_THROUGH | 直通，不做任何处理 | 1 cycle |

### 12.3 Per-Channel 参数存储

PPU 通过查表获取每个输出通道的 requantize 参数：

```
Per-Channel Parameter LUT (SRAM):
┌──────┬──────────┬─────────┬──────────┬────────────┐
│  ch  │  M[15b]  │  S[6b]  │  zp[16b] │ bias_q[64b]│
├──────┼──────────┼─────────┼──────────┼────────────┤
│  0   │  16022   │   22    │    0     │   -1340    │
│  1   │  18901   │   23    │    0     │    562     │
│ ...  │   ...    │   ...   │   ...    │    ...     │
│ N-1  │  12045   │   21    │    0     │     89     │
└──────┴──────────┴─────────┴──────────┴────────────┘

每通道存储: 2(M) + 1(S) + 1(rsvd) + 2(zp) + 8(bias) = 14 bytes/channel
512 通道: 7 KB
```

Add 节点额外存储两组 rescale 参数（共 6 bytes/节点）：

```
Add Parameter Entry:
┌──────────┬─────────┬──────────┬─────────┐
│  M_A[15] │  S_A[6] │  M_B[15] │  S_B[6] │
└──────────┴─────────┴──────────┴─────────┘
= 42 bits，对齐到 6 bytes
```

### 12.4 Add 节点 Rescale 原理

两路输入 scale 不同时，需对齐到输出 scale 再相加：

```
c_q = (a_q × M_A + round) >> S_A  +  (b_q × M_B + round) >> S_B

其中:
    M_A / 2^S_A ≈ scale_A / scale_C
    M_B / 2^S_B ≈ scale_B / scale_C
```

硬件复用 CONV_REQ 的乘法器和移位器，2 cycle 完成一个元素。

### 12.5 硬件资源估算

| 组件 | 规格 | 面积 |
|------|------|------|
| 15-bit 乘法器 | 40×15 → 55-bit | ~150 LUT |
| Barrel Shifter | 55-bit input, 6-bit shift | ~100 LUT |
| ADDER_BIAS | 40-bit | ~60 LUT |
| ADDER_ADD | 16-bit | ~25 LUT |
| ADDER_ZP | 16-bit | ~25 LUT |
| CLAMP + ReLU | 比较器 + MUX | ~40 LUT |
| Control FSM | 模式状态机 + 计数器 | ~100 LUT |
| temp_reg | 16-bit 寄存器 | ~16 FF |
| Parameter LUT 接口 | 地址生成 + 读端口 | ~50 LUT |
| **PPU 总计** | | **~550 LUT** |

PPU 占 16×16 脉动阵列 (~25K LUT) 的 ~2%，面积代价极小。

### 12.6 PPU 与整体流水线的关系

```
Systolic Array                PPU                    SRAM
─────────────        ────────────────────        ──────────
  16×16 PE           (6-stage pipeline)          Activation
  输出 acc   ──────→  bias → mul → shift  ──────→  Buffer
  (每cycle        → zp → clamp → relu        (写回)
   产出1列          (每cycle消耗1个acc)
   = 16个acc)

  吞吐匹配: Array 每 cycle 产出 16 个 acc
            PPU 需 16 组并行 datapath（每列一个）
            或 PPU 串行处理（16 cycle 处理 1 列）
```

**V1.0 选择：每列 1 个 PPU，共 16 组并行**。与脉动阵列输出列对齐，无气泡。

PPU 总面积 = 16 × 550 = ~8,800 LUT，占整芯片 ~6%。

---

## 13. 关键设计参数

### 13.1 时钟频率目标
- FPGA原型：100-200 MHz（Artix-7）
- ASIC目标：200-400 MHz（28nm/40nm）

### 13.2 面积汇总（Pro 配置：16×16, INT16, 40-bit ACC）

| 模块 | LUT | FF | DSP48 | SRAM | 占比(LUT) |
|------|-----|-----|-------|------|----------|
| Systolic Array 16×16 | ~33,000 | ~22,000 | 256 | — | 56.6% |
| DW Conv Module (16ch, 3×3) | ~3,700 | ~2,500 | 144 | 3 KB | 6.3% |
| PPU ×16 | ~8,800 | ~1,000 | — | 8 KB (Param) | 15.1% |
| Control Unit | ~1,220 | ~500 | — | — | 2.1% |
| DMA Engine | ~600 | ~800 | — | — | 1.0% |
| CSR Register File | ~500 | ~400 | — | — | 0.9% |
| Weight Scratchpad 接口 | ~200 | ~100 | — | 32 KB | 0.3% |
| Activation Scratchpad 接口 | ~200 | ~100 | — | 64 KB (2×32) | 0.3% |
| DW Buffer 接口 | ~100 | ~50 | — | 4 KB | 0.2% |
| 总线/胶合逻辑 | ~1,000 | ~500 | — | — | 1.7% |
| **片上逻辑总计** | **~49,320** | **~27,950** | **400** | — | **84.5%** |
| **片上存储总计** | — | — | — | **111 KB** | — |
| **整芯片估算** | **~58,300** | **~30,000** | **400** | **111 KB** | **100%** |

> 注：整芯片 LUT 含约 15% 的布线/时序优化开销（P&R bloat），故 49.3K × 1.18 ≈ 58.3K。

### 13.3 各配置面积对比

| 配置 | SA尺寸 | DW | PPU | CU+DMA | SRAM | LUT总计 | DSP | 等效面积 |
|------|--------|-----|-----|--------|------|---------|-----|---------|
| **Tiny** (8×8, INT8, 32b) | ~8.3K | ~1.8K | ~4.4K | ~1.8K | 39KB | ~20K | 100 | ~0.08mm² |
| **Base** (16×16, INT8, 32b) | ~25K | ~3.7K | ~8.8K | ~1.8K | 75KB | ~45K | 400 | ~0.20mm² |
| **Pro** (16×16, INT16, 40b) | ~33K | ~3.7K | ~8.8K | ~1.8K | 111KB | ~58K | 400 | ~0.23mm² |
| **Pro+** (16×16, INT16, 44b) | ~35K | ~3.7K | ~8.8K | ~1.8K | 111KB | ~60K | 400 | ~0.24mm² |
| **Max** (16×16, INT16, 48b) | ~37K | ~3.7K | ~8.8K | ~1.8K | 139KB | ~63K | 400 | ~0.40mm² |

### 13.4 FPGA 资源映射（Artix-7 200T，Pro 配置）

| 资源 | 使用量 | 芯片总量 | 占比 |
|------|--------|----------|------|
| LUT | ~58,300 | 134,600 | 43% |
| FF | ~30,000 | 269,200 | 11% |
| DSP48E1 | 400 | 740 | 54% |
| BRAM 36Kb | ~28 (111KB) | 365 | 8% |
| BRAM 18Kb | ~6 (Line Buf等) | 730 | <1% |

### 13.5 性能模型
- 16×16阵列 @ 200MHz = 102.4 GOPS (INT8, 2-MAC/PE) 或 51.2 GOPS (INT16, 1-MAC/PE)
- 需要2个阵列交替或1个阵列+高频达到200 GOPS INT8目标
- INT16模式峰值约 100 GOPS（面积换精度，MCU场景可接受）
- DW模块16通道 @ 200MHz = 3.2 GOPS（DW只占3-5%总计算，足够）

---

## 14. 决策依赖链

```
目标场景(MCU IP) → 性能(0.2T) → 工作负载(CNN) → 数据类型(INT8/16)
                                                        ↓
计算架构(脉动阵列+DW) → 存储(分布式Scratchpad) → 总线(Wishbone)
                                                        ↓
                            ISA(层级寄存器配置+后处理流水线)
```

---

## 15. CSR寄存器定义

详见 [npu-register-spec.md](npu-register-spec.md)

- 4KB地址空间，5组寄存器
- Group 0: 控制/状态/中断/性能计数器
- Group 1: 层参数（算子类型、维度、卷积核、步长、padding等）
- Group 2: DMA配置（地址、步长、转置控制）
- Group 3: 后处理配置（shift/scale/clamp/LUT使能）
- Group 4: 256-entry LUT数据（非线性激活函数）
