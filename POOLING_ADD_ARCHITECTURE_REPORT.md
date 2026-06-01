# Open-NPU Pooling and Add Operators - Architecture Specification Report

**Date:** 2026-05-29  
**Source:** Open-NPU Repository `/data/sam/open-npu/design/`  
**Reviewed Files:**
- `architecture-spec.md`
- `npu-register-spec.md`
- `interface-spec.md`

---

## Executive Summary

Pooling (Max/Avg) and Eltwise Add are post-processing operators in Open-NPU that **reuse the PPU (Post-Processing Unit) datapath** rather than using the main Systolic Array. Both operators bypass the SA computation engine and leverage the existing hardware infrastructure for quantization and activation, demonstrating efficient hardware reuse.

---

## 1. POOLING OPERATOR ARCHITECTURE

### 1.1 Operator Type and Hardware Mapping

**OP_TYPE = 3 (OP_POOLING)**

| Aspect | Details |
|--------|---------|
| **Hardware Path** | Pooling Logic (复用PPU datapath) - bypasses Systolic Array |
| **Modes Supported** | Max Pool (0) and Average Pool (1) |
| **Data Type Support** | INT8 and INT16 |
| **Quantization** | Per-output-channel requantize via PPU pipeline |
| **Output Pipeline** | Requantize → Clamp → Optional ReLU/LUT |

**Key Insight:** Pooling does NOT use the Systolic Array. Instead, it implements a dedicated Pooling Logic that feeds results directly into the PPU (Post-Processing Unit) datapath for requantization and activation.

---

### 1.2 Configuration Registers

#### LAYER_MODE (CSR 0x040)
```
Bits [3:0]: OP_TYPE = 3  (identifies this layer as pooling)
Bit  [4]:   DATA_TYPE = 0 for INT8, 1 for INT16
```

#### POOL_CFG (CSR 0x060) - Core Pooling Configuration

| Register Field | Bits | Description |
|---|---|---|
| **POOL_MODE** | [0] | 0=MaxPool, 1=AvgPool |
| **POOL_H** | [7:4] | Pool window height (1-15) |
| **POOL_W** | [11:8] | Pool window width (1-15) |
| **POOL_STRIDE_H** | [15:12] | Pool stride vertical (1-15) |
| **POOL_STRIDE_W** | [19:16] | Pool stride horizontal (1-15) |
| **GLOBAL_POOL** | [20] | 1=Global pooling mode (ignores POOL_H/W) |
| **Reserved** | [31:21] | Reserved (0) |

**Example Configuration (Register Write):**
```c
// For 2×2 max pool with stride 2
REG(POOL_CFG) = (0 << 20)  // GLOBAL_POOL = 0
              | (2 << 16)  // POOL_STRIDE_W = 2
              | (2 << 12)  // POOL_STRIDE_H = 2
              | (2 << 8)   // POOL_W = 2
              | (2 << 4)   // POOL_H = 2
              | (0 << 0);  // POOL_MODE = MaxPool
```

#### Input/Output Dimension Registers
```
IN_DIM_H_W  (0x044): Input feature map height × width
IN_DIM_C    (0x048): Input channels (same as output for pooling)
OUT_DIM_H_W (0x04C): Output feature map height × width
OUT_DIM_C   (0x050): Output channels (same as input for pooling)
```

#### Padding Configuration (for border handling)
```
PADDING (0x05C):
  Bits [7:0]:   PAD_TOP
  Bits [15:8]:  PAD_BOTTOM
  Bits [23:16]: PAD_LEFT
  Bits [31:24]: PAD_RIGHT
```

When `in_zp` is set in descriptor, padding pixels are filled with input zero point value.

---

### 1.3 Interface Specification (Layer Descriptor Format)

In the offline model binary (NPU1 format), pooling layer descriptor contains:

```c
// From interface-spec.md, Layer Fixed Config (62 bytes)
struct layer_descriptor {
    uint8  op_type;          // = 3 (OP_POOLING)
    uint8  data_type;        // 0=INT8, 1=INT16
    uint16 in_h, in_w, in_c; // Input dimensions
    uint16 out_h, out_w, out_c; // Output dimensions
    uint8  pad_top, pad_bottom, pad_left, pad_right; // Padding
    
    // Pooling-specific fields:
    uint8  pool_mode;        // 0=Max, 1=Avg
    uint8  pool_h;           // Window height
    uint8  pool_w;           // Window width
    uint8  pool_stride_h;    // Stride H
    uint8  pool_stride_w;    // Stride W
    uint8  global_pool;      // 1=global mode
    
    // ... other fields (resize, deconv, concat - unused for pooling)
    
    uint8  post_ctrl;        // Post-processing control (for requantize/ReLU)
    uint8  sched_ctrl;       // Scheduling hints (DB_EN, FUSE_*)
    int16  clamp_min, clamp_max;  // Output clamp range
    int8   in_zp;            // Input zero point (for padding)
    uint16 param_ch_count;   // Number of per-channel params (= out_c)
    uint8  has_lut;          // 1 if followed by LUT data
    uint8  has_add;          // 0 for pooling
    // ... rest
};
```

