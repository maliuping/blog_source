+++
date = '2026-06-23T09:56:33+08:00'
draft = false
title = 'Linux 内核调试：ftrace 通用方法（以DRM/显示子系统为例）'
tags = ["Linux", "Ftrace"]
+++

## 1. 为什么要学 ftrace

在内核调试里，经常会遇到这样的问题：

* **这个 callback 到底是谁调的？**
* **某个函数有没有真正执行到？**
* **一次 ioctl / probe / 中断 / commit 的完整调用链是什么？**
* **多个回调的执行顺序是什么？**
* **某个驱动回调是在 probe 阶段调用，还是运行时 atomic/stream on 阶段调用？**

这类问题，如果只靠 `grep` 源码，通常只能知道“**理论上谁可能会调用**”；但实际运行时到底走了哪条路径、有没有进入某个分支、调用顺序是否符合预期，还是需要**运行时观测**。

而 **ftrace** 是 Linux 内核里最通用、最实用的一套函数级 tracing 机制，适合做以下事情：

* 跟踪函数是否被调用
* 观察函数调用顺序
* 观察“谁调用了谁”
* 从一个入口函数出发，查看整条调用树
* 分析 probe / bind / irq / workqueue / atomic commit / stream on 等流程

这篇文章整理一套 **ftrace 的通用使用模板**，并以 **DRM 显示子系统**中的 bridge / atomic callback 为例，说明如何用它排查问题。后续切换到其他子系统时，只需要替换过滤函数名即可。

---

# 2. 先回答一个常见疑问：ftrace 需要写“钩子函数”吗？

**通常不需要。**

这是很多人第一次接触 ftrace 时最容易混淆的地方。

## 2.1 ftrace 和“写钩子”的区别

### ftrace（function / function_graph）

* **不需要改内核代码**
* **不需要写 hook 函数**
* 直接通过 `/sys/kernel/tracing/`（或 `/sys/kernel/debug/tracing/`）配置
* 让内核去跟踪指定函数的调用

### 你自己写 `pr_info()` / `dump_stack()`

* 需要改驱动源码
* 适合快速定位某个点

### tracepoint

* 需要内核本身提供 tracepoint，或者自己在代码里加 tracepoint

### kprobe / kretprobe

* 可以不改源码动态挂到函数入口/返回
* 但比 ftrace 更偏“可编程”，使用复杂度更高

---

## 2.2 自己驱动里的 `static` 函数也能 trace 吗？

**可以。**

例如驱动里有：

```c
static void im_mipi_tx_bridge_mode_set(struct drm_bridge *bridge,
				       const struct drm_display_mode *mode,
				       const struct drm_display_mode *adjusted_mode)
{
	...
}
```

或者：

```c
static int im_mipi_tx_bridge_attach(struct drm_bridge *bridge,
				    enum drm_bridge_attach_flags flags)
{
	...
}
```

这种函数虽然是 `static`，但只要：

* 编译进内核/模块
* 没有被完全优化掉
* 内核开启了 function tracing 相关配置

就可以被 ftrace 跟踪。

**ftrace 跟踪的是“编译后的函数入口”**，不是“导出符号”的概念。

---

# 3. ftrace 能解决什么问题

我一般把 ftrace 的用途归纳成下面几类：

## 3.1 确认某个函数有没有被调用

比如：

* `im_mipi_tx_bridge_mode_set()` 有没有走到？
* `drm_atomic_bridge_chain_enable()` 有没有被执行？
* `clk_set_rate()` 是否真的被某个路径调用？

## 3.2 看调用顺序

比如：

* `mode_set -> pre_enable -> enable` 的顺序是不是这样？
* `disable` 和 `post_disable` 谁先谁后？
* `probe -> component bind -> drm_bridge_attach` 的顺序是什么？

## 3.3 看“谁调用了谁”

比如：

* `im_mipi_tx_bridge_mode_set()` 是谁调的？
* 是 `drm_atomic_bridge_chain_mode_set()` 调的，还是别的 helper？
* 某个中断处理函数是硬中断上下文进来的，还是 threaded irq？

## 3.4 从入口函数出发，看一整条调用链

比如：

