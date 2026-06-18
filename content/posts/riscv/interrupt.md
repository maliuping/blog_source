+++
date = '2026-06-11T00:21:53+08:00'
draft = false
title = 'Riscv Interrubt'
tags = ["RISC-V", "Interrupt"]
+++

1 中断概念
在riscv中，中断属于异常的一种，其处理过程与异常处理类似。大多数riscv处理器同时支持M模式和S模式下的中断。在默认情况下，中断会在M模式下处理。如处理器支持S模式，那么可以有选择的把部分中断委托给S模式来处理。
1.1 中断类型
借mcause寄存器中 中断ID的描述，中断可分为定时器中断，软件中断，外部中断。
- 软件中断：
  - 用于处理器内核之间的通信，处理器核间中断（IPI）
- 定时器中断：
  - 用于操作系统的时钟中断
  - riscv体系规定处理器必须有一个定时器，通常实现在M模式。
  - riscv体系结构为定时器定义了两个64位的寄存器mtime和mtimecmp。它们通常实现在CLINT中
- 外部中断：
  - 通常是指来自处理器外部设备的中断。
[图片]

通常将软件中断和定时器中断概括为本地中断(CLINT)， 将外部中断称为全局中断。通过sifive FU740-C000可以理解下，FU740-C000的中断架构如下。
本地中断直接发送到特定的hart，并具有专用的中断值。这种方式可以减少中断延迟，因为不需要仲裁来确定哪个hart处理给定的请求，也不需要额外的内存访问来确定中断原因。
软件和定时器中断是由Core-Local Interruptor（CLINT）生成的本地中断。FU740-C000不包含其他本地中断源。
相比之下，全局中断通过平台级中断控制器（PLIC）路由，可以将中断导向系统中的任何hart。将全局中断与hart(s)解耦，允许设计PLIC以适应平台需求，从而支持广泛的属性，如中断数量、优先级和路由方案。
默认情况下，所有中断都在机器模式下处理。对于支持特权模式的hart，可以选择将中断委托给特权模式处理。
[图片]

1.2 中断处理过程


1.3 中断处理相关寄存器
寄存器
含义
mtvec（Machine Trap-Vector Base-Address）
它保存发生异常时处理器需要跳转到的地址
mepc（Machine Exception Program Counter）
当 trap 发生时，hart 会将发生 trap 所对应的指令的地址值（pc）保存在 mepc 中
mcause（Machine Cause)
当 trap 发生时，hart 会设置该寄存器通知我们 trap 发生的原因
mtval（Machine Trap value)
它保存了 exception 发生时的附加信息：譬如访问地址出错时 的地址信息、或者执行非法指令时的指令本身，对于其他异常，它的值为 0。
mstatus（Machine Status)
用于跟踪和控制 hart 的当前操作状态（特别地，包括关闭和打开全局中断）。
mscratch（Machine Scratch)
Machine 模式下专用寄存器，可以自己定义其用法
mie（Machine Interrupt Enable)
用于进一步控制（打开和关闭）software interrupt/timer interrupt/external interrupt
mip（Machine Interrupt Pending)
它列出目前已发生等待处理的中断
mideleg

1. mie(Machine Interrupt Enable)
打开（1）或者关闭（0） M/S 模式下对应的 External/Timer/Software 中
[图片]


2. mip(Machine Interrupt Pending)
获取当前 M/S 模式 下对应的 External/Timer/Software 中断是否发
[图片]
2 CLINT
Riscv 处理器一般支持软件中断、时钟中断，这两个本地中断，它们属于处理器内核私有中断，直接发送到处理器内核，而不需要经过中断控制器的路由。如下所示
[图片]
clint支持的中断采用固定优先级策略，高优先级的中断可以抢占低优先级的中断。clint支持的中断如下表所示。中断号越大，优先级越高。可以直接通过中断id号写mie寄存器完成软件中断和时间中断的开关。
名称
中断号
说明
ssip
1
S模式下的软件中断
msip
3
M模式下的软件中断
stip
5
S模式下的时钟中断
mtip
7
M模式下的时钟中断



2.1 寄存器
Nuclei 处理器的clint 寄存器地址来自 iregion + offset, 其中iregion 可以通过寄存器mirgb_info(0x7f7) 得到。
    csrr t0, CSR_IREGION_BASE
    /* iregion base -> t0 */
    srai t0, t0, 10 // t0 = t0 >> 10
    slli t0, t0, 10 // t0 = t0 << 10
