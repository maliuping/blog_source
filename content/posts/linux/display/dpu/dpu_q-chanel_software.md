+++
date = '2026-10-24T1:02:13+08:00'
draft = true
title = 'Linlon q-channel 软件配合说明'
categories = ["Linux"]
tags = ["Low Power", "DPU"]
+++
## 1. 概述

DPU 的 Q-Channel 是 DPU 与外部电源控制器之间的低功耗握手接口。它不是一个单独的“Q-Channel 驱动”，而是由显示驱动、Linux runtime PM/genpd 框架和 PMCTRL 驱动共同配合完成：

- DPU/linlondp 驱动：负责让 DPU 停止工作，进入 `INACTIVE`，断开 TBU，关闭中断和时钟。
- PMCTRL/genpd 驱动：负责发起 Q-Channel 请求，等待 DPU 应答，然后完成时钟、隔离、复位和电源开关控制。
- DPU 硬件：输出 `PWRQACTIVE`、`PWRQACCEPTn`、`PWRQDENY`，表示自身是否已经可以下电。

简而言之：

> DPU 驱动负责“我可以断电了”，PMCTRL 驱动负责“开始执行断电握手”。

## 2. Q-Channel 信号和寄存器

### 2.1 信号语义

Soc VO 电源域使用 DPU 的 PWR Q-Channel：

| 信号 | 方向 | 含义 |
| --- | --- | --- |
| `PWRQREQn` | PMCTRL -> DPU | 电源请求，低有效。`1` 请求上电，`0` 请求下电 |
| `PWRQACCEPTn` | DPU -> PMCTRL | 请求应答，低有效。`1` 表示未接受下电，`0` 表示接受下电 |
| `PWRQDENY` | DPU -> PMCTRL | DPU 拒绝下电请求 |
| `PWRQACTIVE` | DPU -> PMCTRL | DPU 是否仍然活跃。只有为 `0` 时才允许下电 |

### 2.2 SOC 寄存器

PMCTRL 中对应寄存器为 `DPU_PWR_HS`，物理地址为：0x14000000

位定义：

| 位 | 字段 | 说明 |
| --- | --- | --- |
| bit0 | `PWRQREQN` | 软件写；`1` 请求上电，`0` 请求下电 |
| bit8 | `PWRQACCEPTN` | DPU 输出；`1` 未接受下电，`0` 接受下电 |
| bit9 | `PWRQDENY` | DPU 输出；`1` 表示拒绝下电 |
| bit10 | `PWRQACTIVE` | DPU 输出；`1` 表示仍在活动，`0` 表示可以下电 |

定义位置：

- `pmctrl_sc/pmctrl_sc_struct.h`
- `pmctrl_sc/pmctrl_sc.h`

## 3. 软件分层与职责

### 3.1 用户态/显示框架

应用或 compositor 通过 DRM atomic commit 关闭显示输出。该操作最终触发 CRTC disable、pipeline disable 和 runtime PM put。

### 3.2 DPU/linlondp 驱动

DPU 驱动不直接操作 `PWRQREQn`，它负责满足 DPU 下电的前置条件：

- 关闭 layer、scaler、output 等 pipeline 组件。
- flush 配置并等待 flip done。
- 将 GCU 操作模式切换为 `INACTIVE`。
- 断开 IOMMU/TBU。
- 关闭 DPU 中断。
- 关闭 `aclk` 和 `pxclk`。
- 通过 `pm_runtime_put()` 释放 DPU 设备引用。只有 DPU 挂接到 VO power domain 时，最后一个 consumer 释放引用后才会继续触发父域的 genpd power off。

关键代码：

- `linlondp/linlondp_crtc.c:365`：`linlondp_crtc_atomic_disable()`
- `linlondp/linlondp_crtc.c:180`：`linlondp_crtc_unprepare()`
- `linlondp/linlondp_dev.c:452`：`linlondp_dev_suspend()`
- `linlondp/hw/dp_dev.c:747`：`dp_disconnect_iommu()`
- `linlondp/hw/dp_dev.c:382`：`dp_change_opmode()`

### 3.3 PMCTRL/genpd 驱动

PMCTRL 驱动在 VO 电源域关闭时执行 Q-Channel 协议：

- 读取并轮询 `PWRQACTIVE`。
- 写 `PWRQREQn` 发起上电或下电请求。
- 轮询 `PWRQACCEPTn` 等待 DPU 应答。
- 握手完成后继续执行 NoC idle、时钟关闭、isolation、memory low power、reset 和 power switch 控制。

