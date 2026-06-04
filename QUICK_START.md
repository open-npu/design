# Open-NPU 快速上手

本文档提供从 ONNX 模型到 RTL 仿真的完整端到端流程。

---

## 1. 环境准备

### 系统要求

| 依赖 | 用途 | 安装命令 |
|------|------|----------|
| Python 3.8+ | 工具链 | 系统自带或 `apt install python3` |
| numpy, onnx, onnxruntime, Pillow | 模型转换/量化 | `pip install numpy onnx onnxruntime pillow` |
| gcc (C99) + make | C 仿真器编译 | `apt install gcc make` |
| Icarus Verilog | RTL 仿真 | `apt install iverilog` |
| cocotb | RTL cocotb 测试 | `pip install cocotb` |

### 编译 C 仿真器

```bash
cd /data/sam/open-npu/csim
make          # 默认: 16×16 脉动阵列, 128KB SRAM, INT8+INT16

# 自定义配置
make CFLAGS_HW="-DNPU_ARRAY_SIZE=4 -DNPU_SPAD_SIZE_KB=32"
```

---

## 2. 快速验证（跑内置测试）

无需准备 ONNX 模型，直接验证工具链可用：

```bash
cd /data/sam/open-npu/csim && make && cd ../tools && python3 test_mobilenetv2.py
```

预期输出:
```
MobileNetV2-Tiny: 12 layers
BIT-EXACT match! (10 elements)
SUCCESS: MobileNetV2-Tiny 12-layer inference verified!
```

已有 E2E 测试脚本（均在 `tools/` 下）：
- `test_mobilenetv2.py` — MobileNetV2-Tiny (12层, 16×16 输入)
- `test_resnet18_e2e.py` — ResNet-18 (31层, 224×224 输入)
- `test_yolo_tiny_e2e.py` — YOLO-Tiny
- `test_m110_e2e.py` — M110 人脸模型 (63层)
- `test_int16_e2e.py` — INT16 全精度验证

---

## 3. 完整工作流：ONNX → 推理 → 验证

### 3.1 模型转换 + 量化

```bash
cd /data/sam/open-npu/tools

# INT8 量化
python3 onnx_converter.py \
    --model ./model.onnx \
    --calib ./calibration_images/ \
    --input ./test_input.bin \
    --output model.npu1.bin \
    --bits 8 \
    --num-calib 50

# INT16 量化
python3 onnx_converter.py \
    --model ./model.onnx \
    --calib ./calibration_images/ \
    --input ./test_input.bin \
    --output model_int16.npu1.bin \
    --bits 16 \
    --num-calib 50
```

**参数说明：**

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `--model` | 必填 | ONNX float32 模型路径 |
| `--calib` | 必填 | 校准图片目录 (.jpg/.png) |
| `--input` | 必填 | 测试输入 (uint8 NCHW 格式) |
| `--output` | `model.npu1.bin` | 输出 NPU1 二进制 |
| `--bits` | `8` | 量化位宽 (8 或 16) |
| `--num-calib` | `50` | 校准图片数量 |
| `--mean` | 无 | 输入均值 (3 浮点数, 如 ImageNet: `0.485 0.456 0.406`) |
| `--std` | 无 | 输入标准差 (3 浮点数, 如 ImageNet: `0.229 0.224 0.225`) |

**输出文件：**
- `model.npu1.bin` — 编译后的 NPU1 模型 (header + 层描述符 + 权重)
- `model_input.bin` — 量化后的输入张量
- `model_meta.npz` — 元数据 (scale, zero-point, 输入形状)

### 3.2 C 仿真器推理

```bash
cd /data/sam/open-npu/csim
./npu_sim ../tools/model.npu1.bin ../tools/model_input.bin output.bin
```

### 3.3 精度比较

```bash
cd /data/sam/open-npu/tools
python3 compare.py \
    --model ./model.onnx \
    --input ./test_input.bin \
    --npu-output ../csim/output.bin \
    --meta model_meta.npz
```

输出指标: cosine similarity, MSE, MAE, max absolute error, SNR (dB)。

---

## 4. 生成逐层 Golden 数据 (用于 RTL 对比)

```bash
cd /data/sam/open-npu/csim
export DUMP_LAYERS=1
./npu_sim ../tools/model.npu1.bin ../tools/model_input.bin output.bin
# 产出: /tmp/csim_layer_000.bin, /tmp/csim_layer_001.bin, ...
```

读取逐层数据:
```python
import numpy as np

# INT8 输出
out = np.fromfile('/tmp/csim_layer_000.bin', dtype=np.int8).reshape(H, W, C)

# INT16 输出
out = np.fromfile('/tmp/csim_layer_000.bin', dtype=np.int16).reshape(H, W, C)
```

---

## 5. RTL 仿真 (可选)

### 5.1 运行单元测试

```bash
cd /data/sam/open-npu/rtl/tb

# 单个模块测试
make DUT=npu_pe SIM=icarus
make DUT=npu_systolic SIM=icarus
make DUT=npu_ppu SIM=icarus
make DUT=npu_dma SIM=icarus
make DUT=npu_ctrl SIM=icarus

# 端到端 MobileNet 测试
make DUT=npu_top ARRAY_SIZE=4 SPAD_KB=32 SIM=icarus
```

