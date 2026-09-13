+++
date = '2026-09-03T16:18:46+08:00'
draft = true
title = 'Probe fail 排查思路'
categories = ["Debug"]
tags = ["Linux", "Debug"]
+++


# Linux Platform Driver 加载成功但 Probe 未执行问题排查

## 1. 问题背景

在 HAPS FPGA 环境中，通过 `modprobe` 加载 `xx-sc` 驱动模块时，没有看到预期的内核 Log 信息。

最初的问题是：

> 如何确认 `.ko` 模块是否已经成功加载？如果模块已经加载，为什么驱动的 `probe()` 仍然没有执行？

这里需要首先区分 Linux 驱动模型中的几个不同阶段：

```text
加载 .ko 模块
      │
      ▼
执行 module_init()
      │
      ▼
注册 platform_driver
      │
      ▼
匹配 platform_device
      │
      ▼
调用 probe()
      │
      ▼
驱动完成设备初始化
```

因此：

```text
.ko 已加载
    ≠
probe() 已执行
    ≠
设备已经初始化成功
```

本次问题的排查也正是沿着这条链路逐层进行。

---

# 2. 第一阶段：确认内核模块是否成功加载

首先使用：

```bash
lsmod | grep <module_name>
```

例如：

```bash
lsmod | grep xx_sc
```

同时检查：

```bash
ls /sys/module/<module_name>/
```

如果：

* `lsmod` 可以找到对应模块；
* `/sys/module/<module_name>/` 存在；

则可以基本确认：

```text
.ko 文件
   │
   ▼
modprobe
   │
   ▼
模块成功加载到 Kernel
```

但是这一步只能说明：

> **模块已经进入内核。**

并不能说明 `platform_driver` 已经匹配到设备，更不能说明 `probe()` 已经执行。

---

# 3. 第二阶段：确认 Platform Driver 是否注册成功

对于 Platform Driver，可以检查：

```bash
ls /sys/bus/platform/drivers/
```

查找目标 Driver：

```bash
ls /sys/bus/platform/drivers/ | grep -i xx_sc
```

本次问题中可以看到：

```text
xx_sc
```

说明：

```text
modprobe xx_sc
      │
      ▼
module_init()
      │
      ▼
platform_driver_register()
      │
      ▼
/sys/bus/platform/drivers/xx_sc
```

因此可以确认：

> **`xx_sc` Platform Driver 已经成功注册到 Platform Bus。**

此时问题已经不是 `.ko` 加载失败，也不是 Driver 注册失败。

---

# 4. 第三阶段：确认 Platform Device 是否存在

接下来检查：

```bash
ls /sys/bus/platform/devices/
```

本次找到对应设备：

```text
17060000.xx_sc
```

查看设备目录：

```bash
ls -la /sys/bus/platform/devices/17060000.xx_sc/
```

可以看到：

```text
driver_override
modalias
of_node -> ../../../firmware/devicetree/base/xx_sc@17060000
power
subsystem -> ../../../bus/platform
uevent
waiting_for_supplier
```

这说明：

```text
Device Tree
      │
      ▼
xx_sc@17060000
      │
      ▼
Platform Device 已成功创建
      │
      ▼
17060000.xx_sc
```

因此：

> **DTS 节点已经被内核解析，并成功创建了对应的 Platform Device。**

---

# 5. 第四阶段：检查 Driver 是否已经绑定到 Device

正常情况下，如果 Platform Driver 成功绑定到 Platform Device，设备目录中应该存在：

```text
driver -> ../../../bus/platform/drivers/<driver_name>
```

例如：

```text
/sys/bus/platform/devices/17060000.xx_sc/
    │
    └── driver -> ../../../bus/platform/drivers/xx_sc
```

但是本次实际检查结果中：

```text
17060000.xx_sc/
```

目录下没有：

```text
driver
```

软链接。

因此当前状态为：

```text
xx_sc.ko 已加载                ✓
        │
        ▼
xx_sc platform_driver 已注册   ✓
        │
        ▼
17060000.xx_sc device 已创建   ✓
        │
        ▼
Driver 未绑定                  ✗
        │
        ▼
probe() 未执行
```

这一步是整个问题排查中的重要分界点。

可以确定问题不再是：

```text
modprobe 失败
```

而是：

```text
为什么 Driver 没有和 Device 完成绑定？
```

---

# 6. 第五阶段：检查 Device Tree Compatible

首先查看 Platform Device 的 `modalias`：

```bash
cat /sys/bus/platform/devices/17060000.xx_sc/modalias
```

输出：

```text
of:Nxx_scT(null)Cim,xx_sc
```

然后查看运行时 Device Tree 中的 `compatible`：

```bash
tr '\0' '\n' < \
/sys/bus/platform/devices/17060000.xx_sc/of_node/compatible
```

输出：

```text
im,xx_sc
```

说明当前运行的 Device Tree 中：

```dts
compatible = "im,xx_sc";
```

Driver 侧需要存在对应的 OF Match Table，例如：

```c
static const struct of_device_id vo_ss_of_match[] = {
    {
        .compatible = "im,xx_sc",
    },
    {}
};

MODULE_DEVICE_TABLE(of, vo_ss_of_match);
```

对应关系为：

```text
Device Tree
compatible = "im,xx_sc"
           │
           ▼
Platform Driver
of_match_table
           │
           ▼
"im,xx_sc"
```

如果两者不匹配，则 Driver 无法绑定到 Device。

可以通过下面命令检查模块导出的 OF Alias：

```bash
modinfo <module_name> | grep alias
```

---

