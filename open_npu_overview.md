# Open-NPU 整机说明

这份文档是整颗 NPU 的全栈地图：先说明系统定位与一次推理经过的软硬件模块，再给出当前冻结的 64-lane Row 硬件通路、量化、切块与跨层融合机制、工具链与公开基准模型验收口径。

读完后应能回答：一个 ONNX 浮点模型从量化转换到 RISC-V SoC RTL 逐层逐 word 比对，中间每一步由哪个模块负责、数据存放在哪里、满足什么门禁才算通过。

---

## 0. 一句话定位

Open-NPU 是一颗挂载在 RISC-V SoC Wishbone 总线上的 **有符号 INT8 / INT16 定点神经网络加速器**。CPU 不参与乘加运算，只负责配置寄存器（CSR）、触发启动、等待中断，并在验证固件中比对主存输出与参考 golden。

当前冻结硬件配置：

| 项目 | 规格 |
|---|---|
| 主力计算核 | **64-lane K-major MAC 行引擎**（峰值 `64 MACs/cycle`，`12.8 GMAC/s` = `25.6 GOPS` @ `200 MHz`），统一承载 Conv2D、FC 与 Depthwise Conv2D |
| 后处理单元 | **8-lane 分时复用 PPU**（`PPU_LANES = 8`，4 级流水）+ **8-lane 流水双分支 Add** |
| 片上 Scratchpad | **172 KiB 四库 512-bit 架构**（`32 KiB IFM + 48 KiB OFM + 80 KiB WGT + 12 KiB PARAM`） |
| 累加与偏置位宽 | 44-bit 有符号累加器 + 64-bit 有符号逐通道偏置 |
| 数据类型 | 有符号 INT8、有符号 INT16（零点对称默认 `zp = 0`，同时支持非零 `in_zp / out_zp`） |
| 模型描述符 | `NPU1` 离线格式（每层固定头 62 字节 + 逐通道 14 字节参数包） |
| 全芯片真综合 | Yosys `synth -top npu_top` **0 黑盒**，总计 `40,923` cells（`$div = 0`，`$mod = 0`，`$print = 0`） |

公开仓库整网验收矩阵覆盖 **3 个公开代号模型（`model_c`、`model_e`、`model_s`）x 2 种位宽（INT8 / INT16）= 6 格**，每格必须同时通过两道门禁：

1. **门 1（精度门）**：CSIM 反量化输出 vs ONNX Runtime FP32 余弦相似度达标（INT8 >= `0.95`，INT16 >= `0.99`）
2. **门 2（硬件一致性门）**：SoC Verilator 链式推理输出 vs **当场**由当前 `onnx_converter + csim` 生成的逐层 golden **逐 32-bit word 完全一致（bit-exact）**

---

## 1. 仓库分层与职责

从上到下分为四层：

```
ONNX 浮点模型
    │  tools/onnx_converter.py（PTQ 量化、切块、K-major 重排、融合、打包）
    ▼
NPU1 二进制 (*.npu1.bin)  ──────────►  csim/npu_sim（C 参考模拟器，生成逐层 golden）
    │
    │  tools/internal/gen/model_*_golden.py + soc/firmware/gen_soc_test.py
    ▼
NPU2 固件测试镜像 + RISC-V 验证程序
    │
    ▼
soc/sim Verilator 整机仿真：VexRiscv 写 CSR -> NPU RTL 链式推理 -> UART 输出逐层 PASS/FAIL
```

| 目录 | 角色 |
|---|---|
| `tools/` | ONNX 转换器、空间/通道切块、K-major 权重打包、公开模型 golden 生成与回归脚本 |
| `design/` | 架构、数据通路、离线二进制接口与 CSR 寄存器规格文档 |
| `csim/` | 逐比特精确 C 参考模拟器 `npu_sim`（累加位宽、舍入与截断行为与 RTL 严格一致） |
| `rtl/` | 零黑盒可综合 Verilog-2005 实现（控制器、DMA、64-lane Row 计算核、8-lane PPU、四库宽口 SRAM）及 156 项 cocotb 测试 |
| `soc/` | LiteX + VexRiscv SoC 集成、裸机验证固件与 Verilator 全网仿真环境 |
| `driver/` | 面向 MCU / RTOS 集成的独立 HAL 与 `NPU1` 运行时驱动库 |

