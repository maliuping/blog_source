+++
date = '2026-09-23T11:15:14+08:00'
draft = false
title = 'Linlon DPU hw blocks description'
categories = ["SOC"]
tags = ["SOC", "DPU"]
+++

Linlon-D6 显示处理器（DPU）在 **`GLB_CORE_INFO`** 寄存器的 **`NUM_BLOCKS`** 字段中定义了总共有 **`0x23`（即 35 个）** 基础寄存器块。

这些寄存器块按照 `0x00200`（512 字节）的步长，连续分布在偏移地址 `0x00000` 到 `0x045FF` 的内存空间中。具体清单如下：

# 35 个基础寄存器块清单（Block 0x00 ~ Block 0x22）

| 块编号 (Index) | 偏移地址 (Offset Range) | 寄存器块名称 (Block Name) | **BLOCK_TYPE** | 块类型说明 (Block Type Description)  |
| :---:          | :---:                   | :---                      | :---:          | :---                                 |
| **0x00 (0)**   | `0x00000 - 0x001FF`     | **GCU**                   | **`0x00`**     | 全局控制单元 (Global Control Unit)   |
| **0x01 (1)**   | `0x00200 - 0x003FF`     | **LPU0**                  | **`0x01`**     | LPU 控制块 (LPU Control)             |
| **0x02 (2)**   | `0x00400 - 0x005FF`     | **LPU0_LAYER0**           | **`0x10`**     | 输入图层块 (Rich Layer 0)            |
| **0x03 (3)**   | `0x00600 - 0x007FF`     | **LPU0_LAYER1**           | **`0x10`**     | 输入图层块 (Simple Layer 1)          |
| **0x04 (4)**   | `0x00800 - 0x009FF`     | **LPU0_LAYER2**           | **`0x10`**     | 输入图层块 (Rich Layer 2)            |
| **0x05 (5)**   | `0x00A00 - 0x00BFF`     | **LPU0_LAYER3**           | **`0x10`**     | 输入图层块 (Simple Layer 3)          |
| **0x06 (6)**   | `0x00C00 - 0x00DFF`     | **LPU0_LAYER_WR**         | **`0x11`**     | 内存回写图层块 (Write-back Layer)    |
| **0x07 (7)**   | `0x00E00 - 0x00FFF`     | **CU0**                   | **`0x02`**     | CU 合成单元控制块 (Composition Unit) |
| **0x08 (8)**   | `0x01000 - 0x011FF`     | **CU0_LINE_SPLITTER**     | **`0x20`**     | 行拆分器块 (Line Splitter)           |
| **0x09 (9)**   | `0x01200 - 0x013FF`     | **CU0_SCALER0**           | **`0x21`**     | 缩放引擎块 (Scaler Engine 0)         |
| **0x0A (10)**  | `0x01400 - 0x015FF`     | **CU0_SCALER1**           | **`0x21`**     | 缩放引擎块 (Scaler Engine 1)         |
| **0x0B (11)**  | `0x01600 - 0x017FF`     | **CU0_LINE_MERGER**       | **`0x22`**     | 行合并器块 (Line Merger)             |
| **0x0C (12)**  | `0x01800 - 0x019FF`     | **DOU0**                  | **`0x03`**     | DOU 显示输出单元控制块               |
| **0x0D (13)**  | `0x01A00 - 0x01BFF`     | **DOU0_IPS**              | **`0x30`**     | 图像处理子系统 (IPS)                 |
| **0x0E (14)**  | `0x01C00 - 0x01DFF`     | **DOU0_IPS_FCOEFF**       | **`0x32`**     | 前向变换 LUT 系数表 (FT Coeff)       |
| **0x0F (15)**  | `0x01E00 - 0x01FFF`     | **DOU0_BS**               | **`0x31`**     | 后端子系统 (Backend Subsystem)       |
| **0x10 (16)**  | `0x02000 - 0x021FF`     | **RESERVED**              | **`0xFF`**     | 系统保留块 (Reserved Block)          |
| **0x11 (17)**  | `0x02200 - 0x023FF`     | **LPU1**                  | **`0x01`**     | LPU 控制块 (LPU Control)             |
| **0x12 (18)**  | `0x02400 - 0x025FF`     | **LPU1_LAYER0**           | **`0x10`**     | 输入图层块 (Rich Layer 0)            |
| **0x13 (19)**  | `0x02600 - 0x027FF`     | **LPU1_LAYER1**           | **`0x10`**     | 输入图层块 (Simple Layer 1)          |
| **0x14 (20)**  | `0x02800 - 0x029FF`     | **LPU1_LAYER2**           | **`0x10`**     | 输入图层块 (Rich Layer 2)            |
| **0x15 (21)**  | `0x02A00 - 0x02BFF`     | **LPU1_LAYER3**           | **`0x10`**     | 输入图层块 (Simple Layer 3)          |
| **0x16 (22)**  | `0x02C00 - 0x02DFF`     | **LPU1_LAYER_WR**         | **`0x11`**     | 内存回写图层块 (Write-back Layer)    |
| **0x17 (23)**  | `0x02E00 - 0x02FFF`     | **CU1**                   | **`0x02`**     | CU 合成单元控制块 (Composition Unit) |
| **0x18 (24)**  | `0x03000 - 0x031FF`     | **CU1_LINE_SPLITTER**     | **`0x20`**     | 行拆分器块 (Line Splitter)           |
| **0x19 (25)**  | `0x03200 - 0x033FF`     | **CU1_SCALER0**           | **`0x21`**     | 缩放引擎块 (Scaler Engine 0)         |
| **0x1A (26)**  | `0x03400 - 0x035FF`     | **CU1_SCALER1**           | **`0x21`**     | 缩放引擎块 (Scaler Engine 1)         |
| **0x1B (27)**  | `0x03600 - 0x037FF`     | **CU1_LINE_MERGER**       | **`0x22`**     | 行合并器块 (Line Merger)             |
| **0x1C (28)**  | `0x03800 - 0x039FF`     | **DOU1**                  | **`0x03`**     | DOU 显示输出单元控制块               |
| **0x1D (29)**  | `0x03A00 - 0x03BFF`     | **DOU1_IPS**              | **`0x30`**     | 图像处理子系统 (IPS)                 |
| **0x1E (30)**  | `0x03C00 - 0x03DFF`     | **DOU1_IPS_FCOEFF**       | **`0x32`**     | 前向变换 LUT 系数表 (FT Coeff)       |
| **0x1F (31)**  | `0x03E00 - 0x03FFF`     | **DOU1_BS**               | **`0x31`**     | 后端子系统 (Backend Subsystem)       |
| **0x20 (32)**  | `0x04000 - 0x041FF`     | **GLB_LT_COEFF0**         | **`0x05`**     | 全局图层变换 LUT 系数表 0            |
| **0x21 (33)**  | `0x04200 - 0x043FF`     | **GLB_LT_COEFF1**         | **`0x05`**     | 全局图层变换 LUT 系数表 1            |
| **0x22 (34)**  | `0x04400 - 0x045FF`     | **GLB_LT_COEFF2**         | **`0x05`**     | 全局图层变换 LUT 系数表 2            |

