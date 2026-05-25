# Open-NPU Design Specifications

[中文版](README_CN.md)

Hardware architecture and interface specifications for the Open-NPU neural network accelerator.

## Documents

| Document | Description |
|----------|-------------|
| [architecture-spec.md](architecture-spec.md) | Full architecture specification — datapath, memory hierarchy, micro-architecture of each module |
| [interface-spec.md](interface-spec.md) | Port-level interface timing and protocol definitions |
| [npu-register-spec.md](npu-register-spec.md) | CSR register map, bit fields, access types |

## Key Specs

- 16×16 INT8 systolic array (configurable)
- INT8 / INT16 mixed precision support
- Per-channel requantization with INT64 bias accumulation
- 128KB activation SRAM + 64KB weight SRAM + 8KB parameter SRAM
- Wishbone-B4 bus interface
- Target: ~0.2 TOPS @ 200MHz

## Related Repositories

- [open-npu/rtl](https://github.com/open-npu/rtl) — Synthesizable Verilog implementation
- [open-npu/csim](https://github.com/open-npu/csim) — C cycle-approximate simulator
- [open-npu/tools](https://github.com/open-npu/tools) — ONNX converter & quantization toolchain

## License

Apache-2.0