**No weight data needed** - Pooling is parameter-free.

---

### 1.4 PPU Post-Processing for Pooling

After pooling window reduction (max or average), results flow through the PPU datapath:

#### POST_CTRL Register (CSR 0x180) - Control Flags

| Bit | Field | Pooling Typical Value |
|-----|-------|-------|
| [1:0] | PPU_MODE | 0 (CONV_REQ mode - standard requantize, not ADD mode) |
| [2] | RELU_EN | Often 1 (apply ReLU after pool in fused ops) |
| [3] | RELU6_EN | 0 for standard pooling |
| [4] | LUT_EN | 0 or 1 if using LUT activation |
| [5] | ZP_EN | 1 if output has zero point offset |
| [6] | BIAS_EN | 0 for pure pooling (no per-channel bias) |
| [7] | INT16_OUT | Matches input data type |

#### PPU Requantize Pipeline

```
Pooling Result (int32 for max, accumulated sum for avg)
    ↓
[×M] Multiply by per-channel scale factor M
    ↓
[>>S] Right shift by S bits (with rounding)
    ↓
[+ZP] Add output zero point (if ZP_EN=1)
    ↓
[Clamp] Clamp to [clamp_min, clamp_max]
    ↓
[ReLU] Optional ReLU (max(0, x)) if RELU_EN=1
    ↓
Output (INT8/INT16)
```

**Important:** Pooling results are treated as **pre-quantized accumulators** that require requantization to match output scale. The per-channel M (multiplier) and S (shift) values come from the parameter SRAM loaded by DMA.

---

### 1.5 DMA Configuration for Pooling

#### Input/Output Data Flow

```
External SRAM (Input)
    ↓ DMA_IN_ADDR
Activation Scratchpad (ping-pong buffer)
    ↓ Pooling Logic reads tile
Pooling computation (local sliding window)
    ↓ Results → PPU
Activation Scratchpad (output bank)
    ↓ DMA_OUT_ADDR
External SRAM (Output)
```

**Key Registers:**
- `DMA_IN_ADDR (0x100)` - Input feature map base address
- `DMA_OUT_ADDR (0x104)` - Output feature map base address
- `DMA_IN_STRIDE (0x110)` - Row/channel stride for input fetching
- `DMA_OUT_STRIDE (0x114)` - Row/channel stride for output writing
- `DMA_IN_SIZE (0x128)` - Total input data size
- `DMA_OUT_SIZE (0x130)` - Total output data size

**No weight DMA needed** for pooling (DMA_WEIGHT_ADDR unused).

#### Parameter Loading for Requantization

```
DMA_PARAM_ADDR (0x10C)  or  POST_PARAM_ADDR (0x184)
    ↓ Both registers are dual-mapped (same physical register)
Per-channel parameter SRAM (8KB)
    ↓ Indexed by output channel
PPU reads M[ch], S[ch], zp[ch] during drain
```

**Per-Channel Parameter Format (14 bytes per output channel):**
```
Offset [0:1]  → M[14:0]     (15-bit unsigned multiplier)
Offset [2]    → S[5:0]      (6-bit shift amount)
Offset [3]    → reserved
Offset [4:5]  → zp[15:0]    (output zero point)
Offset [6:13] → bias_q[63:0] (bias - typically 0 for pooling)
```

---

### 1.6 Tiling and Scratchpad Mapping

For large feature maps, pooling is tiled to fit in Activation Scratchpad:

```
Tiling Parameters (from descriptor):
  tile_h, tile_w       - Output tile size
  tile_num_h, tile_num_w - Number of tiles

Per tile:
  Input needed: (tile_h × stride_h + (pool_h-1)) × (tile_w × stride_w + (pool_w-1)) × C × bytes_per_elem
  Output: tile_h × tile_w × C × bytes_per_elem
```

**Ping-Pong Buffering:**
- While computing Tile[i], DMA loads Tile[i+1] into opposite bank
- No explicit weight ping-pong needed (no weights for pooling)
- Both input and output can be in same Activation Scratchpad banks

