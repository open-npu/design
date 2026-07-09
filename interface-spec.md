# Open-NPU 接口规格真相表 (Interface Specification)

> **本文件是离线模型二进制格式的唯一权威定义。**
> csim、tools、RTL驱动均应以本文件为准。
>
> 硬件CSR寄存器定义见 `npu-register-spec.md`（RTL实现时使用）。
> 两者的区分见 §6。

版本：V1.0
更新：2026-05-22

---

## 1. 模型二进制格式 (NPU1)

```
┌────────────────────────────────────────────────────────┐
│  Header (16 bytes)                                      │
├────────────────────────────────────────────────────────┤
│  Layer Descriptor #0 (variable-length)                  │
│  Layer Descriptor #1                                    │
│  ...                                                    │
│  Layer Descriptor #(N-1)                                │
├────────────────────────────────────────────────────────┤
│  Weight Blob (at weight_offset, contiguous)             │
└────────────────────────────────────────────────────────┘
```

### 1.1 Header (16 bytes)

| Offset | Field         | Type   | Value/Description                |
|--------|---------------|--------|----------------------------------|
| 0-3    | magic         | uint32 | `0x4E505531` (ASCII "NPU1")      |
| 4-7    | num_layers    | uint32 | 层数 N                           |
| 8-11   | weight_offset | uint32 | 权重数据相对文件起始的字节偏移   |
| 12-15  | weight_size   | uint32 | 权重数据总字节数                 |

### 1.2 Layer Descriptor (变长)

每层描述符由以下部分顺序拼接：

```
[Fixed Config: 62 bytes]
[Per-Channel Params: param_ch_count × 14 bytes]  (仅当param_ch_count > 0)
[Add Params: 8 bytes]                             (仅当has_add = 1)
[LUT INT8: 256 bytes + LUT INT16: 512 bytes]      (仅当has_lut = 1)
```

---

## 2. Layer Fixed Config (62 bytes)

| Offset | Field          | Type   | Description                                          |
|--------|----------------|--------|------------------------------------------------------|
| 0      | op_type        | uint8  | 算子类型 (0-7, 见§2.1)                               |
| 1      | data_type      | uint8  | 0=INT8, 1=INT16                                      |
| 2-3    | in_h           | uint16 | 输入高度                                             |
| 4-5    | in_w           | uint16 | 输入宽度                                             |
| 6-7    | in_c           | uint16 | 输入通道数                                           |
| 8-9    | out_h          | uint16 | 输出高度                                             |
| 10-11  | out_w          | uint16 | 输出宽度                                             |
| 12-13  | out_c          | uint16 | 输出通道数                                           |
| 14     | kernel_h       | uint8  | 卷积核高度                                           |
| 15     | kernel_w       | uint8  | 卷积核宽度                                           |
| 16     | dilation_h     | uint8  | 膨胀系数 H                                           |
| 17     | dilation_w     | uint8  | 膨胀系数 W                                           |
| 18     | stride_h       | uint8  | 步长 H                                               |
| 19     | stride_w       | uint8  | 步长 W                                               |
| 20     | pad_top        | uint8  | 上方 padding                                         |
| 21     | pad_bottom     | uint8  | 下方 padding                                         |
| 22     | pad_left       | uint8  | 左侧 padding                                        |
| 23     | pad_right      | uint8  | 右侧 padding                                        |
| 24     | pool_mode      | uint8  | 池化模式 (0=Max, 1=Avg)                              |
| 25     | pool_h         | uint8  | 池化核高度                                           |
| 26     | pool_w         | uint8  | 池化核宽度                                           |
| 27     | pool_stride_h  | uint8  | 池化步长 H                                           |
| 28     | pool_stride_w  | uint8  | 池化步长 W                                           |
| 29     | global_pool    | uint8  | 1=全局池化                                           |
| 30     | resize_mode    | uint8  | 0=nearest, 1=bilinear                                |
| 31     | scale_h        | uint8  | Resize缩放因子 H (Q4.4 定点)                        |
| 32     | scale_w        | uint8  | Resize缩放因子 W (Q4.4 定点)                        |
| 33     | insert_h       | uint8  | Deconv垂直插零数                                     |
| 34     | insert_w       | uint8  | Deconv水平插零数                                     |
| 35-36  | concat_offset  | uint16 | Concat通道偏移                                       |
| 37-38  | concat_total_c | uint16 | Concat后总通道数                                     |
| 39-40  | tile_h         | uint16 | Tile输出高度 (0=不tiling)                            |
| 41-42  | tile_w         | uint16 | Tile输出宽度                                         |
| 43-44  | tile_num_h     | uint16 | 垂直方向tile数量                                     |
| 45-46  | tile_num_w     | uint16 | 水平方向tile数量                                     |
| 47     | post_ctrl      | uint8  | 后处理控制 (见§5)                                    |
| 48     | sched_ctrl     | uint8  | 调度控制 (见§4)                                      |
| 49-50  | clamp_min      | int16  | 输出下限 (INT8: -128, INT16: -32768)                 |
| 51-52  | clamp_max      | int16  | 输出上限 (INT8: 127, INT16: 32767)                   |
| 53     | in_zp          | int8   | 输入零点 (padding填充值)                             |
| 54     | _pad1          | uint8  | 保留 (0x00)                                          |
| 55-56  | param_ch_count | uint16 | Per-channel参数数量 (通常=out_c, 0表示无)            |
| 57     | has_lut        | uint8  | 1=后跟LUT数据                                        |
| 58     | has_add        | uint8  | 1=后跟Add参数                                        |
| 59     | residual_src   | int8   | Add残差输入层索引 (-1=无)                            |
| 60-61  | input_src      | int16  | 输入来源层索引 (-1=上一层)                           |

