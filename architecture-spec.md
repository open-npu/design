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
│  │              Post-Processing Pipeline                        │   │
│  │  [Shift] → [Multiply] → [Add] → [Compare/Clamp] → [LUT]   │   │
│  │                                                256-entry     │   │
│  │  (~500-2000 LUT, 可编程, 非CPU核)              非线性激活     │   │
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
| **Post-Processing Pipeline** | 可编程后处理：shift/mul/add/clamp/LUT激活 | 接计算输出 → 写回Activation Buffer | ~500-2000 LUT |

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

## 8. 关键设计参数

### 8.1 时钟频率目标
- FPGA原型：100-200 MHz（Artix-7）
- ASIC目标：200-400 MHz（28nm/40nm）

### 8.2 FPGA资源估算（Artix-7 200T）
| 资源 | 估计用量 | 芯片总量 | 占比 |
|------|----------|----------|------|
| DSP48 | ~256-512 | 740 | 35-70% |
| LUT | ~30-50K | 134K | 22-37% |
| BRAM (36Kb) | ~36-72 | 365 | 10-20% |
| FF | ~20-40K | 269K | 7-15% |

### 8.3 性能模型
- 16×16阵列 @ 200MHz = 102.4 GOPS (INT8, 2-MAC/PE) 或 51.2 GOPS (INT16, 1-MAC/PE)
- 需要2个阵列交替或1个阵列+高频达到200 GOPS INT8目标
- INT16模式峰值约 100 GOPS（面积换精度，MCU场景可接受）
- DW模块16通道 @ 200MHz = 3.2 GOPS（DW只占3-5%总计算，足够）

---

## 9. 决策依赖链

```
目标场景(MCU IP) → 性能(0.2T) → 工作负载(CNN) → 数据类型(INT8/16)
                                                        ↓
计算架构(脉动阵列+DW) → 存储(分布式Scratchpad) → 总线(Wishbone)
                                                        ↓
                            ISA(层级寄存器配置+后处理流水线)
```

---

## 10. CSR寄存器定义

详见 [npu-register-spec.md](npu-register-spec.md)

- 4KB地址空间，5组寄存器
- Group 0: 控制/状态/中断/性能计数器
- Group 1: 层参数（算子类型、维度、卷积核、步长、padding等）
- Group 2: DMA配置（地址、步长、转置控制）
- Group 3: 后处理配置（shift/scale/clamp/LUT使能）
- Group 4: 256-entry LUT数据（非线性激活函数）
