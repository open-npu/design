# Open-NPU 离线模型二进制接口规格 (Interface Specification)

> **本文件是离线模型二进制格式（`NPU1`）的唯一权威定义。**
> `csim`、`tools` 与 `driver` 均以本文件定义的字节偏移与字段语义为准。
> 硬件 Wishbone CSR 寄存器定义见 `npu-register-spec.md`。

版本：V1.3
更新：2026-11-26

---

## 1. 模型二进制格式 (`NPU1`)

```
┌────────────────────────────────────────────────────────┐
│  Header (16 bytes)                                     │
├────────────────────────────────────────────────────────┤
│  Layer Descriptor #0 (variable-length)                 │
│  Layer Descriptor #1                                   │
│  ...                                                   │
│  Layer Descriptor #(N-1)                               │
├────────────────────────────────────────────────────────┤
│  Weight Blob (at weight_offset, contiguous)            │
└────────────────────────────────────────────────────────┘
```

### 1.1 Header (16 bytes, Little-Endian)

| Offset | Field | Type | Value / Description |
|--------|-------|------|---------------------|
| `0-3` | `magic` | `uint32` | `0x4E505531` (ASCII `"NPU1"`) |
| `4-7` | `num_layers` | `uint32` | 层数 `N` |
| `8-11` | `weight_offset` | `uint32` | 权重 blob 相对文件起始的字节偏移 |
| `12-15` | `weight_size` | `uint32` | 权重数据总字节数 |

### 1.2 Layer Descriptor（变长拼接）

```
[Fixed Config: 62 bytes]
[Per-Channel Params: param_ch_count * 14 bytes]   (仅当 param_ch_count > 0)
[Add Params: 8 bytes]                              (仅当 has_add == 1)
```

> 注：为保持二进制向下兼容，固定头第 57 字节保留 `has_lut` 字段，但现行硬件与工具链固定置 `has_lut = 0`，不再附带 LUT 表数据。

---

## 2. Layer Fixed Config (62 bytes)

