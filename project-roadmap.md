# Open-NPU 项目路线图

> 本文档是一个渐进式的项目规划。每个阶段包含需要调研和决策的关键问题。
> 我们逐步推进：调研 → 决策 → 记录结论 → 进入下一步。
>
> **当前进度：阶段 2 - 软硬件并行开发**

---

## 阶段 0：项目定位与目标（调研中）

### 0.1 目标定位
- [x] **决策：NPU的目标场景是什么？** → **端侧推理，低算力MCU挂载IP**
  - 定位：作为NPU IP核挂载到低算力CPU上，不做完整独立芯片
  - 场景：IoT/嵌入式/可穿戴/智能家居

- [x] **决策：工作负载是什么？** → **CNN为主，V2.0再加轻量Transformer支持**
  - V1.0：纯CNN（Conv2D, DW Conv, FC, Pooling, ReLU, Add, Resize, DeConv, Concat）
  - V2.0：添加Softmax LUT加速（~1000 LUT），MatMul复用MAC阵列
  - LayerNorm/GELU永远CPU回退（硬件开销太大：额外35-75%面积）
  - 理由：Ethos-U55同级也不支持Transformer，0.2T+有限SRAM跑Transformer不实际
  - 详见：cnn-vs-transformer-hardware-analysis.md

- [x] **决策：语音场景支持？** → **V1.0支持KWS+命令识别，连续ASR作为stretch goal**
  - KWS（关键词检测）：DS-CNN/TC-ResNet，~80 MMACs/推理，占0.2T不到0.5%，**极轻松**
  - 命令识别（~100词）：DS-CNN变体，< 1% 算力，**完全可行**
  - 连续ASR：wav2letter/QuartzNet纯CNN可做，计算可行但存储是瓶颈（6+MB模型）
  - TTS：纯CNN NPU**无法支持**（所有现代TTS都需attention/RNN），留给CPU用传统方案
  - 算子方面：标准CNN算子（Conv2D/DW Conv/ReLU/Pooling/FC/Add）即可覆盖所有语音CNN模型
  - Conv1D可映射为Conv2D(kernel=[1,K])，不需额外硬件
  - MFCC特征提取和CTC解码由CPU/DSP处理
  - 详见：asr-tts-feasibility-research.md

- [x] **决策：性能目标量级？** → **0.2 TOPS (200 GOPS)**
  - 对标Arm Ethos-U55中档配置
  - TinyML甜蜜点

- [x] **决策：是否考虑未来商业化？** → **开源但保留商业化可能**
  - License选择：Apache 2.0（商业友好，不会GPL污染）
  - 代码质量从一开始按可综合标准写
  - 不急于商业化，但架构和代码为后续转商业留空间

- [x] **决策：是否需要CPU Demo SoC？** → **需要，选RISC-V**
  - 所有成功的开源加速器项目都提供CPU集成示例
  - RISC-V许可证允许完全开源再分发，Cortex-M3不行
  - 首选VexRiscv（MIT许可），备选Ibex（Apache 2.0）
  - 使用LiteX SoC框架快速搭建
  - 详见：cpu-demo-soc-research.md

### 0.2 竞品与参考项目调研
- [ ] **调研：已有开源NPU/加速器项目**
  - NVDLA（NVIDIA，已停维护但架构经典）
  - VTA（TVM配套，教学导向）
  - Gemmini（UC Berkeley，systolic array）
  - CFU-Playground（Google，FPGA上的自定义加速器）
  - DIANA（混合数模计算）
  - Eyeriss（MIT，学术参考）
  - 调研要点：各项目的架构选择、优缺点、社区活跃度、可复用的部分

- [ ] **调研：商业NPU架构参考**
  - ARM Ethos系列
  - Hailo
  - 寒武纪
  - Google TPU（公开论文）
  - 调研要点：这些产品的架构选择和trade-off，为自己的决策提供参考

### 0.3 技术路线预判
- [x] **决策：数据类型支持** → **INT8 + INT16，可配置参数化**
  - INT8：主力推理精度，per-channel量化，支持非对称(scale+zero point)
  - INT16：精度敏感场景，per-channel量化，对称(纯移位requantize，无zp)
  - 不支持FP16/BF16：浮点MAC面积大，MCU场景INT量化已够用
  - 累加器位宽可配置（32~48-bit），推荐40-bit（支持1×1 Conv C_in≤512）
  - MAC单元设计：INT8模式下可拆分为2个INT8乘法或1个INT16乘法（复用逻辑）
  - 后处理流水线：acc → +bias → ×M(15-bit) → >>S → (+zp, INT8 only) → clamp
  - V1.0 INT16 requantize：纯移位 `(acc * M) >> S`
  - 实测精度：INT16定点requantize cosine=0.999999（vs FP32），INT8定点cosine=0.927