---

### 1.7 Global Pooling

When `GLOBAL_POOL = 1`:
- Window size is entire spatial dimension: H_pool = input_h, W_pool = input_w
- Stride and kernel fields are ignored
- Output size: 1 × 1 × C per sample
- Used in classification networks for final spatial reduction

---

## 2. ELTWISE ADD OPERATOR ARCHITECTURE

### 2.1 Operator Type and Hardware Mapping

**OP_TYPE = 4 (OP_ELTWISE_ADD)**

| Aspect | Details |
|--------|---------|
| **Hardware Path** | PPU (ADD mode) - bypasses Systolic Array |
| **Operation** | Element-wise addition: `out[i] = rescale_A(in_A[i]) + rescale_B(in_B[i])` |
| **Data Type Support** | INT8 and INT16 |
| **Dual-Branch Rescaling** | Independent M/S/zp for each input branch |
| **Typical Use Case** | Residual connections, feature fusion |

**Key Insight:** Add uses the PPU's dedicated ADD mode, which processes two independent input branches with separate rescale parameters before summation.

---

### 2.2 Configuration Registers

#### LAYER_MODE (CSR 0x040)
```
Bits [3:0]: OP_TYPE = 4  (identifies this layer as Add)
Bit  [4]:   DATA_TYPE = 0 for INT8, 1 for INT16
```

#### DMA Configuration for Dual Inputs

| Register | Address | Purpose |
|----------|---------|---------|
| **DMA_IN_ADDR** | 0x100 | Primary input (branch A) base address |
| **DMA_ADD_B_ADDR** | 0x120 | Secondary input (branch B) base address |
| **POST_ADD_INPUT_ADDR** | 0x198 | Same as DMA_ADD_B_ADDR (dual-mapped) |

#### Add-Specific Stride Configuration

```
DMA_OUT_STRIDE (0x114):
  Bits [15:0]: OUT_STRIDE_ROW  - Row stride for output

POST_ADD_STRIDE (0x19C):
  Bits [15:0]: ADD_IN_STRIDE_ROW  - Row stride for branch B input
```

---

### 2.3 Rescale Parameter Configuration

#### POST_CTRL for Add Mode (CSR 0x180)

```
Bits [1:0]: PPU_MODE = 1  (ADD mode, not CONV_REQ)
Bit  [2]:   RELU_EN       (1=apply ReLU after add)
Bit  [7]:   INT16_OUT     (matches input data type)
```

#### Add Parameter Storage

**DMA_ADD_PARAM_ADDR (0x124) or POST_ADD_PARAM_ADDR (0x194)** - both dual-mapped

Points to 8 bytes of Add rescale parameters per layer:

```
Byte [0:1]  → M_A[14:0]     (Branch A multiplier)
Byte [2]    → S_A[5:0]      (Branch A shift)
Byte [3]    → reserved
Byte [4:5]  → M_B[14:0]     (Branch B multiplier)
Byte [6]    → S_B[5:0]      (Branch B shift)
Byte [7]    → reserved
```

**Note:** Add parameters are separate from per-channel params. Only 8 bytes per Add node (vs 14 bytes per channel for Conv). Can be loaded once at layer start.

---

### 2.4 Interface Specification (Layer Descriptor)

```c
struct layer_descriptor {
    uint8  op_type;           // = 4 (OP_ELTWISE_ADD)
    uint8  data_type;         // 0=INT8, 1=INT16
    uint16 in_h, in_w, in_c;  // Input A dimensions
    uint16 out_h, out_w, out_c; // Output (usually = in_h, in_w, in_c)
    
    // ... padding, tiling fields (mostly unused for element-wise add) ...
    
    uint8  post_ctrl;         // bit[1:0]=PPU_MODE (must be 1 for ADD)
    uint8  sched_ctrl;        // Scheduling hints
    int16  clamp_min, clamp_max;
    int8   in_zp;             // For padding if needed (usually not)
    uint16 param_ch_count;    // For per-channel params (0 for pure add)
    uint8  has_add;           // = 1 (followed by 8 bytes of add params)
    int8   residual_src;      // Source layer index for input B (-1 if not used)
    int16  input_src;         // Source layer index for input A (-1 = previous layer)
    
    // Followed by (if has_add):
    // [8 bytes] add_param_t with M_A, S_A, M_B, S_B
};
```

---

### 2.5 Add Computation Pipeline

#### Input Data Flow