| Offset | Field | Type | Description |
|--------|-------|------|-------------|
| `0` | `op_type` | `uint8` | 算子类型（`0-5, 7`，见 §2.1） |
| `1` | `data_type` | `uint8` | `0` = INT8，`1` = INT16 |
| `2-3` | `in_h` | `uint16` | 输入高度 |
| `4-5` | `in_w` | `uint16` | 输入宽度 |
| `6-7` | `in_c` | `uint16` | 输入通道数 |
| `8-9` | `out_h` | `uint16` | 输出高度 |
| `10-11` | `out_w` | `uint16` | 输出宽度 |
| `12-13` | `out_c` | `uint16` | 输出通道数 |
| `14` | `kernel_h` | `uint8` | 卷积核高度 |
| `15` | `kernel_w` | `uint8` | 卷积核宽度 |
| `16` | `dilation_h` | `uint8` | 空洞膨胀系数 H |
| `17` | `dilation_w` | `uint8` | 空洞膨胀系数 W |
| `18` | `stride_h` | `uint8` | 垂直步长 H |
| `19` | `stride_w` | `uint8` | 水平步长 W |
| `20` | `pad_top` | `uint8` | 上方 padding |
| `21` | `pad_bottom` | `uint8` | 下方 padding |
| `22` | `pad_left` | `uint8` | 左侧 padding |
| `23` | `pad_right` | `uint8` | 右侧 padding |
| `24` | `pool_mode` | `uint8` | 池化模式（`0` = Max，`1` = Avg） |
| `25` | `pool_h` | `uint8` | 池化窗口高度 |
| `26` | `pool_w` | `uint8` | 池化窗口宽度 |
| `27` | `pool_stride_h` | `uint8` | 池化垂直步长 |
| `28` | `pool_stride_w` | `uint8` | 池化水平步长 |
| `29` | `global_pool` | `uint8` | `1` = 全局池化 |
| `30` | `resize_mode` | `uint8` | `0` = 最近邻，`1` = 双线性 |
| `31` | `scale_h` | `uint8` | Resize 垂直缩放因子（Q4.4 定点） |
| `32` | `scale_w` | `uint8` | Resize 水平缩放因子（Q4.4 定点） |
| `33-34` | `_reserved_33` | `uint8[2]` | 保留字节（固定填 `0`，原转置卷积插零字段已停用） |
| `35-36` | `concat_offset` | `uint16` | Concat 通道起始偏移 |
| `37-38` | `concat_total_c` | `uint16` | Concat 拼接后总通道数 |
| `39-40` | `tile_h` | `uint16` | 空间分块输出高度（`0` = 不切块） |
| `41-42` | `tile_w` | `uint16` | 空间分块输出宽度 |
| `43-44` | `tile_num_h` | `uint16` | 垂直方向 tile 数 |
| `45-46` | `tile_num_w` | `uint16` | 水平方向 tile 数 |
| `47` | `post_ctrl` | `uint8` | 后处理控制位（见 §5） |
| `48` | `sched_ctrl` | `uint8` | 调度控制位（见 §4） |
| `49-50` | `clamp_min` | `int16` | 输出下限（INT8: `-128`，INT16: `-32768`） |
| `51-52` | `clamp_max` | `int16` | 输出上限（INT8: `127`，INT16: `32767`） |
| `53` | `in_zp` | `int8` | 输入零点（用于边界 padding 填充） |
| `54` | `_pad1` | `uint8` | 对齐保留（`0x00`） |
| `55-56` | `param_ch_count` | `uint16` | 逐通道参数个数（通常等于 `out_c`，无参数时为 `0`） |
| `57` | `has_lut` | `uint8` | 保留字段（固定为 `0`） |
| `58` | `has_add` | `uint8` | `1` = 描述符末尾附带 8 字节 Add 参数 |
| `59` | `residual_src` | `int8` | Add 残差输入来源层索引（`-1` = 无） |
| `60-61` | `input_src` | `int16` | 主输入来源层索引（`-1` = 紧邻上一层输出） |

### 2.1 算子类型编码 (`op_type`)

| 值 | 名称 | 说明 |
|----|------|------|
| `0` | `OP_CONV2D` | 标准卷积（含空洞卷积） |
| `1` | `OP_DW_CONV` | 深度卷积（Depthwise Conv2D） |
| `2` | `OP_FC` | 全连接 |
| `3` | `OP_POOLING` | 池化（Max / Avg / Global） |
| `4` | `OP_ELTWISE_ADD` | 双分支逐元素缩放加法 |
| `5` | `OP_RESIZE` | 空间缩放（最近邻 / 双线性） |
| `6` | `RESERVED` | 已停用保留编码 |
| `7` | `OP_CONCAT` | 通道拼接 |

---

## 3. Per-Channel Parameter Pack (14 bytes / channel)

用于 `Conv2D / DWConv / FC` 的逐通道偏置与重量化：

```
┌──────────────┬──────────────────────────────────────────────────┐
│  byte offset │ 字段定义                                          │
├──────────────┼──────────────────────────────────────────────────┤
│  [0:1]       │ M[14:0] (15-bit 无符号乘数, bit15 = 0)            │
│  [2]         │ S[5:0]  (6-bit 右移位数, bit7:6 = 0)              │
│  [3]         │ reserved (0x00)                                  │
│  [4:5]       │ zp[15:0] (16-bit 有符号输出零点)                  │
│  [6:13]      │ bias_q[63:0] (64-bit 有符号偏置, little-endian)   │
└──────────────┴──────────────────────────────────────────────────┘
```

PPU 固定算术顺序（与 `rtl/src/npu_ppu.v` 和 `csim/src/postproc.c` 逐比特一致）：
```
acc(44-bit) -> + bias_q[ch] -> * M[ch] + (1 << (S[ch] - 1)) -> >>> S[ch] -> + zp[ch] -> clamp -> ReLU/ReLU6 -> out
```

