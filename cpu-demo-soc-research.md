# CPU Demo SoC 调研报告

## 目标
为 Open-NPU（~0.2 TOPS，MCU级）项目评估是否需要提供 CPU Demo SoC，以及 CPU 核选型（Cortex-M3 vs RISC-V）。

---

## 1. 开源加速器 IP 项目如何处理 CPU 集成

| 项目 | CPU 集成方式 | CPU 类型 | 总线接口 |
|------|-------------|---------|---------|
| **NVDLA** | 提供虚拟平台（QEMU + SystemC），使用 ARMv8 CPU 模拟；也有社区将其集成到 RISC-V SoC（RocketChip/Chipyard） | ARM（官方）/ RISC-V（社区） | CSB（配置总线，类AXI-Lite）+ DBBIF（数据总线，AXI） |
| **Gemmini** | 作为 RoCC 协处理器直接集成到 Chipyard/RocketChip SoC | RISC-V (Rocket/BOOM) | RoCC 接口 + TileLink 系统总线 |
| **VTA (TVM)** | 部署在 FPGA（Pynq/Zynq）上，依赖 ARM 硬核 PS 端或 RISC-V 软核 | ARM (Zynq PS) / RISC-V | AXI |
| **CFU-Playground** | 基于 LiteX SoC 框架 + VexRiscv RISC-V 软核，加速器作为 CFU 挂载 | RISC-V (VexRiscv) | Wishbone |

### 结论
**几乎所有开源加速器 IP 项目都提供某种形式的 CPU 集成示例**。原因：
- 用户需要一个可运行的参考系统来验证加速器功能
- 没有 CPU 的加速器 IP 很难被评估和采用
- Demo SoC 大幅降低用户的集成门槛

**建议：Open-NPU 应提供一个轻量级 Demo SoC。**

---

## 2. Cortex-M3 方案分析

### 2.1 DesignStart 计划

ARM 通过 DesignStart 计划提供 Cortex-M3 的可综合 RTL：
- **DesignStart Eval**：免费下载，固定配置的 Cortex-M3（不可修改微架构），包含 SSE-050 子系统
- **DesignStart Pro**：需要签署协议，提供完全可配置的 Cortex-M3 RTL
- **DesignStart Pro Academic**：面向学术机构，click-through 许可，可流片最多 1000 片

### 2.2 许可证限制（关键问题）

| 条目 | 状态 |
|------|------|
| RTL 是否开源 | **否**，RTL 是加密/混淆的 Verilog，受 NDA 保护 |
| 能否在 GitHub 公开 RTL | **不能**，违反许可协议 |
| 能否在开源项目中再分发 | **不能**，用户必须自行从 ARM 获取 |
| 商业流片是否免费 | Eval 版免预付费；Pro 版需按 Arm Flexible Access 协议，流片时付费 |
| 能否修改 RTL | Eval 版不可修改内核配置；Pro 版可配置 |

### 2.3 生态优势
- **工具链成熟**：Keil MDK、IAR、GCC (arm-none-eabi)
- **RTOS 支持广泛**：FreeRTOS、RT-Thread、Zephyr、Mbed OS 均原生支持
- **CMSIS 标准**：统一的外设访问层、DSP 库、NN 库（CMSIS-NN）
- **社区巨大**：大量教程、参考设计、IP 核生态
- **调试工具**：J-Link、ULINK、DAP-Link 等

### 2.4 总线接口
- Cortex-M3 原生使用 **AHB-Lite**（指令/数据总线）
- 外设总线通过 AHB-to-APB bridge 连接 **APB** 外设
- 可通过 bridge 连接 AXI 系统

### 2.5 核心问题
**Cortex-M3 RTL 不能在开源项目中再分发**。如果选择 Cortex-M3 作为 Demo CPU：
- 项目仓库不能包含 CPU RTL
- 用户必须自行注册 ARM 账号下载 DesignStart 包
- 集成脚本需引导用户手动放置 ARM IP
- 项目的"开源"属性会打折扣

