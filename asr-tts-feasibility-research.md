# CNN-Only NPU (0.2 TOPS) ASR/TTS 可行性研究

## 摘要

本文研究一个 **0.2 TOPS、INT8、MCU级、仅支持CNN（无Transformer）** 的NPU是否能支持简单的ASR（自动语音识别）和TTS（文字转语音）工作负载。

---

## 1. CNN-Only ASR模型（无Transformer/Attention）

### 1.1 关键词检测（Keyword Spotting, KWS）

KWS是最轻量的语音任务——检测少量预定义的唤醒词/命令词（如"Hey Siri"、"OK Google"）。

#### DS-CNN（Depthwise Separable CNN）

来源：ARM "Hello Edge: Keyword Spotting on Microcontrollers" (2017)

| 约束等级 | Flash | RAM | MACs/推理 | 精度 |
|---------|-------|-----|-----------|------|
| Small (S) | ~80 KB | ~30 KB | ~6 MMACs | ~94.4% |
| Medium (M) | ~200 KB | ~50 KB | ~20 MMACs | ~94.9% |
| Large (L) | ~500 KB | ~100 KB | ~80 MMACs | ~95.4% |

- **架构**：纯CNN，使用深度可分离卷积（Depthwise + Pointwise Conv）
- **数据集**：Google Speech Commands（12类关键词）
- **目标硬件**：Cortex-M4/M7，在70KB总内存（权重+激活+特征提取）下可以10次推理/秒
- **算子需求**：Conv2D、Depthwise Conv2D、Pointwise Conv（1×1 Conv）、BatchNorm、ReLU、Average Pooling、FC

#### TC-ResNet（Temporal Convolution ResNet）

来源：Hyperconnect (Interspeech 2019)

- **核心思想**：用1D时域卷积替代2D频域卷积，直接处理原始音频特征
- **优势**：比DS-CNN快 ~385x（在Google Pixel 1上）
- **模型大小**：极小（数十KB级别）
- **精度**：与DS-CNN相当或略优
- **算子需求**：Conv1D、Residual Add、BatchNorm、ReLU、Average Pooling、FC

#### 对0.2 TOPS NPU的评估

- KWS（DS-CNN Large）仅需 ~80 MMACs/推理
- 0.2 TOPS = 200 GOPS = 200,000 MMACs/秒
- **结论**：**极其轻松**。一次KWS推理仅消耗 0.2 TOPS 的 0.04% 算力
- 即使考虑每秒10次推理：800 MMACs/秒，仍只占 0.4% 算力

---

### 1.2 命令识别（Command Recognition, ~100 commands）

比KWS稍复杂——需要区分更多类别（100+命令短语）。

- **方法**：仍使用DS-CNN/TC-ResNet架构，增加输出类别数
- **模型变化**：主要是最后的全连接层增大（12类→100类）
- **额外计算**：约增加 20-50% MACs（主要在FC层）
- **预估总计算量**：~100-150 MMACs/推理
- **对0.2 TOPS的占用**：< 0.1%
- **结论**：**完全可行**，无任何压力

---

### 1.3 连续语音识别（Continuous ASR）

#### Jasper（NVIDIA, 2019）

- **架构**：纯1D卷积，BxR结构（B个Block，每Block R个重复子层）
- **Jasper 10x5**：~333M参数，计算量极大（~30+ GFLOPS/秒音频）
- **结论**：远超0.2 TOPS能力，**不可行**

#### QuartzNet（NVIDIA, 2019）

- **架构**：基于Jasper改进，使用1D Time-Channel Separable Convolutions
- **QuartzNet 5x5**：~6.7M 参数
- **QuartzNet 15x5**：~18.9M 参数
- **计算量估算**：
  - QuartzNet 5x5：~0.5-1 GFLOPS/秒音频（INT8约 1-2 GOPS）
  - QuartzNet 15x5：~1.5-3 GFLOPS/秒音频（INT8约 3-6 GOPS）
- **LibriSpeech WER**：~3.9%（15x5），优秀的识别精度
- **算子需求**：Conv1D、Depthwise Conv1D、Pointwise Conv1D、BatchNorm、ReLU、Residual Add