- [x] **决策：计算架构风格** → **脉动阵列 (Systolic Array) + Depthwise Conv 专用模块**
  - 主阵列：16×16 Weight Stationary INT8 脉动阵列
  - DW模块：16通道并行，专用处理Depthwise Conv（避免脉动阵列利用率暴跌）
  - 后处理流水线：ReLU + Pool + BN融合 + 量化
  - 理由：面积效率最高，设计2-4人月可完成核心，编译器简单（im2col+tiling）
  - 开源参考：Gemmini（Chisel）、tinyTPU（Verilog）
  - 详见：compute-architecture-comparison.md

---

## 阶段 1：架构定义

> 进入条件：阶段0所有决策项完成

### 1.1 整体架构设计
- [x] 顶层架构框图 → 详见 [architecture-spec.md](architecture-spec.md)
- [x] 模块划分与接口定义 → 7个核心模块，接口已定义
- [x] 数据通路设计 → Ping-Pong双缓冲 + DMA流水线
- [x] **决策：数据格式 (Tensor Layout)**
  - 外部存储：NCHW（兼容ONNX/PyTorch/ncnn）
  - NPU内部计算：NHWC（脉动阵列WS模式效率最高）
  - NCHW→NHWC转换：DMA地址生成器跨步模式（~100-200 LUT），仅第一层需要
  - 后续层输出已是NHWC，直接给下一层使用
- [x] **决策：图像预处理方案**
  - V1.0：CPU软算预处理（YUV→RGB、Resize、归一化）
  - V1.1：独立IPU IP（Image Preprocessing Unit，~2500-4000 LUT）
  - IPU功能：CSC色彩转换 + 仿射变换(含Resize/Crop/旋转) + 归一化
  - IPU与NPU完全独立，通过SRAM中转，作为可选模块
  - NPU内部的Resize算子保留（特征图upsample，用于FPN/U-Net层间操作）

### 1.2 指令集架构（ISA）设计
- [x] **决策：指令集风格** → **层级寄存器配置 + 可编程后处理单元**
  - 控制方式：层级寄存器配置（一组寄存器描述一整层参数：输入尺寸、核大小、步长、padding等）
  - 后处理：可编程后处理流水线（~500-2000 LUT），5-10条固定操作
  - 支持256-entry LUT非线性激活（1 cycle查表，覆盖ReLU/Sigmoid/Swish/GELU等任意函数）
  - 不是CPU核：无取指/译码/分支预测，纯数据流管道（shift→multiply→add→compare→clamp→LUT）
  - 灵活性：90%+新算子不需要改硬件，只需调整寄存器配置和后处理LUT内容
  - 不支持的操作（需CPU介入）：排序、动态分支、变长循环（如NMS）

### 1.3 存储架构
- [x] **决策：片上存储层次** → **分布式Scratchpad**
  - 权重Scratchpad: 32-64KB（装一层tiled权重）
  - 激活Scratchpad: 32-64KB（ping-pong双缓冲，装输入/输出tile）
  - DW模块私有小buffer
  - 总片上SRAM: 64-128KB
  - DMA负责外部SRAM ↔ 本地scratchpad显式搬运
  - 理由：带宽无冲突（各模块独立端口），MAC利用率>95%，行为可预测，复杂度低
  - 全局共享SRAM在脉动阵列满载+DMA并行时有23-44%的bank冲突导致stall，不适合本设计
  - 参考：Gemmini也采用分布式scratchpad方案

### 1.4 总线与外部接口
- [x] **决策：总线协议** → **Wishbone（LiteX原生），商业化时加AXI Wrapper**
  - 开发阶段：Wishbone直连VexRiscv/LiteX，零桥接零开销
  - NPU DMA使用Wishbone B4 pipeline模式，支持连续传输
  - 商业化阶段：加Wishbone↔AXI桥（LiteX现成，~200行），对外暴露AXI4接口
  - TileLink排除：VexRiscv/LiteX生态不使用
  - 理由：与Demo SoC原生兼容，开发最快，未来AXI包装成本极低

---

## 阶段 2：软硬件并行开发

> 进入条件：架构定义文档完成并评审

### 2.1 C功能仿真器
- [ ] 实现NPU的纯C行为模型
- [ ] 支持从模型转换工具输出的指令序列
- [ ] 作为golden model用于RTL验证

### 2.2 RTL实现
- [ ] 核心计算单元
- [ ] 控制器/调度器
- [ ] 片上存储与DMA
- [ ] 顶层集成

