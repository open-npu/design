# Open-NPU 设计规格

[English](README.md)

Open-NPU 神经网络加速器的硬件架构、数据通路、二进制格式与寄存器规格文档。

## 文档列表

| 文档 | 说明 |
|------|------|
| [open_npu_overview.md](open_npu_overview.md) | 整机地图 — ONNX -> CSIM -> SoC RTL 全链路、切块、融合与公开基准模型矩阵（`model_c`、`model_e`、`model_s`） |
| [rtl_design_overview.md](rtl_design_overview.md) | 信号级 RTL 实现总览（`npu_top`、64-lane Row、8-lane PPU、四库 512-bit SRAM） |
| [mac_array_and_datapath.md](mac_array_and_datapath.md) | 64-lane K-major 行引擎、`wrow/abuf/avec` 单写口分库、DW 折叠复用与激活喂入数据通路 |
| [interface-spec.md](interface-spec.md) | 离线 `NPU1` 模型二进制与层描述符权威定义 |
| [npu-register-spec.md](npu-register-spec.md) | 完整 CSR 寄存器手册（`0x000-0x4FF`）、位域定义、性能直方图寄存器与编程示例 |
| [QUICK_START.md](QUICK_START.md) | 从 ONNX 量化转换到 CSIM、RTL cocotb 与 SoC 仿真的端到端快速上手指南 |

## 核心硬件指标

- **计算核心**：统一 64-lane K-major MAC 行引擎（`64 MACs/cycle`），统一承载 Conv2D、FC 与 Depthwise Conv2D（DW 复用 `wrow` 与 `racc`）
- **后处理通路**：8-lane 分时复用 PPU（`PPU_LANES=8`，4 级流水，INT64 bias + 逐通道乘移截断）+ 8-lane 流水双分支 Elementwise Add
- **数据精度**：有符号 INT8 / INT16 激活与权重，44-bit 有符号累加器
- **片上存储**：172 KB 独立四库 Scratchpad（`32 KB IFM + 48 KB OFM + 80 KB WGT + 12 KB PARAM`，计算侧 512-bit 宽口，DMA 侧 32-bit 端口）
- **总线接口**：Wishbone-B4 32-bit CSR 从接口 + Wishbone-B4 32-bit 流水 DMA 主接口
- **峰值算力**：`12.8 GMAC/s`（`25.6 GOPS`）@ `200 MHz`

## 支持的算子（RTL 逐比特验证通过）

| `op_type` | 算子 | 硬件路径 | RTL 状态 |
|-----------|------|----------|----------|
| 0 | Conv2D | 64-lane K-major 行引擎（支持任意卷积核、步长、空洞率与填充） | 已验证 |
| 1 | DWConv | 折叠复用 `wrow_b[0..31]` + `racc[0..7]`（8 通道/组，支持全局 DW 切片流式） | 已验证 |
| 2 | FC | 64-lane K-major 行引擎（复用 1x1 卷积路径） | 已验证 |
| 3 | Pooling | 流式池化状态机（Max / Avg / Global，内置 40 拍串行恢复余数倒数器） | 已验证 |
| 4 | Eltwise Add | 8-lane 流水双路重缩放加法 + PPU 钳位 | 已验证 |
| 5 | Resize | 流式缩放状态机（最近邻 / 双线性） | 已验证 |
| 6 | Reserved | 已裁撤保留（`OP_DECONV` 已停用，硬件守卫拦截） | — |
| 7 | Concat | 通道偏移寻址拼接 + 可选重缩放 | 已验证 |

## RTL 验证与零黑盒综合基线

| 测试集 / 指标 | 范围 | 结果 |
|---------------|------|------|
| 单元 cocotb 测试 | CSR、SRAM、PPU、DMA、Ctrl、Compute | `50/50 通过` |
| 集成 cocotb 测试 | Top、Super-block CSR、Param 预取、Tiling、利用率 | `61/61 通过` |
| 鲁棒性与 E2E cocotb 测试 | WB 协议、Abort/Reset、边界异常、DMA E2E（`model_c/e/s` 及合成算子链） | `45/45 通过` |
| SoC Verilator 整网链式 E2E | 公开代号模型（`model_c`、`model_e`、`model_s` 的 INT8 与 INT16） | 逐层逐 word bit-exact 通过 |
| Yosys 全芯片真综合（`synth -top npu_top`，0 黑盒） | 全部可综合 RTL 模块（`172 KB` SRAM + `102,400`-bit 内部 bank 全部推断为 `$mem_v2`） | **`40,923` cells**（`npu_compute`: `28,973`，`$adff`: `886`，`$mul`: `489`，`$div`: `0`，`$mod`: `0`） |

## 相关仓库

- [open-npu/rtl](https://github.com/open-npu/rtl) — 可综合 Verilog-2005 实现与 cocotb 验证套件
- [open-npu/soc](https://github.com/open-npu/soc) — LiteX + VexRiscv SoC 集成与 Verilator 整机仿真
- [open-npu/csim](https://github.com/open-npu/csim) — 逐比特一致 C 参考仿真器
- [open-npu/tools](https://github.com/open-npu/tools) — ONNX 转换、INT8/INT16 量化、K-major 打包与切块工具链
- [open-npu/driver](https://github.com/open-npu/driver) — Bare-metal 与 FreeRTOS C 驱动 SDK

## 许可证

Apache-2.0