**对0.2 TOPS的评估：**
- QuartzNet 5x5 INT8量化：~1-2 GOPS/秒音频
- 0.2 TOPS = 200 GOPS/秒
- 实时处理1秒音频需 1-2 GOPS → **理论上可以实时**（占用 ~1% 算力）
- **但是**：模型参数 6.7M × 1字节(INT8) = ~6.7 MB，需要足够的模型存储
- **结论**：**计算上边缘可行，但存储是瓶颈**

#### Citrinet（NVIDIA, 2021）

- **架构**：改进QuartzNet + Squeeze-and-Excitation + Sub-word编码
- **特点**：比QuartzNet精度更高，但计算量也更大
- **Squeeze-and-Excitation**：需要全局平均池化 + FC层（标准CNN算子）
- **结论**：如果QuartzNet边缘可行，Citrinet更难

#### wav2letter（Facebook/Meta）

- **架构**：纯1D卷积网络
- **重要性**：ARM Ethos-U55官方ASR demo使用此模型
- **输入**：MFCC特征 (296×39)
- **输出**：148×29（29字符的概率分布）
- **激活缓冲区**：~766 KB
- **在Ethos-U55上**：~28.5M NPU cycles/推理
- **结论**：是目前MCU级NPU部署连续ASR的主要参考

---

### 1.4 KWS vs 完整ASR的本质区别

| 维度 | KWS（关键词检测） | 完整ASR（连续语音识别） |
|------|------------------|----------------------|
| 词汇量 | 10-20个固定词 | 无限（开放词汇） |
| 输入长度 | 1秒音频片段 | 任意长度连续音频 |
| 输出 | 分类标签（哪个词） | 字符/词序列 |
| 解码器 | argmax | CTC/Beam Search |
| 模型大小 | 50-500 KB | 5-50 MB |
| 计算量 | 6-80 MMACs/推理 | 0.5-30+ GFLOPS/秒 |
| 实时性要求 | 每秒1-10次 | 持续处理 |

---

## 2. CNN-Based TTS模型

### 2.1 现有CNN-Based TTS

#### WaveNet（DeepMind, 2016）
- **架构**：纯CNN（Dilated Causal Convolutions）
- **计算量**：原始版本极其庞大，无法实时
- **改进版**：Parallel WaveNet 可以实时，但仍需GPU级算力
- **结论**：**完全不适合MCU**

#### LPCNet（Mozilla/Xiph, 2019）
- **架构**：轻量级RNN+DSP混合（不是纯CNN）
- **计算量**：~3 GFLOPS（原始），优化后可降低
- **FARGAN（后继者）**：仅需 **600 MFLOPS**
- **特点**：结合传统线性预测编码(LPC)和神经网络
- **MCU可行性**：ARMv7实现有gap，但FARGAN的600 MFLOPS在0.2 TOPS能力范围内
- **问题**：不是纯CNN，包含RNN组件

#### EfficientSpeech（2023）
- **架构**：浅层非自回归金字塔结构Transformer（U-Net形式）
- **参数量**：**266K**（26.6万）
- **计算量**：**90 MFLOPS**
- **性能**：在Raspberry Pi 4 ARM CPU上实时因子 104.3x
- **质量**：接近FastSpeech2，轻微退化
- **问题**：虽然极轻量，但使用了Transformer结构（attention）
- **结论**：**计算上可行（90 MFLOPS << 200 GOPS），但需要Transformer支持**

#### TinyTTS（2026）
- **参数量**：**1.6M**（160万）
- **模型大小**：3.4 MB (FP16 ONNX)
- **架构**：混合（包含attention层 + flow-based组件）
- **性能**：ONNX推理 92ms生成4.88秒音频（Intel CPU），实时因子 ~53x
- **采样率**：44.1 kHz
- **问题**：包含attention机制，**不是纯CNN**
- **结论**：**不适合CNN-only NPU**

### 2.2 TTS在0.2 TOPS的可行性总结

| 方案 | 计算量 | 纯CNN? | 0.2 TOPS可行? |
|------|--------|--------|--------------|
| WaveNet | 数百GFLOPS | ✅是 | ❌不可行 |
| LPCNet/FARGAN | 600 MFLOPS | ❌含RNN | ⚠️计算可行，但需RNN |
| EfficientSpeech | 90 MFLOPS | ❌含Transformer | ⚠️计算可行，但需attention |
| TinyTTS | ~数百MFLOPS | ❌含attention | ⚠️计算可行，但需attention |
| 拼接合成（ESP32方案） | 极低 | N/A | ✅可行（非神经网络） |

