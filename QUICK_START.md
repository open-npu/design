# Open-NPU 快速上手

本文档提供从 ONNX 模型量化打包、CSIM 参考仿真到 RTL cocotb 与 RISC-V SoC 端到端验证的完整流程。

---

## 1. 环境准备

### 系统依赖

| 依赖 | 用途 | 安装命令 |
|------|------|----------|
| Python 3.8+ | 工具链与测试脚本 | `apt install python3` |
| numpy, onnx, onnxruntime, Pillow | 模型转换与校准量化 | `pip install numpy onnx onnxruntime pillow` |
| gcc (C99) + make | CSIM 编译 | `apt install gcc make` |
| Icarus Verilog / Verilator | RTL 与 SoC 仿真 | `apt install iverilog verilator` |
| cocotb | RTL 验证平台 | `pip install cocotb` |

### 编译 C 参考仿真器（CSIM）

```bash
cd csim
make clean all test    # 默认配置: 64-lane MAC Row, 44-bit 累加器, INT8+INT16
```

---

## 2. 快速自检（打包器与 CSIM 一致性）

无需外部数据文件，直接验证 `NPU1` 二进制打包器与 CSIM 逐比特一致性：

```bash
cd tools
python3 model_packer.py test
```

预期输出：
```
ALL TESTS PASSED
```

---

## 3. 完整工作流：ONNX -> NPU1 -> CSIM 推理

### 3.1 模型转换与训练后量化（PTQ）

```bash
cd tools

# INT8 量化
python3 onnx_converter.py \
    --model ./model_e.onnx \
    --calib ./calib_images/ \
    --input ./sample_input.bin \
    --output model_e_int8.npu1.bin \
    --bits 8 \
    --num-calib 50

# INT16 量化
python3 onnx_converter.py \
    --model ./model_e.onnx \
    --calib ./calib_images/ \
    --input ./sample_input.bin \
    --output model_e_int16.npu1.bin \
    --bits 16 \
    --num-calib 50
```

**常用参数说明：**

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `--model` | 必填 | ONNX float32 模型路径 |
| `--calib` | 必填 | 校准图像目录（`.jpg`/`.png`） |
| `--input` | 必填 | 测试输入（uint8 NCHW 二进制或图像） |
| `--output` | `model.npu1.bin` | 输出 `NPU1` 二进制路径 |
| `--bits` | `8` | 量化位宽（`8` 或 `16`） |
| `--num-calib` | `50` | 校准样本数量 |
| `--mean` | 无 | 可选逐通道归一化均值（3 个浮点数，如 `0.485 0.456 0.406`） |
| `--std` | 无 | 可选逐通道归一化标准差（3 个浮点数，如 `0.229 0.224 0.225`） |

**转换产物：**
- `model_e_int8.npu1.bin` — 编译后的 `NPU1` 模型（文件头 + 逐层描述符 + 权重 blob）
- `model_e_int8.npu1_input.bin` — 量化后的输入张量
- `model_e_int8.npu1_meta.npz` — 量化元数据（输入/输出 scale、形状等）

### 3.2 CSIM 仿真推理

```bash
../csim/npu_sim \
    model_e_int8.npu1.bin \
    model_e_int8.npu1_input.bin \
    output.bin
```

---

## 4. RTL 与 RISC-V SoC 端到端验证

### 4.1 公开基准模型一键 SoC 端到端验证

公开回归覆盖 `model_c`、`model_e`、`model_s` 三个代号模型（各含 INT8 与 INT16 两种精度）：

```bash
# 打包固件并运行 Verilator SoC 整网链式比对（默认 32-bit，可通过 DMA_BUS_WIDTH=64|128 切换宽口总线）
./soc/run_model_e2e.sh model_c_int8
./soc/run_model_e2e.sh model_e_int8
DMA_BUS_WIDTH=128 ./soc/run_model_e2e.sh model_s_int8
```

### 4.2 RTL cocotb 模块与集成回归

```bash
cd rtl/tb

# 单模块测试
make unit_csr
make unit_ppu
make unit_dma
make unit_ctrl
make unit_compute

# 顶层与 DMA 端到端测试
make integration
make e2e
```

---

## 5. 驱动 SDK（Bare-metal / FreeRTOS）

驱动位于 `driver/`，直接解析 `NPU1` 模型结构并驱动 Wishbone CSR：

```bash
riscv32-unknown-elf-gcc -Wall -std=c11 -ffreestanding \
    -DNPU_BASE_ADDR=0x80000000 \
    -c baremetal/npu_driver.c -o npu_driver.o
```

```c
npu_init();
npu_status_t st = npu_run_model(model_bin, ext_mem_base, io_buf_base);
```

---

## 6. 支持的算子与硬件规格速查

| `op_type` | 算子 | 硬件路径 |
|-----------|------|----------|
| 0 | Conv2D | 64-lane K-major 行引擎（任意卷积核/步长/空洞率/padding） |
| 1 | DWConv | 复用 `wrow_b[0..31]` + `racc[0..7]`（含全局 DW 流式切片） |
| 2 | FC | 64-lane K-major 行引擎（复用 1x1 卷积路径） |
| 3 | Pooling | Pooling FSM（Max / Avg / Global + 40 拍串行倒数器） |
| 4 | Eltwise Add | 8-lane 流水双分支缩放加法 + PPU |
| 5 | Resize | Resize FSM（最近邻 / 双线性） |
| 7 | Concat | 通道偏移寻址拼接 |

| 参数 | 指标 |
|------|------|
| 峰值算力 | `12.8 GMAC/s` (`25.6 GOPS`) @ `200 MHz` |
| 数据类型 / 累加器 | 有符号 INT8 / INT16，44-bit 有符号累加器 |
| 计算与后处理 | 64-lane K-major MAC Row + 8-lane 分时复用 PPU |
| 片上 Scratchpad | `172 KB`（`32 KB IFM + 48 KB OFM + 80 KB WGT + 12 KB PARAM`，512-bit 宽读口） |
| 外部总线规格 | `32-bit` Wishbone-B4 CSR 从口 + 参数化 `32/64/128-bit` `Wishbone-B4` 或 `AXI4` 突发 DMA 主口 |
| 全芯片综合规模 | Yosys 零黑盒 `npu_top`：`44,626` cells（`32b WB`）/ `45,441` cells（`128b AXI4`，`+1.83%`） |