---

## 3. RISC-V 方案分析

### 3.1 候选核心对比

| 核心 | 许可证 | ISA | 流水线 | Artix-7 LUT | 特点 | 成熟度 |
|------|--------|-----|--------|-------------|------|--------|
| **PicoRV32** | ISC (MIT-like) | RV32IMC | 无流水线(多周期) | ~750-1500 | 极小面积，单总线 | 高，GitHub 2.8K stars |
| **VexRiscv** | MIT | RV32IMAC | 5级流水线 | 504-1935 (可配置) | 高度可配置，支持 CFU | 高，LiteX 默认核，活跃社区 |
| **SERV** | ISC | RV32I(M) | 串行(bit-serial) | ~200-400 | 最小面积，1-bit 串行执行 | 中，极端面积优化 |
| **Ibex** | Apache 2.0 | RV32IMC | 2级流水线 | ~2000-3000 | 工业级质量，安全特性 | 高，lowRISC/OpenTitan 验证 |
| **CV32E40P** | Solderpad 0.51 | RV32IMFC | 4级流水线 | ~3000-4000 | 高性能，可选 FPU | 高，OpenHW Group 验证 |

### 3.2 VexRiscv 详细面积数据（Artix-7）

| 配置 | DMIPS/MHz | LUT | FF | Fmax |
|------|-----------|-----|-----|------|
| Small (RV32I, 无bypass) | 0.52 | 504 | 505 | 243 MHz |
| Small+Productive (RV32I) | 0.82 | 816 | 534 | 232 MHz |
| Full (RV32IM, 4KB I$/D$) | 1.21 | 1840 | 1158 | 199 MHz |
| Full MaxPerf (RV32IM, 8KB I$/D$) | 1.38 | 1935 | 1216 | 200 MHz |

### 3.3 许可证兼容性

| 核心 | 许可证 | 可否开源再分发 | 商业使用 |
|------|--------|---------------|---------|
| PicoRV32 | ISC | ✅ 完全自由 | ✅ 无限制 |
| VexRiscv | MIT | ✅ 完全自由 | ✅ 无限制 |
| SERV | ISC | ✅ 完全自由 | ✅ 无限制 |
| Ibex | Apache 2.0 | ✅ 自由（需保留声明） | ✅ 有专利授权条款 |
| CV32E40P | Solderpad 0.51 | ✅ 自由 | ✅ 类似 Apache 2.0 |

**所有候选 RISC-V 核心均可自由在开源项目中使用和再分发。**

### 3.4 生态成熟度

| 维度 | 状态 |
|------|------|
| **GCC 工具链** | 成熟，riscv32-unknown-elf-gcc 稳定可用 |
| **LLVM/Clang** | 完善支持 |
| **RTOS** | FreeRTOS、Zephyr、RT-Thread 均支持 RISC-V |
| **调试** | OpenOCD + GDB，JTAG 调试链完整 |
| **仿真** | Verilator、Spike、QEMU 均支持 |
| **SoC 框架** | LiteX 提供完整的 SoC 生成能力 |
| **IDE** | VS Code + PlatformIO、Eclipse、自定义 Makefile |

### 3.5 总线接口

| 核心 | 原生总线 | 可桥接到 |
|------|---------|---------|
| PicoRV32 | Native mem interface (简单valid/ready) | Wishbone, AXI-Lite |
| VexRiscv | Wishbone (分离 I/D) | AXI, AHB (通过 LiteX bridge) |
| SERV | Wishbone (分离 I/D) | AXI-Lite |
| Ibex | OBI (Open Bus Interface) | Wishbone, AXI, AHB |
| CV32E40P | OBI | Wishbone, AXI, AHB |

---

## 4. 推荐方案

### 4.1 对于开源项目：**强烈推荐 RISC-V**