* 从 `drm_atomic_commit()` 出发，到底会走哪些 bridge / encoder / connector 回调？
* 从 `spi_sync()` 出发，控制器驱动里哪些函数被调用？
* 从 `regulator_enable()` 出发，底层 PMIC 驱动怎么走？

---

# 4. 内核配置要求

要使用 function / function_graph tracer，内核通常至少需要这些配置：

```config
CONFIG_DEBUG_FS=y
CONFIG_TRACING=y
CONFIG_FTRACE=y
CONFIG_FUNCTION_TRACER=y
CONFIG_FUNCTION_GRAPH_TRACER=y
CONFIG_DYNAMIC_FTRACE=y
```

建议再打开：

```config
CONFIG_STACKTRACE=y
CONFIG_KALLSYMS=y
CONFIG_KALLSYMS_ALL=y
```

这样输出会更完整。

## 4.1 查看当前内核配置

如果系统支持 `/proc/config.gz`：

```bash
zcat /proc/config.gz | egrep 'CONFIG_(FTRACE|FUNCTION_TRACER|FUNCTION_GRAPH_TRACER|DEBUG_FS|TRACING|DYNAMIC_FTRACE)'
```
---

# 5. tracing 目录在哪里

新内核通常在：

```bash
/sys/kernel/tracing
```

有些系统也可能在：

```bash
/sys/kernel/debug/tracing
```

建议统一用一个变量：

```bash
TR=/sys/kernel/tracing
[ -d "$TR" ] || TR=/sys/kernel/debug/tracing
echo "$TR"
```

如果 tracefs/debugfs 没挂载，可以手动挂载：

```bash
mount -t tracefs nodev /sys/kernel/tracing 2>/dev/null || true
mount -t debugfs none /sys/kernel/debug 2>/dev/null || true
```

---

# 6. ftrace 两种最常用模式

ftrace 里我最常用的是两种 tracer：

1. **function**
2. **function_graph**

---

# 7. function tracer：看“有没有被调、谁调了它”

## 7.1 适用场景

适合回答：

* 某个函数到底有没有执行
* 某个 callback 是谁直接调的
* 多个关键函数的粗略顺序是什么

例如在 DRM 里排查：

* `im_mipi_tx_bridge_mode_set`
* `im_mipi_tx_bridge_atomic_enable`
* `drm_atomic_commit`

---

## 7.2 基本步骤

### 第一步：准备 tracing 环境

```bash
TR=/sys/kernel/tracing
[ -d "$TR" ] || TR=/sys/kernel/debug/tracing

echo 0 > $TR/tracing_on
echo nop > $TR/current_tracer
echo > $TR/trace
echo > $TR/set_ftrace_filter
```

### 第二步：写入要跟踪的函数列表

例如以 DRM bridge callback 为例：

```bash
cat > /tmp/ftrace_filter.txt <<'EOF'
drm_atomic_commit
drm_atomic_helper_commit_tail
drm_atomic_helper_commit_modeset_disables
drm_atomic_helper_commit_modeset_enables
drm_atomic_bridge_chain_pre_enable
drm_atomic_bridge_chain_enable
drm_atomic_bridge_chain_disable
drm_atomic_bridge_chain_post_disable
drm_bridge_attach
devm_drm_of_get_bridge
im_mipi_tx_chan_attach
im_mipi_tx_bridge_attach
im_mipi_tx_bridge_mode_set
im_mipi_tx_bridge_atomic_pre_enable
im_mipi_tx_bridge_atomic_enable
im_mipi_tx_bridge_atomic_disable
EOF

cat /tmp/ftrace_filter.txt > $TR/set_ftrace_filter
```

### 第三步：打开 function tracer

```bash
echo function > $TR/current_tracer
echo 1 > $TR/tracing_on
```

### 第四步：触发你要观察的流程

例如：

* 跑一次 `modetest`
* 让 Weston 切一个显示模式
* 插拔显示器
* 触发某个驱动的 stream on/off

### 第五步：停止 tracing 并读取结果

```bash
echo 0 > $TR/tracing_on
cat $TR/trace
```

---

## 7.3 输出怎么看

你可能看到类似这样的内容：