**关键发现**：
- 所有现代高质量TTS方案都包含attention或RNN组件
- **纯CNN TTS要么计算量巨大（WaveNet），要么不存在**
- MCU级设备上的TTS主要使用**拼接合成**（concatenative synthesis），不是神经网络

### 2.3 行业现状：MCU级设备TTS

- **Espressif ESP32-SR**：使用拼接合成（concatenative），非神经网络TTS，仅支持中文
- **MicroASR**：传统TTS引擎，需 100 DMIPS + 2MB Flash + 256KB RAM，支持10万词汇
- **结论**：MCU级TTS主要依赖传统方法（拼接/参数合成），不是神经网络

---

## 3. 实际可行性分析（0.2 TOPS NPU）

### 3.1 关键词检测（10-20个词）

| 指标 | 数值 |
|------|------|
| 模型 | DS-CNN / TC-ResNet |
| 参数量 | 20K-500K |
| 模型大小(INT8) | 20-500 KB |
| 计算量 | 6-80 MMACs/推理 |
| 推理频率 | 10次/秒 |
| 总计算需求 | 60-800 MMACs/秒 |
| 0.2 TOPS占用 | < 0.5% |
| **可行性** | ✅ **非常轻松** |

### 3.2 命令识别（~100个命令）

| 指标 | 数值 |
|------|------|
| 模型 | DS-CNN Large变体 |
| 参数量 | 200K-1M |
| 模型大小(INT8) | 200KB-1MB |
| 计算量 | 80-200 MMACs/推理 |
| 推理频率 | 5-10次/秒 |
| 总计算需求 | 400-2000 MMACs/秒 |
| 0.2 TOPS占用 | < 1% |
| **可行性** | ✅ **轻松可行** |

### 3.3 连续语音识别（自由语音）

| 指标 | 数值 |
|------|------|
| 模型 | QuartzNet 5x5 / wav2letter |
| 参数量 | 6-19M |
| 模型大小(INT8) | 6-19 MB |
| 计算量 | 1-6 GOPS/秒音频 |
| 0.2 TOPS占用 | 0.5-3% (计算) |
| 存储需求 | 6-19 MB Flash |
| RAM需求 | 1-2 MB（激活缓冲） |
| CTC解码 | 需CPU辅助 |
| **可行性** | ⚠️ **计算边缘可行，存储和RAM是主要瓶颈** |

**挑战**：
- 模型存储：6-19 MB，MCU Flash通常只有1-4 MB
- 激活RAM：~1 MB，MCU SRAM通常 256-512 KB
- 需要外部Flash/PSRAM
- CTC解码（Beam Search）需要CPU辅助
- 实际WER（错误率）可能因量化而退化

### 3.4 TTS（文字转语音）

| 方案 | 可行性 |
|------|--------|
| 神经网络TTS（纯CNN） | ❌ 不存在实用的纯CNN轻量TTS |
| 神经网络TTS（含attention） | ❌ NPU不支持attention |
| 拼接合成 | ✅ 可行，但不需要NPU |
| FARGAN vocoder | ⚠️ 600 MFLOPS可行，但含RNN |

**结论**：CNN-only NPU **无法有效支持神经网络TTS**。TTS应由CPU使用传统拼接合成方案完成。

---

## 4. 算子需求分析

### 4.1 CNN-Based ASR所需算子

#### 核心卷积算子
| 算子 | KWS | 命令识别 | 连续ASR | 备注 |
|------|-----|---------|---------|------|
| Conv2D | ✅ | ✅ | - | DS-CNN使用 |
| Conv1D | ✅ | ✅ | ✅ | TC-ResNet/QuartzNet |
| Depthwise Conv2D | ✅ | ✅ | - | DS-CNN核心 |
| Depthwise Conv1D | - | - | ✅ | QuartzNet/Citrinet |
| Pointwise Conv (1×1) | ✅ | ✅ | ✅ | 所有深度可分离架构 |

#### 激活和归一化
| 算子 | 必要性 | 备注 |
|------|--------|------|
| ReLU | ✅必须 | 所有模型 |
| BatchNorm | ✅必须 | 推理时融合为Scale+Bias |
| Residual Add | ✅必须 | ResNet/QuartzNet |