---

# 额外高地址块（Special / Secure Register Pages）
除了上述 35 个基础 Block 外，手册在更位高地址空间还单独定义了以下扩展/安全块：

| 偏移地址 (Offset) | 寄存器块名称 (Block Name) | **BLOCK_TYPE** | 块类型说明                          |
| :---              | :---                      | :---:          | :---                                |
| `0x0FE00`         | **PERIPHERAL**            | **`0xFF`**     | 外设 ID 标识块                      |
| `0x10000`         | **LPU0_TRUSTED**          | **`0x09`**     | LPU0 受信任控制页 (Trusted Control) |
| `0x10200`         | **LPU1_TRUSTED**          | **`0x09`**     | LPU1 受信任控制页 (Trusted Control) |

* **`0x10000`**：**LPU0_TRUSTED**（LPU0 受信任图层控制页，仅允许 Secure APB 访问）
* **`0x10200`**：**LPU1_TRUSTED**（LPU1 受信任图层控制页，仅允许 Secure APB 访问）


**编码规律与硬件逻辑**：
* **`0x01` ~ `0x03`**：核心控制顶层块（`0x01` LPU, `0x02` CU, `0x03` DOU）。
* **`0x10` / `0x11`**：DMA 图层数据管线（`0x10` 普通读取图层, `0x11` 内存写回图层）。
* **`0x20` ~ `0x22`**：CU 合成管线子模块（`0x20` Splitter, `0x21` Scaler, `0x22` Merger）。
* **`0x30` ~ `0x32`**：DOU 输出显示管线子模块（`0x30` IPS, `0x31` BS, `0x32` FT_COEFF）。
* **`0x09`**：受信任图层安全配置块。
* **`0xFF`**：保留或非逻辑块。

