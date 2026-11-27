# Open-NPU RTL 设计总览（信号与模块级）

本文档描述 **当前 `rtl/src/` 可综合硬件的实际模块结构、接口信号、状态机握手、硬件守卫与零黑盒综合指标**。进行 RTL 修改、编写底层固件驱动或评估 PPA 时，以本文档与 `npu-register-spec.md` 为准。

代码基线：`MAC_LANES = 64`，`PPU_LANES = 8`，`SPAD_KB = 172`（`32 KB IFM + 48 KB OFM + 80 KB WGT + 12 KB PARAM`），`SRAM_B_WIDTH = 512`，`ACC_WIDTH = 44`，外部 DMA 主口编译期可配置 `DMA_BUS_WIDTH = 32 / 64 / 128` 与 `DMA_BUS_PROTO = 0 (Wishbone-B4) / 1 (AXI4)`（默认 `32-bit Wishbone-B4`）。

---

## 1. 模块清单与层次结构

`rtl/src/` 共包含 10 个 Verilog-2005 源文件，全芯片单时钟域、异步低有效复位，**零黑盒**通过 Yosys 综合：

| 模块文件 | 行数 | 职责 |
|---|---|---|
| `npu_top.v` | `~1,300` | 顶层封装；例化读写双通道 DMA、4 块片上 SRAM 与可选 AXI4 桥，管理端口仲裁、双缓冲与跨层互换 |
| `npu_csr.v` | `~650` | Wishbone-B4 从接口寄存器堆、4 槽 Super-Block 描述符窗与性能计数器直方图 |
| `npu_ctrl.v` | `~2,840` | 层级主控制器：DMA 读写解耦调度、空间 tile 循环、WGT/PARAM 乒乓预取与 E1-E9 硬件守卫 |
| `npu_dma.v` | `~520` | 参数化 `32/64/128-bit` 突发主控 DMA，支持任意 4B 对齐首尾掩码、1D 连续与 2D strided 搬运 |
| `npu_wb_to_axi4.v` | `~240` | 可选可综合 AXI4 `INCR` 突发桥（16-beat 读写突发、4KB 边界裁剪、写响应 `BVALID` 严格保序） |
| `npu_compute.v` | `~4,300` | 核心计算微定序器：64-bank 缓冲、64-lane K-major 行引擎、DW 折叠复用、Pool / Add / Resize / Concat |
| `npu_ppu_bank.v` | `96` | 8-lane 后处理阵列封装（`PPU_LANES = 8`） |
| `npu_ppu.v` | `317` | 单通道 4 级后处理流水线（INT64 bias 对齐累加、15-bit 乘数 + 6-bit 算术右移舍入、零点与钳位） |
| `npu_sram_wide.v` | `~105` | 混合位宽双端口同步 SRAM（Port A `32/64/128-bit` 字掩码 DMA 写，Port B 512-bit 计算读写） |
| `npu_sram.v` | `~70` | 参数化 Port A `32/64/128-bit` 字掩码写 + Port B 32-bit 读双端口同步 SRAM |

> 已退役模块：历史版本的 `npu_systolic.v`、`npu_pe.v`、`npu_dw_conv.v`、`npu_dw_bank.v` 均已彻底移除；稠密卷积、全连接与深度可分离卷积统一由 `npu_compute.v` 的 64-lane 行引擎承载。

模块实例层次：

```
npu_top
├── npu_csr                  (Wishbone Slave <- CPU)
├── npu_dma                  (Wishbone Master -> DDR)
├── npu_sram_wide x 4        (IFM 32KB / OFM 48KB / WGT 80KB / PARAM 12KB)
├── npu_ctrl                 (层级 DMA 与 Tiling 调度器)
└── npu_compute              (统一算子计算引擎)
    └── npu_ppu_bank
        └── npu_ppu x 8      (8-lane 分时复用后处理流水线)
```

---

## 2. 顶层参数与对外接口（`npu_top.v`）

