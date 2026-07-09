# RTL-CSIM Bit-Exact 对齐调试记录

**日期：** 2026-06-20 ~ 2026-06-22  
**模型：** MODEL_B INT16 Layer 0 (Conv 7×7, 112×112×1 → 56×56×64, 28 tiles)  
**结果：** 2048/2048 words PASS (bit-exact)  
**仿真次数：** ~12 次 Verilator E2E (每约 16 分钟)

---

## 修复概览

### RTL (5 bugs)
| # | 文件 | Bug | 影响 |
|---|------|-----|------|
| 1 | `npu_top.v` | DB_EN 下 `effective_act_base` 缺少 ping_pong bank 偏移 | compute 始终读 Bank[0] |
| 2 | `npu_ctrl.v` | DMA `next_tile_word_off` 累积偏移溢出 | 多 tile 数据覆盖错位 |
| 3 | `npu_compute.v` | ~~PPU bias 包含 word[1][31:16]~~ **→ 回滚** | 原始正确，bias 跨 3 个 word |
| 4 | `npu_ppu.v` | `PROD_W=55` 太窄 (应为 56) | 大乘积截断为 0 |
| 5 | `npu_compute.v` | `next_pos_blk` 使用全图坐标而非 tile-local | 多 pass 激活地址错 |
| 6 | `npu_compute.v` | padding 硬编码为 0（非 in_zp） | 非零 in_zp 边界不对齐 |

### CSIM (3 bugs, 同一模式)
| # | 文件 | Bug | 根因 |
|---|------|-----|------|
| 1 | `conv2d.c/dwconv.c/fc.c` | `trunc40` 零扩展 | `(int64_t)((uint64_t)v << 24 >> 24)` 无符号扩展 |
| 2 | `postproc.c` | `trunc_40bit` 零扩展 | 同上 |
| 3 | `postproc.c` | 56-bit 乘积截断零扩展 | `(int64_t)((uint64_t)v << 8 >> 8)` 无符号扩展 |
| 4 | `postproc.c` | 55→56 bit 宽度修复 | 匹配 RTL PROD_W=56 |

---

## 零扩展 Bug 详解（CSIM 核心问题）

`(int64_t)((uint64_t)v << N >> N)` 在 uint64 上执行：
1. `<< N` — 左移 N 位，低 N 位补 0
2. `>> N` — uint64 右移为逻辑移位，高 N 位补 0
3. `(int64_t)` — 仅重新解释位模式，**不做符号扩展**

结果是提取了低 (64-N) 位并**零扩展**为 int64。负数的位 39（或 55）为 1，但在零扩展后变为正大数。

**修复：** 在掩码后增加算术右移符号扩展：
```c
v <<= N; v >>= N;  // sign-extend bit (64-N-1)
```

该模式在 CSIM 中出现 3 次，全部修复后与 RTL 对齐。

---

## 调试历程

```
初始状态:  832/2048 (40.6%)  ← 黄金数据不一致 (不同 model conversion)
  ↓ 统一 golden 数据 + RTL bug 1-2 修复
           832/2048 (40.6%)  ← 仍有 mismatch
  ↓ RTL bug 4-5 修复 + CSIM bug 1 修复
           832/2048 (40.6%)  ← 不影响当前像素
  ↓ RTL bug 3 回滚 (bias 原始正确)
           105/2048 ( 5.1%)  ← 大幅改善! 87% 减少
  ↓ CSIM bug 2 修复 (postproc 40-bit)
           105/2048 ( 5.1%)  ← 不变
  ↓ CSIM bug 4 修复 (55→56 宽度)
           105/2048 ( 5.1%)  ← 不变
  ↓ CSIM bug 3 修复 (56-bit 符号扩展)
             0/2048 ( 0.0%)  ← ALL PASS! ✓
  ↓ RTL bug 6 修复 (in_zp padding)
             0/2048 ( 0.0%)  ← 回归通过
```

### 关键转折点
- **105→0**：56-bit 乘积零扩展修复。对于 S=48 等大移位量通道，负乘积被零扩展纠正数 ÷ 2^48 产生非零输出(210)，RTL 正确产生 0。修复后 CSIM 也产出 0，对齐。

---

## Commit 记录

### RTL (`open-npu/rtl`)
```
1a2fa6f fix: use cfg_in_zp for activation padding instead of hardcoded 0
d57d411 test: update MODEL_B INT16 layer 0 golden to match fixed CSIM
993f80b fix: revert incorrect PPU bias fix — bias spans words [1:3]
5d844a8 fix: 5 critical RTL bugs from DB_EN and PPU debug session
```

### CSIM (`open-npu/csim`)
```
bbf7bed csim: fix 56-bit product truncation sign-extension in postproc
bd20ed8 csim: add debug traces + postproc trunc_40bit sign-ext fix
2c1424b csim: fix trunc40 sign-extension + postproc 55→56 bit
```

---

## 已知局限

| 项目 | 状态 |
|------|------|
| MODEL_B Layer 0 INT16 | ✅ PASS |
| MODEL_B 其他层 (Pooling, DW, FC...) | 未测试 |
| INT8 模式 | 未测试 |
| 其他模型 (MODEL_A, MobileNet...) | 未测试 |
| DW Conv 独立路径 | 未测试 |
| 非零 in_zp 模型 | RTL 已修复，待模型验证 |