[图片]
[图片]
[图片]
Nuclei 处理器的clint的寄存器如下表，其中MSIP寄存器主要用来触发软件中断，用于处理器硬件线程之间的通信，即IPI。MTIMECMP和MTIME是M模式下与定时器相关的寄存器。MTIME寄存器返回系统的时钟周期数。MTIMECMP寄存器用来设置时间间隔，当MTIME返回的时间大于或者等于MTIMECMP寄存器的值时，便会触发定时器中断。
[图片]
[图片]
[图片]
[图片]

3 PLIC
一个专为RISC-V系统环境工作而设计的中断控制器。PLIC将各种设备中断多路复用到Hart上下文的外部中断线上，并支持硬件中断优先级。
PLIC支持多达1023个中断（0是保留的）和15872个上下文，但实际的中断数和上下文数取决于PLIC的具体实现。
[图片]
然而，实现必须遵循PLIC操作参数内每个寄存器的偏移量。声称符合PLIC标准的PLIC-Compliant标准PLIC应遵循下述部分提到的实现。
3.1 中断源

3.2 PLIC编程接口

3.2.1 优先级寄存器
寄存器
功能
内存映射地址
priority
设置某一路中断源的优先级
Base + 4 * interrupt-id
- 每个 PLIC 中断源对应一个寄存器，用于配置该中断源的优 先级。
- plic标准, 0 表示对该中断源禁用中断。 其余优先级，1 最低，随着数值的增大优先级越高。
- 如果两个中断源优先级相同，则根据中断源的 ID 值进一步 区分优先级，ID 值越小的优先级越高。

PLIC Interrupt Source Priority Memory Map
0x000000: Reserved (interrupt source 0 does not exist)
0x000004: Interrupt source 1 priority
0x000008: Interrupt source 2 priority
...
0x000FFC: Interrupt source 1023 priority

3.2.2 Pending寄存器
寄存器
功能
内存映射地址
pending
用于指示某一路中断 源是否发生
BASE + 0x1000 + ((interrupt-id) / 32) * 4
- 标准 PLIC 包含 32 个 32 位的 Pending 寄存器，每一个 bit 对应 一个中断源，如果为 1 表示该中断源上发生了中断（进入 Pending 状态），有待 hart 处理，否则表示该中断源上当前无 中断发生。
- Pending 寄存器中断的 Pending 状态可以通过 claim 方式清除。
- 第一个 Pending 寄存器的第 0 位对应不存在的 0 号中断源，其 值永远为 0

Plic interrupt pending Bits memory map
0x001000: Interrupt Source #0 to #31 Pending Bits
...
0x00107C: Interrupt Source #992 to #1023 Pending Bits

3.2.3 Enable寄存器
寄存器
功能
内存映射地址
enable
针对某个 hart 开启或者关闭某 一路中断源。
BASE + 0x2000 +
每个中断源对应 Enable 寄存器的一个 bit， 将对应的 bit 位设置为 1 表示使能该中断源，否则表示关 闭该中断源。

PLIC Interrupt Enable Bits Memory Map
0x002000: Interrupt Source #0 to #31 Enable Bits on context 0
...
0x00207C: Interrupt Source #992 to #1023 Enable Bits on context 0
0x002080: Interrupt Source #0 to #31 Enable Bits on context 1
...
0x0020FC: Interrupt Source #992 to #1023 Enable Bits on context 1
0x002100: Interrupt Source #0 to #31 Enable Bits on context 2
...
0x00217C: Interrupt Source #992 to #1023 Enable Bits on context 2
0x002180: Interrupt Source #0 to #31 Enable Bits on context 3
...
0x0021FC: Interrupt Source #992 to #1023 Enable Bits on context 3
...
...
...
0x1F1F80: Interrupt Source #0 to #31 on context 15871
...
0x1F1FFC: Interrupt Source #992 to #1023 on context 15871

NUCLEI PLIC Interrupt Enable Bits Memory Map
[图片]
3.2.4 threshold寄存器
寄存器
功能
内存映射地址
threshold

