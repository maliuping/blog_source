+++
date = '2026-10-12T11:46:13+08:00'
draft = false
title = 'Linlon q-channel'
categories = ["SOC"]
tags = ["Low Power", "DPU"]
+++

在linlon-d6 显示处理器中，**Q-Channel** 是遵循 AMBA 标准的硬件低功耗握手接口协议。它用于 DPU 与 SOC 外部的时钟/电源控制器之间进行自动化、安全的**外部时钟门控（Clock Gating）**与**电源门控（Power Gating）**握手，确保 DPU 在空闲时能切断时钟或电源以降低功耗，而在有总线访问或显示任务时能被安全唤醒。

在linlon-d6 中，全局控制单元（GCU）实现了两个独立的 Q-Channel 接口：

---

## 1. 两个 Q-Channel 接口
1. **`ACLK` 时钟 Q-Channel（External Clock Gating Interface）**：
   用于管理 DPU 主处理时钟 `ACLK` 树的外部门控。当 DPU 所有模块处于空闲状态且所有图层数据已取完时，可关停外部 ACLK 时钟树以节省动态功耗。
2. **电源 Q-Channel（Power Gating Interface）**：
   用于管理 DPU 全局电源域（`PWRGLB`）的上电与下电序列。当 DPU 彻底进入 INACTIVE 状态且从 SMMU 转换缓冲区（TBU）断开后，可通过此接口关闭 DPU 电源。

---

## 2. 核心 4 信号握手机制（4-Wire Handshake Protocol）
每个 Q-Channel 接口均包含以下 4 个标准信号：

* **`QACTIVE`**（DPU -> 控制器，高有效）：**活动状态/唤醒指示**。
  * `1`：DPU 内部有任务正在运行或有总线唤醒请求，无法关闭时钟/电源。
  * `0`：DPU 已进入完全空闲状态，允许进入低功耗模式。
* **`QREQn`**（控制器 -> DPU，低有效）：**低功耗请求信号**。
  * `1`：正常工作模式（时钟/电源开启）。
  * `0`：控制器向 DPU 发起进入低功耗模式（关时钟或断电）的请求。
* **`QACCEPTn`**（DPU -> 控制器，低有效）：**请求接受应答**。
  * `0`：DPU 确认已处于空闲状态，同意进入低功耗模式。
* **`QDENY`**（DPU -> 控制器，高有效）：**请求拒绝应答**。
  * `1`：DPU 拒绝低功耗请求（例如握手期间收到了新的 APB 访问或预取请求），阻止系统切断电源/时钟。

*(注：信号名前缀根据接口类型分为 `ACLK` 或 `PWR`，如 `ACLKQREQn` / `PWRQREQn`)。*

---

## 3. 典型握手与工作流程

### (1) 进入低功耗（关时钟 / 下电）过程：
1. **空闲指示**：当 DPU 内部所有硬件单元（GCU、LPU、CU、DOU）完成当前帧处理并停机时，DPU 将 `QACTIVE` 拉低 (`0`)。
2. **发起请求**：SoC 侧控制器检测到 `QACTIVE=0`，将 `QREQn` 拉低 (`0`)，向 DPU 请求关时钟或断电。
3. **硬件响应判断**：
   * **同意（ACCEPT）**：若 DPU 仍处于空闲状态，DPU 将 `QACCEPTn` 拉低 (`0`)，控制器随后切断 ACLK 或关闭电源。
   * **拒绝（DENY）**：若在握手期间 CPU 发起了新的 APB 寄存器读写（拉高 `PWAKEUP`），DPU 硬件会自动将 `QDENY` 拉高 (`1`) 拒绝请求，终止下电流程。

### (2) 退出低功耗（唤醒 / 上电）过程：
1. 当 CPU 需要重新访问 DPU 或开启显示时，控制器将 `QREQn` 拉高 (`1`)，并恢复时钟/电源。
2. DPU 完成复位与初始化后，将 `QACCEPTn` 拉高 (`1`) 完成上电握手，恢复正常工作。

---

## 4. 硬件自动保护逻辑（`PWRQACTIVE` 的生成）
在 DPU 硬件内部，`PWRQACTIVE` 信号是由一个**组合逻辑 OR 门**输出的。其输入汇总了：
* APB 编程接口的唤醒信号（`PWAKEUPx`）；
* DTI 内存转换接口唤醒信号（`TCUTWAKEUPx`）；
* DOU 的帧预取标志（`doux_prefetch`）；
* 各个硬件子模块的运行标志（`gcu_active`, `lpu_active`, `cu_active`, `dou_active`）。

只要上述任何一个输入为 `1`，`PWRQACTIVE` 就会保持高电平，防止在驱动访问或图像合成过程中发生误断电。

---

