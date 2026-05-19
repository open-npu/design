# CNN-only vs CNN+Transformer 硬件复杂度分析

> 为 Open-NPU 0.2 TOPS MCU级NPU 的工作负载决策提供调研支持

---

## 1. CNN-only 需要的运算

CNN推理的核心运算都可以映射到 **MAC阵列 + 简单控制逻辑**：

| 运算 | 硬件映射 | 复杂度 |
|------|----------|--------|
| Conv2D | MAC阵列 + 权重加载 | 核心计算单元，规则数据流 |
| Depthwise Conv | MAC阵列（通道独立） | 同上，利用率可能略低 |
| Pooling (Max/Avg) | 比较器/加法器 | 极简单 |
| Fully Connected | MAC阵列（矩阵乘） | 与Conv共享硬件 |
| ReLU | 比较器 max(0,x) | 1个比较器，~1 LUT |
| BatchNorm (fused) | 推理时融合为 scale+bias | 1乘1加，融合进后处理 |
| Add/Concat | 加法器 / 数据搬运 | 几乎无开销 |

**关键特征**：
- 数据访问模式规则（滑动窗口），适合SRAM tiling
- 所有非线性运算极简单（ReLU = 比较器）
- BatchNorm推理时已融合为乘加，无需额外硬件

---

## 2. Transformer 额外需要的运算

在CNN基础上，Transformer**额外**需要：

| 运算 | 与CNN共享？ | 额外硬件需求 | 复杂度评估 |
|------|------------|-------------|-----------|
| MatMul (Q×K^T, Attn×V) | ✅ 可复用FC/MAC阵列 | 无额外（但访问模式不同） | 低 |
| **Softmax** | ❌ 全新 | exp()、求和、除法 | **高** |
| **LayerNorm** | ❌ 全新 | 均值、方差、sqrt、除法 | **高** |
| **GELU/SiLU** | ❌ 全新 | 非线性近似（多项式/LUT） | **中** |
| Transpose/Reshape | 部分 | 地址生成器调整 | 低 |
| 大矩阵非规则访问 | — | 更复杂的地址生成 | 中 |

### 各运算的硬件实现细节

#### Softmax: `softmax(x_i) = exp(x_i) / Σexp(x_j)`

需要3个子运算：
1. **指数运算 exp(x)**：最复杂部分
   - 查表法（LUT）：精度好但存储开销大
   - 分段线性近似：精度与面积折中
   - Base-2替代：用 2^x 替代 e^x，简化为移位操作
   - 二阶Taylor近似：资源最少但精度有限
2. **累加 Σexp(x_j)**：需要全向量遍历，加法树
3. **除法**：需要除法器或倒数近似

#### LayerNorm: `(x - μ) / sqrt(σ² + ε) * γ + β`

需要的子运算：
1. **均值计算**：累加 + 除法（或移位近似）
2. **方差计算**：差的平方 → 累加 → 除法
3. **平方根倒数 1/sqrt()**：迭代法或LUT
4. **乘加**：scale和bias

对比BatchNorm（推理时）：仅需1乘1加（预计算好的γ/sqrt(σ²+ε)和β-μγ/sqrt(σ²+ε)）

#### GELU: `x * 0.5 * (1 + erf(x/sqrt(2)))`

- 比ReLU复杂得多（ReLU = 1个比较器）
- I-BERT方案：二阶多项式近似，仅需整数乘法和移位
- 硬件：可用分段线性近似或复用Softmax的exp单元

---

## 3. FPGA资源定量分析

### 3.1 MAC阵列（CNN核心）

基于已发表论文的FPGA综合数据（Xilinx UltraScale XCZU3EG）：

| 实现 | 规模 | LUT | FF | DSP | 备注 |
|------|------|-----|-----|-----|------|
| tinyTPU | 14×14 INT8 | 120 | 129 | 196 | 基础实现 |
| Libano参考 | 14×14 INT8 | 23,080 | 60,422 | 196 | 带加法树 |
| DSP优化方案 | 14×14 INT8 | 167 | 4,516 | 210 | DSP内移位优化 |
| Xilinx DPU B1024 | ~32×32等效 | 1,280 | 7,856 | 192 | 商用参考 |

**估算16×16 INT8 systolic array**：
- 使用DSP48E2打包INT8乘法：**128-256个DSP**
- LUT开销（控制+加法树）：**1,000-5,000 LUT**
- FF（流水线寄存器）：**5,000-10,000 FF**
- 总计约 **5,000-15,000 LUT等效面积**

### 3.2 Softmax单元

基于Hyft论文（Xilinx xc7z030，8输入向量）：

| 实现方案 | LUT | FF | 频率 | 延迟 |
|----------|-----|-----|------|------|
| APCCAS'18 查表法 | 2,564 | 2,794 | 436 MHz | — |
| ISCAS'20 近似 | 2,229 | 224 | 154 MHz | — |
| TCAS-I'22 优化 | 1,476 | 698 | 500 MHz | — |
| Hyft16 (16-bit) | 1,072 | 824 | 625 MHz | 12.4ns |
| Hyft32 (32-bit) | 2,399 | 1,528 | 526 MHz | 19ns |
| Xilinx官方FP32 | 13,254 | 18,664 | 435 MHz | 232ns |