```verilog
module npu_top #(
    parameter ARRAY_SIZE  = `ARRAY_SIZE,     // 8 (DW 通道组宽度兼容参数)
    parameter IFM_KB      = `IFM_KB,         // 32
    parameter OFM_KB      = `OFM_KB,         // 48
    parameter WGT_KB      = `WGT_KB,         // 80
    parameter PARAM_KB    = `PARAM_KB,       // 12
    parameter SRAM_B_W    = `SRAM_B_WIDTH,   // 512
    parameter MAC_LANES_P = `MAC_LANES,      // 64
    parameter PPU_LANES_P = `PPU_LANES,      // 8
    parameter DMA_DATA_W  = `DMA_BUS_WIDTH   // 32 (默认) / 64 / 128
)
```

| 信号组 | 方向 | 说明 |
|---|---|---|
| `clk`, `rst_n` | in | 单时钟域，异步低有效复位 |
| `wb_slv_cyc_i, wb_slv_stb_i, wb_slv_we_i, wb_slv_adr_i[11:0], wb_slv_dat_i[31:0], wb_slv_sel_i[3:0]` | in | Wishbone-B4 CSR 从端口（4 KB 地址空间） |
| `wb_slv_dat_o[31:0], wb_slv_ack_o` | out | Wishbone-B4 CSR 读数据与单拍应答 |
| `wb_mst_*`（默认未定义 `DMA_AXI4`） | in/out | 参数化 `32/64/128-bit` Wishbone-B4 DMA 突发主端口 |
| `m_axi_aw* / w* / b* / ar* / r*`（定义 `DMA_AXI4` 时） | in/out | 参数化 `32/64/128-bit` 标准 AXI4 `INCR` 突发主端口 |
| `irq_o` | out | 电平中断输出：`|(IRQ_STATUS & IRQ_EN)` |

---

## 3. 控制与计算握手协议（`npu_ctrl` <-> `npu_compute`）

CPU 只配置 CSR 并写 `CTRL_START`，层内所有分块与重载均由硬件握手完成：

| 信号 | 方向（Compute 视角） | 语义 |
|---|---|---|
| `start` | in | 启动当前层的计算状态机 |
| `done` | out | 整层（全部空间 tile 与全部 `oc_group`）执行完毕 |
| `tile_done` | out | 单拍脉冲：当前非末尾空间 tile 计算完成，通知控制器写出或切换双缓冲 |
| `db_prefetch_done` | in | 控制器通知：下一空间 tile 的输入已预取至另一半 IFM bank |
| `oc_group_done` | out | 单拍脉冲：当前 `oc_group` 计算完毕，请求控制器 DMA 重载下一组权重 |
| `oc_group_out[15:0]` | out | 当前请求重载的 `oc_group` 索引 |
| `wgt_reload_done` | in | 控制器通知：下一组权重已写入 WGT SRAM |
| `tile_out_h_actual / _w_actual` | out | 当前 tile 裁剪边界后的实际输出高宽，供控制器计算 2D per-tile store 主存偏移 |

### 硬件合法性守卫（`ERROR_CODE`）

`npu_ctrl.v` 在接收 `START` 时对层配置做静态守卫检查，违规时拒绝启动、置 `STATUS.ERROR = 1` 并写入错误码：

| 错误码 | 触发条件 |
|---|---|
| `E1` | 保留编码（流式切片模式已支持逐片参数重载） |
| `E2` | 窄通道多像素打包模式与逐 OC 参数重载冲突 |
| `E3` | 全局 DW 流式输出区越界 `DW_STREAM_OUT_BASE` |
| `E4` | 输入激活单次加载字节数超出 `IFM` 容量（`DB_EN` 下超出半库 `16 KB`） |
| `E5` | 输出激活写入范围超出 `OFM` 容量（`48 KB`） |
| `E6` | `Conv2D / FC` 归约深度 `k_depth > 65535` 或不支持的保留算子（如已停用的 `op_type = 6`） |
| `E7` | `AvgPool` 窗口面积为 0 |
| `E8 / E9` | 初始权重或参数 DMA 长度超出 `WGT`（`80 KB`）或 `PARAM`（`12 KB`）容量 |

---

## 4. 数据通路关键设计要点

1. **四库独立 512-bit SRAM（`npu_sram_wide.v`）**：
   - 计算侧每拍可同时从 `IFM`（512-bit）、`WGT`（512-bit）与 `PARAM`（512-bit）读数，并向 `OFM`（512-bit 字节使能）写回，彻底消除单激活库读写争用。
   - 跨层 SRAM 融合时，通过翻转 `act_role_swap` 直接将上一层的 `OFM` 切换为下一层的输入读取库（零拷贝）。
2. **64-Bank 单写口暂存阵列**：
   - `wrow_b`、`abuf_b`、`avec_b` 全部采用单写口 + 同步读结构，使 Yosys / Vivado / ASIC RAM 编译器能稳定推断为紧凑的分布式或块存储，避免生成数万个异步复位触发器。
3. **8-Lane PPU（`npu_ppu_bank.v`）舍入一致性**：
   - 累加器与偏置均为有符号对齐（44-bit 有效累加），乘法后加 `1 << (S - 1)` 再做算术右移 `>>> S`（round-half-up），与 `csim/src/postproc.c` 及 `tools/model_packer.py` 逐比特完全一致。
4. **全芯片零组合除法与取模**：
   - `npu_ctrl.v` 与 `npu_compute.v` 中的通道组对齐全部为 2 的幂移位掩码；AvgPool 均值采用 40 拍 17-bit 串行恢复余数倒数器计算。

---

## 5. Yosys 全芯片零黑盒综合实测（`npu_top`）

综合命令：`yosys -p "read_verilog -Irtl/include rtl/src/*.v; synth -top npu_top; stat"`（0 黑盒，含全部控制、计算、DMA、CSR 与四库 SRAM）：

| 指标 | 重构前基线 | 当前 RTL 实测 | 变化 |
|---|---|---|---|
| **`npu_top` 总单元数（Cells）** | `70,735` | **`44,626`** | **`-36.9%`** |
| **异步复位触发器（`$adff`）** | `4,826` | **`994`** | **`-79.4%`** |
| **硬件乘法器原语（`$mul`）** | `718` | **`526`** | **`-26.7%`** |
| **组合除法 / 取模（`$div` / `$mod`）** | `18` | **`0` / `0`** | **100% 消除** |
| **推断片上存储块（`$mem_v2`）** | — | 4 块主 SRAM（`172 KB`）+ 192 个单写口小 bank（`102,400` bits） | 零触发器膨胀 |
| **Verilator SoC C++ 编译产物** | `14.3 MB` | **`6.40 MB`**（全量编译 `13.0s`） | **`-55.2%`** |

---

## 6. 编译期参数化配置（SRAM 容量与外部 DMA 总线）

修改硬件片上存储预算或外部主存接口规格只需通过编译宏 / 工具链参数统一配置：
1. **片上 SRAM 容量**：修改 `rtl/include/npu_defines.vh`（或编译参数 `-DIFM_KB=... -DOFM_KB=... -DWGT_KB=... -DPARAM_KB=...`）与 `tools/hw_config.py` 中的对应容量；
2. **外部 DMA 总线位宽与协议**：
   - `DMA_BUS_WIDTH`：`32`（默认）、`64` 或 `128`（控制 DMA 数据位宽与 4 块片上 SRAM Port A 写入位宽，硬件自动处理任意 4B 对齐起止地址与字节/字写掩码）；
   - `DMA_BUS_PROTO`：`0` = Wishbone-B4 Master（默认），`1` / `-DDMA_AXI4` = AXI4 `INCR` 突发主接口（经 `npu_wb_to_axi4` 输出标准 `m_axi_*` 端口；`128-bit AXI4` 全芯片 Yosys 综合为 `45,441` cells，较默认 `32-bit WB` 仅增加 `+1.83%`）；
   - LiteX SoC 仿真可通过 `DMA_BUS_WIDTH=32|64|128` 一键切换“32-bit CPU 控制总线 + 宽口主存数据总线”双层架构。