```
Branch A (from previous layer output or external SRAM)
    ↓ DMA_IN_ADDR → Activation Scratchpad
    
Branch B (from another layer output, typically residual connection)
    ↓ DMA_ADD_B_ADDR → Activation Scratchpad (different bank or location)

Control Unit: Load both branches into Scratchpad in parallel (if different sources)
    or sequentially if tiling needed
    
PPU reads both branches, applies rescaling, then adds
```

#### PPU Add Mode Computation

**Per Element Computation:**
```
rescaled_A = (A × M_A) >> S_A  (with rounding)
rescaled_B = (B × M_B) >> S_B  (with rounding)
sum = rescaled_A + rescaled_B
clipped = clamp(sum, clamp_min, clamp_max)
output = ReLU(clipped) if RELU_EN else clipped
```

**Hardware Implementation:**
- PPU operates in parallel on 16 output channels per cycle (16×PPU parallelism)
- Each PPU pipeline element receives one pair of (A[ch], B[ch]) values
- Applies dual rescale in parallel, then addition
- Results clamp and activate before writing to Activation Scratchpad

---

### 2.6 DMA Configuration for Add

#### Dual-Source Loading

```
Example: Residual Add in MobileNet
  Main branch input:     DMA_IN_ADDR    (e.g., 0x20000000)
  Skip/Residual input:   DMA_ADD_B_ADDR (e.g., 0x20004000)
  Output:                DMA_OUT_ADDR   (e.g., 0x20008000)

  DMA_IN_SIZE:    tile_h × tile_w × C_in × 2 bytes
  DMA_OUT_SIZE:   tile_h × tile_w × C_out × 2 bytes (usually = in_size)
```

#### No Weight DMA

```
DMA_WEIGHT_ADDR:  Unused (0 or ignored)
DMA_WEIGHT_SIZE:  0 (no weights for Add)
```

---

### 2.7 Tiling for Add

Input dimensions for both branches must be identical (element-wise requirement):

```
Tiling constraint:
  in_h = out_h
  in_w = out_w
  in_c = out_c (same channels for both inputs)

Tile size: tile_h × tile_w × C (spatial only, no channel tiling needed)

Both branches loaded into same or adjacent tile regions in Scratchpad:
  Branch A: [base_A, base_A + tile_size)
  Branch B: [base_B, base_B + tile_size) - different region or ping-pong bank
```

---

### 2.8 Residual Connection Integration

In the descriptor, residual input source specified:

```
int8 residual_src;  // Layer index (-1 if not used)
```

**Execution Flow:**
1. Control Unit identifies Add layer with residual_src ≠ -1
2. Loads output of layer `residual_src` into DMA_ADD_B_ADDR during tiling setup
3. May reorder DMA timing: fetch both branches before starting computation
4. Or, if fusion enabled (FUSE_* flags), keep residual in Scratchpad from previous layer

---

## 3. COMPARATIVE SUMMARY: Pooling vs Add vs Systolic Array

| Feature | Pooling | Add | Systolic Array (Conv/FC) |
|---------|---------|-----|---------|
| **Hardware Module** | Pooling Logic | PPU (ADD mode) | Systolic Array (16×16 PE) |
| **Weight Storage** | None | None | Weight Buffer (32-64KB) |
| **Computation Type** | Sliding window reduction | Element-wise fusion | Matrix multiply |
| **Parameter Need** | Per-channel requantize | Dual rescale | Per-channel requantize |
| **Parallelism** | 16 ch/cycle (PPU-limited) | 16 ch/cycle (PPU-limited) | 256 MACs/cycle (16×16 SA) |
| **Output Pathway** | → PPU → Activation SRAM | → PPU → Activation SRAM | → PPU → Activation SRAM |
| **Typical Ops/sec** | ~0.2-1 GOPS | ~0.2-1 GOPS | ~200 GOPS (peak) |

**Key Architectural Principle:** Pooling and Add are I/O-bound post-processing operations that fit naturally into the PPU datapath designed for requantization. Using dedicated hardware (vs. trying to use Systolic Array inefficiently) maximizes throughput per Watt.

---

## 4. PPU (POST-PROCESSING UNIT) SHARED DATAPATH

### 4.1 PPU Architecture Overview

The PPU is the unified post-processing engine serving:
1. **CONV_REQ mode**: Requantize Conv/DW outputs
2. **ADD mode**: Add dual-rescale + requantize
3. **RELU_ONLY mode**: Pure activation (no requantization)
4. **PASS_THROUGH mode**: Bypass (identity)

### 4.2 PPU Pipeline Stages

