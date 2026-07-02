+++
date = '2026-06-30T09:52:25+08:00'
draft = true
title = 'Kernel_mode_setting'
+++

驱动程序必须通过在 DRM 设备上调用 `drmm_mode_config_init()` 来初始化 **Mode Setting Core**。
该函数会初始化 `struct drm_device` 中的 `mode_config` 成员，并且不会返回失败。初始化完成后，
还需要通过设置以下字段来完成 **Mode Configuration** 的配置。

- `int min_width, min_height`; `int max_width, max_height`; 以像素（pixel）为单位，
  指定 Frame Buffer（帧缓冲）的最小和最大宽度、高度。这些值用于告诉 DRM Core：
  支持创建的 framebuffer 最小尺寸是多少；支持创建的 framebuffer 最大尺寸是多少。
- `struct drm_mode_config_funcs *funcs`; 指向 Mode Setting 回调函数（Mode Setting Functions）的指针。
  这里需要由驱动提供一组 `drm_mode_config_funcs` 回调，用于完成 mode setting 相关的核心操作，例如：
  - 创建 framebuffer（fb_create）
  - Atomic Commit 检查（atomic_check）
  - Atomic Commit 执行（atomic_commit）
  - 输出轮询（output_poll_changed）等

# 1. overview
KMS 向用户空间（userspace）提供的基本对象模型相对简单。Framebuffer（帧缓冲）（由 `struct drm_framebuffer` 表示，
参见 Frame Buffer Abstraction）作为图像数据的来源，提供像素数据给 Plane。Plane（由`struct drm_plane`表示，详细
内容参见 Plane Abstraction）负责引用一个 Framebuffer，并定义该图像如何显示。一个 CRTC（由`struct drm_crtc`表示，
参见 CRTC Abstraction）可以接收一个、多个，甚至零个 Plane 的像素数据，并将它们进行混合（blending），生成最终的显示输出。

关于 Plane 之间如何进行混合（Blending）的具体过程，将在 Plane Composition Properties 及其相关章节中进行更详细的介绍。

---

如果结合 DRM 的数据流来理解，这段话实际上描述的是 KMS 最核心的数据流：
```c
Framebuffer
      │
      ▼
   Plane 0 ───┐
              │
   Plane 1 ───┼──► CRTC (Blend) ───► Encoder ───► Bridge ───► Connector ───► Panel
              │
   Plane 2 ───┘
```
各对象的职责可以概括为：

Framebuffer：存放图像数据（显存中的像素）。
Plane：决定使用哪个 Framebuffer，以及图像的位置、缩放、格式、透明度等显示属性。
CRTC：将多个 Plane 按照层级和混合规则进行合成，并产生最终的扫描输出时序（Timing）。
Encoder / Bridge / Connector / Panel：负责将 CRTC 输出的数据转换、传输并最终显示到屏幕上。

这也是 DRM/KMS 中最经典的对象关系：Framebuffer → Plane → CRTC → Encoder → Connector → Display。