# Open-NPU Design Specifications

[中文版](README_CN.md)

Hardware architecture and interface specifications for the Open-NPU neural network accelerator.

## Documents

| Document | Description |
|----------|-------------|
| [QUICK_START.md](QUICK_START.md) | **Getting started guide** — ONNX → quantization → CSIM → RTL verification, end-to-end workflow |
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

## Supported Operators (RTL Verified)

All 8 CNN operator types are implemented in synthesizable Verilog and verified bit-exact against the C simulator golden reference.

| op_type | Operator | Hardware Path | RTL Status |
|---------|----------|---------------|------------|
| 0 | Conv2D | Systolic Array (1×1, 3×3) | Verified |
| 1 | DWConv | DW Conv Module (any kernel) | Verified |
| 2 | FC | Systolic Array (reuses Conv2D) | Verified |
| 3 | Pooling | Pooling FSM (Max / Avg / Global) | Verified |
| 4 | Eltwise Add | Add FSM + dual rescale | Verified |
| 5 | Resize | Resize FSM (nearest / bilinear) | Verified |
| 6 | Deconv | Conv2D path + zero-insertion skip | Verified |
| 7 | Concat | Reuses Add FSM + offset addressing | Verified |

## RTL Verification Status

| Test Suite | Scope | Result |
|------------|-------|--------|
| Unit tests (PE, Systolic, PPU, CSR, SRAM, DMA, DW, Ctrl, Top, Compute) | 90+ tests | All PASS |
| DMA E2E INT8 10-layer MobileNetV2-Tiny | Per-layer bit-exact | PASS |
| DMA E2E INT16 10-layer MobileNetV2-Tiny | Per-layer bit-exact | PASS |
| Spatial Tiling E2E (32×32 input, 6 layers) | All tiling variants | PASS |
| Individual operator E2E (Pooling, Add, Resize, Deconv, Concat) | 15+ tests | All PASS |
| **AllOps-Mini full model** (18 layers, 7 op types, 16×16 input) | **End-to-end bit-exact** | **PASS** |
| **AllOps-128 full model** (18 layers, 22 invocations, 128×128 input) | **E2E bit-exact + DMA tiling** | **PASS** |
| **INT16 full regression** (10 tests, all 7 op types + AllOps-Mini INT16) | **10/10 bit-exact** | **PASS** |
| **Auto-next multi-layer** (3-layer auto inference, single START) | **FSM auto-restart bit-exact** | **PASS** |

## Synthesis Results (Yosys `synth_xilinx`)

Yosys 0.65 Xilinx 7-series synthesis (`synth_xilinx -top npu_top -family xc7`, ARRAY_SIZE=16, SPAD_KB=128):

| Resource | Count |
|----------|-------|
| LUT (total) | 52,804 |
| FF (flip-flops) | 39,145 |
| BRAM36E1 | 26 (104KB SRAM) |
| DSP48E1 | 333 |
| CARRY4 | 8,022 |
| Estimated Logic Cells | 43,726 |

Recommended FPGA targets:

| Device | LUT Utilization | Fit? |
|--------|----------------|------|
| XC7A200T (Artix-7) | 39% | **Best value** |
| XC7K160T (Kintex-7) | 52% | High frequency |
| XC7Z045 (Zynq) | 24% | ARM SoC integrated |

## Related Repositories

- [open-npu/rtl](https://github.com/open-npu/rtl) — Synthesizable Verilog implementation
- [open-npu/csim](https://github.com/open-npu/csim) — C cycle-approximate simulator
- [open-npu/tools](https://github.com/open-npu/tools) — ONNX converter & quantization toolchain

## License

Apache-2.0