```text
# tracer: function
#
# entries-in-buffer/entries-written: 20/20   #P:4
#
#                                _-----=> irqs-off/BH-disabled
#                               / _----=> need-resched
#                              | / _---=> hardirq/softirq
#                              || / _--=> preempt-depth
#                              ||| / _-=> migrate-disable
#                              |||| /     delay
#           TASK-PID     CPU#  |||||  TIMESTAMP  FUNCTION
#              | |         |   |||||     |         |
        modetest-22892   [000] ...1. 405429.747737: drm_atomic_commit <-drm_atomic_helper_set_config
        modetest-22892   [000] ...1. 405429.747995: drm_atomic_helper_commit_modeset_disables <-linlondp_kms_commit_tail
        modetest-22892   [000] ...1. 405429.748016: im_mipi_tx_bridge_mode_set <-drm_bridge_chain_mode_set
        modetest-22892   [000] ...1. 405429.748028: drm_atomic_bridge_chain_pre_enable <-linlondp_atomic_helper_commit_modeset_enables
        modetest-22892   [000] ...1. 405429.748034: im_mipi_tx_bridge_atomic_pre_enable <-drm_atomic_bridge_call_pre_enable
        modetest-22892   [000] ...1. 405429.912283: drm_atomic_bridge_chain_enable <-linlondp_atomic_helper_commit_modeset_enables
        modetest-22892   [000] ...1. 405429.912300: im_mipi_tx_bridge_atomic_enable <-drm_atomic_bridge_chain_enable
        modetest-22892   [000] ...1. 405429.913302: drm_atomic_commit <-drm_mode_obj_set_property_ioctl
        modetest-22892   [000] ...1. 405429.913503: drm_atomic_helper_commit_modeset_disables <-linlondp_kms_commit_tail
        modetest-22892   [000] ...1. 405430.004372: drm_atomic_commit <-drm_mode_obj_set_property_ioctl
        modetest-22892   [000] ...1. 405430.004573: drm_atomic_helper_commit_modeset_disables <-linlondp_kms_commit_tail
        modetest-22892   [000] ...1. 405430.095519: drm_atomic_commit <-drm_mode_obj_set_property_ioctl
        modetest-22892   [000] ...1. 405430.095690: drm_atomic_helper_commit_modeset_disables <-linlondp_kms_commit_tail
     kworker/3:1-142     [003] ...1. 405431.995661: drm_atomic_commit <-drm_framebuffer_remove
     kworker/3:1-142     [003] ...1. 405431.995815: drm_atomic_helper_commit_modeset_disables <-linlondp_kms_commit_tail
        modetest-22892   [003] ...1. 405432.013936: drm_atomic_commit <-drm_atomic_helper_disable_all
        modetest-22892   [003] ...1. 405432.014079: drm_atomic_helper_commit_modeset_disables <-linlondp_kms_commit_tail
        modetest-22892   [003] ...1. 405432.014091: drm_atomic_bridge_chain_disable <-disable_outputs
        modetest-22892   [003] ...1. 405432.014249: im_mipi_tx_bridge_atomic_disable <-drm_atomic_bridge_chain_disable
        modetest-22892   [003] ...1. 405432.014368: drm_atomic_bridge_chain_post_disable <-disable_outputs
```

这个输出至少能回答三件事：

1. `im_mipi_tx_bridge_mode_set()` **确实被调用了**
2. 它的直接调用者是 `drm_bridge_chain_mode_set()`
3. `mode_set` 发生在 `enable` 之前

---

# 8. function_graph tracer：看完整调用树

如果你不只是想看“有没有调用”，而是想看：

> **从某个入口函数开始，到底一路调用了哪些函数、层级关系是什么**

那我更推荐 **function_graph**。

---

## 8.1 适用场景

特别适合排查：

* `drm_atomic_commit()` 的完整 commit 时序
* probe / bind / attach 流程
* 某个 ioctl 进来后，内核里走了哪些 helper / callback
* 某个子系统的 enable / disable / runtime PM 调用链

---

## 8.2 基本步骤

### 第一步：清理现场