```
Stage 1: Input MUX (select source: Systolic Array / DW Conv / Pooling Logic / Add)
Stage 2: Parameter lookup (fetch M[ch], S[ch], zp[ch] from Parameter SRAM)
Stage 3: Multiply (acc × M or rescale branches)
Stage 4: Shift (>> S with rounding)
Stage 5: Add/Clamp (+ zp, clamp to range, optional +bias)
Stage 6: Activation (ReLU / ReLU6 / LUT output)
    ↓
Output register → Write to Activation Scratchpad
```

**Throughput:** 1 cycle latency, 1 output channel per cycle per PPU instance.  
**Parallelism:** 16 instances (PPU×16), one per output column → 16 channels per cycle total.

### 4.3 How Pooling Integrates

```
Pooling Logic (slide window on Activation SRAM)
    ↓ Extract POOL_H × POOL_W window
    ↓ Apply max() or sum()
    ↓ 40-bit accumulator result (treated as pre-quantized)
    ↓ Write to PPU input MUX (one result per cycle, 16 results across 16 PPU units)
    ↓ PPU applies requantize pipeline
    ↓ Output written back to Activation SRAM
```

### 4.4 How Add Integrates

```
Branch A data → PPU
Branch B data → PPU
    ↓ Multiplexed to dual rescale stage
    ↓ PPU_MODE[1:0] = 1 (ADD mode) routes to addition logic
    ↓ rescaled_A + rescaled_B
    ↓ Optional clamp/ReLU
    ↓ Output to Activation SRAM
```

---

## 5. CONTROL FLOW AND EXECUTION SEQUENCING

### 5.1 Dispatch Logic (Control Unit)

From `architecture-spec.md`:

```
┌────────────────────────────────────────────────────────────┐
│  mode (from CSR)        Dispatch                           │
├────────────────────────────────────────────────────────────┤
│  CONV2D / FC            → Systolic Array (im2col mapping)  │
│  DW_CONV                → DW Conv Module (bypass SA)       │
│  POOLING (avg/max)      → Pooling Logic (复用PPU datapath)  │
│  ADD (element-wise)     → PPU (ADD mode, bypass SA)        │
│  RELU_ONLY              → PPU (RELU_ONLY mode)             │
│  RESIZE (nearest/bilinear) → Resize FSM + PPU + RMW WB    │
│  CONCAT                 → DMA only (重排写地址)             │
└────────────────────────────────────────────────────────────┘
```

### 5.2 State Machine for Pooling Execution

```
IDLE
  ↓ (START triggered)
LOAD_INPUT  (DMA fetches input tile from external SRAM → Activation SRAM Bank[0])
  ↓
POOL_COMPUTE  (Pooling Logic slides window, produces results stream)
  ↓ Each window result → PPU pipeline
  ↓
PPU_PROCESS (Requantize pipeline 6-stage: bias, mul, shift, zp, clamp, relu)
  ↓ Results stream into Activation SRAM Bank[1]
  ↓
DMA_WRITEOUT (DMA transfers output tile from Bank[1] → external SRAM)
  ↓
Ping-pong flag toggle
  ↓ (More tiles?) → LOAD_INPUT else → DONE
DONE  (Interrupt CPU)
```

### 5.3 State Machine for Add Execution

```
IDLE
  ↓ (START triggered)
LOAD_BOTH_INPUTS  
  ├─ DMA loads Branch A → Activation SRAM Bank[0]
  └─ DMA loads Branch B → Activation SRAM Bank[1] (or same bank different region)
  ↓
ADD_COMPUTE  
  ├─ PPU_MODE set to 1 (ADD mode)
  ├─ Branch A data → PPU, Apply rescale M_A, S_A
  ├─ Branch B data → PPU, Apply rescale M_B, S_B
  ├─ Addition: A' + B'
  └─ Optional ReLU
  ↓
PPU_OUTPUT → Activation SRAM Bank for output (or directly write)
  ↓
DMA_WRITEOUT (Transfer output to external SRAM)
  ↓ (More tiles?) → LOAD_BOTH_INPUTS else → DONE
DONE  (Interrupt CPU)
```

---

## 6. KEY HARDWARE DESIGN DECISIONS

### 6.1 Why Reuse PPU for Pooling?

| Reason | Benefit |
|--------|---------|
| **Operator frequency** | Pooling appears in many CNN architectures but not in hot path |
| **Datapath alignment** | Both Pooling and Conv outputs need requantization |
| **Silicon efficiency** | Dedicated Pooling hardware < reusing existing PPU |
| **Design simplicity** | Unified requantize pipeline, no duplicated logic |

**Trade-off:** Pooling throughput is limited to 16 channels/cycle (PPU parallelism) vs. potential dedicated pipelined version. But for edge inference (single-model deployment), this is acceptable.

