+++
date = '2026-08-19T20:20:11+08:00'
draft = false
title = 'AXI Read Signal 中 Burst 的理解'
categories = ["Bus"]
tags = ["AXI"]
+++

## 1. Burst 是什么

在 AXI / DDR / DPU 等 SoC 数据通路中，**Burst 可以理解为一次连续的数据传输事务**。

一次 Read Transaction 并不一定只读取一个数据，而是可以从某个起始地址开始，连续读取多个数据。

例如 AXI Data Width 为 128 bit：

```text
一次 Read Burst
┌─────────────────────────────┐
│ Beat 0 : 16 Bytes           │
│ Beat 1 : 16 Bytes           │
│ Beat 2 : 16 Bytes           │
│ Beat 3 : 16 Bytes           │
└─────────────────────────────┘

总数据量 = 4 × 16 Bytes = 64 Bytes
```

因此：

* **Burst**：一次完整的连续传输
* **Beat**：Burst 中的一次数据传输
* **Burst Length**：一个 Burst 包含多少个 Beat

可以简单记忆：

> **Burst 是“一次搬一串数据”，Beat 是“这一串里面的一拍”。**

---

## 2. Burst、Beat 和 Transaction 的关系

可以把三者理解成：

```text
Transaction
    │
    └── Read Burst
            │
            ├── Beat 0
            ├── Beat 1
            ├── Beat 2
            └── Beat 3
```

在 AXI Read 中：

```text
AR Channel
    │
    │  发起一次 Read Burst
    ▼
Read Data Channel
    │
    ├── Beat 0
    ├── Beat 1
    ├── Beat 2
    └── Beat 3
             │
             └── RLAST = 1
```

因此：

```text
ARVALID && ARREADY
```

表示 **Read Address 请求握手**，它描述的是一个 Read Burst 的开始。

而：

```text
RVALID && RREADY
```

表示一次 **Read Data Beat 的传输**。

最后一个 Beat 通常通过：

```text
RLAST = 1
```

来标识。

---

## 3. AXI 中 Burst Length 怎么表示

AXI 中常见的 Read Address 信号包括：

```text
ARADDR
ARLEN
ARSIZE
ARBURST
```

其中：

### ARLEN

`ARLEN` 表示 Burst 中 Beat 数量减 1：

```text
Burst Length = ARLEN + 1
```

例如：

```text
ARLEN = 0  → 1 Beat
ARLEN = 1  → 2 Beats
ARLEN = 3  → 4 Beats
ARLEN = 7  → 8 Beats
```

这是看 RTL 或波形时非常容易弄错的地方。

---

### ARSIZE

`ARSIZE` 表示每个 Beat 的数据大小，通常：

```text
Beat Bytes = 2 ^ ARSIZE
```

例如：

```text
ARSIZE = 4
```

表示：

```text
2^4 = 16 Bytes / Beat
```

如果：

```text
ARLEN  = 7
ARSIZE = 4
```

那么：

```text
Burst Length = 7 + 1 = 8 Beats

每 Beat = 16 Bytes

总数据量 = 8 × 16 = 128 Bytes
```

---

## 4. 一个完整的 Read Burst

假设：

```text
ARADDR  = 0x1000
ARLEN   = 3
ARSIZE  = 4
ARBURST = INCR
```

那么：

```text
Burst Length = 4 Beats
Beat Size    = 16 Bytes
```

地址连续递增：

```text
Beat 0 → 0x1000 ~ 0x100F
Beat 1 → 0x1010 ~ 0x101F
Beat 2 → 0x1020 ~ 0x102F
Beat 3 → 0x1030 ~ 0x103F
```

总共：

```text
4 × 16 = 64 Bytes
```

可以理解成：

```text
ARADDR = 0x1000
           │
           ▼
      ┌─────────┐
      │ Beat 0  │ 16B
      ├─────────┤
      │ Beat 1  │ 16B
      ├─────────┤
      │ Beat 2  │ 16B
      ├─────────┤
      │ Beat 3  │ 16B
      └─────────┘
           │
           └── RLAST = 1
```

---

## 5. Burst 和 AXI Handshake

AXI 使用 VALID / READY 进行握手。

### Address Channel

```text
ARVALID && ARREADY
```

表示：

> Master 发起的 Read Address 请求被 Slave/Interconnect 接收。

这个请求描述的是**整个 Burst**。

例如：

```text
ARADDR  = 0x1000
ARLEN   = 7
ARSIZE  = 4
```

并不是只请求 `0x1000` 的 16 Bytes，而是请求：

```text
0x1000 ~ 0x107F
```

共：

```text
8 Beats × 16 Bytes = 128 Bytes
```

---

### Read Data Channel

之后 Slave 返回数据：

```text
RVALID && RREADY
```

每发生一次：

```text
RVALID && RREADY
```

就是一个 Beat 的数据传输。

例如：

```text
RVALID    ┌────┐    ┌────┐    ┌────┐    ┌────┐
          │    │    │    │    │    │    │    │
RREADY  ─────────────────────────────────────────

          Beat0     Beat1     Beat2     Beat3
                                             │
                                             └─ RLAST
```

所以需要注意：

> **一个 AR handshake 对应一个 Burst，而一个 Burst 可以对应多个 R channel handshake。**

---

## 6. 为什么需要 Burst

Burst 的主要目的之一是提高连续内存访问效率。

例如 DPU 需要读取一段连续的 framebuffer 数据。

如果每 16 Bytes 单独发起一次 Read：

```text
Read 0 → 0x1000
Read 1 → 0x1010
Read 2 → 0x1020
Read 3 → 0x1030
...
```

需要多次独立的 Read Transaction。

使用 Burst：