---

## 4. Scheduling Control (`sched_ctrl`, byte 48)

| Bit | 名称 | 说明 |
|-----|------|------|
| `[0]` | `DB_EN` | 空间 tile 双缓冲（ping-pong 预取） |
| `[1]` | `FUSE_START` | 跨层片上融合首层（输出留在 OFM bank，不写回主存） |
| `[2]` | `FUSE_MID` | 跨层片上融合中间层（输入输出均在 IFM/OFM bank 间互换） |
| `[3]` | `FUSE_END` | 跨层片上融合末层（从片上读取输入，完成后写回主存） |
| `[4]` | `PER_TILE_STORE` | 空间切块按 2D 行跨度直接拼回主存 NHWC 输出张量 |
| `[7:5]` | `reserved` | 保留（`0`） |

---

## 5. `POST_CTRL` (byte 47)

| Bit | 名称 | 说明 |
|-----|------|------|
| `[1:0]` | `PPU_MODE` | `0`=CONV_REQ, `1`=ADD, `2`=RELU_ONLY, `3`=PASSTHROUGH |
| `[2]` | `RELU_EN` | ReLU 截断（`max(0, x)`） |
| `[3]` | `RELU6_EN` | ReLU6 截断（钳位至 `[0, clamp_max]`） |
| `[4]` | `reserved` | 保留（`0`） |
| `[5]` | `ZP_EN` | 启用输出零点加法 |
| `[6]` | `BIAS_EN` | 启用逐通道偏置加法 |
| `[7]` | `INT16_OUT` | `1` = 输出 INT16，`0` = 输出 INT8 |

---

## 6. Add Parameter Pack (8 bytes)

用于 `OP_ELTWISE_ADD` 的双分支对齐缩放：

| Byte Offset | 字段 |
|---|---|
| `[0:1]` | `M_A[14:0]`（分支 A 15-bit 无符号乘数） |
| `[2]` | `S_A[5:0]`（分支 A 6-bit 右移位数） |
| `[3]` | `0x00` |
| `[4:5]` | `M_B[14:0]`（分支 B 15-bit 无符号乘数） |
| `[6]` | `S_B[5:0]`（分支 B 6-bit 右移位数） |
| `[7]` | `0x00` |

算术定义：`out = clamp(round_half_up(A * M_A, S_A) + round_half_up(B * M_B, S_B) + zp)`。

---

## 7. 切块预算与张量/权重布局

- **片上 Scratchpad 默认预算（`172 KB`）**：
  - `IFM`：`32 KB`（`DB_EN = 1` 时每半区 `16 KB`）
  - `OFM`：`48 KB`
  - `WGT`：`80 KB`
  - `PARAM`：`12 KB`（开启参数 ping/pong 预取时每半区 `6 KB`）
- **激活布局**：CSIM 内部与 RTL SRAM / 主存 DMA 统一为 **NHWC**。
- **权重布局**：
  - `NPU1` 离线文件内按 `[out_c, kernel_h, kernel_w, in_c]`（OHWI）紧凑存储；
  - 生成 RTL / SoC 载荷时，`model_packer.pack_conv_weights_kmajor()` 将稠密卷积与 FC 权重按 64 通道 OC 组重排为 **K-major** 顺序（每个 K 步连续存放组内 64 路通道权重），匹配 512-bit WGT SRAM 宽读口。

---

## 附录：权威实现对照表

| 章节 | 对应代码文件 |
|------|--------------|
| §1 - §2 | `tools/model_packer.py` (`pack_fixed`, `pack_model`) |
| §3, §5, §6 | `csim/include/npu_types.h`、`csim/src/postproc.c`、`rtl/src/npu_ppu.v` |
| §4, §7 | `tools/tiling.py`、`tools/layer_fusion.py`、`tools/hw_config.py` |