关键代码：

- `pmctrl_sc/pmctrl_pm_domain.c:439`：`pmu_qchan_lp_ctrl()`
- `pmctrl_sc/pmctrl_pm_domain.c:832`：`pmu_pd_disable()`
- `pmctrl_sc/pmctrl_pm_domain.c:922`：`pmu_pd_enable()`

## 4. 设备树配置

VO 电源域在设备树中声明 Q-Channel 参数：

```dts
pd_vo_subsys: pmctrl-pd-vo-subsys {
	#power-domain-cells = <0>;
	regmap-pmctrl = <&pmctrl_sc>;

	qchan_ctrl_info = <DPU_PWR_HS_OFFSET \
			DPU_PWR_HS_PWRQREQN_OFFSET \
			DPU_PWR_HS_PWRQACCEPTN_OFFSET \
			DPU_PWR_HS_PWRQDENY_OFFSET \
			DPU_PWR_HS_PWRQACTIVE_OFFSET>;
};
```

该配置定义了：

1. Q-Channel 状态寄存器偏移。
2. `PWRQREQn` 位偏移。
3. `PWRQACCEPTn` 位偏移。
4. `PWRQDENY` 位偏移。
5. `PWRQACTIVE` 位偏移。

驱动解析逻辑位于：

- `pmctrl_sc/pmctrl_pm_domain.c:1389`

## 5. 下电握手流程

```text
用户态/compositor
        |
        v
DRM atomic commit / CRTC disable
        |
        v
linlondp_crtc_atomic_disable()
  - pipeline disable
  - flush + wait flip done
        |
        v
linlondp_crtc_unprepare()
  - GCU mode = INACTIVE
  - pxlclk off
        |
        v
pm_runtime_put()
        |
        v
linlondp_dev_suspend()
  - disconnect TBU/IOMMU
  - IRQ off
  - aclk off
        |
        v
VO genpd power off（前提：DPU 是 VO 域的 consumer；当前 DT 尚未满足）
        |
        v
pmu_pd_disable()
        |
        v
pmu_qchan_lp_ctrl(is_lp=true)
  1. poll PWRQACTIVE == 0
  2. write PWRQREQN = 0
  3. poll PWRQACCEPTN == 0
        |
        v
NoC idle -> clocks off -> isolation on -> memory low power
-> reset assert -> power switch off -> regulator off
```

### 5.1 VO genpd power off 的触发入口

linlondp 驱动不会直接调用 `pmu_pd_disable()`。VO 电源域下电由 Linux genpd 框架判定并调用 PMCTRL 注册的 `power_off` 回调：

```text
genpd core 判定 VO-SUBSYS 可以断电
        |
        v
genpd->power_off
        |
        v
pmctrl_pd_power_off()
        |
        v
pmctrl_pd_power(false)
        |
        v
pmu_pd_disable()
        |
        v
pmu_qchan_lp_ctrl(is_lp=true)
```

回调注册位置：

- `pmctrl_sc/pmctrl_pm_domain.c:1574`：`genpd.power_off = pmctrl_pd_power_off`
- `pmctrl_sc/pmctrl_pm_domain.c:1101`：`pmctrl_pd_power_off()`
- `pmctrl_sc/pmctrl_pm_domain.c:832`：`pmu_pd_disable()`

genpd 框架只有在以下条件满足时才会真正调用 `power_off`：

- 电源域当前处于 on 状态，且系统 suspend 没有进行中。
- 域没有配置为 `GENPD_FLAG_ALWAYS_ON` 或 `GENPD_FLAG_RPM_ALWAYS_ON`。
- 没有仍处于 on 状态的子域。
- 域内所有 consumer 都已经 runtime suspend。

对应判断位于 `linux-6.6/drivers/base/power/domain.c:669`。当前 VO 节点没有配置 `pd-always-on` 或 `pd-rpm-always-on`。

接入正常 runtime PM 后，下电触发链应为：

```text
CRTC disable
        |
        v
pm_runtime_put()
        |
        v
DPU 作为 VO 域最后一个 consumer runtime suspend
        |
        v
genpd_runtime_suspend()
        |
        v
genpd_power_off()
        |
        v
pmctrl_pd_power_off()
        |
        v
pmu_pd_disable()
```

除 runtime PM 外，以下路径也可能触发 `power_off`：