### 6.2 Why Separate Add Mode?

| Reason | Benefit |
|--------|---------|
| **Dual-input requirement** | Add needs independent rescaling for each branch |
| **Residual fusion opportunity** | Can avoid writing/reading intermediate tensor |
| **Common pattern** | ResNet, MobileNetV3, etc. use residual adds everywhere |
| **Parameter reuse** | M_A/S_A/M_B/S_B fit in 8 bytes, separate from Conv params |

### 6.3 Activation Scratchpad as Central Hub

All post-SA computation (Pooling, Add, even ReLU-only) reads/writes Activation Scratchpad:

```
Systolic Array → {Pooling Logic, Add PPU, DW Output} → Requantize PPU → Activation SRAM
```

This avoids external memory round-trips for intermediate tensors, enabling layer fusion.

---

## 7. QUANTIZATION AND REQUANTIZATION

### 7.1 Pooling Quantization

**Max Pool:** Integer result directly from window max, no precision loss in pooling itself.

**Avg Pool:** Integer division (or scaled integer arithmetic) produces sum that needs rescaling.

Both require per-channel requantization because:
- Input may have input_scale ≠ output_scale
- Channels may have different quantization parameters

### 7.2 Add Quantization

Independent scales for each branch:
```
A_quantized ∈ [int8_min, int8_max] with scale S_A
B_quantized ∈ [int8_min, int8_max] with scale S_B

Dequantized sum: (A_q × S_A) + (B_q × S_B)

Re-quantize to output: out_q = (sum / S_out), rounded
```

PPU implements: `out = round((A × M_A >> S_A) + (B × M_B >> S_B)) + zp`

Where M_A, M_B, S_A, S_B chosen at compile time to match quantization.

---

## 8. HARDWARE RESOURCE UTILIZATION

### 8.1 PPU Area Breakdown

Per PPU instance (one of 16):
- Multiplier: ~100 LUT
- Shift/Round: ~80 LUT
- Adder (+bias, +zp, clamp): ~150 LUT
- Activation (ReLU/LUT MUX): ~120 LUT
- Control FSM + registers: ~100 LUT
- **Total per PPU:** ~550 LUT

**Total (16×PPU):** ~8,800 LUT, represents ~15% of total NPU area.

### 8.2 Pooling Logic Area

Estimate (not explicitly quantified in spec, inferred):
- Window sliding buffers (line buffers): ~200 LUT (holds POOL_H×POOL_W elements)
- Max comparator tree: ~300 LUT (16 parallel max trees for 16 channels)
- Avg accumulator: ~150 LUT (16 parallel accumulators)
- Control: ~100 LUT
- **Estimated:** ~750 LUT (modest, supports ~0.2 GOPS throughput)

### 8.3 Add Datapath Area

Incorporated into PPU; no separate hardware:
- MUX for dual inputs: ~50 LUT (already counted in PPU)
- ADD logic: dual rescale stage part of PPU pipeline

---

## 9. PERFORMANCE CHARACTERISTICS

### 9.1 Pooling Performance

| Metric | Value |
|--------|-------|
| **Throughput (channels)** | 16 ch/cycle (PPU limited) |
| **Latency** | ~6 cycles (PPU pipeline stages) + window computation |
| **Typical latency per output pixel** | ~7-10 cycles |
| **Memory BW requirement** | SRAM internal (input + output tiles) |

Example (8×8 tile, 2×2 pool, stride 2):
- Input: 8×8 = 64 pixels/tile
- Output: 4×4 = 16 pixels/tile
- PPU drain: 16 outputs × ~7 cycles ≈ 112 cycles per tile
- DMA overhead: minimal (ping-pong hides it)

### 9.2 Add Performance

| Metric | Value |
|--------|-------|
| **Throughput (channels)** | 16 ch/cycle (same as Conv requantize) |
| **Latency** | ~6 cycles (PPU pipeline stages) |
| **Typical latency per output pixel** | ~6-8 cycles |
| **Memory BW requirement** | 2× input (dual branches) + output |