**结论**：一个高效的INT8 Softmax单元约需 **1,000-2,500 LUT**

### 3.3 LayerNorm单元

基于HAAN论文（Xilinx Alveo U280，INT8配置）：

| 配置 | LUT | FF | DSP | 功耗 |
|------|-----|-----|-----|------|
| (256, 256) | 58K | 21K | 1536 | 3.5W |
| (32, 512) | 86K | 25K | 1025 | 6.4W |

注意：上述是高并行度设计（面向服务器级）。对于低功耗MCU级NPU，缩减并行度后估算：

**单路LayerNorm单元估算**：
- 均值计算（累加+除法）：~500 LUT
- 方差计算（平方差+累加+除法）：~800 LUT
- 平方根倒数（迭代/LUT）：~500 LUT
- Scale+Bias：~200 LUT（或复用DSP）
- **总计：~2,000-3,000 LUT**

对比BatchNorm（推理时）：仅需1个乘法器+1个加法器 ≈ **1个DSP或~100 LUT**

### 3.4 GELU/SiLU

基于论文数据（45nm标准单元，但可推算FPGA等效）：
- 复用Softmax的exp单元：额外开销仅 **9-11%面积增加**
- 独立实现：分段线性近似约 **500-1,000 LUT**
- 对比ReLU：**1个LUT**

### 3.5 开销汇总

| 模块 | LUT估算 | 占MAC阵列的比例 |
|------|---------|----------------|
| 16×16 INT8 MAC阵列（核心） | 5,000-15,000 | 基准100% |
| Softmax单元 | 1,000-2,500 | 10-25% |
| LayerNorm单元 | 2,000-3,000 | 15-30% |
| GELU激活 | 500-1,000 | 5-10% |
| 地址生成器增强 | 500-1,000 | 5-10% |
| **Transformer支持总额外开销** | **4,000-7,500** | **35-75%** |

**结论：支持Transformer的硬件开销约为CNN-only设计的 35%-75% 额外面积。**

---

## 4. 现有小型NPU的做法

### 4.1 Arm Ethos-U55（对标产品）

**Ethos-U55 支持的算子**（Vela编译器确认）：
- ✅ Conv2D, Depthwise Conv, FC, Pooling
- ✅ ReLU, ReLU6, Sigmoid, Tanh, Hard-Swish, Leaky-ReLU
- ✅ Add, Sub, Mul, Concatenation
- ✅ **Softmax**（硬件加速，有约束条件）
- ✅ **EXP**（硬件加速）
- ✅ MEAN（归约操作）
- ❌ **BATCH_MATMUL**（不支持！回退到CPU）
- ❌ LayerNorm（不支持，回退到CPU）
- ❌ GATHER, TRANSPOSE（不支持）

**关键发现**：
- Ethos-U55 **不支持Transformer**，缺少MATMUL、TRANSPOSE、GATHER
- Arm为此专门开发了 **Ethos-U85**（2024年发布），性能4倍提升，新增5个关键算子以支持Transformer
- U85算力范围：256 GOPS ~ 4 TOPS（远超我们的0.2 TOPS目标）

### 4.2 Gemmini（UC Berkeley，开源学术参考）

- 核心：systolic array，主攻矩阵乘
- Transformer支持：通过 **I-BERT** 方案
  - 非线性运算（Softmax/GELU/LayerNorm）用**整数多项式近似**
  - 在加速器的累加器中用整数算术完成
  - 不是专用硬件单元，而是巧妙利用已有的乘加能力
- 结果：比纯CPU实现快40倍

### 4.3 行业通行做法

| 方案 | 代表产品 | 适用场景 |
|------|----------|----------|
| 非线性运算回退CPU | Ethos-U55, 多数小型NPU | 小算力(<1 TOPS)，面积敏感 |
| 硬件加速非线性 | Ethos-U85, 大型NPU | 大算力(>1 TOPS)，带宽敏感 |
| 整数近似复用MAC | Gemmini (I-BERT) | 学术/灵活设计 |
| 可编程向量单元 | NVDLA后处理管线 | 中等规模，可扩展 |

**小结**：0.2 TOPS级别的NPU，业界做法是 **非线性运算回退CPU或用整数近似复用MAC**。

---

## 5. 存储/带宽影响分析

### 5.1 Attention的O(n²)问题

CNN（Conv2D）:
- 访问模式：规则滑动窗口
- 片上缓存需求：输入tile + 权重tile + 输出tile
- 对于典型层：几KB到几十KB即可tiling

Transformer（Self-Attention）:
- Q×K^T 产生 n×n attention矩阵
- 假设序列长度 n=64，embedding_dim=64：
  - Q, K, V 各 64×64 = 4KB (INT8)
  - Attention矩阵：64×64 = 4KB
  - 总计单头：~16KB
