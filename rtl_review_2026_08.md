# Open-NPU RTL 审查报告（2026-08）

范围：`rtl/src/` 全部 9 个模块，交叉核对 `csim/`、`tools/model_packer.py`、
`soc/firmware/main.c`、`driver/baremetal/`、`design/npu-register-spec.md`。

分级：

- **A 级** —— 已实测/已验证，会真的咬人，或直接挡住既定路线（缩小 SRAM、改阵列尺寸、上 FPGA）
- **B 级** —— 软件当前兜住了，硬件不自保；换个前端/驱动就会踩
- **C 级** —— 结构上不安全但当前时序碰不到，或纯文档/面积问题
- **已排除** —— 审查中提出但核实后不成立

---

## A 级

### A1. 脉动阵列实测利用率 0.82%

`npu_compute.v:1257` 的 `S_ACT_EMIT` 每拍只往第 `act_cnt` 行放一个标量，其余 15 行喂 0。

实测（`rtl/tb/integration/test_array_util.py`，Icarus）：

| 形状 | 周期 | 层 MAC | 有效 MAC/拍 | 利用率 |
|---|---|---|---|---|
| 8×8×16→16 k3 | 58,884 | 147,456 | 2.10 | **0.82%** |
| 8×8×16→16 k1 | 7,528 | 16,384 | 2.18 | 0.85% |
| 8×8×16→32 k1 | 15,052 | 32,768 | 2.18 | 0.85% |

峰值 256 MAC/拍。周期去向：drain 路径（`S_DRAIN_WAIT`+`S_REDUCE`）48%，
喂数路径（`S_ACT_EMIT`+`S_ACT_FLUSH`+`S_ACT_LOAD`）35%，权重装载 12%。

`tools/internal/perf_model.py` 假设 256 MAC/拍，**比 RTL 乐观约 120 倍**。
优化路线见 `rtl_design_overview.md` §5.10（都是调度改动，不影响 bit-exact）。

---

### A2. `npu_ctrl` 的容量守卫混用宏和参数 —— 缩小 SRAM 会静默越界

`npu_ctrl.v:21-23` 定义了模块参数：

```verilog
parameter integer ACT_DEPTH   = `SPAD_KB * 64,
parameter integer WGT_DEPTH   = `SPAD_KB * 128,
parameter integer PARAM_DEPTH = `SPAD_KB * 16
```

`npu_top` 确实把正确的深度传下来了。但守卫里**一半用参数、一半直接用宏**：

| 用参数（正确跟随） | 用 `` `SPAD_KB `` 宏（永远是 192） |
|---|---|
| `:175` `:594` E5 | `:156` `param_needs_reload` |
| `:609` E8 | `:224` `:236` slice_stream 判定 |
| `:612` E9 | `:242` `grp8_mode` 判定 |
| | `:561` E1、`:566` E2 |