```bash
TR=/sys/kernel/tracing
[ -d "$TR" ] || TR=/sys/kernel/debug/tracing

echo 0 > $TR/tracing_on
echo nop > $TR/current_tracer
echo > $TR/trace
echo > $TR/set_ftrace_filter
echo > $TR/set_graph_function
```

### 第二步：设置“调用树入口”

比如想看一次 DRM atomic commit：

```bash
echo drm_atomic_commit > $TR/set_graph_function
```

这表示：
**从 `drm_atomic_commit()` 开始，画调用树。**

### 第三步：设置过滤函数

如果不加过滤，输出可能会非常大。
例如只保留 DRM bridge / atomic 主线 + 自己驱动函数：

```bash
cat > $TR/set_ftrace_filter <<'EOF'
drm_atomic_commit
drm_atomic_helper_commit_tail
drm_atomic_helper_commit_modeset_disables
drm_atomic_helper_commit_modeset_enables
drm_atomic_bridge_chain_pre_enable
drm_atomic_bridge_chain_enable
drm_atomic_bridge_chain_disable
drm_atomic_bridge_chain_post_disable
drm_bridge_attach
devm_drm_of_get_bridge
im_mipi_tx_chan_attach
im_mipi_tx_bridge_attach
im_mipi_tx_bridge_mode_set
im_mipi_tx_bridge_atomic_pre_enable
im_mipi_tx_bridge_atomic_enable
im_mipi_tx_bridge_atomic_disable
EOF
```

### 第四步：启用 function_graph tracer

```bash
echo function_graph > $TR/current_tracer
echo 1 > $TR/tracing_on
```

### 第五步：触发显示流程并查看结果

```bash
echo 0 > $TR/tracing_on
cat $TR/trace
```

---

## 8.3 典型输出示意

你会看到类似这样的层级结构：

```text
# tracer: function_graph
#
# CPU  DURATION                  FUNCTION CALLS
# |     |   |                     |   |   |   |
 0)               |  drm_atomic_commit() {
 0)               |    drm_atomic_helper_commit_modeset_disables() {
 0)   6.250 us    |      im_mipi_tx_bridge_mode_set [im_mipi_tx]();
 0) + 40.364 us   |    }
 0)               |    drm_atomic_bridge_chain_pre_enable() {
 0)               |      im_mipi_tx_bridge_atomic_pre_enable [im_mipi_tx]() {
 1) * 16769.01 us |      } /* im_mipi_tx_bridge_atomic_pre_enable [im_mipi_tx] */
 0) @ 164444.2 us |    }
 0)               |    drm_atomic_bridge_chain_enable() {
 0) ! 147.656 us  |      im_mipi_tx_bridge_atomic_enable [im_mipi_tx]();
 0) ! 220.052 us  |    }
 0) @ 165581.7 us |  }
 0)               |  drm_atomic_commit() {
 0) + 19.270 us   |    drm_atomic_helper_commit_modeset_disables();
 0) * 90886.71 us |  }
 0)               |  drm_atomic_commit() {
 0) + 19.270 us   |    drm_atomic_helper_commit_modeset_disables();
 0) * 90976.82 us |  }
 0)               |  drm_atomic_commit() {
 0) + 24.739 us   |    drm_atomic_helper_commit_modeset_disables();
 0) * 91021.61 us |  }
 ------------------------------------------
 1) modetes-22861  => kworker-27127
 ------------------------------------------

 1)               |  drm_atomic_commit() {
 1) + 19.271 us   |    drm_atomic_helper_commit_modeset_disables();
 1) * 20695.83 us |  }
 ------------------------------------------
 1) kworker-27127  => modetes-22861
 ------------------------------------------

 1)               |  drm_atomic_commit() {
 1)               |    drm_atomic_helper_commit_modeset_disables() {
 1)               |      drm_atomic_bridge_chain_disable() {
 1) + 68.229 us   |        im_mipi_tx_bridge_atomic_disable [im_mipi_tx]();
 1) ! 231.771 us  |      }
 1) * 28403.12 us |      drm_atomic_bridge_chain_post_disable();
 2) @ 182573.1 us |    } /* drm_atomic_helper_commit_modeset_disables */
 2) @ 182787.7 us |  } /* drm_atomic_commit */
```

这个输出可以非常直观地回答：