---

## 2. 一次推理中的三方协作

```
CPU 固件 (soc/firmware/main.c)
    逐层配置 CSR -> 写 CTRL_START -> 等待 IRQ_DONE -> 校验 DDR 输出与 golden 逐 word 一致

NPU 控制器 (rtl/src/npu_ctrl.v)
    调度权重 / 激活 / 参数 DMA -> 控制 OC 分组权重重载、空间 tile 循环、双缓冲预取与跨层 bank 切换

NPU 计算核 (rtl/src/npu_compute.v)
    面向片上四库 512-bit SRAM 工作
    统一在 64-lane K-major Row 与 8-lane PPU 上完成 Conv2D / DWConv / FC / Pool / Add / Resize / Concat
```

核心概念：
- **DDR / 主存**：SoC 仿真中为 LiteX `main_ram`（起始地址 `0x40000000`），存放完整模型权重、层间特征图与参考输出。
- **Scratchpad SRAM**：NPU 内部紧耦合便签存储器，所有算子只读写片上 SRAM，不直接访问主存。
- **CSR**：基地址 `0x80000000` 的 Wishbone 从设备寄存器空间（4 KB）。
- **DMA**：Wishbone 主设备，负责主存与四块片上 SRAM 之间的 1D 连续搬运或 2D 带行距（strided）搬运。
- **链式推理**：第 N 层写回主存（或驻留片上交换 bank）的输出直接作为第 N+1 层的真实输入，不逐层注入独立 golden 输入。

---

## 3. 硬件微架构概览

顶层为 `rtl/src/npu_top.v`，对外仅暴露一个 Wishbone Slave（CSR）、一个 Wishbone Master（DMA）与一根电平中断线 `irq`：

```
                    Wishbone Slave          Wishbone Master (DDR)
     CPU ─────────────────────────────────► npu_csr            ▲
                                               │               │
                                               ▼               │
                                            npu_ctrl ──────► npu_dma
                                               │               │
                         ┌─────────────────────┼───────────────┴──────┐
                         ▼                     ▼                      ▼
                 IFM / OFM SRAM           WGT SRAM               PARAM SRAM
               (32 KiB + 48 KiB)          (80 KiB)                (12 KiB)
                 512-bit 读写口          512-bit 读口            512-bit 读口
                         │                     │                      │
                         └─────────────────────┼──────────────────────┘
                                               ▼
                                          npu_compute
                         ┌─────────────────────┴─────────────────────┐
                         ▼                                           ▼
              64-Lane K-Major Row Engine                   npu_ppu_bank (8 Lanes)
          (Conv2D / FC / DW 复用 wrow + racc)          (INT64 Bias + 乘移重量化 + Add)
```

### 3.1 四块独立片上 SRAM（172 KiB）

默认配置下（`IFM_KB=32`、`OFM_KB=48`、`WGT_KB=80`、`PARAM_KB=12`），`npu_top.v` 例化 4 个 `npu_sram_wide`：

| SRAM 库 | 深度（32-bit word） | 容量 | 端口配置与职责 |
|---|---|---|---|
| `IFM` (`u_sram_act`) | `8,192` | `32 KiB` | A 口 32-bit DMA 写，B 口 512-bit 计算读；双缓冲时分为 `2 x 16 KiB` |
| `OFM` (`u_sram_ofm`) | `12,288` | `48 KiB` | A 口 32-bit DMA 读，B 口 512-bit RMW 读 / 按字节掩码写；跨层融合时可通过 `act_role_swap` 与 IFM 角色互换 |
| `WGT` (`u_sram_wgt`) | `20,480` | `80 KiB` | A 口 32-bit DMA 写，B 口 512-bit 计算读；存放 K-major 打包权重或 DW 权重 |
| `PARAM` (`u_sram_param`) | `3,072` | `12 KiB` | A 口 32-bit DMA 写，B 口 512-bit 读；支持前后半区 ping/pong 下一层参数预取 |

### 3.2 统一 64-Lane K-Major 行引擎与 DW 折叠复用

`npu_compute.v` 内部不再包含独立的二维脉动阵列或独立的 `npu_dw_conv` 模块，而是统一使用一套高密度行引擎与分库缓冲：

