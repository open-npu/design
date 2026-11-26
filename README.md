# Open-NPU Design Specifications

[中文版](README_CN.md)

Hardware architecture, datapath, binary format, and CSR specifications for the Open-NPU neural network accelerator.

## Documents

| Document | Description |
|----------|-------------|
| [open_npu_overview.md](open_npu_overview.md) | Whole-chip map — ONNX -> CSIM -> SoC RTL, tiling, fusion, and public model matrix (`model_c`, `model_e`) |
| [rtl_design_overview.md](rtl_design_overview.md) | Signal-level RTL implementation guide (`npu_top`, 64-lane Row, 8-lane PPU, 4-bank 512-bit SRAM) |
| [mac_array_and_datapath.md](mac_array_and_datapath.md) | 64-lane K-major Row Engine, `wrow/abuf/avec` banking, DW folding, and activation feed datapath |
| [architecture-spec.md](architecture-spec.md) | System architecture specification — design rationale, precision analysis, tiling, and double buffering |
| [interface-spec.md](interface-spec.md) | Offline `NPU1` binary format and descriptor layout specification |
| [npu-register-spec.md](npu-register-spec.md) | Complete CSR register map (`0x000-0x4FF`), bit fields, performance counters, and programming examples |
| [QUICK_START.md](QUICK_START.md) | End-to-end quick-start guide from ONNX quantization to CSIM, RTL cocotb, and SoC simulation |

## Key Hardware Specifications

- **Compute Core**: Unified 64-lane K-major MAC Row Engine (`64 MACs/cycle`), executing Conv2D, FC, and Depthwise Conv2D (reusing `wrow` and `racc`)
- **Post-Processing**: 8-lane time-multiplexed PPU (`PPU_LANES=8`, 4-stage pipeline, INT64 bias + per-channel multiply-shift-clamp) + 8-lane pipelined Elementwise Add
- **Precision**: Signed INT8 and INT16 activations/weights with 44-bit signed accumulators
- **Scratchpad SRAM**: 172 KB split 4-bank architecture (`32 KB IFM + 48 KB OFM + 80 KB WGT + 12 KB PARAM`, 512-bit compute ports, 32-bit DMA ports)
- **Bus Interface**: Wishbone-B4 32-bit CSR slave + Wishbone-B4 32-bit pipelined DMA master
- **Peak Throughput**: `12.8 GMAC/s` (`25.6 GOPS`) @ `200 MHz`

## Supported Operators (RTL Bit-Exact Verified)

| `op_type` | Operator | Hardware Path | RTL Status |
|-----------|----------|---------------|------------|
| 0 | Conv2D | 64-lane K-major Row Engine (arbitrary kernel, stride, dilation, padding) | Verified |
| 1 | DWConv | Folded onto `wrow_b[0..31]` + `racc[0..7]` (8 channels/group, global-DW streaming) | Verified |
| 2 | FC | 64-lane K-major Row Engine (1x1 spatial reuse) | Verified |
| 3 | Pooling | Streaming Pooling FSM (Max / Avg / Global with 40-cycle restoring-division reciprocal) | Verified |
| 4 | Eltwise Add | 8-lane pipelined dual-rescale Add + PPU clamp | Verified |
| 5 | Resize | Streaming Resize FSM (nearest / bilinear) | Verified |
| 6 | Reserved | Retired (`OP_DECONV` disabled; rejected by hardware guard) | — |
| 7 | Concat | Channel-offset DMA / SRAM concatenation + optional rescale | Verified |

## RTL Verification & Synthesis Baseline

| Suite / Metric | Scope | Result |
|----------------|-------|--------|
| Unit cocotb tests | CSR, SRAM, PPU, DMA, Ctrl, Compute | `50/50 PASS` |
| Integration cocotb tests | Top, Super-block CSR, Param prefetch, Tiling, Util | `61/61 PASS` |
| Robustness & E2E cocotb tests | WB protocol, Abort/Reset, Edge cases, DMA E2E (`model_c/e` + synthetic chains) | `45/45 PASS` |
| SoC Verilator full-network E2E | Public coded models (`model_c`, `model_e` in INT8 & INT16) | Layer-by-layer bit-exact PASS |
| Full-chip Yosys synthesis (`synth -top npu_top`, 0 black boxes) | All 8 active RTL modules (`172 KB` SRAM + `102,400`-bit compute bank RAMs inferred as `$mem_v2`) | **`40,963` cells** (`npu_compute`: `29,010`, `$adff`: `886`, `$mul`: `488`, `$div`: `0`, `$mod`: `0`) |

## Related Repositories

- [open-npu/rtl](https://github.com/open-npu/rtl) — Synthesizable Verilog-2005 RTL & cocotb verification suite
- [open-npu/soc](https://github.com/open-npu/soc) — LiteX + VexRiscv SoC integration & Verilator simulation
- [open-npu/csim](https://github.com/open-npu/csim) — Bit-exact C reference simulator
- [open-npu/tools](https://github.com/open-npu/tools) — ONNX converter, INT8/INT16 PTQ, K-major packer & tiler
- [open-npu/driver](https://github.com/open-npu/driver) — Bare-metal & FreeRTOS C driver SDK

## License

Apache-2.0