* `mode_set / pre_enable / enable` 的顺序
* `disable / post_disable` 在哪个阶段
* 某个 callback 是在 atomic helper 的哪个阶段触发的

---

# 9. 一个非常重要的技巧：先看 available_filter_functions

很多时候你写进 `set_ftrace_filter` 的函数名不生效，不是 ftrace 坏了，而是：

* 函数名写错了
* 内核里实际名字和你想的不一样
* 函数被编译优化成别的形式
* 模块还没加载，函数暂时不可见

所以我一般都会先查一下：

```bash
grep im_mipi_tx /sys/kernel/debug/tracing/available_filter_functions
grep drm_atomic_bridge_chain /sys/kernel/debug/tracing/available_filter_functions
```

如果你在调别的子系统，就换成：

```bash
grep spi_ /sys/kernel/debug/tracing/available_filter_functions
grep clk_ /sys/kernel/debug/tracing/available_filter_functions
grep regulator_ /sys/kernel/debug/tracing/available_filter_functions
```

这个步骤非常重要，它能帮你确认：**这个函数到底能不能被 ftrace 追踪。**

---

# 10. 一套通用排查思路：从“阶段”切 trace，而不是一口气全追

这是我最推荐的工作流。

不要一上来就把整个子系统全 trace 了，否则输出量会非常大，而且难以看出重点。

更好的方法是：

## 10.1 先把问题拆成阶段

例如 DRM 显示链路，可以拆成：

### 阶段 A：probe / bind / attach

关注：

* 设备 probe
* component bind, `component_bind_all`
* `devm_drm_of_get_bridge`
* `drm_bridge_attach`

### 阶段 B：mode 校验 / atomic check

关注：

* `drm_atomic_check_only`
* `drm_atomic_helper_check_modeset`
* bridge / encoder 的 `mode_valid`、`atomic_check`

### 阶段 C：mode_set / pre_enable / enable

关注：

* `drm_atomic_commit`
* `drm_atomic_bridge_chain_mode_set`
* `drm_atomic_bridge_chain_pre_enable`
* `drm_atomic_bridge_chain_enable`

### 阶段 D：disable / post_disable

关注：

* `drm_atomic_helper_commit_modeset_disables`
* `drm_atomic_bridge_chain_disable`
* `drm_atomic_bridge_chain_post_disable`

---

## 10.2 其他子系统也可以这样拆

比如 SPI 控制器：

* probe 阶段
* `spi_setup`
* `spi_sync` / `transfer_one_message`
* DMA / IRQ 完成
* runtime PM suspend/resume

比如时钟子系统：

* probe / clk_hw_register
* `clk_prepare_enable`
* `clk_set_rate`
* 父时钟切换
* disable/unprepare

比如 regulator：

* probe
* `regulator_get`
* `regulator_enable`
* `set_voltage`
* disable

**核心思路都是一样的：按阶段缩小观察范围。**

---

# 11. 一个通用 ftrace 模板脚本

下面这个脚本可以作为通用模板，适合以后迁移到别的子系统。
思路是：

* 统一清理 tracing 环境
* 按函数列表写 filter
* 选择 `function` 或 `function_graph`
* 可选设置 graph 入口函数
* 启动 tracing

---

## 11.1 通用脚本 `trace_template.sh`

```bash
#!/bin/sh
TR=/sys/kernel/tracing
if [ ! -d "$TR" ]; then
        TR=/sys/kernel/debug/tracing
fi
TR=/sys/kernel/debug/tracing

MODE="${1:-graph}"          # function / graph
ENTRY="${2:-}"              # function_graph 的入口函数，可为空
FILTER_FILE="${3:-/tmp/ftrace_filter.txt}"

echo "[trace] using tracing path: $TR"
echo "[trace] mode=$MODE entry=$ENTRY filter=$FILTER_FILE"

echo 0 > $TR/tracing_on
echo nop > $TR/current_tracer
echo > $TR/trace
echo > $TR/set_ftrace_filter
echo > $TR/set_graph_function

if [ -f "$FILTER_FILE" ]; then
	cat "$FILTER_FILE" > $TR/set_ftrace_filter
fi

case "$MODE" in
	function)
		echo function > $TR/current_tracer
		;;
	graph)
		echo function_graph > $TR/current_tracer
		if [ -n "$ENTRY" ]; then
			echo "$ENTRY" > $TR/set_graph_function
		fi
		;;
	*)
		echo "Unknown mode: $MODE"
		exit 1
		;;
esac

echo 1 > $TR/tracing_on

echo
echo "Tracing started."
echo "Do your test now, then stop with:"
echo "  echo 0 > $TR/tracing_on"
echo "Dump trace with:"
echo "  cat $TR/trace > /tmp/trace.log"
```

