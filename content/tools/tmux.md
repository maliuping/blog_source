+++
date = '2026-07-15T10:15:14+08:00'
draft = false
title = 'Tmux'
categories = ["Tools"]
tags = ["Tmux"]
+++

# 1. overviw

之前没搞懂tmux的本质，用的时候也很难受，用着用着就放弃了。最近发现tmux原来是这样的组织结构。它包含三层层级结构：

```Plain Text
tmux
 ├─ Session（会话）→ 一个独立工作区
 │   ├─ Window（窗口）→ 相当于标签页
 │   │   ├─ Pane（面板）→ 分割窗格
```

举个例子，我一般这么用：

- **Window 0**: auto_test build/edit

- **Window 1**：左右分两个面板，左边写代码，右边build

- **Window 2**: blog source edit

全在一个终端窗口里搞定，不用在terminal 的好几个标签之间跳来跳去。



# 2. 最常用的操作（前缀键 Ctrl\+b）

## 2.1 注意前缀键的使用方式是：

1. 先按Ctrl \+ b

2. 松开

3. 再按其他组合键

## 2.2 会话管理

- 创建新会话：`tmux new -s mysession`

- 列出所有会话：`tmux ls`

- 脱离当前会话：Ctrl\+b d

- 重新接入会话：`tmux attach -t mysession`

- 重命名会话：Ctrl\+b $

## 2.3 窗口管理

- 新建窗口：Ctrl\+b c

- 切换窗口：Ctrl\+b \+ 数字键

- 上/下一个：Ctrl\+b p / Ctrl\+b n

- 重命名窗口：Ctrl\+b ,

- 关闭窗口：Ctrl\+b \&

## 2.4 面板管理

- 水平分割：Ctrl\+b "

- 垂直分割：Ctrl\+b %

- 切换面板：Ctrl\+b \+ 方向键

- 调整大小：Ctrl\+b \+ 按住方向键

- 关闭面板：Ctrl\+b x

**滚动看历史？** Ctrl\+b \[ 进入复制模式，方向键翻页，q 退出。简单粗暴。

## 2.5 窗口名字
1. 创建会话时指定窗口名字
```Text
tmux new -s work -n editor
```

2. 创建窗口时指定名字
```Text
tmux new-window -n editor
```
3. 重命名窗口
```Text
快捷键：
Ctrl+b ,

然后输入：
editor
```

# 3. 实际使用感受

```Markdown
# 连上服务器
$ tmux new -s dev                # 建个叫 dev 的会话

# 在 dev 里
Ctrl+b c                         # 新开窗口 1：写代码
Ctrl+b %                         # 垂直分屏：左边写，右边build

Ctrl+b c                         # 新开窗口 2：blog edit

Ctrl+b c                         # 新开窗口 3：auto_test

Ctrl+b d                         # 脱离会话，关电脑回家

# 第二天到公司
$ tmux attach -t dev             # 一切原封不动
```





# 4. 进阶玩法

## 4.1 鼠标支持

在 `~/.tmux.conf` 里加一行：

```Plain Text
set -g mouse on
```

然后就能用鼠标点击切窗口、拖拽调面板大小了。