#### 池化和全连接
| 算子 | 必要性 | 备注 |
|------|--------|------|
| Average Pooling | ✅必须 | 全局或局部 |
| Max Pooling | 可选 | 部分模型 |
| Fully Connected (FC) | ✅必须 | 分类头/CTC输出 |
| Softmax | ✅必须 | 输出概率 |

### 4.2 特殊需求（NPU外处理）

| 组件 | 处理位置 | 说明 |
|------|---------|------|
| **MFCC特征提取** | DSP/CPU | 音频→梅尔频率倒谱系数，纯数学运算 |
| **CTC解码器** | CPU | 贪心解码或Beam Search，非神经网络 |
| **Beam Search** | CPU | 搜索算法，需要语言模型（可选） |
| **语言模型** | CPU | N-gram或小型模型（连续ASR可选） |
| **VAD（语音活动检测）** | DSP/CPU/NPU | 判断是否有语音输入 |

### 4.3 需要Transformer/Attention的场景

| 场景 | 是否必须Attention? |
|------|-------------------|
| KWS（关键词检测） | ❌ 不需要 |
| 命令识别 | ❌ 不需要 |
| 连续ASR（QuartzNet） | ❌ 不需要 |
| 连续ASR（Citrinet SE模块） | ⚠️ SE需要全局池化+FC，可映射为标准算子 |
| TTS（高质量神经网络） | ✅ 几乎所有现代TTS都需要 |
| 连续ASR（Conformer/Whisper） | ✅ 需要，但这些是最新最强模型 |

**结论**：**KWS和命令识别完全不需要Transformer**。连续ASR可以用纯CNN（QuartzNet）实现，但精度不如Transformer方案。

### 4.4 Conv1D 与 Conv2D 的映射关系

**关键问题**：NPU如果只支持Conv2D，能否运行Conv1D模型？

**答案：可以**。Conv1D(input=[B,C,T], kernel=[K]) 等价于 Conv2D(input=[B,C,1,T], kernel=[1,K])。
只需将一个空间维度设为1即可。所有1D操作都可以映射到2D算子。

---

## 5. 竞品分析

### 5.1 Arm Ethos-U55

| 项目 | 规格 |
|------|------|
| 计算性能 | 64-512 GOPS（配置依赖MAC数量） |
| MAC配置 | 32/64/128/256 MAC |
| 支持框架 | TensorFlow Lite Micro |
| 面积 | ~0.1 mm² |
| 功耗 | 声称90%能耗降低vs纯Cortex-M |
| 支持网络 | CNN、RNN |

**语音支持**：
- ✅ **KWS（关键词检测）**：官方demo，DS-CNN模型
- ✅ **ASR（连续语音识别）**：官方demo，使用 **wav2letter**（纯CNN）模型
- ✅ **TTS**：官方宣传支持（但未公开具体TTS模型）
- 工具链：Vela编译器，支持INT8量化、剪枝、算子融合

**关键参考**：Ethos-U55 官方ML评估套件包含完整的KWS+ASR级联demo：
- 先用DS-CNN做KWS检测唤醒词
- 检测到后用wav2letter做连续ASR
- wav2letter在U55上运行：~28.5M cycles/推理，激活缓冲~766KB

**Ethos-U85（下一代）**：
- 性能：256 GOPS到 4 TOPS @1GHz
- 新增：原生Transformer支持
- 目标：更长上下文ASR、更流畅TTS、小型LLM对话

### 5.2 Syntiant NDP200

| 项目 | 规格 |
|------|------|
| 计算性能 | **6.4 GOPS**（0.0064 TOPS） |
| 功耗 | < 1 mW |
| 处理器 | Syntiant Core 2 + Cortex-M0+ + HiFi-3 DSP |
| 模型支持 | CNN、RNN、FC，>7M参数 |
| 接口 | SPI、I2C、I2S、双PDM、11线图像接口 |
| 价格 | $10（万片） |

**语音支持**：
- ✅ 语音命令识别（Voice Control）
- ✅ 声学事件检测
- ✅ 多传感器融合
- ❌ 未宣传完整ASR
- ❌ 未宣传TTS

**注意**：NDP200仅6.4 GOPS，比我们的0.2 TOPS（200 GOPS）**低30倍**。它主要定位超低功耗always-on场景。

### 5.3 对比总结