---

# 12. 以 DRM 显示子系统为例：三套常用 filter

下面给三套我实际会用的 filter 文件示例。

---

# 13. 示例一：追 bridge attach / 拓扑搭建阶段

适合排查：

* `devm_drm_of_get_bridge()` 找到了谁
* `drm_bridge_attach()` 有没有执行
* 自己的 `bridge_attach()` 是谁调的

## 13.1 `drm_attach_filter.txt`

```text
drm_bridge_attach
drm_bridge_attach_next_bridge
devm_drm_of_get_bridge
im_mipi_tx_chan_attach
im_mipi_tx_bridge_attach
```

## 13.2 使用方式

```bash
./trace_template.sh graph drm_bridge_attach /tmp/drm_attach_filter.txt
```

然后触发：

* 驱动 probe
* component bind
* connector 初始化
* 你的 channel attach 流程

---

# 14. 示例二：追 atomic modeset 主线

适合排查：

* `mode_set / pre_enable / enable`
* callback 的执行顺序
* 自己的 bridge callback 是谁调的

## 14.1 `drm_atomic_filter.txt`

```text
drm_atomic_commit
drm_atomic_helper_commit_tail
drm_atomic_helper_commit_modeset_disables
drm_atomic_helper_commit_modeset_enables
drm_atomic_bridge_chain_mode_set
drm_atomic_bridge_chain_pre_enable
drm_atomic_bridge_chain_enable
drm_atomic_bridge_chain_disable
drm_atomic_bridge_chain_post_disable
im_mipi_tx_bridge_mode_set
im_mipi_tx_bridge_atomic_pre_enable
im_mipi_tx_bridge_atomic_enable
im_mipi_tx_bridge_atomic_disable
```

## 14.2 使用方式

```bash
./trace_template.sh graph drm_atomic_commit /tmp/drm_atomic_filter.txt
```

然后触发：

* `modetest`
* 桌面环境切换分辨率
* 屏幕开关 / 热插拔 / modeset

---

# 15. 示例三：只看“有没有被调用”

适合快速验证：

* 某个 callback 到底有没有走到
* 不关心整棵调用树，只想知道顺序和直接调用者

## 15.1 `drm_quick_filter.txt`

```text
drm_atomic_commit
drm_atomic_bridge_chain_mode_set
drm_atomic_bridge_chain_enable
im_mipi_tx_bridge_mode_set
im_mipi_tx_bridge_atomic_enable
```

## 15.2 使用方式

```bash
./trace_template.sh function "" /tmp/drm_quick_filter.txt
```

---

# 16. 如何把这个模板迁移到其他子系统

真正需要替换的，通常只有两样东西：

1. **入口函数**
2. **过滤函数列表**

---

# 17. 迁移示例：SPI 子系统

如果你要排查 SPI 传输，可以把入口函数换成：

```text
spi_sync
```

过滤函数可以换成类似：

```text
spi_sync
__spi_sync
spi_transfer_one_message
xxx_spi_transfer_one
xxx_spi_irq
```

然后：

```bash
./trace_template.sh graph spi_sync /tmp/spi_filter.txt
```

---

# 18. 迁移示例：Clock 子系统

如果你要看时钟 enable / set_rate 路径，可以尝试：

入口函数：

```text
clk_set_rate
```

或者：

```text
clk_prepare_enable
```

过滤函数示例：

```text
clk_prepare_enable
clk_core_prepare
clk_core_enable
clk_set_rate
clk_change_rate
xxx_clk_set_rate
```

---

# 19. 迁移示例：Regulator 子系统

入口函数：

```text
regulator_enable
```

