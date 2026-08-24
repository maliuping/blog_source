+++
date = '2026-08-01T14:39:17+08:00'
draft = true
title = 'phy protocol interface'
categories = ["Display"]
tags = ["DPHY"]
+++

ppi(phy protocol interface)用于实现phy lane mode和controller之间的连接。这里描述的接口旨在具有通用性，能够适用于各种情况。

下表中定义了ppi中使用的信号，对于具有多个数据通道的phy, 每个通道都是用一组ppi信号。每个信号已被分配到下面六个类别之一：
- 1）High-Speed transmit signals
- 2）High-Speed receive signals
- 3）Escape mode transmit signals
- 4）Escape mode receive signals
- 5）control signals
- 6）error signals

| symbol                      | Dir      | categories | des                                                                                                       |
| --------                    | -------- | --------   | --------                                                                                                  |
| High-Speed transmit signals |          |            |                                                                                                           |
| TxDataWidthHS \[1:0\]       | I        |            | 高速传输数据总线宽度选择                                                                                  |
| TxWordClkHS                 | O        |            | 高速传输字时钟。该信号用于同步高速传输时钟域中的 PPI 信号, 建议所有传输Lane 模块共享一个 TxWordClkHS 信号 |
| TxDataHS\[7:0\]             | I        |            | 高速传输数据总线宽度                                                                                      |
| TxWordValidHS\[0\]          | I        |            | 高速发送字数据有效                                                                                        |
| TxRequestHs                 |          |            |                                                                                                           |
|                             |          |            |                                                                                                           |
|                             |          |            |                                                                                                           |
|                             |          |            |                                                                                                           |
|                             |          |            |                                                                                                           |
|                             |          |            |                                                                                                           |
|                             |          |            |                                                                                                           |
|                             |          |            |                                                                                                           |
|                             |          |            |                                                                                                           |
|                             |          |            |                                                                                                           |
|                             |          |            |                                                                                                           |
|                             |          |            |                                                                                                           |
|                             |          |            |                                                                                                           |
|                             |          |            |                                                                                                           |
|                             |          |            |                                                                                                           |



# TxWordValidHS 和 TxDataHS的关系
为啥需要 TxWordValidHS ?
- 如果 TxDataHS width = 8 bit, 那么很简单, 每个 TxWordClkHS 周期只能发送 1 byte, 此时 TxWordValidHS[0] 如果为1 表示 TxDataHS[7:0]有效，如果为0 表示 无效。
  ```text
  TxDataHS[7:0]
       |
       v
    一个byte
  ```

- 但是为了提高PPI的吞吐量，DPHY支持更宽的数据接口，比如 TxDataHS[31:0], 一个 TxWordClkHS 周期可以提供 byte0, byte1, byte2, byte3 共4个byte 。
  这里有一个问题如果最后剩余的数据不足4 byte, 例如dsi packet: **AA BB CC** 只有3 byte。那么如下，其中 byte3 无效。
  ```text
   TxDataHS[31:0]
   [31:24]  XX
   [23:16]  CC
   [15:8]   BB
   [7:0]    AA
  ```
  所以需要 TxWordValidHS 用于告诉 DPHY 哪些 byte 要发送。

- 可以这样理解它们之间的关系：
  ```text
               TxWordClkHS
                  |
                  |
       +----------+----------+
       |                     |
       v                     v

     TxDataHS[31:0]       TxWordValidHS[3:0]


       byte0   <----> bit0
       byte1   <----> bit1
       byte2   <----> bit2
       byte3   <----> bit3
  ```
  TxWordValidHS[n] 就是 TxDataHS 每 8-bit byte slice 的 valid 标志。当 PPI 数据宽度超过 8bit 时，一个 TxWordClkHS 周期可以携带多个 byte，通过 TxWordValidHS 指示哪些 byte 需要被发送。