| 触发来源 | 主要路径 | 说明 |
| --- | --- | --- |
| Runtime PM | `genpd_runtime_suspend()` -> `genpd_power_off()` | 域内最后一个 consumer runtime suspend |
| 系统 suspend | `genpd_suspend_noirq()` -> `genpd_sync_power_off()` -> `_genpd_power_off()` | 域内 consumer 全部 suspended 时 |
| 未使用域回收 | `genpd_power_off_unused()` -> `genpd_power_off_work` | `late_initcall` 尝试关闭没有 consumer 的域；是否发生取决于 probe 时序和 `pd_ignore_unused` 启动参数 |
| 强制关闭 | `pmctrl_pd` sysfs 写 `off` 或 `atool pmctrl -d` | 直接调用 `pmu_pd_disable()`，绕过 genpd consumer 判定 |
| probe 默认关闭 | `pd-default-off` -> `pmu_pd_disable()` | 当前 VO 节点未配置该属性 |

### 5.2 为什么必须先进入 INACTIVE

`PWRQACTIVE` 是 DPU 内部多个活跃标志的汇总，包括 GCU、LPU、CU、DOU 的活动状态、DOU prefetch 以及接口 wakeup 信号。只要显示链路仍在取数或输出，`PWRQACTIVE` 就不会落下。

因此软件必须先：

- 停止 layer 取数和输出。
- 等待当前帧/配置生效。
- 将 GCU 模式设置为 `INACTIVE`。
- 断开 TBU，避免 IOMMU 侧仍有未完成事务。

否则 PMCTRL 会在 `wait qchannel active off timeout!` 处超时。

### 5.3 访问 DPU 寄存器会重新拉高 PWRQACTIVE

每次 APB 寄存器访问都会通过 `PWAKEUP` 拉高 `PWRQACTIVE`。因此：

- 下电握手开始后，不能再访问 DPU 寄存器。
- 上电握手完成前，也不能访问 DPU 寄存器。
- 调试时读取 `0x17000000` 范围内的 DPU 寄存器会破坏低功耗状态判断。

读取 PMCTRL 的 `0x14000000` 不受此限制。

## 6. 上电握手流程

```text
VO genpd power on
        |
        v
pmu_pd_enable()
  - regulator on
  - power switch on
  - clock/reset/isolation/memory restore
        |
        v
pmu_qchan_lp_ctrl(is_lp=false)
  1. write PWRQREQN = 1
  2. poll PWRQACCEPTN == 1
        |
        v
linlondp_dev_resume()
  - aclk on
  - IRQ on
  - TBU/IOMMU connect
  - restore lost HW state
        |
        v
CRTC enable -> pipeline configure -> flush
        |
        v
DPU 开始工作，PWRQACTIVE 重新变为 1
```

## 8. 当前代码状态与注意事项

### 8.1 PWR Q-Channel 与 ACLK Q-Channel 分开处理

DPU 还实现了一套 ACLK Q-Channel，用于外部 ACLK 时钟门控。它与本文讨论的 PWR Q-Channel 不是同一套握手：

- PWR Q-Channel：控制整个 DPU 电源域上下电，由 PMCTRL/genpd 驱动执行。
- ACLK Q-Channel：控制 ACLK 时钟域外部 gating，通常由时钟控制器或 SoC 时钟逻辑处理。

当前 linlondp 驱动中只有 `clk_prepare_enable()/clk_disable_unprepare()` 对 `aclk` 的管理，没有直接操作 `ACLKQREQn/ACLKQACCEPTn/ACLKQACTIVE` 的代码。ACLK 的 Q-Channel 需要结合 SoC 时钟控制器实现确认。

## 9. 常见问题定位

| 现象 | 可能原因 | 检查点 |
| --- | --- | --- |
| `wait qchannel active off timeout!` | DPU 仍处于工作状态，或仍有寄存器访问 | GCU mode 是否为 `INACTIVE`；DOU 是否停止输出；是否还在访问 DPU 寄存器 |
| `qchannel off timeout!` | DPU 未接受下电请求 | `PWRQREQN` 是否成功写 0；`PWRQACCEPTn` 是否一直为 1；`PWRQDENY` 是否为 1 |
| `qchannel on timeout!` | 上电请求未完成 | `PWRQREQN` 是否成功写 1；电源、时钟、复位是否已经恢复 |
| 下电后读 DPU 寄存器异常或功耗回升 | APB 访问重新拉高 `PWRQACTIVE` | 下电后只读 PMCTRL 状态，不读 DPU 寄存器 |
| 自动 runtime suspend 未触发 Q-Channel | DPU 未挂接 VO 电源域 | 检查 DPU 节点 `power-domains` 是否启用 |