过滤函数示例：

```text
regulator_enable
_regulator_enable
regulator_set_voltage
xxx_regulator_enable
```

---

# 20. function 与 function_graph 如何选择

我自己的经验是：

## 用 `function` 的场景

* 想快速确认某个函数有没有执行
* 想知道某个 callback 的直接调用者
* 想看少量关键函数的调用顺序
* 希望输出尽量简单

## 用 `function_graph` 的场景

* 想看完整调用树
* 想看一个 ioctl / commit / transfer / enable 的整条路径
* 想搞清楚多个 helper / callback 的层级关系
* 在分析一个子系统的主流程

如果拿不准，建议优先用 **function_graph**，然后通过 **缩小 filter** 控制输出规模。

---

# 21. 调试时的实战建议

## 21.1 先用 `available_filter_functions` 确认函数名

这个步骤很关键，能避免很多无效折腾。

## 21.2 不要一次 trace 整个子系统

先拆阶段，再 trace。

## 21.3 先看 attach / probe，再看 runtime 流程

很多问题其实在拓扑搭建阶段就已经埋下了，运行时只是“表现出来”。

## 21.4 function_graph 输出太大时，先缩小入口函数

比如只从：

* `drm_atomic_commit`
* `spi_sync`
* `clk_set_rate`

开始画。

## 21.5 必要时和 `dump_stack()` 联合使用

如果某个 callback 你只想知道“这一处到底是谁调的”，在驱动里临时加：

```c
pr_info("%s\n", __func__);
dump_stack();
```

通常比大面积 trace 更快出答案。

---

# 22. 以 DRM bridge callback 为例的典型问题与解法

## 问题 1：`im_mipi_tx_bridge_mode_set()` 是谁调的？

### 方法

* `function` 模式
* filter 加：

  * `drm_atomic_bridge_chain_mode_set`
  * `im_mipi_tx_bridge_mode_set`

### 观察点

看输出中：

```text
im_mipi_tx_bridge_mode_set <- drm_atomic_bridge_chain_mode_set
```

---

## 问题 2：`mode_set / pre_enable / enable` 顺序是什么？

### 方法

* `function_graph`
* 入口：`drm_atomic_commit`
* filter 加：

  * `drm_atomic_bridge_chain_mode_set`
  * `drm_atomic_bridge_chain_pre_enable`
  * `drm_atomic_bridge_chain_enable`
  * 自己的 bridge callback

### 观察点

确认顺序是否为：

```text
mode_set -> pre_enable -> enable
```

---

## 问题 3：`im_mipi_tx_bridge_attach()` 是在 probe 阶段还是 commit 阶段调用？

### 方法

* 单独追 attach 阶段
* 入口：`drm_bridge_attach`
* filter 加：

  * `im_mipi_tx_bridge_attach`
  * `im_mipi_tx_chan_attach`
  * `devm_drm_of_get_bridge`

### 观察点

看 attach 是在：

* 驱动 probe / bind 时调用
* 还是某个 connector 初始化阶段调用

---

# 23. 总结

ftrace 是 Linux 内核里非常适合“**学习调用链**”和“**定位 callback 实际调用路径**”的工具。
相比单纯看源码，它最大的价值在于：

* 能告诉你**实际运行时**走了哪条路径
* 能确认**某个函数到底有没有被调用**
* 能看清楚**回调的执行顺序**
* 能把复杂子系统拆成若干阶段逐步观察

在使用上，我建议把它当成一个通用模板，而不是“只给某个子系统用的技巧”：

1. **先确定阶段**（probe / attach / check / enable / disable）
2. **再确定入口函数**
3. **最后列出少量关键 filter 函数**
4. **优先用 function_graph 看主线**
5. **必要时再配合 `dump_stack()` 或 dynamic debug**

对于 DRM 显示、SPI、I2C、Clock、Regulator、IRQ、Net 等子系统，这套方法都适用。
真正变化的只是：**你要追踪的入口函数和过滤函数列表**。

如果把它用熟了，后面你在看一个陌生子系统时，效率会高很多：
不是先陷在源码里翻半天，而是先用 ftrace 把“活的调用链”跑出来，再回过头去看代码，就会清楚很多。
