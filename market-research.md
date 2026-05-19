# MCU NPU 加速器市场调研报告

> 调研时间：2025年5月 | 目标：~0.2 TOPS NPU IP，面向MCU的AI加速器

---

## 一、市场需求分析

### 驱动力

MCU集成NPU已成为明确的行业趋势，核心驱动力包括：

1. **tinyML爆发**：在MCU上运行神经网络推理（关键词检测、异常检测、图像分类等）需求激增
2. **隐私与实时性**：数据不出设备，本地推理延迟更低
3. **功耗约束**：电池供电设备要求毫瓦级AI推理，纯CPU软件方案能效比不足
4. **成本下沉**：AI从高端芯片下沉到1美元级MCU（TI已在Cortex-M0+上集成NPU）

### MCU厂商动态

**所有主流MCU厂商都在积极集成NPU：**

| 厂商 | 状态 |
|------|------|
| STMicroelectronics | 已发布STM32N6（自研Neural-ART NPU） |
| NXP | 已发布i.MX RT700 + MCX N94（eIQ Neutron NPU） |
| Texas Instruments | 2026年发布MSPM0G5187（TinyEngine NPU，<1美元级） |
| Renesas | DRP-AI集成在RA8系列 |
| Infineon | 采用Arm Ethos-U55 |
| 国芯科技 | CCR4001S 自研NPU RISC-V MCU |

**结论：市场需求真实且强劲，MCU+NPU已不是"是否需要"的问题，而是"如何实现"的问题。**

---

## 二、主要商业产品

### 2.1 STMicroelectronics — STM32N6

| 指标 | 规格 |
|------|------|
| NPU名称 | ST Neural-ART Accelerator（自研） |
| 算力 | **600 GOPS (0.6 TOPS)** |
| NPU频率 | 1 GHz |
| CPU | Arm Cortex-M55 @ 800MHz |
| 内存 | 4.2MB SRAM |
| ISP | 支持5MP@30fps |
| 发布时间 | 2024年12月 |
| 应用 | 计算机视觉、音频处理 |

### 2.2 NXP — eIQ Neutron NPU

| 指标 | 规格 |
|------|------|
| 搭载芯片 | i.MX RT700, MCX N94x/N54x |
| CPU | Cortex-M33 双核 @ 150MHz（MCX），Cortex-M33 @ 325MHz（RT700） |
| NPU提升 | AI推理速度提升172倍（vs 纯CPU） |
| 内存 | 7.5MB SRAM（RT700） |
| 发布时间 | 2024年9月 |
| 特点 | 自研NPU架构，专注边缘AI |

### 2.3 Texas Instruments — TinyEngine NPU

| 指标 | 规格 |
|------|------|
| 搭载芯片 | MSPM0G5187, AM13Ex |
| 算力 | **2.56 GOPS** |
| CPU | Cortex-M0+ @ 80MHz |
| Flash/SRAM | 128KB / 32KB |
| 精度 | 8/4/2位混合精度 |
| 价格 | **<1美元**（1K起订） |
| 能效 | vs无NPU方案能耗降至1/120 |
| 发布时间 | 2026年3月 |
| 意义 | **业界首次在M0+级MCU集成NPU** |

### 2.4 Arm Ethos-U系列

| 型号 | Ethos-U55 | Ethos-U65 | Ethos-U85 |
|------|-----------|-----------|-----------|
| 算力 | 32-512 GOPS | ~1 TOPS | 256 GOPS - 4 TOPS |
| 目标 | Cortex-M55/M85 MCU | Cortex-M/A混合 | 下一代高端 |
| 授权伙伴 | 20+ | 多家 | Alif, Himax, Nuvoton, Infineon |
| 特点 | 最成熟的MCU NPU IP | 支持Linux | 4x U65性能 |

### 2.5 国产MCU厂商

- **国芯科技**：CCR4001S，基于自主RISC-V架构+自研NPU，面向智能家居/安防
- **芯来科技**：推出NACC Micro-NPU IP（2025年7月），与RISC-V CPU紧耦合，1-32核可配置，最高4 TOPS
- **Espressif**：ESP32-S3内置向量指令加速（非独立NPU），ESP32-P4增强AI能力

### 2.6 其他

- **Syntiant**：NDP200/NDP250 专用AI芯片（非MCU内集成），NDP200 6.4 GOPS @ 1mW，专注always-on语音/传感
- **Himax**：采用Ethos-U55，面向视觉AI

---

## 三、市场规模

### TinyML市场

| 来源 | 2025年规模 | 预测 | CAGR |
|------|-----------|------|------|
| NMSC | $17.6亿 | 2026年$24.9亿 | — |
| DataM Intelligence | $15.3亿 | 2035年$96.5亿 | 20.2% |
| Technavio | — | 2025-2030增长$80.6亿 | 35.7% |

### RISC-V AI SoC市场

- 预计到2030年，基于RISC-V的AI SoC市场超过**420亿美元**，CAGR 49.2%（芯来科技引用数据）

### 关键判断

- TinyML市场2025年已达15-18亿美元规模
- 年增长率20-36%
- 到2030年将达80-100亿美元级别
- **这是一个真实的、快速增长的市场**