**后果**：把 `npu_top` 用 `SPAD_KB=64` 例化而不改 `npu_defines.vh` 里的 `` `define ``，
E1/E2 和 slice_stream / grp8 判定仍按 192KB 放行，DMA 写穿实际只有 1/3 大的 SRAM，
**不报 `hw_error`**。这正好挡在"把 SRAM 缩到 52KB"这条路上。

修法：把这 6 处宏换成模块参数。

---

### A3. `npu_systolic` drain MUX 硬编码到 16 列 —— 改阵列尺寸会静默出错

`npu_systolic.v:240-257`：

```verilog
case (drain_col_sel)
    4'd0:  acc_mux = pe_acc_out[r][0];
    ...
    4'd15: acc_mux = pe_acc_out[r][15];
    default: acc_mux = {ACC_W{1'b0}};
endcase
```

`ARRAY_SIZE=32` 时 `drain_col_sel` 变 5 位，第 16–31 列全部落到 `default` 输出 0。
不报错，结果直接是错的。应改成 generate 循环生成的通用 MUX。

同一类的 `ARRAY_SIZE≠16` 硬编码还有一批：

| 位置 | 常量 | 含义 |
|---|---|---|
| `npu_compute.v:440` | `grp_oc = 5'd16` | 输出通道分组宽度 |
| `npu_compute.v:376` | `px_acc_buf [0:255]` | 16 像素 × 16 列；`ARRAY_SIZE=32` 时列索引 `[3:0]` 会把 16–31 列混叠到 0–15 |
| `npu_compute.v:308` | `param_cache [0:63]` | 16 通道 × 4 字；`ARRAY_SIZE=32` 需要 128 |
| `npu_ctrl.v:736,1079` | `16'd64` | param 重载字数 = 16ch × 4 |
| `npu_ctrl.v:1077` | `* 32'd256` | oc_group 的 param 字节步长 = 16ch × 16B |

**结论：`ARRAY_SIZE` 现在实际上不是可配置的。** 要么修完这一批，要么在
`npu_defines.vh` 里加一句"仅支持 16"并加 elaboration 断言。

---

### A4. `HW_CONFIG` 的 SPAD 字段单位，RTL/固件 与 规格书/驱动 不一致

`npu_csr.v:192-201` 把 `SPAD_KB` 原值（192）打进 `[23:16]`，注释却写 "units of 4KB"。

| 谁 | 怎么解读 `[23:16]` |
|---|---|
| `npu_csr.v` 实际打包 | 原始 KB 数（192） |
| `soc/firmware/main.c:95` | 当 KB 读 ✅ 与 RTL 一致 |
| `design/npu-register-spec.md` | 4KB 单位 |
| `driver/baremetal/npu_driver.c:59` | `HW_CFG_SPAD_4KB(v) * 4` ❌ 得到 768KB |

固件路径自洽，**baremetal 驱动路径错 4 倍**。上 FPGA 走驱动就会踩。
三方选一个定死（建议按规格书打 `SPAD_KB/4`，同时改固件），并加一条回归。

---

### A5. `POST_CLAMP` 的 max/min 位域，固件 与 驱动 不一致

`npu_top.v:616` 把 `reg_post_clamp[15:0]` 当 `clamp_max` 接给 PPU。

| 谁 | max 放哪 |
|---|---|
| `soc/firmware/main.c:376` | `[15:0]`（注释明写 "clamp_max in [15:0]"）✅ 与 RTL 一致 |
| `driver/baremetal/npu_hal.h:193` `POST_CLAMP_PACK(min,max)` | min 在 `[15:0]`，max 在 `[31:16]` ❌ |

也就是说**用 baremetal 驱动按规格书打包描述符，PPU 会把 CLAMP_MIN 当成 ReLU6 的上界**。
现在跑得通纯粹因为固件绕开了驱动。

顺带：`clamp_min` **根本没接到 PPU**。`npu_ppu.v` 的下界是写死的 −128 / −32768。
转换器目前总是设 `clamp_min = qmin`，值刚好对得上，但寄存器是个摆设。

---

### A6. 没有累加器溢出守卫

`npu_ctrl.v` 有 E1–E9 九条守卫（param SRAM、act/wgt SRAM 容量、k_depth 16 位、
AvgPool 倒数 LUT），**唯独没有一条检查 44 位累加器**。

44 位有符号范围 ±8.8×10¹²。INT16 最坏情况每项 ~2³⁰：

| 层 | MAC 数 | 最坏累加 | 结果 |
|---|---|---|---|
| 3×3×512 | 4,608 | 4.9×10¹² | 安全 |
| 3×3×1024 | 9,216 | 9.9×10¹² | **溢出**（勉强越界） |
| 5×5×1024 | 25,600 | 2.7×10¹³ | **溢出** |
| 1×1×65535（E6 放行） | 65,535 | 7.0×10¹³ | **溢出** |

PE 里是裸的 `acc_reg <= acc_reg + (act_in * weight_reg)`，没有饱和也没有 sticky 标志，
溢出静默回绕。建议加 E10：`k_depth × 2^(2·bits-2) > 2^(ACC_W-1)` 时报错。

---

### A7. `AUTO_NEXT` 不清双缓冲 / tiling 状态

`npu_ctrl.v:1298` 的 `S_DONE` 自动跳回 `S_LOAD_WGT`，但**不复位**
`tile_x_seq`、`tile_y_seq`、`ping_pong_flag`、`prefetch_active`、`prefetch_pending`、
`db_prefetch_done`、`next_tile_ddr_addr`、`store_bank`、`r_tile_*`。

现在没暴露，是因为：

- 固件走的是每层软复位（`main.c:487`），不用 AUTO_NEXT
- 唯一的 E2E 测试 `test_auto_next_3layer` 是 3 层 untiled 1×1 Conv，**碰不到 DB/tiling 状态**

但 `driver/baremetal/npu_hal.h:59` 把 `CTRL_AUTO_NEXT` 暴露出去了。
FPGA 上要靠 AUTO_NEXT 降低 CPU 干预时，第一个带 tiling 的多层模型就会拿到错误的
bank / tile 索引，或者卡死在 `S_TILE_WAIT_DB`。

---

### A8. `npu_dw_conv` 的复位循环 → 6.5 万个触发器

```verilog
// npu_dw_conv.v:66
for (wi = 0; wi < MAX_KSZ*MAX_KSZ; wi = wi + 1)
    weights[wi] <= {DATA_W{1'b0}};
```

`MAX_KSZ=16` → 256 项 × 16 位。异步复位遍历整个数组，综合工具**没法推成 RAM**，
只能推成 4096 个触发器；例化 16 个通道就是 **65536 个触发器**。

`weights` 的访问模式（单写口 + 组合读）本来是标准的分布式 RAM。
去掉复位循环即可 —— 权重在用之前一定会被 `wgt_load` 写满，不需要复位值。

对"成本敏感、SRAM 要做小"的目标来说，这是白扔的面积，而且比 SRAM 本身还贵。

---

### A9. 小 SPAD 的硬下限：`DW_STREAM_OUT_BASE`

```verilog
// npu_defines.vh:46
`define DW_STREAM_OUT_BASE (`SPAD_KB * 64 - 1024)
```

全局池化 / 全局 DW 的输出固定放在 act SRAM 顶部，预留 1024 字。

- `SPAD_KB = 16` → base = 0，输出直接盖住 SRAM[0] 的输入切片
- `SPAD_KB < 16` → 负数回绕

E3 守卫只检查"切片 ≤ base"，不检查 base 本身是否合理。
**这是 `SPAD_KB` 的实际下限**。计划的 `SPAD_KB=64` 有余量（base=3072），但要写进文档。

---

### A10. `npu_ctrl` 无法通过 Yosys 综合，而回归关卡是空的

两个问题叠在一起，互相掩盖了。

**其一**：`rtl/Makefile` 的 `syn` recipe 末尾是 `yosys ... 2>&1 | tee synth.log`。
make 拿到的是管道最后一个命令（`tee`）的退出码，**永远为 0**。
`phase1_regression.py` 里那道 `run("RTL synthesis", ["make", "syn"], RTL)` 因此从不失败。
已修：改为先写日志、失败时 `tail` 出错误并 `exit 1`。

**其二**：真正的错误是

```
ERROR: Multiple edge sensitive events found for this signal!   (npu_ctrl.wgt_reload_done)
```

根因是 `npu_ctrl.v:326` 和 `:494`：

```verilog
always @(posedge clk or negedge rst_n) begin
    if (!rst_n || ctrl_soft_rst) begin   // ← ctrl_soft_rst 不在敏感列表里
```

软复位是同步信号（CSR 位），却折进了异步复位分支。Yosys 判定为混合边沿事件，
直接拒绝整个模块。**全仓库只有 `npu_ctrl.v` 用了这个写法**，其余 8 个模块单独综合都是 0 错误。

修法（保行为）：

```verilog
    if (!rst_n) begin
        <异步复位赋值>
    end else if (ctrl_soft_rst) begin
        <同样的赋值，同步生效>
    end else begin
```

**在此之前，"这份 RTL 能综合"这句话是没有证据的。** 已修复并验证。

**其三**（修前两条时暴露出来的）：把 `make syn` 变成真关卡后，回归卡在综合上
超过一小时。原因是 `synth` 的 `memory_map` pass 把 192KB SPAD 展开成约 150 万个
触发器再丢给 ABC。这在物理上也没意义——SPAD 在任何真实流程里都是 SRAM 宏或 BRAM。

已拆成两个目标：

| 目标 | 内容 | 耗时 | 用途 |
|---|---|---|---|
| `make syn` | `hierarchy; proc; opt_clean; check -assert` | ~31s | 回归关卡 |
| `make syn-area` | 完整 tech-map，`blackbox npu_sram` | 分钟级 | 面积分析 |

快速关卡足以抓到上面那个 `npu_ctrl` 错误（它在 `proc` 阶段就报）。

---

### A11. `npu_sram` 注释与实现相反

文件头写 "Write-first: if read and write to same address on same port,
read returns new data"，代码是

```verilog
mem[a_addr] <= a_wdata;
a_rdata     <= mem[a_addr];   // 非阻塞 → 拿到旧值
```

实际是 **read-first**。`npu_dma` 的 store 路径和 `npu_compute` 的四处
读-改-写（pool / add / DW / resize）都是按 read-first 写的，行为一致 ——
**只有注释是错的**，但按注释改代码会出事。

---

## B 级：软件兜着，硬件不自保

| # | 位置 | 现象 | 当前为什么没事 |
|---|---|---|---|
| B1 | `npu_compute.v:2151` | DW 卷积 padding 喂字面 0，不是 `cfg_in_zp`（Conv 路径 `:1220` 是对的） | 转换器走对称量化，`in_zp` 恒为 0。加非对称量化立刻炸 |
| B2 | `npu_ppu.v:187` | INT8 + ReLU6 时 `out_data <= {{8{clamp_hi[7]}}, clamp_hi[7:0]}`，只取低 8 位 | 转换器 `cfg.clamp_max = min(qmax, relu6_qmax)` 保证 ≤127。驱动写 200 进去会输出 −56 |
| B3 | `npu_ctrl.v:204` | `skip_act_load`/`skip_store` 在 `cfg_tile_h!=0` 时静默失效，不报错 | 固件已在 `main.c:444` 主动 strip FUSE 位。直接写 CSR 会静默损坏数据 |
| B4 | `npu_compute.v:155` | AvgPool 倒数 LUT 只覆盖 {1..16, 25, 36, 49, 64}，`default` 回落到 1/256 | E7 守卫会拦住未列举的 count。但 LUT 空洞（17–24、26–35…）意味着这些窗口尺寸**不被支持**，不是"精度差一点" |
| B5 | `npu_compute.v:2008` | `cfg_kernel_h[3:0] * cfg_kernel_w[3:0]`，kh/kw ≥16 会回绕 | `MAX_KSZ=16` 的注释宣传 16×16 DW，实际 `[3:0]` 到 15 就满了。转换器不产生 ≥16 的核 |
| B6 | `npu_csr.v:342+` | 引擎 busy 时写层/DMA/PPU 寄存器立即生效，无写保护 | 固件不这么干。中断驱动的驱动很容易踩 |
| B7 | `npu_dma.v` | `wb_ack_i` 不来就永久挂住，无超时、无错误上报 | 规格书定义了错误码 4「总线超时」，RTL 没实现。仿真里 DDR 总会 ack；FPGA 上真会挂 |

---

## C 级：结构不安全但当前碰不到 / 文档 / 面积

- **C1** `npu_ctrl.v:809` PTS 的 `tile_done` 分支带 `!prefetch_active` 条件。
  理论上"compute 比 prefetch 先完成"会丢掉一次 tile store。当前时序下 prefetch 总是
  先完成（compute 慢得多，见 A1），所以碰不到 —— 但这个安全裕度来自性能缺陷，
  A1 优化后要重新审。
- **C2** `npu_ctrl.v:809-921` `S_WAIT_COMP` 里同一拍可能有多处 `state <=` 赋值，
  靠源码顺序决定胜者。prefetch 完成与 `tile_done` 同拍时会跳过 PTS store。同上，
  当前时序碰不到。
- **C3** `npu_ctrl.v:821` PTS 行长回退分支 `r_tile_row_len <= tile_row_len * tile_row_count`
  之后又乘一次高度，长度会平方。仅在 `tile_out_w_actual==0` 时触发，实际由 compute 常驱动。
- **C4** `npu_ctrl.v:494-536` 软复位块有重复赋值，且漏了 `store_bank`、`r_tile_ddr_offset`、
  `r_nhwc_row_stride`、`r_tile_row_len/count`。
- **C5** `npu_compute.v:549-637` 复位覆盖不全：`feed_left`、`px_remaining`、`blk_px_cnt`、
  `tree_col`、`px_acc_buf`、`param_cache` 等都没清。整芯片复位没问题，软 abort 有风险。
- **C6** `npu_compute.v:1517-1694` `S_PARAM_LOAD → S_PPU_FEED → S_WRITEBACK` 这条约 180 行的
  旧 PPU 通路在 conv 流程里**不可达**（活路径走 `S_PPU_STREAM`）。维护陷阱，建议删。
- **C7** `npu_ppu.v` 注释写 40 位、字面量写 `56'sd...`，实际 `ACC_W=44`、`PROD_W=60`。
  位宽够用，功能对，但全是陈旧信息。
- **C8** `npu_compute.v:26,28` 默认 `ACT_ADDR_W=13` / `PARAM_ADDR_W=11` 是按 `SPAD_KB=128`
  写死的注释和值。`npu_top` 会覆盖，独立例化则静默截断。
- **C9** `npu_csr.v:209` `DMA_STATUS` 的 `[2:0]` 三位全部镜像同一个 `hw_dma_busy`，
  规格书定义的是 IN/OUT/WEIGHT 三个独立位 + `XFER_COUNT`。软件无法区分 DMA 阶段。
- **C10** `npu_dma.v` 单拍 Wishbone classic，`burst_cfg` 未使用，无背靠背传输。
  仿真里不显眼，真 DDR 上是主要带宽损失。
- **C11** `architecture-spec.md` §8 描述的阵列与 RTL 实现不符（行/列语义、drain 周期、
  INT8 双倍吞吐、累加器位宽全对不上）。详见 `rtl_design_overview.md` §9。

---

## 已排除（审查中提出，核实后不成立）

| 提出的问题 | 核实结论 |
|---|---|
| "64 位 bias 被截断到 44 位再相加，与 CSIM 的先加后截不符" | **不成立**。CSIM 的 `trunc_40bit` 实际按 `get_acc_width()`=44 截断，且 `(a+b) mod 2^44 ≡ (a + (b mod 2^44)) mod 2^44`，先截后加与先加后截**算术等价** |
| "`byte_off[17:2]` 只有 64KiB 寻址跨度，大层会混叠" | **不成立**。`byte_off` 是 32 位，`[17:2]` 取 16 位字索引 = 256KB 字节跨度；当前最大的 wgt SRAM 是 96KB，安全到 `SPAD_KB≈512` |
| "PPU 的 `>>>` 对负数舍入错误（经典 requant bug）" | **不成立**。S2 先加 `2^(S-1)` 再算术右移，与 `model_packer.ref_postproc_perchannel` 和 CSIM 的 `(product + (1<<(S-1))) >> S` 一致。是 round-half-up，负数的 .5 向 +∞，三方一致 |
| "流水化 drain 在 `S_DRAIN_OUT` 期间下发 DRAIN 命令，阵列不接受" | **不成立**。`sa_cmd_valid` 是寄存器输出，脉冲实际出现在捕获后一拍，那时阵列已回到 `S_READY` |
| "DMA store 会重复送字" | **不成立**。`S_STORE_RD` 给地址、下一拍取数，与 `npu_sram` 的 read-first 行为一致 |

---

## 建议的修复顺序

**挡路的先修**（不修就没法缩 SRAM / 上 FPGA）：

1. A2 —— 守卫里的 6 处 `` `SPAD_KB `` 宏换成模块参数
2. A4 + A5 —— 定死 `HW_CONFIG` SPAD 单位和 `POST_CLAMP` 位域，三方（RTL / 固件 / 驱动 / 规格书）对齐，加回归
3. A8 —— 删掉 `npu_dw_conv` 的复位循环（一行，省 6.5 万触发器）
4. A6 —— 加 E10 累加器溢出守卫

**决定要不要参数化再修**：

5. A3 —— `ARRAY_SIZE≠16` 那一批。如果短期不打算改阵列尺寸，就加断言锁死 16，别留假的可配置性

**性能**：

6. A1 —— 按 `rtl_design_overview.md` §5.10 的顺序做，1（跨 pass 不 drain）性价比最高

**卫生**：

7. A10、C6、C7、C11 —— 注释和死代码