```text
ARADDR  = 0x1000
ARLEN   = 7
ARSIZE  = 4
ARBURST = INCR
```

一次请求即可完成：

```text
0x1000
0x1010
0x1020
0x1030
0x1040
0x1050
0x1060
0x1070
```

即：

```text
8 Beats × 16 Bytes = 128 Bytes
```

对于 DDR / Memory Controller 来说，这种连续访问通常更加高效。

---

## 7. RTL 中如何理解 Burst Signal

在 RTL 中经常会看到类似：

```text
read_req
read_addr
read_burst_len
read_burst_cnt
read_data
read_data_valid
read_last
```

可以按照下面的逻辑理解：

```text
                 Read Request
                      │
                      ▼
              ┌──────────────┐
              │ Burst Config │
              │              │
              │ addr         │
              │ burst_len    │
              └──────┬───────┘
                     │
                     ▼
               Read Burst
                     │
       ┌─────────────┼─────────────┐
       ▼             ▼             ▼
    Beat 0         Beat 1        Beat N
       │             │             │
       └─────────────┴─────────────┘
                     │
                   Last
```

如果看到：

```text
read_burst_cnt
```

通常就是在记录：

> 当前这个 Burst 已经传输了多少个 Beat。

例如一个 8 Beat Burst：

```text
burst_cnt

0 → 1 → 2 → 3 → 4 → 5 → 6 → 7
                              │
                              └── Last
```

具体实现会根据 RTL 的计数方式有所不同，有些设计可能从 `0` 计到 `burst_len-1`，也有些从 `1` 计到 `burst_len`。

---

## 8. Burst 和压缩数据不要混淆

在 DPU / FBDC / AFBC 等场景中，Burst 属于**内存访问层**，而 Tile / Compression 属于**数据组织和压缩层**。

例如 DPU_PREPARE：

```text
GPU / ISP
    │
    │ Compressed Tile
    ▼
   DDR
    │
    │ AXI Read Burst
    ▼
┌───────────────┐
│ DPU_PREPARE   │
│               │
│ AXI Read      │
└───────┬───────┘
        │
        ├── Beat 0
        ├── Beat 1
        ├── Beat 2
        ├── ...
        └── Beat N
              │
              ▼
        Decompress
              │
              ▼
             T2R
              │
              ▼
           Raster
              │
              ▼
             DPU
```

这里：

**Compressed Tile**

描述的是：

> DDR 中存储的数据是什么格式。

而：

**AXI Burst**

描述的是：

> 如何从 DDR 把这些数据搬回来。

所以二者是不同层次的概念。

---

## 9. 看波形时的分析方法

以后看到一个 `read signal`，可以按照下面几个问题逐层分析：

### 第一步：谁发起 Read？

找到：

```text
ARVALID
ARREADY
```

确认：

```text
ARVALID && ARREADY
```

发生的时刻。

---

### 第二步：这个 Burst 从哪里开始？

看：

```text
ARADDR
```

例如：

```text
ARADDR = 0x1000
```

---

### 第三步：一个 Burst 有多少 Beat？

看：

```text
ARLEN
```

计算：

```text
Burst Length = ARLEN + 1
```

---

### 第四步：一个 Beat 有多大？

看：

```text
ARSIZE
```

计算：

```text
Beat Size = 2^ARSIZE Bytes
```

---

### 第五步：整个 Burst 有多少数据？

计算：

```text
Total Bytes
    = Burst Length × Beat Size
```

例如：

```text
ARLEN  = 7
ARSIZE = 4

=> 8 Beats
=> 16 Bytes / Beat
=> 128 Bytes / Burst
```

---

### 第六步：数据什么时候真正传输？

观察：

```text
RVALID
RREADY
```

每一次：

```text
RVALID && RREADY
```

就是一个 Beat。

---

### 第七步：什么时候结束？

看：

```text
RLAST
```

最后一个 Beat 一般：

```text
RLAST = 1
```

表示这个 Burst 结束。

---

## 10. 最终形成一个完整的心智模型

看 AXI Read Burst 时，可以脑子里直接建立下面这个模型：

```text
                    AXI READ
                       │
                       ▼
              ┌────────────────┐
              │ Address Channel │
              │                │
              │ ARADDR         │
              │ ARLEN          │
              │ ARSIZE         │
              │ ARBURST        │
              └───────┬────────┘
                      │
                一个 Burst 请求
                      │
                      ▼
              ┌────────────────┐
              │   Read Burst   │
              │                │
              │ Beat 0         │
              │ Beat 1         │
              │ Beat 2         │
              │ ...            │
              │ Beat N         │
              └───────┬────────┘
                      │
                      ▼
              Read Data Channel
                      │
                RVALID & RREADY
                      │
                      ▼
                   RLAST=1
                      │
                      ▼
                 Burst 完成
```

### 核心概念

| 概念          | 含义                   |
| ------------- | -----------------      |
| Transaction   | 一次完整的 AXI 事务    |
| Burst         | 一次连续的多 Beat 传输 |
| Beat          | 一次数据传输           |
| ARADDR        | Burst 起始地址         |
| ARLEN         | Beat 数量 - 1          |
| ARSIZE        | 每个 Beat 的数据大小   |
| ARBURST       | 地址递增方式           |
| RVALID/RREADY | Read Data Beat 握手    |
| RLAST         | Burst 最后一个 Beat    |

最重要的三个公式：

```text
Burst Length = ARLEN + 1

Beat Bytes = 2^ARSIZE

Burst Total Bytes
    = (ARLEN + 1) × 2^ARSIZE
```

> **“这一次内存 Read 请求，要连续搬多少个 Beat；每个 Beat 搬多少字节；什么时候搬完。”**