- 如果 n=128：attention矩阵 = 16KB，总计 ~48KB
- 如果 n=256：attention矩阵 = 64KB，**已超出多数MCU NPU的片上SRAM**

### 5.2 对0.2 TOPS NPU的影响

典型片上SRAM预算（参考Ethos-U55）：**16-128 KB**

| 序列长度 | Attention矩阵大小(INT8) | 总Buffer需求(单头) | 是否可行 |
|----------|------------------------|-------------------|----------|
| 32 | 1 KB | ~6 KB | ✅ 可行 |
| 64 | 4 KB | ~16 KB | ✅ 勉强可行 |
| 128 | 16 KB | ~48 KB | ⚠️ 需要较大SRAM |
| 256 | 64 KB | ~192 KB | ❌ 不实际 |
| 512 | 256 KB | ~768 KB | ❌ 完全不可行 |

### 5.3 TinyFormer实际案例

根据TinyFormer论文（MCU部署实测）：
- 目标平台：STM32F746（320KB SRAM, 1MB Flash）
- 成功部署Transformer，peak memory = **120-300 KB**
- 但 Softmax 占推理延迟的 **94%**（纯CPU执行）
- 总推理时间：**3.9秒**（Cortex-M7 @ 216MHz）
- 使用bitmap查表优化Softmax后：6.9-19×加速

**结论**：Transformer在MCU上**可行但受限**，序列长度必须很短（32-64），且Softmax是性能瓶颈。

---

## 6. 实际建议

### 6.1 推荐方案：CNN-first + 有限Transformer加速

**不建议**在0.2 TOPS NPU上做完整的Transformer硬件加速，原因：
1. 面积开销35-75%过高（Softmax + LayerNorm + GELU）
2. Ethos-U55同级产品也不支持Transformer
3. 0.2 TOPS算力 + 有限SRAM，Transformer实际可跑的模型很小
4. Arm为支持Transformer专门做了U85（4x性能提升）

**建议的分层策略**：

```
┌─────────────────────────────────────────────────┐
│ 硬件加速（NPU）                                   │
│  • MAC阵列：Conv2D, Depthwise Conv, FC, MatMul   │
│  • 简单激活：ReLU, ReLU6                          │
│  • 池化：Max/Average Pooling                      │
│  • 元素运算：Add, Mul                             │
│  • （可选）Softmax查表加速                         │
├─────────────────────────────────────────────────┤
│ CPU回退（RISC-V）                                  │
│  • LayerNorm                                      │
│  • GELU/SiLU                                      │
│  • Softmax（如果NPU未实现）                        │
│  • Transpose, Gather                              │
│  • 其他长尾算子                                    │
└─────────────────────────────────────────────────┘
```

### 6.2 关键设计决策

| 决策点 | 建议 | 理由 |
|--------|------|------|
| MatMul | ✅ NPU硬件加速 | 复用MAC阵列，零额外硬件 |
| Softmax | ⚠️ 可选轻量加速 | Base-2查表法 ~1000 LUT，解决94%瓶颈 |
| LayerNorm | ❌ CPU回退 | 硬件复杂（除法+sqrt），使用频率低 |
| GELU | ❌ CPU回退 | 可用ReLU替代，或I-BERT整数近似在CPU跑 |
| Transpose | ✅ DMA/地址生成 | 无计算开销，仅改变访问模式 |

### 6.3 最终建议

**Phase 1（V1.0）**：纯CNN-only设计
- 覆盖95%的TinyML应用场景
- 最小面积，最快上线
- 目标：MobileNet-v2, MCUNet, ResNet-8等

**Phase 2（V2.0）**：添加轻量Transformer支持
- 添加Softmax LUT加速（~1000 LUT，仅10%面积增加）
- 确保MatMul可被高效调度（灵活tiling）
- 支持混合CNN+Attention模型（如MobileViT-tiny）
- LayerNorm/GELU仍在CPU执行

**不建议（除非升级到>1 TOPS）**：
- 完整LayerNorm硬件单元
- GELU硬件单元
- 支持长序列(>128)的attention

---

## 参考文献

1. Hyft: Softmax Acceleration with Adaptive Numeric Format (arXiv:2311.13290) — Softmax FPGA资源数据
2. DSP Optimization for FPGA Systolic Arrays (arXiv:2409.03508) — MAC阵列FPGA资源数据
3. HAAN: Holistic Approach for Accelerating Normalization (arXiv:2502.11832) — LayerNorm资源数据
4. I-BERT: Integer-only BERT Quantization (arXiv:2101.01321) — 整数近似方法
5. TinyFormer: Efficient Transformer Design for MCU (arXiv:2311.01759) — MCU部署实测
6. Arm Ethos-U55/U85 Operator Support — 商用NPU参考
7. Gemmini OSCAR Workshop Presentation — 开源加速器Transformer支持
8. Reusing Softmax Hardware for GELU (arXiv:2402.10118) — 硬件复用方案