---

## 四、NPU IP授权竞争格局

### 4.1 主要NPU IP供应商

| 供应商 | 产品 | 算力范围 | 定位 | 授权模式 |
|--------|------|---------|------|---------|
| **Arm** | Ethos-U55/U65/U85 | 32 GOPS - 4 TOPS | MCU/边缘 | 商业授权 |
| **Ceva** | NeuPro-Nano | 10-200 GOPS | TinyML | 商业授权 |
| **Cadence** | Tensilica Neo | 8 GOPS - 80 TOPS | 全范围 | 商业授权 |
| **Synopsys** | ARC NPX6 | 1K-96K MACs | 高性能 | 商业授权 |
| **芯原(VeriSilicon)** | VIP9000 | 0.5-20 TOPS | IoT/汽车 | 商业授权 |
| **安谋科技** | 周易Z/X2 | 到320 TOPS | 中高端 | 商业授权 |
| **英业达** | Minima | <32 GOPS | MCU专用 | 商业授权 |
| **芯来科技** | NACC | 到4 TOPS | RISC-V MCU | 商业授权 |

### 4.2 开源NPU IP

| 项目 | 架构 | 状态 | 特点 |
|------|------|------|------|
| **Google Coral NPU** | RISC-V 32位 | 2025年开源 | 芯原合作商业化版本，超低功耗，面向可穿戴 |
| **open-npu (popovych-labs)** | — | GitHub开源 | 模块化设计 |

### 4.3 开源替代空间分析

**有明确空间，理由如下：**

1. **商业IP授权费高昂**：Arm Ethos需要Arm架构授权+NPU授权双重费用，中小芯片公司负担重
2. **RISC-V生态崛起**：RISC-V MCU厂商需要配套的NPU IP，但选择有限
3. **Google验证了方向**：Coral NPU开源项目证明了开源NPU IP的可行性和市场需求
4. **中国市场需求**：国产替代背景下，开源NPU IP有独特价值
5. **TI定位参考**：TI的TinyEngine证明2.56 GOPS级NPU在1美元MCU上已有商业价值
6. **空白地带**：目前开源方案集中在超低功耗（Coral NPU微瓦级），~0.2 TOPS级别的开源NPU IP尚无成熟方案

**竞争风险：**
- Arm Ethos生态极为成熟（20+合作伙伴），工具链完善
- Ceva NeuPro-Nano直接针对TinyML市场
- 芯来NACC已在RISC-V赛道占位
- Google Coral NPU有巨头背书

---

## 五、关键应用场景

### 按优先级排列

| 优先级 | 场景 | 典型模型 | 算力需求 | 市场成熟度 |
|--------|------|---------|---------|-----------|
| 1 | **语音唤醒词/KWS** | 小型CNN/DS-CNN | 10-100 GOPS | 已成熟 |
| 2 | **异常检测/预测性维护** | 自编码器/1D-CNN | 1-50 GOPS | 快速增长 |
| 3 | **手势/活动识别** | CNN/LSTM | 10-100 GOPS | 增长中 |
| 4 | **图像分类（小型）** | MobileNet/MCUNet | 50-500 GOPS | 增长中 |
| 5 | **人脸检测/识别** | 轻量CNN | 100-600 GOPS | 需求强 |
| 6 | **心率/生物信号分析** | 1D-CNN/RNN | 1-30 GOPS | 新兴 |
| 7 | **环境声音分类** | CNN | 10-100 GOPS | 新兴 |

### 0.2 TOPS定位分析

**0.2 TOPS (200 GOPS) 可覆盖的场景：**
- 语音唤醒词 ✓
- 异常检测/预测性维护 ✓
- 手势/活动识别 ✓
- 小型图像分类（CIFAR-10级别）✓
- 环境声音分类 ✓
- 简单人脸检测（可能需要优化）△

**这个算力定位合理，对标Arm Ethos-U55的中档配置（128-256 GOPS），属于TinyML的甜蜜点。**

---

## 六、总结与建议

### 市场判断

1. **需求确认**：MCU+NPU是明确趋势，所有主流厂商已入场
2. **时机合适**：市场正处于爆发早期（2025-2030 CAGR 20-36%）
3. **开源有空间**：Google Coral NPU验证了方向，但0.2 TOPS级别的开源方案尚未出现

### 产品定位建议

- **算力**：0.2 TOPS (200 GOPS) 是合理的甜蜜点
- **精度**：必须支持INT8，可选INT4/INT16
- **架构**：面向RISC-V MCU生态（差异化）
- **功耗**：目标 < 50mW（对标Ceva NeuPro-Nano同档）
- **面积**：尽量小，适配低成本MCU
- **竞争对手**：主要是Arm Ethos-U55（商业）、Ceva NeuPro-Nano（商业）、Google Coral NPU（开源但定位更低）

### 差异化机会

1. 开源 + RISC-V 生态绑定
2. 完整工具链（编译器、量化工具、模型zoo）
3. 面向中国RISC-V MCU厂商生态
4. 比Coral NPU更高算力（0.2 TOPS vs 微瓦级）
5. 比商业IP更低成本的集成方案