### 5.2 E2E RTL 测试

```bash
cd /data/sam/open-npu/rtl/tb

# INT8 MobileNetV2-Tiny
make DUT=mobilenet_e2e SIM=icarus

# INT16 MobileNetV2-Tiny
make DUT=mobilenet_e2e_int16 SIM=icarus

# 全模型 (需要 Verilator)
make DUT=dma_e2e SIM=verilator
```

RTL 测试通过与 `/tmp/csim_layer_*.bin` golden 数据对比确认 bit-exact。

---

## 6. 驱动 SDK (用于 SoC/FPGA)

驱动位于 `/data/sam/open-npu/driver/`，支持 baremetal 和 FreeRTOS。

编译:
```bash
# Baremetal (RISC-V)
riscv32-unknown-elf-gcc -Wall -std=c11 -ffreestanding \
    -DNPU_BASE_ADDR=0x80000000 \
    -c baremetal/npu_driver.c -o npu_driver.o

# FreeRTOS (RISC-V)
riscv32-unknown-elf-gcc -Wall -std=c11 -ffreestanding \
    -DNPU_BASE_ADDR=0x80000000 \
    -I/path/to/freertos/include \
    -c freertos/npu_rtos.c -o npu_rtos.o
```

API 示例:
```c
// 初始化
npu_init();

// 跑完整 NPU1 模型
npu_status_t st = npu_run_model(model_bin, ext_mem_base, io_buf_base);

// FreeRTOS 异步推理
npu_inference_req_t req = { .model_bin = model_bin, ... };
npu_rtos_run_model_async(&req);
npu_rtos_wait(&req, 10000);
```

---

## 7. NPU1 二进制格式

```
Header (16 字节):
  uint32 magic     = 0x4E505531 ("NPU1")
  uint32 num_layers
  uint32 weight_offset    (权重 blob 偏移)
  uint32 weight_size

层描述符 (变长, 逐层拼接):
  fixed_config (62 字节) + per-channel params + [add params]

权重 blob (位于 weight_offset)
```

---

## 8. 支持的算子

| op_type | 算子 | 硬件路径 |
|---------|------|----------|
| 0 | Conv2D | 脉动阵列 (1×1, 3×3) |
| 1 | DWConv | DW 卷积模块 (任意 kernel) |
| 2 | FC | 脉动阵列 (复用 Conv2D) |
| 3 | Pooling | Pooling FSM (Max/Avg/Global) |
| 4 | Eltwise Add | Add FSM + 双路 rescale |
| 5 | Resize | Resize FSM (nearest/bilinear) |
| 6 | Deconv | Conv2D 路径 + zero-insert skip |
| 7 | Concat | 复用 Add FSM + offset 寻址 |

---

## 9. 架构概要

| 参数 | 指标 |
|------|------|
| 峰值算力 | 0.2 TOPS @ 200MHz |
| 数据类型 | INT8 / INT16 |
| 累加器 | 40-bit signed |
| 脉动阵列 | 16×16 INT8, weight-stationary |
| 片上 SRAM | 128KB Activation + 64KB Weight + 8KB Param |
| 总线 | Wishbone-B4 |
| FPGA 目标 | Artix-7 XC7A200T (~52K LUT, 39K FF, 26 BRAM, 333 DSP) |

---

## 10. 目录结构

```
open-npu/
├── design/              ← 架构文档
│   ├── architecture-spec.md     架构详细设计
│   ├── interface-spec.md        NPU1 二进制接口规范
│   └── npu-register-spec.md     CSR 寄存器手册
├── tools/               ← Python 工具链
│   ├── onnx_converter.py        ONNX → NPU1 转换
│   ├── model_packer.py          NPU1 打包
│   ├── compare.py               精度比较
│   └── test_*.py                E2E 测试脚本
├── csim/                ← C 仿真器
│   ├── src/                     源码
│   └── Makefile
├── rtl/                 ← 可综合 Verilog
│   ├── src/                     RTL 源码
│   └── tb/                      cocotb 测试平台
├── driver/              ← 驱动 SDK
│   ├── baremetal/               Baremetal 驱动
│   └── freertos/                FreeRTOS 驱动
└── soc/                 ← LiteX SoC 集成
    ├── litex/                    LiteX Builder
    ├── firmware/                 SoC 固件
    └── sim/                      Verilator 仿真
```

---

## 11. 更多文档

- [架构详细设计](architecture-spec.md) — 模块级微架构、时序图、状态机
- [NPU1 接口规范](interface-spec.md) — 二进制格式字段定义
- [寄存器手册](npu-register-spec.md) — 全部 CSR 位定义与编程示例
- [驱动 SDK](../driver/README.md) — 驱动 API 参考与编译说明
- [逐层提取快速指南](../LAYER_EXTRACTION_QUICK_START.md) — RTL per-layer golden 数据生成