- **单写口 64-Bank 片内缓冲**：权重行缓冲 `wrow_b[0..31][0..63]`、激活 gather 缓冲 `abuf_b[0..31][0..63]` 与激活向量缓冲 `avec_b[0..31][0..63]` 按 64 个单写口 bank 组织（总计 `102,400` bit），综合工具直接推断为紧凑的 `$mem_v2` 存储块，彻底消除多写口多路选择器爆炸。
- **Conv2D / FC 执行**：每拍从 512-bit 权重口读入当前 K 下标的 64 路权重，与广播激活相乘并累加至 64 个 44-bit 寄存器 `racc[0..63]`（峰值 `64 MACs/cycle`）。当单组输出通道数 `G < 64`（如 `16` 或 `32`）时，自动将 `P = 64 / G` 个相邻输出像素拼入同一拍并行乘加。
- **Depthwise Conv2D 折叠复用**：DW 卷积按每组 8 通道执行，权重直接缓存在 `wrow_b[0..31][0..63]`（支持最大 `256` 个卷积核 tap），通道独立乘加复用 `racc[0..7]`。整图覆盖的全局 DW（输出 `1x1xC`）仍支持按 16 通道组流式切片，无需整张特征图同时驻留片上。
- **零组合除法器**：AvgPool 的窗口面积倒数由一个 **40 拍 17-bit 串行恢复余数倒数单元** 在层启动阶段一次性算出，全芯片 `$div = 0`、`$mod = 0`。

### 3.3 8-Lane 分时复用后处理单元（`npu_ppu_bank.v` + `npu_ppu.v`）

64 个累加器结果无需 64 条并行 PPU 同时排空：写回阶段按每拍 8 个通道送入 8 条 4 级流水 `npu_ppu`，8 拍完成 64 路输出并与 512-bit OFM 写出带宽精确匹配。单条 PPU 流水执行：

```
acc (44-bit) -> + bias_q (截断对齐) -> * M[ch] + round -> >>> S[ch] -> + zp[ch] -> clamp -> ReLU / ReLU6 -> INT8/INT16
```

Elementwise Add 同样采用 **每拍 8 路（16 个乘法器）** 与 PPU 流水重叠执行，消除单拍 128 乘法器峰值开销。

---

## 4. 内存布局与切块 / 双缓冲 / 跨层融合

- **激活张量布局**：CSIM、DMA 与片上 SRAM 统一采用 **NHWC** 连续排布；ONNX 端的 NCHW 输入在转换与校准阶段完成转换。
- **权重布局**：`REG_WGT_LAYOUT = 1` 时采用 **K-major** 排布（每个 K 步连续存放当前 OC 组的 64 个通道权重，自然对齐 512-bit 宽读口）。
- **空间切块（Spatial Tiling）**：当整层激活超出 IFM/OFM 预算时，离线编译器 `tools/tiling.py` 计算 `tile_h / tile_w / tile_num_h / tile_num_w`；写回主存时开启 `PER_TILE_STORE`（PTS），由 2D DMA 按整图行距拼接回主存 NHWC 张量。
- **双缓冲（`DB_EN`）**：IFM 对半切分为两个 `16 KiB` 缓冲，计算块 N 的同时预取块 N+1，通过 `tile_done` 与 `db_prefetch_done` 握手同步。
- **跨层融合（`FUSE_START / MID / END`）**：不切块且满足片上驻留条件的相邻层，通过 `IFM` 与 `OFM` 物理库角色互换（`act_role_swap`）直接在片上传递中间特征图，省去主存往返读写。

---

## 5. 支持的算子编码

编码位于 `LAYER_MODE[3:0]`，与离线 `NPU1` 描述符及 CSIM 一致：

