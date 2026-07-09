+++
date = '2026-07-09T15:23:56+08:00'
draft = false
title = 'Dphy clocks overview'
categories = ["Display"]
tags = ["DPHY"]
+++

在 MIPI D-PHY 中，经常会看到 **Lane Rate、HS Clock、Byte Clock、Pixel Clock** 等概念，它们分别位于不同的模块，不要混为一谈。

# 1. Pixel Clock（像素时钟）

**作用对象：DPU / DPI / IPI**

Pixel Clock 决定像素数据进入 DSI Host 的速度。

例如：

* 1920×1080@60 RGB888
* Pixel Clock = **148.5 MHz**

表示：

> Controller 每秒接收 148.5M 个 Pixel。

它属于**显示时序**，与 D-PHY 串行发送方式无关。

# 2. Lane Rate（Bit Rate）

**作用对象：D-PHY Serial Link**

Lane Rate 又叫：

* Data Rate
* Bit Rate
* HS Bit Rate

表示：

> 一条 Data Lane 每秒传输多少 bit。

例如：

```text
Lane Rate = 1 Gbps
```

表示：

```text
1 秒发送 10^9 bit
```

如果使用 4 Lane，则总带宽约为：

```text
4 × 1Gbps = 4Gbps
```

Lane Rate 是 D-PHY 最核心的参数。

# 3. HS Clock（High-Speed Clock）

HS Clock 是 D-PHY Serializer 内部使用的高速时钟。

由于 D-PHY 采用 **DDR（Double Data Rate）** 传输：

* 上升沿发送 1 bit
* 下降沿发送 1 bit

因此：

```text
Lane Rate = 2 × HS Clock
```

例如：

| Lane Rate | HS Clock |
| --------: | -------: |
|    1 Gbps |  500 MHz |
|  1.5 Gbps |  750 MHz |
|    2 Gbps |    1 GHz |

# 4. Byte Clock（PPI Clock）

Byte Clock 是 **Controller 与 D-PHY(PPI Interface)** 之间工作的时钟。

它不是 PHY 线上的时钟，而是：

> Serializer 每隔一个 Byte Clock，从 Controller 获取一次并行数据。

对于 Synopsys DSI Host（16-bit PPI）：

每个 Byte Clock 周期传输：

```text
16 bit
```

因此：

```text
Byte Clock = Lane Rate / 16
```

也可以写成：

```text
Byte Clock = HS Clock / 8
```

例如：

| Lane Rate | Byte Clock |
| --------: | ---------: |
|    1 Gbps |   62.5 MHz |
|  1.5 Gbps |  93.75 MHz |
|    2 Gbps |    125 MHz |

# 5. 四个 Clock 所在位置

可以用下面这张图理解它们的位置：

```text
              Display Pipeline

+--------------------+
| DPU / DPI / IPI    |
| Pixel Clock        |
+--------------------+
          │
          ▼
+--------------------+
| DSI Controller     |
| Byte Clock (PPI)   |
+--------------------+
          │ 16-bit Parallel
          ▼
+--------------------+
| D-PHY Serializer   |
| HS Clock (DDR)     |
+--------------------+
          │ Serial Bit Stream
          ▼
+--------------------+
| MIPI D-PHY Lane    |
| Lane Rate          |
+--------------------+
```

---

# 6. 四者关系总结

对于 **16-bit PPI** 的 DSI Host，可记住下面三个公式：

```text
Lane Rate = 2 × HS Clock
```

```text
Byte Clock = HS Clock / 8
```

```text
Byte Clock = Lane Rate / 16
```

如果再结合 Pixel Clock，则可以得到整个显示链路：

```text
Pixel Clock
        │
        ▼
DSI Controller
(Byte Clock)
        │
        ▼
Serializer
(HS Clock)
        │
        ▼
MIPI Lane
(Lane Rate)
```


| PPI Width |     HS Clock |                    PPI Clock |
| --------: | -----------: | ---------------------------: |
|     8 bit | LaneRate / 2 |   LaneRate / 8 = HSClock / 4 |
|    16 bit | LaneRate / 2 |  LaneRate / 16 = HSClock / 8 |
|    32 bit | LaneRate / 2 | LaneRate / 32 = HSClock / 16 |


其中：

* **Pixel Clock**：决定像素进入 Controller 的速度。
* **Byte Clock**：决定 Controller 向 D-PHY 输出并行数据的速度。
* **HS Clock**：Serializer 的内部工作时钟，采用 DDR。
* **Lane Rate**：最终在 MIPI Data Lane 上传输的串行比特率，也是 D-PHY 最常见的标称速率。

> **补充说明：** `Byte Clock = Lane Rate / 16` 并不是 MIPI D-PHY 规范固定规定的公式，而是 **16-bit PPI 接口宽度** 下得到的结果。如果某些 D-PHY IP 使用 **8-bit** 或 **32-bit** PPI，则关系会分别变为 `Lane Rate / 8` 或 `Lane Rate / 32`。因此，这个公式应理解为 **"PPI 位宽决定了 Byte Clock 与 Lane Rate 的换算关系"**，而不是 D-PHY 的通用公式。这样也能解释为什么不同厂商的 Databook 中会出现不同的 Byte Clock 定义。