针对某个 hart 设置中断源 优先级的阈值。
BASE + 0x200000 + (hart) * 0x1000
- 每个 context 有 1 个 Threshold 寄存器用于设置中断优先级 的阈值。
- 所有小于或者等于（<=）该阈值的中断源即使发生了也会 被 PLIC 丢弃。特别地，当阈值为 0 时允许所有中断源上发 生的中断；当阈值为 7 时丢弃所有中断源上发生的中断
PLIC Interrupt Priority Thresholds Memory Map
0x200000: Priority threshold for context 0
0x201000: Priority threshold for context 1
0x202000: Priority threshold for context 2
0x203000: Priority threshold for context 3
...
...
...
0x3FFF000: Priority threshold for context 15871

3.2.5 claim/complete寄存器
寄存器
功能
内存映射地址
Claim/Complete

BASE + 0x200004 + (hart) * 0x1000
- Claim 和 Complete 是同一个寄存器，每个 context 一个。
- 对该寄存器执行读操作称之为 Claim，即获取当前发生的最高优先 级的中断源 ID。Claim 成功后会清除对应的 Pending 位。
- 对该寄存器执行写操作称之为 Complete。所谓 Complete 指的 是通知 PLIC 对该路中断的处理已经结束

PLIC Interrupt Claim/Completion Process Memory Map
0x200004: Interrupt Claim/Completion Process for context 0
0x201004: Interrupt Claim/Completion Process for context 1
0x202004: Interrupt Claim/Completion Process for context 2
0x203004: Interrupt Claim/Completion Process for context 3
...
...
...
0x3FFF004: Interrupt Claim/Completion Process for context 15871
NUCLEI PLIC Interrupt Claim/Completion Process Memory Map
[图片]
3.3 PLIC操作流程
[图片]
1. 中断优先级设置，这是针对中断源设置的。优先级值 0 保留表示“中断禁用”，优先级随着整数值的增加而增加；相同优先级的全局中断之间的平局通过中断ID来打破；ID最低的中断具有最高的有效优先级。
2. 设置context enable 寄存器
3. 设置优先级阈值寄存器

3.4 DTS 中断解析
plic:plic@1c000000 {
        compatible = "riscv,plic0";
        reg = <0x0 0x1c000000 0x0 0x3000000>;
        #interrupt-cells = <0x1>;
        #address-cells = <0x0>;
        interrupts-extended = <&cpu0_intc 0xb &cpu0_intc 0x9>;
        interrupt-controller;
        riscv,ndev = <92>;
};
plic:plic@1c000000：
- 这是该设备节点的标签（plic）和地址（@1c000000）。它表示 PLIC 设备位于地址 0x1c000000。
compatible = "riscv,plic0";：
- 这个字段表示该设备的兼容性标识符。riscv,plic0 表示该设备是一个 RISC-V 系统的中断控制器（PLIC，即平台中断控制器）。操作系统通过这个字段来识别硬件设备，并加载合适的驱动程序。
reg = <0x0 0x1c000000 0x0 0x210000>;：
- reg 描述了设备的物理地址和大小。在这里：
  - 0x0 0x1c000000 是设备的起始地址（即 PLIC 的基地址是 0x1c000000）。
  - 0x0 0x210000 是该设备的大小，表示设备占用了 0x210000 字节的地址空间。
#interrupt-cells = <0x1>;：
- #interrupt-cells 表示该设备支持的中断描述符的单元数。在这个例子中，每个中断描述符由一个单元组成，表示中断使用一个单元来传递。
#address-cells = <0x0>;：
- #address-cells 表示该设备节点下的地址字段所使用的单元数。在此例中，#address-cells 设置为 0，表示没有子地址需要描述，通常用于没有子设备的简单设备。
interrupts-extended = <&cpu0_intc 0xb &cpu0_intc 0x9>;：
- interrupts-extended 定义了该设备与 CPU 中断控制器之间的中断映射关系。它使用 &cpu0_intc 来引用 cpu0_intc 中断控制器，并分别指定了两个中断源：
  - 0xb 和 0x9 分别表示 PLIC 控制的两个中断号。0xb表示M模式下的外部中断， 0x09表示S模式下的外部中断
  - 这些值通常表示在中断控制器中注册的中断源，可能对应某些硬件事件。
interrupt-controller;：
- 该字段表示当前设备节点是一个中断控制器设备。它告诉操作系统这个设备负责处理中断，并且该设备会控制中断的分发和管理。
riscv,ndev = <92>;：
- riscv,ndev 字段表示该 RISC-V 系统中 PLIC 控制的设备数量。<92> 表示 PLIC 中总共有 92 个中断源。这些设备和中断源通常是与硬件外设相关的中断请求（IRQ）
参考：
riscv-plic-spec