| 编码 | 算子 | 硬件执行路径 | 说明 |
|---|---|---|---|
| `0` | `CONV2D` | 64-lane Row + 8-lane PPU | 支持任意卷积核、步长、空洞率（dilation）与四边非对称 padding |
| `1` | `DW_CONV` | `wrow` + `racc[0..7]` + 8-lane PPU | 支持常规空间滑窗与 `1x1` 全局 DW 切片流式执行 |
| `2` | `FC` | 64-lane Row + 8-lane PPU | 等价于 `1x1` 卷积路径 |
| `3` | `POOLING` | Pooling FSM | MaxPool / AvgPool / GlobalAvgPool（40 拍串行倒数器） |
| `4` | `ELTWISE_ADD` | 8-lane 流水双路缩放 + PPU | 残差双分支独立 `(M_A, S_A)` / `(M_B, S_B)` 对齐相加 |
| `5` | `RESIZE` | Resize FSM | 最近邻 / 双线性插值（切块时带全局坐标原点） |
| `6` | 保留 | — | 历史转置卷积编码已停用，硬件守卫返回错误码拦截 |
| `7` | `CONCAT` | 通道偏移写入 + 可选重缩放 | 多分支按通道偏移拼接到同一输出张量 |

---

## 6. 公开基准模型矩阵（`model_c` / `model_e` / `model_s`）

公开回归测试包含以下三个代号模型（共 6 个精度单元）：

| 单元 | 模型代号 | 典型算子拓扑特征 | 余弦门禁 | SoC RTL 一致性状态 |
|---|---|---|---|---|
| `C8 / C16` | `model_c` | 大分辨率多尺度特征、MaxPool、上采样 Resize、通道 Concat（17 层） | `0.95 / 0.99` | 逐层逐 word bit-exact **PASS** |
| `E8 / E16` | `model_e` | 深层常规卷积链、跨步下采样、双分支残差 Add（31 层） | `0.95 / 0.99` | 逐层逐 word bit-exact **PASS** |
| `S8 / S16` | `model_s` | 倒残差深度可分离卷积长链、ReLU6、全局池化、全连接（64 层） | `0.95 / 0.99` | 逐层逐 word bit-exact **PASS** |

常用回归入口：

```bash
# 跑单个公开代号模型的全链路（转换 + CSIM golden + 固件编译 + Verilator SoC 仿真）
soc/run_model_e2e.sh model_c_int8
soc/run_model_e2e.sh model_e_int8
soc/run_model_e2e.sh model_s_int8

# 跑公开 6 格矩阵（余弦门 + SoC RTL bit-exact 门）
python3 tools/internal/phase1_model_matrix.py --rtl

# 跑全套门禁回归（单元测试 + 转换器守卫 + RTL cocotb + 公开矩阵）
python3 tools/internal/phase1_regression.py --mode quick
```

---

## 7. 关键文件索引

**硬件（`rtl/`）**：
- `rtl/include/npu_defines.vh` — 通道宽度（`MAC_LANES=64`、`PPU_LANES=8`）、四库 SRAM 容量（`172 KB`）与寄存器基址
- `rtl/src/npu_top.v` — 顶层互连、四库 `npu_sram_wide` 实例化与 bank 切换
- `rtl/src/npu_csr.v` — Wishbone CSR 寄存器堆、性能直方图与 Super-Block 3 槽描述符窗
- `rtl/src/npu_ctrl.v` — 层级 DMA / 计算主状态机、OC 重载、双缓冲与硬件合法性守卫
- `rtl/src/npu_compute.v` — 64-lane Row 乘加、DW 折叠复用、Pooling / Add / Resize 微定序器
- `rtl/src/npu_ppu_bank.v` / `rtl/src/npu_ppu.v` — 8-lane 4 级后处理流水线
- `rtl/src/npu_dma.v` — 1D 连续与 2D strided Wishbone 主控 DMA

**工具链与仿真（`tools/`、`csim/`、`soc/`）**：
- `tools/onnx_converter.py` — ONNX 图归一化、PTQ 量化、切块与 `NPU1` 生成
- `tools/model_packer.py` — `NPU1` 描述符与 K-major 权重打包
- `tools/tiling.py` / `tools/layer_fusion.py` / `tools/hw_config.py` — 硬件约束感知切块与层间融合分析
- `tools/internal/gen/model_{c,e,s}_int{8,16}_golden.py` — 公开模型逐层 golden 生成脚本
- `csim/src/` — C 参考模拟器实现
- `soc/firmware/main.c` / `soc/firmware/gen_soc_test.py` / `soc/run_model_e2e.sh` — RISC-V SoC 链式验证固件与一键运行脚本