# 7. 第六阶段：检查 Supplier Dependency

由于 Device 已经存在、Driver 也已经注册，但仍然没有完成绑定，因此继续检查：

```bash
cat /sys/bus/platform/devices/17060000.xx_sc/waiting_for_supplier
```

结果为：

```text
1
```

这说明该 Device 当前存在：

```text
Supplier Dependency
```

即：

```text
Consumer Device
      │
      │ 等待
      ▼
Supplier Device
```

Linux 内核不会立即让 Consumer 完成 Probe，而是等待相关的 Supplier 准备完成。

---

# 8. 使用 devices_deferred 定位具体依赖

这是本次排查中最关键的一步。

执行：

```bash
cat /sys/kernel/debug/devices_deferred
```

得到：

```text
17060000.xx_sc  platform: wait for supplier \
/mipi_tx1@160e2000/ports/port@0/endpoint
```

至此，可以得到当前最明确的依赖关系：

```text
                 Device Tree Graph
                        │
                        ▼

         17060000.xx_sc
              │
              │ Consumer
              │
              │ waiting for supplier
              ▼
      mipi_tx1@160e2000
              │
              ▼
       ports/port@0/endpoint
```

因此当前问题已经从：

```text
xx_sc Driver 为什么没有 Probe？
```

进一步收敛为：

```text
xx_sc 正在等待 mipi_tx1
对应的 Device Tree Endpoint Supplier
```

---

# 9. Device Tree Graph 与 Supplier Dependency

本次 DTS 结构中，`xx_sc` 下存在多个 Channel：

```text
xx_sc@17060000
│
├── vo-ch@0
│
└── vo-ch@1
```

每个 Channel 下存在：

```text
ports
```

这种结构通常用于描述显示 Pipeline 中的硬件连接关系。

逻辑上可能类似：

```text
         xx_sc
           │
           │ Device Tree Graph
           ▼
       vo-ch@X
           │
           ▼
        endpoint
           │
           │ remote-endpoint
           ▼
       mipi_tx1
```

Linux 内核会根据 Device Tree 中的：

```text
ports
port
endpoint
remote-endpoint
```

以及其他 phandle 引用关系建立设备之间的依赖关系。

因此，虽然：

```text
xx_sc Driver
```

可能没有显式调用：

```text
mipi_tx1 Driver
```

但是 Device Tree 中存在设备连接关系时，内核的设备依赖机制仍可能形成：

```text
xx_sc
   │
   │ Consumer
   ▼
mipi_tx1
   │
   │ Supplier
   ▼
准备完成后
   │
   ▼
xx_sc 才允许继续 Probe
```

---

# 10. 当前问题状态

目前已经确认：

```text
┌───────────────────────────────────────┐
│ 1. xx_sc.ko 模块加载                  │
│               ✓                       │
├───────────────────────────────────────┤
│ 2. xx_sc Platform Driver 注册         │
│               ✓                       │
├───────────────────────────────────────┤
│ 3. xx_sc Platform Device 创建         │
│               ✓                       │
├───────────────────────────────────────┤
│ 4. Driver 与 Device 绑定              │
│               ✗                       │
├───────────────────────────────────────┤
│ 5. xx_sc probe() 执行                 │
│               ✗                       │
├───────────────────────────────────────┤
│ 6. waiting_for_supplier               │
│               1                       │
├───────────────────────────────────────┤
│ 7. 当前等待的 Supplier                │
│ mipi_tx1@160e2000/ports/.../endpoint │
└───────────────────────────────────────┘
```

---
# 12. 通用排查流程总结

以后遇到：

> `modprobe` 成功，但驱动没有 Log，或者 `probe()` 没有执行

可以按照下面的顺序排查：

```text
① .ko 是否加载？
    │
    ├── lsmod
    └── /sys/module
          │
          ▼
② Driver 是否注册？
    │
    └── /sys/bus/<bus>/drivers/
          │
          ▼
③ Device 是否存在？
    │
    └── /sys/bus/<bus>/devices/
          │
          ▼
④ Driver 是否绑定 Device？
    │
    └── device/driver -> ...
          │
          ▼
⑤ compatible 是否匹配？
    │
    ├── modalias
    ├── of_node/compatible
    └── modinfo alias
          │
          ▼
⑥ 是否存在 Deferred Probe？
    │
    └── /sys/kernel/debug/devices_deferred
          │
          ▼
⑦ 是否正在等待 Supplier？
    │
    ├── waiting_for_supplier
    └── devices_deferred
          │
          ▼
⑧ 沿 Supplier Dependency 继续向上排查
```

---

# 13. 本次排查的核心经验

本次问题最大的收获是：

> **不要仅仅通过 `modprobe` 的返回结果判断驱动是否正常。**

Linux 驱动加载过程中至少需要区分：

```text
模块加载
    ↓
Driver 注册
    ↓
Device 创建
    ↓
Driver/Device Match
    ↓
Supplier Dependency 满足
    ↓
probe()
    ↓
驱动初始化成功
```

任何一个阶段出现问题，都可能表现为：

```text
modprobe 没有报错
但是没有任何驱动 Log
```

因此，当遇到类似问题时，推荐优先检查：

```bash
lsmod
ls /sys/module/
ls /sys/bus/platform/drivers/
ls /sys/bus/platform/devices/
cat <device>/modalias
cat <device>/waiting_for_supplier
cat /sys/kernel/debug/devices_deferred
dmesg
```

其中：

```text
/sys/kernel/debug/devices_deferred
```

对于 Deferred Probe 和 Supplier Dependency 问题，往往是定位问题的关键信息来源。