| NPU | 算力 | KWS | 命令识别 | 连续ASR | TTS |
|-----|------|-----|---------|---------|-----|
| **我们的设计** | 0.2 TOPS | ✅ | ✅ | ⚠️边缘 | ❌(纯CNN) |
| Ethos-U55 (128MAC) | ~0.1-0.5 TOPS | ✅ | ✅ | ✅(wav2letter) | 宣传支持 |
| Syntiant NDP200 | 0.0064 TOPS | ✅ | ✅ | ❌ | ❌ |
| Ethos-U85 | 0.25-4 TOPS | ✅ | ✅ | ✅ | ✅ |

### 5.4 MicroASR（商业方案参考）

- **类型**：传统算法（非神经网络）
- **ASR**：200 DMIPS + 700KB ROM + 256KB RAM，支持10万词汇，>99%精度
- **TTS**：100 DMIPS + 2MB Flash + 256KB RAM
- **平台**：Cortex-M4/M7、ESP32等
- **意义**：证明了MCU级设备可以做完整ASR/TTS，但使用传统方法而非深度学习

---

## 6. 结论与建议

### 6.1 明确可行的功能

| 功能 | 推荐 | 理由 |
|------|------|------|
| **KWS（关键词检测）** | ✅ **强烈推荐** | 极低计算需求，成熟方案，是NPU核心卖点 |
| **命令识别（100词）** | ✅ **推荐** | 计算需求仍很低，市场需求大 |
| **VAD（语音活动检测）** | ✅ **推荐** | 可用小型CNN实现，always-on场景必需 |
| **声学事件检测** | ✅ **推荐** | 类似KWS的CNN分类任务 |

### 6.2 有条件可行的功能

| 功能 | 评估 | 条件 |
|------|------|------|
| **连续ASR** | ⚠️ | 需要6+MB模型存储、1+MB RAM、CPU辅助CTC解码 |
| **FARGAN vocoder** | ⚠️ | 600 MFLOPS计算可行，但包含RNN组件 |

### 6.3 不可行的功能

| 功能 | 原因 |
|------|------|
| **神经网络TTS** | 所有实用TTS都需要attention/RNN，纯CNN方案(WaveNet)计算量过大 |
| **高精度Transformer ASR** | 需要attention支持 |

### 6.4 NPU设计建议

**必须支持的算子（用于KWS/命令识别）**：
1. Conv2D（含stride、padding）
2. Depthwise Conv2D
3. Pointwise Conv（1×1 Conv2D）
4. ReLU / ReLU6
5. Average Pooling / Global Average Pooling
6. Fully Connected (FC / Dense)
7. Softmax
8. Element-wise Add（Residual连接）
9. Quantize / Dequantize

**建议支持的算子（用于连续ASR）**：
- Conv1D → 可映射为Conv2D([1,K])
- Larger model支持（需足够的模型缓冲带宽）

**不需要NPU支持的组件**：
- MFCC特征提取 → DSP/CPU
- CTC解码/Beam Search → CPU
- 语言模型 → CPU

### 6.5 市场定位建议

参考竞品定位：
- **vs Syntiant NDP200**（6.4 GOPS）：我们算力高30x，可以做更复杂的命令识别
- **vs Ethos-U55**（100-500 GOPS）：算力相当，应对标其KWS+ASR能力
- **核心卖点**：KWS + 命令识别 + VAD + 声学事件检测
- **差异化机会**：如果支持连续ASR（wav2letter/QuartzNet），将与Ethos-U55直接竞争

---

## 7. 参考资料

1. ARM, "Hello Edge: Keyword Spotting on Microcontrollers", arXiv:1711.07128, 2017
2. NVIDIA, "QuartzNet: Deep Automatic Speech Recognition with 1D Time-Channel Separable Convolutions", arXiv:1910.10261, 2019
3. NVIDIA, "Citrinet: Closing the Gap between Non-Autoregressive and Autoregressive End-to-End Models", 2021
4. Facebook, "wav2letter: a fast open source speech recognition system", 2016
5. Mozilla/Xiph, "LPCNet: Improving Neural Speech Synthesis through Linear Prediction", 2019
6. ARM ML Embedded Evaluation Kit - KWS+ASR use case
7. Syntiant NDP200 Product Brief
8. EfficientSpeech: An On-Device Text to Speech Model, arXiv:2305.13905, 2023
9. TinyTTS: The Smallest English TTS Model, 2026
10. MicroASR: Commercial embedded speech recognition solution