### 2.1 算子类型编码 (op_type)

| 值 | 名称         | 说明            |
|----|--------------|-----------------|
| 0  | OP_CONV2D    | 标准卷积        |
| 1  | OP_DW_CONV   | 深度可分离卷积  |
| 2  | OP_FC        | 全连接          |
| 3  | OP_POOLING   | 池化            |
| 4  | OP_ELTWISE_ADD | 逐元素加法    |
| 5  | OP_RESIZE    | 上采样/缩放     |
| 6  | OP_DECONV    | 反卷积          |
| 7  | OP_CONCAT    | 通道拼接        |

---

## 3. Per-Channel Parameter Pack (14 bytes/channel)

用于 Conv2D / DWConv / FC 层的 per-channel requantize + bias。

```
┌──────────────────────────────────────────────────────────────────┐
│  byte offset │ 字段                                              │
├──────────────┼──────────────────────────────────────────────────┤
│  [0:1]       │ M[14:0] (15-bit unsigned multiplier, bit15=0)    │
│  [2]         │ S[5:0]  (6-bit shift amount, bit7:6=0)           │
│  [3]         │ reserved (0x00)                                   │
│  [4:5]       │ zp[15:0] (16-bit signed output zero point)       │
│  [6:13]      │ bias_q[63:0] (64-bit signed, little-endian)      │
└──────────────┴──────────────────────────────────────────────────┘

总大小: OUT_C × 14 bytes
示例: 512通道 = 7,168 bytes
```

后处理管道执行顺序（硬件PPU固定流水线）：

```
acc(40-bit) → +bias_q[ch] → ×M[ch] → >>S[ch](+round) → +zp[ch] → clamp → ReLU/LUT → out
```

**设计说明**：使用 int64 bias 而非 int32 是因为 INT8 量化中大通道卷积的 bias 累加精度需要 64-bit 才能避免溢出。代价是参数 SRAM 需要额外 40% 容量（14B vs 10B），但参数 SRAM 总量仍可控（8KB 足够 512 通道）。

---

## 4. Scheduling Control (`sched_ctrl`, descriptor byte 48)

此字段是**离线编译器设置的调度提示**，告诉运行时（csim 或 RTL Control Unit）如何执行该层。

| Bit   | 名称       | 说明                                    |
|-------|------------|----------------------------------------|
| [0]   | DB_EN      | 双缓冲使能 (DMA ping-pong overlap)     |
| [1]   | FUSE_START | 融合块第一层                           |
| [2]   | FUSE_MID   | 融合块中间层                           |
| [3]   | FUSE_END   | 融合块最后一层                         |
| [7:4] | reserved   | 保留 (0)                               |

**重要**：此字段与硬件 CSR `DMA_CTRL (0x118)` 无关。两者是完全不同的概念：
- `sched_ctrl`：存在于离线模型二进制中，编译器决定
- `DMA_CTRL CSR`：RTL内部DMA引擎的配置寄存器，由Control Unit根据descriptor内容编程

### 融合执行语义

当 RTL/csim 看到 `FUSE_START` 时：
1. 连续读取 3 层 descriptor (START + MID + END)
2. 以 START 层的 tile_h/tile_w 为公共 tile 尺寸
3. 中间张量在 SRAM banks 内传递，不写回 DRAM
4. DW 层 border padding 使用 MID 层的 `in_zp`

---

## 5. POST_CTRL (descriptor byte 47)

| Bit   | 名称      | 说明                                     |
|-------|-----------|------------------------------------------|
| [1:0] | PPU_MODE  | 0=CONV_REQ, 1=ADD, 2=RELU_ONLY, 3=PASSTHROUGH |
| [2]   | RELU_EN   | ReLU激活 (max(0, x))                     |
| [3]   | RELU6_EN  | ReLU6 (clamp to [0, clamp_max])          |
| [4]   | LUT_EN    | 查表激活 (替代ReLU)                      |
| [5]   | ZP_EN     | 加输出零点                               |
| [6]   | BIAS_EN   | 加per-channel bias                       |
| [7]   | INT16_OUT | 输出为INT16 (否则INT8)                   |