| 考量因素 | Cortex-M3 | RISC-V | 结论 |
|---------|-----------|--------|------|
| 许可证自由度 | ❌ 不能再分发 RTL | ✅ 完全开源 | RISC-V 胜 |
| 项目完整性 | ❌ 用户需额外获取 ARM IP | ✅ 一键克隆即可使用 | RISC-V 胜 |
| 社区接受度 | ⚠️ 开源社区对封闭 IP 有抵触 | ✅ 开源社区拥抱 | RISC-V 胜 |
| 生态成熟度 | ✅ 极成熟 | ✅ 已足够成熟 | 平手 |
| 面积效率 | ✅ Cortex-M3 面积优化好 | ✅ 多种选择可匹配 | 平手 |

### 4.2 推荐核心选型

**首选：VexRiscv（MIT 许可）**
- 理由：
  - 高度可配置，可根据 NPU demo 需求选择最合适的配置
  - LiteX SoC 生态完善，快速搭建完整 demo 系统
  - CFU-Playground（Google）已验证此方案用于 ML 加速器集成
  - MIT 许可证完全无限制
  - 面积从 504 LUT（最小）到 1935 LUT（全功能），灵活可调
  - 活跃的社区和维护

**备选：Ibex（Apache 2.0）**
- 理由：
  - 工业级质量，经 OpenTitan 项目充分验证
  - 面积适中（RV32IMC，2级流水线）
  - 适合未来如需走商业化路线时做为参考
  - 缺点：SystemVerilog 编写，工具链要求较高

### 4.3 商业化考量

| 方案 | 商业化风险 |
|------|-----------|
| VexRiscv (MIT) | 无任何风险，可自由商用 |
| Ibex (Apache 2.0) | 极低风险，需保留版权声明，有专利互惠条款 |
| PicoRV32 (ISC) | 无任何风险 |
| Cortex-M3 | 需要与 ARM 签署商业许可，按流片量付费 |

### 4.4 建议的 Demo SoC 架构

```
┌─────────────────────────────────────────┐
│              Demo SoC                    │
│                                         │
│  ┌──────────┐    Wishbone    ┌────────┐ │
│  │ VexRiscv │◄──────────────►│Open-NPU│ │
│  │ (RV32IM) │    Bus         │  Core  │ │
│  └──────────┘                └────────┘ │
│       │                          │      │
│       ▼                          ▼      │
│  ┌──────────┐              ┌─────────┐  │
│  │  SRAM    │              │  SRAM   │  │
│  │(Instr/Data)│            │(Weight) │  │
│  └──────────┘              └─────────┘  │
│       │                                 │
│       ▼                                 │
│  ┌──────────┐  ┌──────┐  ┌──────────┐  │
│  │   UART   │  │ GPIO │  │  SPI     │  │
│  └──────────┘  └──────┘  └──────────┘  │
└─────────────────────────────────────────┘
```

基于 LiteX 框架，可快速生成完整的可综合 SoC。

### 4.5 NPU 总线接口建议

对于 Open-NPU 加速器自身的总线接口：
- **配置接口（寄存器访问）**：APB 或 AXI-Lite（简单，面积小）
- **数据接口（权重/激活值搬运）**：AXI4 或 简单 Wishbone（取决于带宽需求）
- 如果选择 LiteX + VexRiscv 方案，**Wishbone** 是最自然的选择（或内部用简单接口 + 提供 Wishbone wrapper）

---

## 5. 总结

| 问题 | 答案 |
|------|------|
| 需要 Demo SoC 吗？ | **需要**，所有成功的开源加速器项目都提供 |
| 选 Cortex-M3 还是 RISC-V？ | **RISC-V**，许可证是决定性因素 |
| 具体选哪个核？ | **VexRiscv**（首选）或 Ibex（备选） |
| 用什么 SoC 框架？ | **LiteX**（Python 生态，快速迭代） |
| NPU 总线接口？ | Wishbone 或 AXI-Lite（配置）+ Wishbone/AXI（数据） |