### 2.3 模型转换工具（最小版本）
- [x] **决策：技术路线** → **方案G：ncnn前端 + 自研NPU后端**
  - 前端：复用ncnn工具链（onnx2ncnn/pnnx → ncnnoptimize → ncnn2int8）
  - 图优化：ncnn已完成（Conv+BN融合、Conv+ReLU融合、Dropout消除、常量折叠）
  - INT8量化：ncnn2int8 + KL散度校准，直接输出INT8模型
  - 自研后端（~1000行Python）：解析ncnn param → tiling切分 → 权重重排 → CSR配置生成 → DMA描述符生成
  - 输出：program.bin（CSR配置序列+DMA地址表）+ weights.bin（重排后的INT8权重）
  - 理由：ncnn前端成熟（百万级用户验证），V1.0只需专注NPU特有的tiling+codegen
  - 未来可迁移：V2.0算子扩展后可切换到TVM BYOC或MLIR，后端逻辑可复用
- [ ] 支持ONNX输入（经ncnn转换）
- [ ] 基础图优化（ncnn已覆盖）
- [ ] NPU指令生成（自研后端）

### 2.4 验证
- [ ] 单元测试
- [ ] C仿真器 vs RTL对比验证
- [ ] 基础回归测试集

---

## 阶段 3：首个端到端验证

> 里程碑：在仿真器上跑通一个完整模型

### 3.1 选择目标模型
- [ ] **决策：第一个demo模型**
  - MobileNet-v2（轻量，算子少）
  - ResNet-18（经典，算子规整）
  - 小型自定义模型（几层conv+pool+fc）
  - 调研要点：先跑通比性能重要

### 3.2 端到端打通
- [ ] 模型转换 → 指令序列 → C仿真器执行 → 输出正确

---

## 阶段 4：FPGA原型

### 4.1 FPGA平台
- [ ] **决策：目标FPGA平台**
  - Xilinx系列（生态好，工具成熟）
  - Intel/Altera
  - 国产FPGA
  - 调研要点：考虑成本、资源量、工具链

### 4.2 FPGA验证
- [ ] 综合与时序收敛
- [ ] 板级调试
- [ ] 实际模型推理验证

---

## 阶段 5：驱动与运行时

### 5.1 驱动开发
- [ ] **决策：初期驱动形式**
  - bare-metal驱动（FPGA上直接跑）
  - Linux内核驱动
  - 先bare-metal后Linux
- [ ] 寄存器访问层
- [ ] DMA管理
- [ ] 中断处理

### 5.2 运行时
- [ ] 任务提交接口
- [ ] 内存管理
- [ ] 基础调度

---

## 阶段 6：完善与扩展

### 6.1 扩展算子支持
### 6.2 性能优化
### 6.3 IPU（图像预处理单元）实现 — V1.1
- [ ] IPU寄存器定义与接口规格
- [ ] 仿射变换引擎（覆盖Resize/Crop/旋转/透视矫正）
- [ ] 色彩空间转换（YUV422→RGB, RGB565→RGB888）
- [ ] 归一化（减mean/除scale/量化到INT8）
- [ ] 通道重排（HWC↔CHW）
- [ ] Wishbone总线集成
### 6.4 INT16 scale模式可配置 — V1.2
- [ ] INT16 requantize支持scale+zp模式（非对称量化），作为可配置选项
- [ ] 后处理流水线增加mode位：mode=0纯移位(V1.0)，mode=1带scale+zp
- [ ] 量化工具链支持非对称INT16校准
- [ ] 理由：部分模型ReLU后激活最小值>0，非对称量化可提升有效精度
### 6.5 更多应用示例
### 6.6 文档完善
### 6.7 社区建设

---

## 阶段 7：商业化准备（如适用）

### 7.1 IP加固
- [ ] RTL lint清理、CDC/RDC检查
- [ ] DFT设计
- [ ] 低功耗设计
- [ ] 安全加固（ECC等）

### 7.2 验证体系升级
- [ ] UVM环境搭建
- [ ] 覆盖率驱动验证（>95%）
- [ ] 形式验证
- [ ] 门级仿真

### 7.3 软件栈产品化
- [ ] 编译器完善（全算子覆盖、量化工具）
- [ ] 驱动产品化（多OS、稳定性）
- [ ] SDK封装（API + 文档 + 示例）

### 7.4 物理设计与流片
- [ ] 工艺选择、综合、布局布线
- [ ] 物理验证、signoff
- [ ] 流片、bring-up

### 7.5 法务与合规
- [ ] IP审计、license检查
- [ ] 专利布局
- [ ] 出口管制合规