Example (14×14 tile, 256 channels):
- Input A: 14×14×256 × 2B = 100 KB/tile (doesn't fit in tile SRAM if too large)
- Input B: same 100 KB
- Output: 100 KB
- With ping-pong: 200 KB activation SRAM sufficient for two 100 KB banks

---

## 10. USAGE EXAMPLES

### 10.1 Configuring Max Pool Layer (CPU C code)

```c
// Layer: 2×2 MaxPool with stride 2, input 28×28×64, output 14×14×64

// 1. Layer type and dimension configuration
REG(LAYER_MODE)    = (0 << 4) | 3;  // INT8, OP_POOLING
REG(IN_DIM_H_W)    = (28 << 16) | 28;
REG(IN_DIM_C)      = 64;
REG(OUT_DIM_H_W)   = (14 << 16) | 14;
REG(OUT_DIM_C)     = 64;

// 2. Pooling configuration
REG(POOL_CFG)      = (0 << 20)   // GLOBAL_POOL = 0
                   | (2 << 16)   // POOL_STRIDE_W = 2
                   | (2 << 12)   // POOL_STRIDE_H = 2
                   | (2 << 8)    // POOL_W = 2
                   | (2 << 4)    // POOL_H = 2
                   | (0 << 0);   // POOL_MODE = MaxPool

// 3. Padding (typically 0 for pooling)
REG(PADDING)       = 0;

// 4. DMA configuration
REG(DMA_IN_ADDR)   = 0x20000000;   // Input address
REG(DMA_OUT_ADDR)  = 0x20010000;   // Output address
REG(DMA_IN_SIZE)   = 28 * 28 * 64 * 1;  // INT8
REG(DMA_OUT_SIZE)  = 14 * 14 * 64 * 1;

// 5. Per-channel requantize parameters
REG(DMA_PARAM_ADDR)    = 0x08000000;
REG(POST_PARAM_COUNT)  = 64;

// 6. Post-processing: requantize + optional ReLU
REG(POST_CTRL)     = (1 << 6)    // BIAS_EN = 0 (no bias)
                   | (0 << 5)    // ZP_EN = 0 (symmetric)
                   | (1 << 2)    // RELU_EN = 1 (apply ReLU)
                   | (0 << 7)    // INT16_OUT = 0 (INT8)
                   | 0x00;       // PPU_MODE = CONV_REQ

REG(POST_CLAMP)    = (127 << 16) | (uint16_t)(-128);  // INT8 range

// 7. Kick off computation
REG(NPU_CTRL)      = 0x01;  // START

// 8. Wait for completion
while (!(REG(NPU_IRQ_STATUS) & 0x01));
REG(NPU_IRQ_STATUS) = 0x01;  // Clear interrupt
```

### 10.2 Configuring Eltwise Add Layer (CPU C code)

```c
// Layer: Residual Add, 14×14×256 (branch A + branch B → output)
// Branch A from previous layer (DMA_IN_ADDR)
// Branch B from skip connection (DMA_ADD_B_ADDR)

// 1. Layer type and dimensions
REG(LAYER_MODE)    = (1 << 4) | 4;  // INT16, OP_ELTWISE_ADD
REG(IN_DIM_H_W)    = (14 << 16) | 14;
REG(IN_DIM_C)      = 256;
REG(OUT_DIM_H_W)   = (14 << 16) | 14;
REG(OUT_DIM_C)     = 256;

// 2. DMA: Branch A (primary input)
REG(DMA_IN_ADDR)      = 0x20000000;   // Main branch
REG(DMA_IN_SIZE)      = 14 * 14 * 256 * 2;  // INT16

// 3. DMA: Branch B (skip/residual input)
REG(DMA_ADD_B_ADDR)   = 0x20004000;   // Skip branch
//    (DMA_ADD_STRIDE could be different if non-standard layout)

// 4. Output address
REG(DMA_OUT_ADDR)     = 0x20008000;
REG(DMA_OUT_SIZE)     = 14 * 14 * 256 * 2;

// 5. Add rescale parameters (8 bytes)
REG(DMA_ADD_PARAM_ADDR)  = 0x08001000;  // M_A, S_A, M_B, S_B

// 6. Per-channel params (if output quantization differs from input)
REG(DMA_PARAM_ADDR)      = 0x08002000;
REG(POST_PARAM_COUNT)    = 256;  // 256 channels

// 7. Post-processing: PPU in ADD mode + ReLU
REG(POST_CTRL)     = (1 << 2)    // RELU_EN = 1
                   | (1 << 7)    // INT16_OUT = 1
                   | (0x01);     // PPU_MODE = ADD (dual rescale)

REG(POST_CLAMP)    = (32767 << 16) | (uint16_t)(-32768);  // INT16 range

// 8. Kick off
REG(NPU_CTRL)      = 0x01;  // START

// 9. Wait
while (!(REG(NPU_IRQ_STATUS) & 0x01));
REG(NPU_IRQ_STATUS) = 0x01;
```

---

## 11. DESIGN TRADE-OFFS AND LIMITATIONS

### 11.1 Pooling Limitations

| Limitation | Reason | Mitigation |
|-----------|--------|-----------|
| **16 ch/cycle throughput** | PPU parallelism; sliding window is inherently serial | Accept for edge inference; not critical path in CNN |
| **No weight data** | Good! Pooling is parameter-free | N/A |
| **Tiling overhead** | Large feature maps require multiple tiles | Compile-time tiling; pipeline hides DMA |

### 11.2 Add Limitations

| Limitation | Reason | Mitigation |
|-----------|--------|-----------|
| **Dual-input bandwidth** | Both branches must be loaded; can exceed SRAM I/O | Layer fusion (FUSE_* flags) avoids writing intermediate |
| **Synchronization latency** | Branch A and B must arrive synchronized to PPU | Control Unit coordinates tile loading |
| **Output quantization** | May differ from either input scale | Separate per-channel params handle mismatches |

### 11.3 PPU Throughput Constraints

```
Systolic Array:    256 MACs/cycle (16×16 = 200 GOPS @ 200MHz)
PPU (Pooling/Add): 16 outputs/cycle  (bottleneck for post-processing)

Ratio: 256/16 = 16× imbalance
```

**Implication:** Pooling/Add layers are I/O-bound, not compute-bound. For typical CNN workloads (few pooling/add layers), this is acceptable.

---

## 12. REFERENCES

**Source Documents:**

1. `/data/sam/open-npu/design/architecture-spec.md` - Core microarchitecture definitions
   - §8: Systolic Array Microarchitecture
   - §10: Control Unit Microarchitecture
   - §11: DW Conv Module Microarchitecture
   - §12: Post-Processing Unit (PPU) Microarchitecture
   - §13: Hardware Resource Utilization

2. `/data/sam/open-npu/design/npu-register-spec.md` - CSR Register Definitions
   - §4: Group 1 Layer Parameters (POOL_CFG at 0x060)
   - §5: Group 3 Post-Processing Config (POST_CTRL, DMA_ADD_*)
   - §9: Typical Usage Flows (examples 9.2 Add, 9.4 Sigmoid via LUT)

3. `/data/sam/open-npu/design/interface-spec.md` - Offline Model Binary Format
   - §2: Layer Descriptor (Fixed Config)
   - §5: POST_CTRL encoding (descriptor byte 47)
   - §7: Add Parameter Pack (8 bytes)
   - §8: Tiling Ownership

---

## Appendix: Register Map Quick Reference

**Pooling Key Registers:**
```
0x040: LAYER_MODE         → OP_TYPE=3, DATA_TYPE
0x044-0x050: Dimensions   → IN/OUT H/W/C
0x05C: PADDING            → PAD_TOP/BOTTOM/LEFT/RIGHT
0x060: POOL_CFG           → POOL_MODE, POOL_H/W, STRIDE_H/W, GLOBAL_POOL
0x100: DMA_IN_ADDR        → Input feature map base
0x104: DMA_OUT_ADDR       → Output feature map base
0x110: DMA_IN_STRIDE      → Input row/channel stride
0x128: DMA_IN_SIZE        → Input total bytes
0x130: DMA_OUT_SIZE       → Output total bytes
0x180: POST_CTRL          → PPU_MODE=0(CONV_REQ), RELU_EN, etc.
0x184: POST_PARAM_ADDR    → Per-channel params (M/S/zp/bias)
0x188: POST_PARAM_COUNT   → Number of output channels
0x18C: POST_CLAMP         → CLAMP_MIN/MAX
```

**Add Key Registers:**
```
0x040: LAYER_MODE         → OP_TYPE=4, DATA_TYPE
0x044-0x050: Dimensions   → IN/OUT H/W/C (same for both branches)
0x100: DMA_IN_ADDR        → Branch A base
0x104: DMA_OUT_ADDR       → Output base
0x120: DMA_ADD_B_ADDR     → Branch B base
0x124: DMA_ADD_PARAM_ADDR → Add rescale params (M_A/S_A/M_B/S_B)
0x19C: POST_ADD_STRIDE    → Branch B row stride
0x180: POST_CTRL          → PPU_MODE=1(ADD), RELU_EN, INT16_OUT
0x184: POST_PARAM_ADDR    → Per-channel output params (if needed)
0x198: POST_ADD_INPUT_ADDR→ Same as DMA_ADD_B_ADDR (dual-mapped)
0x194: POST_ADD_PARAM_ADDR→ Same as DMA_ADD_PARAM_ADDR (dual-mapped)
```

---

**Report Generation Date:** 2026-05-29  
**Status:** Complete - Ready for RTL/Driver Implementation Review