此字段同时映射到硬件 CSR `POST_CTRL (0x180)`，语义完全一致。

---

## 6. 硬件 DMA_CTRL CSR (0x118) — 仅RTL使用

此寄存器由 RTL Control Unit 根据 descriptor 内容编程，不在离线模型二进制中出现。

| Bit   | 名称             | 复位值 | 说明                               |
|-------|------------------|--------|------------------------------------|
| [0]   | TRANSPOSE_EN     | 0      | 1=输入搬运时做NCHW→NHWC转置       |
| [1]   | OUT_TRANSPOSE_EN | 0      | 1=输出时做NHWC→NCHW转置           |
| [5:4] | BURST_LEN        | 0      | Burst长度: 0=4, 1=8, 2=16, 3=32   |
| [7:6] | PRIORITY         | 0      | DMA优先级: 0=低, 3=高             |
| [8]   | CIRCULAR_EN      | 0      | 1=循环缓冲模式                    |
| [31:9]| —                | 0      | 保留                               |

---

## 7. Add Parameter Pack (8 bytes)

用于 `OP_ELTWISE_ADD` 层的双分支 rescale。

```
┌──────────────────────────────────────────────────────────────────┐
│  byte offset │ 字段                                              │
├──────────────┼──────────────────────────────────────────────────┤
│  [0:1]       │ M_A[14:0] (分支A 15-bit multiplier)              │
│  [2]         │ S_A[5:0]  (分支A 6-bit shift)                    │
│  [3]         │ reserved (0x00)                                   │
│  [4:5]       │ M_B[14:0] (分支B 15-bit multiplier)              │
│  [6]         │ S_B[5:0]  (分支B 6-bit shift)                    │
│  [7]         │ reserved (0x00)                                   │
└──────────────┴──────────────────────────────────────────────────┘
```

Add 执行公式：`out = clamp(round((A × M_A >> S_A) + (B × M_B >> S_B) + zp))`

---

## 8. Tiling Ownership

> **V1.0 决策：Tiling 由离线编译器计算。**

- 编译器 (`tools/tiling.py`) 根据 SRAM 预算计算最优 tile 尺寸
- 结果写入 descriptor 的 tile_h / tile_w / tile_num_h / tile_num_w 字段 (offsets 39-46)
- RTL Control Unit 按 descriptor 指定的 tile 参数执行循环，**不自行推导 tile 尺寸**
- 当 tile_h = 0 时表示该层无需 tiling（整层 fit in SRAM）

SRAM 预算约束：
- Activation Buffer: 2 × 32KB (双bank，双缓冲时每bank 16KB)
- Weight Buffer: 64KB
- Parameter SRAM: 8KB (max 512ch × 14B = 7KB)

---

## 9. Input Data Format

| 项目         | 定义                                              |
|--------------|---------------------------------------------------|
| 像素输入     | uint8, NCHW layout                                |
| 归一化公式   | `float_val = (pixel_uint8 - 127.5) / 255.0`       |
| 量化公式     | `input_q = round(float_val / input_scale)`         |
| csim内部布局 | NHWC (对齐硬件脉动阵列数据流)                     |
| debug.bin    | 量化后 INT8/INT16 值, NCHW layout, 直接feed csim  |

**注意**：csim 在加载 debug.bin 时按 NCHW 读取后内部转为 NHWC 执行。

---

## 10. 权重数据格式

权重按层顺序紧密排列于 weight blob 中：

- **Conv2D / FC**: `[out_c × kernel_h × kernel_w × in_c]`，INT8/INT16, OHWI layout
- **DWConv**: `[channels × kernel_h × kernel_w × 1]`，INT8/INT16
- 每层权重大小 = `out_c × kernel_h × kernel_w × in_c × elem_size`
- DWConv 权重大小 = `out_c × kernel_h × kernel_w × 1 × elem_size`

---

## 附录：源文件对应关系

| 本规格章节 | 实现文件                                         |
|-----------|--------------------------------------------------|
| §1-2      | `tools/model_packer.py` (pack_fixed, pack_model) |
| §3        | `csim/include/npu_types.h` (perchannel_param_t)  |
| §4        | `csim/include/npu_types.h` (SCHED_CTRL_*)        |
| §5        | `csim/include/npu_types.h` (POST_*)              |
| §6        | `design/npu-register-spec.md` (CSR 0x118)        |
| §7        | `csim/include/npu_types.h` (add_param_t)         |
| §8        | `tools/tiling.py`                                |
| §9        | `tools/onnx_converter.py` (preprocess)           |
| §10       | `tools/model_packer.py` (weight packing)         |