---

## 决策记录

> 每次做出决策后，在这里记录结论和理由。

| # | 决策项 | 结论 | 理由 | 日期 |
|---|--------|------|------|------|
| 1 | 目标场景 | 端侧推理，低算力MCU挂载NPU IP | 面向IoT/嵌入式市场，做IP而非完整芯片 | 2025-05-17 |
| 2 | 性能目标 | 0.2 TOPS (200 GOPS) | TinyML甜蜜点，对标Ethos-U55中档 | 2025-05-17 |
| 3 | CPU Demo | 需要，选RISC-V (VexRiscv) | Cortex-M3许可证不允许开源再分发 | 2025-05-17 |
| 4 | 工作负载 | CNN为主，V2.0加轻量Transformer | Transformer额外35-75%面积，U55同级也不支持 | 2025-05-17 |
| 5 | 商业化 | 开源但保留商业化可能 | Apache 2.0许可，代码按可综合标准写 | 2025-05-17 |
| 6 | 语音支持 | V1.0支持KWS+命令识别，连续ASR为stretch goal，TTS不支持 | KWS仅占0.5%算力；TTS需attention/RNN，纯CNN无法做 | 2025-05-17 |
| 7 | 数据类型 | INT8 + INT16 | INT8主力推理，INT16用于精度敏感/中间累加；不做浮点 | 2025-05-18 |
| 8 | 计算架构 | 脉动阵列16×16 WS + DW Conv专用模块 | 面积效率最高，设计简单，FPGA友好，DW模块解决利用率问题 | 2025-05-18 |
| 9 | ISA风格 | 层级寄存器配置 + 可编程后处理单元 | 编程简单(一组寄存器=一层)，后处理流水线(非CPU)灵活覆盖激活函数，90%+算子无需改硬件 | 2025-05-18 |
| 10 | 存储架构 | 分布式Scratchpad（64-128KB总量） | 带宽无冲突，MAC利用率>95%，ping-pong隐藏搬运延迟，全局共享SRAM有23-44% bank冲突 | 2025-05-18 |
| 11 | 总线协议 | Wishbone原生，商业化加AXI Wrapper | 与VexRiscv/LiteX直连零开销，B4 pipeline模式满足DMA带宽，AXI包装留给商业化阶段 | 2025-05-18 |
| 12 | 模型转换工具 | ncnn前端 + 自研NPU后端 | ncnn图优化/量化成熟(百万级用户)，自研后端仅~1000行专注tiling+codegen，V2.0可迁移TVM/MLIR | 2025-05-18 |
| 13 | 数据格式 | 外部NCHW + 内部NHWC，DMA跨步转换 | NCHW兼容ONNX/PyTorch，NHWC脉动阵列效率最高，仅第一层需转换(~100-200 LUT) | 2025-05-19 |
| 14 | 图像预处理 | V1.0 CPU软算，V1.1独立IPU IP | 预处理非瓶颈(3-5ms vs 推理0.5-5ms)，IPU含仿射变换/CSC/归一化(~2500-4000 LUT)，可选模块 | 2025-05-19 |
| 15 | 量化策略 | INT8:per-ch+scale+zp, INT16:per-ch+移位(对称) | INT16精度已足够(cos=0.999999)无需zp；INT8加zp利用全[0,255]范围有效精度翻倍；定点requantize(M*acc>>S)精度无损 | 2025-05-19 |
| 16 | 参数化设计 | 累加器/数据位宽/阵列大小可配置 | 一份RTL按parameter生成不同SKU(Tiny/Base/Pro/Pro+/Max)，面积从0.08到0.40mm² | 2025-05-19 |

---

## 调研笔记

> 每次调研的结果记录在这里，供后续决策参考。

- [市场调研报告](market-research.md) — MCU NPU市场前景、竞品、应用场景分析
- [CPU Demo SoC调研](cpu-demo-soc-research.md) — Cortex-M3 vs RISC-V选型，Demo SoC架构方案
- [CNN vs Transformer硬件复杂度](cnn-vs-transformer-hardware-analysis.md) — 支持Transformer的额外面积开销分析
- [ASR/TTS可行性研究](asr-tts-feasibility-research.md) — 语音识别和语音合成在0.2T CNN-only NPU上的可行性
- [CNN vs Transformer硬件复杂度分析](cnn-vs-transformer-hardware-analysis.md) — 工作负载决策支持，FPGA LUT定量对比
- [整体架构规格](architecture-spec.md) — 顶层架构框图、模块清单、数据流、资源估算
- [NPU寄存器规格](npu-register-spec.md) — CSR寄存器位域定义、地址映射、使用示例
