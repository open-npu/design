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

## 相关仓库

- [open-npu/rtl](https://github.com/open-npu/rtl) — 可综合 Verilog 实现
- [open-npu/csim](https://github.com/open-npu/csim) — C 周期近似模拟器
- [open-npu/tools](https://github.com/open-npu/tools) — ONNX 转换器与量化工具链

## 许可证

Apache-2.0
