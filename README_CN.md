# Open-NPU 设计规格

[English](README.md)

Open-NPU 神经网络加速器的硬件架构与接口规格文档。

## 文档列表

| 文档 | 说明 |
|------|------|
| [architecture-spec.md](architecture-spec.md) | 完整架构规格 — 数据通路、存储层次、各模块微架构 |
| [interface-spec.md](interface-spec.md) | 端口级接口时序与协议定义 |
| [npu-register-spec.md](npu-register-spec.md) | CSR 寄存器映射、位域、访问类型 |

## 核心指标

- 16×16 INT8 脉动阵列（可配置）
- INT8 / INT16 混合精度支持
- Per-channel 重量化，INT64 bias 累加
- 128KB 激活 SRAM + 64KB 权重 SRAM + 8KB 参数 SRAM
- Wishbone-B4 总线接口
- 目标算力：~0.2 TOPS @ 200MHz

## 支持的算子（RTL 已验证）

全部 8 种 CNN 算子类型均已在可综合 Verilog 中实现，并通过 C 仿真器 golden 数据逐比特验证。

| op_type | 算子 | 硬件路径 | RTL 状态 |
|---------|------|----------|----------|
| 0 | Conv2D | 脉动阵列 (1×1, 3×3) | 已验证 |
| 1 | DWConv | DW Conv 模块 (任意 kernel) | 已验证 |
| 2 | FC | 脉动阵列 (复用 Conv2D) | 已验证 |
| 3 | Pooling | Pooling FSM (Max / Avg / Global) | 已验证 |
| 4 | Eltwise Add | Add FSM + 双路 rescale | 已验证 |
| 5 | Resize | Resize FSM (最近邻 / 双线性) | 已验证 |
| 6 | Deconv | Conv2D 路径 + 零插入跳过逻辑 | 已验证 |
| 7 | Concat | 复用 Add FSM + 偏移寻址 | 已验证 |

## RTL 验证状态

| 测试集 | 范围 | 结果 |
|--------|------|------|
| 单元测试 (PE, Systolic, PPU, CSR, SRAM, DMA, DW, Ctrl, Top, Compute) | 90+ 测试 | 全部通过 |
| DMA E2E INT8 10层 MobileNetV2-Tiny | 逐层 bit-exact | 通过 |
| DMA E2E INT16 10层 MobileNetV2-Tiny | 逐层 bit-exact | 通过 |
| Spatial Tiling E2E (32×32 输入, 6 层) | 所有 tiling 变体 | 通过 |
| 独立算子 E2E (Pooling, Add, Resize, Deconv, Concat) | 15+ 测试 | 全部通过 |
| **AllOps-Mini 全模型** (18 层, 7 种算子, 16×16 输入) | **端到端 bit-exact** | **通过** |
| **AllOps-128 全模型** (18 层, 22 次调用, 128×128 输入) | **E2E bit-exact + DMA tiling** | **通过** |

## 综合结果 (Yosys Generic)

Yosys 0.65 generic 综合 (`synth -top npu_top`, ARRAY_SIZE=16, SPAD_KB=128)：

| 资源 | 数量 |
|------|------|
| 逻辑门 (AND/OR/XOR/NOT/MUX) | 846,550 |
| 触发器 (DFF) | 38,346 |
| 存储块 | 3 (ACT 32KB + WGT 64KB + PARAM 8KB = 104KB) |
| 综合耗时 | 258s, 峰值内存 4.17 GB |

FPGA 适配估算 (Artix-7 XC7A100T)：FF ~30%, BRAM ~43%, LUT 需进一步优化或使用更大器件。

## 相关仓库

- [open-npu/rtl](https://github.com/open-npu/rtl) — 可综合 Verilog 实现
- [open-npu/csim](https://github.com/open-npu/csim) — C 周期近似模拟器
- [open-npu/tools](https://github.com/open-npu/tools) — ONNX 转换器与量化工具链

## 许可证

Apache-2.0
