+++
date = '2026-11-20T11:07:17+08:00'
draft = false
title = 'Dumb Buffer 从 DRM Core 到 Display Driver 的完整处理流程'
tags = ["Linux", "DRM"]
+++
## 1. 概述

在 Linux DRM（Direct Rendering Manager）框架中，Dumb Buffer 是 DRM 提供的一种简单 Buffer 分配机制，主要用于显示场景下的基础图像缓冲区管理。

用户态可以通过 libdrm 提供的 `drmModeCreateDumb()` 创建 Buffer，通过 `mmap()` 映射到用户空间，完成图像数据填充，最后通过 `drmModeAddFB2()` 将 Buffer 封装为 Framebuffer，再通过 DRM KMS 的 Plane、CRTC 等对象完成显示。

整个过程涉及三个重要概念：

- **Dumb Buffer**：DRM UAPI 提供的基础 Buffer 分配接口。
- **GEM Object**：DRM 内核中的通用图形内存管理对象。
- **Framebuffer**：KMS 中描述图像数据格式、尺寸、Stride 以及底层 GEM Object 的对象。

需要特别注意：Dumb Buffer 并不是一种独立的内存管理机制，它通常建立在 GEM 内存管理框架之上。

DRM Core 负责通用的 UAPI、对象管理和调用流程，具体的内存分配、DMA 地址管理以及显示硬件配置则由 DRM Driver 负责。

## 2. Dumb Buffer 整体处理流程

以用户态通过 libdrm 的 `drmModeCreateDumb()` 创建一个 XRGB8888 Buffer 为例，完整调用流程如下：

```text
+------------------------------------------+
|                Userspace                 |
|                                          |
|          drmModeCreateDumb()             |
|                                          |
|          DRM_IOCTL_MODE_CREATE_DUMB      |
+--------------------+---------------------+
                     |
                     v
+------------------------------------------+
|                 DRM Core                 |
|                                          |
|          drm_ioctl()                     |
|          drm_mode_create_dumb_ioctl()    |
|          drm_mode_create_dumb()          |
|                                          |
|          参数检查、计算、分发             |
+--------------------+---------------------+
                     |
                     v
+------------------------------------------+
|              DRM Driver                  |
|                                          |
|          drm_driver.dumb_create          |
|                                          |
|          创建 GEM Object                 |
|          分配 Buffer 内存                |
|          创建 GEM Handle                 |
+--------------------+---------------------+
                     |
                     v
+------------------------------------------+
|            GEM Memory Management         |
|                                          |
|          GEM Object                     |
|          DMA Address                     |
|          Kernel Virtual Address          |
|          mmap Offset                     |
+--------------------+---------------------+
                     |
                     v
+------------------------------------------+
|                Userspace                 |
|                                          |
|          DRM_IOCTL_MODE_MAP_DUMB         |
|          mmap()                          |
|                                          |
|          CPU 访问 Buffer                 |
+--------------------+---------------------+
                     |
                     v
+------------------------------------------+
|                 DRM Core                 |
|                                          |
|          DRM_IOCTL_MODE_ADDFB2           |
|          Driver fb_create()              |
|                                          |
|          创建 drm_framebuffer            |
+--------------------+---------------------+
                     |
                     v
+------------------------------------------+
|               DRM KMS                    |
|                                          |
|          Plane / CRTC / Connector       |
|          Atomic Check                    |
|          Atomic Commit                   |
+--------------------+---------------------+
                     |
                     v
+------------------------------------------+
|              Display Driver              |
|                                          |
|          获取 DMA 地址                   |
|          配置 Layer                      |
|          更新显示硬件                    |
+--------------------+---------------------+
                     |
                     v
+------------------------------------------+
|              Display HW                  |
|                                          |
|          LPU -> CU -> DOU                |
|          Display Output                  |
+------------------------------------------+
```

从整体流程可以看出，Dumb Buffer 创建只是显示流程的第一步，真正提交到显示硬件还需要经过 GEM、Framebuffer 和 KMS 等多个层次。

## 3. CREATE_DUMB：DRM Core 如何处理 Buffer 创建请求

### 3.1 用户态接口

用户态通常通过 libdrm 提供的接口创建 Dumb Buffer：

```c
struct drm_mode_create_dumb create = {
    .width = 1920,
    .height = 1080,
    .bpp = 32,
};

ret = drmModeCreateDumb(fd,
                        create.width,
                        create.height,
                        create.bpp,
                        &create.handle,
                        &create.pitch,
                        &create.size);
```

该接口最终通过 `DRM_IOCTL_MODE_CREATE_DUMB` 进入内核。

对应的 UAPI 数据结构：

```c
struct drm_mode_create_dumb {
    uint32_t height;
    uint32_t width;
    uint32_t bpp;

    uint32_t flags;

    uint32_t handle;
    uint32_t pitch;
    uint64_t size;
};
```

输入参数主要包括：

| 参数 | 说明 |
|---|---|
| width | Buffer 宽度 |
| height | Buffer 高度 |
| bpp | 每像素位数 |
| flags | 分配标志 |

输出参数包括：

| 参数 | 说明 |
|---|---|
| handle | GEM Handle |
| pitch | 每行字节数，即 Stride |
| size | Buffer 总大小 |

### 3.2 DRM Core 入口

相关源码：

```text
drivers/gpu/drm/drm_dumb_buffers.c
```

入口函数：

```c
drm_mode_create_dumb_ioctl()
```

其主要工作是从用户态获取参数，调用内部处理函数：

```c
int drm_mode_create_dumb_ioctl(struct drm_device *dev,
                               void *data,
                               struct drm_file *file_priv)
{
    struct drm_mode_create_dumb *args = data;
    int err;

    err = drm_mode_create_dumb(dev, args, file_priv);

    if (err) {
        args->handle = 0;
        args->pitch = 0;
        args->size = 0;
    }

    return err;
}
```

核心处理函数：

```c
static int drm_mode_create_dumb(struct drm_device *dev,
                                struct drm_mode_create_dumb *args,
                                struct drm_file *file_priv)
{
    if (!dev->driver->dumb_create)
        return -ENOSYS;

    /*
     * 参数检查及基础尺寸计算
     */

    args->handle = 0;
    args->pitch = 0;
    args->size = 0;

    return dev->driver->dumb_create(file_priv, dev, args);
}
```

可以看出，DRM Core 并不直接分配 Buffer，而是通过：

```c
dev->driver->dumb_create
```

将具体的内存分配工作交给 Driver。

DRM Core 主要负责：

1. 检查输入参数。
2. 进行尺寸计算和溢出检查。
3. 检查 Driver 是否实现 `dumb_create`。
4. 调用 Driver 的 `dumb_create`。
5. 将最终的 Handle、Pitch 和 Size 返回用户态。

这体现了 DRM 框架的一个重要设计思想：

**DRM Core 负责通用流程，Driver 负责硬件相关实现。**

## 4. Driver 如何分配 Dumb Buffer

### 4.1 Driver 的 dumb_create 回调

DRM Driver 通过 `struct drm_driver` 中的 `dumb_create` 回调实现具体的 Buffer 分配：

```c
struct drm_driver {
    ...

    int (*dumb_create)(struct drm_file *file,
                       struct drm_device *dev,
                       struct drm_mode_create_dumb *args);

    ...
};
```

不同 Driver 可以根据自身硬件架构选择不同的内存管理方案，例如：

- CMA：分配连续物理内存。
- DMA GEM：使用 DMA API 管理 Buffer。
- Shmem GEM：使用普通页作为 Buffer 的 backing storage。
- TTM：管理显存资源。

如果 Driver 使用 DRM DMA GEM Helper，可以直接使用：

```c
drm_gem_dma_dumb_create()
```

### 4.2 DMA GEM Dumb Buffer 创建

以 `drm_gem_dma_dumb_create()` 为例，其核心工作包括：

1. 根据 bpp 计算每像素字节数。
2. 计算 Buffer Pitch。
3. 根据硬件要求进行 Pitch 对齐。
4. 计算 Buffer Size。
5. 创建 DMA GEM Object。
6. 创建用户态 GEM Handle。

例如，创建一个 1920×1080 的 XRGB8888 Buffer：

```text
Width  = 1920
Height = 1080
BPP    = 32
CPP    = 4
```

其中：

```text
CPP = BPP / 8

Pitch = Width * CPP
      = 1920 * 4
      = 7680 Bytes

Size  = Pitch * Height
      = 7680 * 1080
      = 8294400 Bytes
```

即 Buffer 大小约为 7.91 MiB。

需要注意，实际 Pitch 可能根据硬件要求进行额外对齐，因此最终值不一定等于 `width * cpp`。

### 4.3 GEM DMA Object

DMA GEM Helper 创建的对象通常使用：

```c
struct drm_gem_dma_object {
    struct drm_gem_object base;

    dma_addr_t dma_addr;
    void *vaddr;
};
```

其中：

| 成员 | 作用 |
|---|---|
| base | DRM GEM 通用对象 |
| dma_addr | 设备访问 Buffer 时使用的 DMA 地址 |
| vaddr | CPU 访问 Buffer 时使用的内核虚拟地址 |

这里有一个重要概念：

`dma_addr` 不一定等于物理地址。

在存在 IOMMU 的系统中，DMA 地址可能是 IOVA。因此，显示硬件应当使用 DMA API 提供的 DMA 地址，而不是直接使用 CPU 物理地址。

## 5. GEM Object 与 GEM Handle

GEM Object 和 GEM Handle 是两个不同的概念。

### 5.1 GEM Object

GEM Object 是内核中的实际 Buffer 管理对象。

它负责管理：

- Buffer 的内存资源。
- Buffer 的大小。
- 内存映射。
- 引用计数。
- DMA 地址等内存相关信息。

### 5.2 GEM Handle

GEM Handle 是用户态用于引用 GEM Object 的整数 ID。

Driver 创建 GEM Object 后，通常调用：

```c
drm_gem_handle_create(file_priv,
                      &obj->base,
                      &handle);
```

建立用户态 Handle 与内核 GEM Object 之间的映射。

关系如下：

```text
             Kernel
       +-------------------+
       |    GEM Object     |
       |                   |
       |  Buffer Memory    |
       |  DMA Address      |
       |  Size              |
       +---------+---------+
                 |
                 |
          GEM Handle Table
                 |
          +------+------+
          |             |
          v             v
      Process A     Process B
      Handle = 5    Handle = 12
```

同一个 GEM Object 可以被不同 DRM File 持有不同的 Handle。

因此：

- GEM Object 是内核中的实际对象。
- Handle 是用户态访问对象的标识。
- Handle 不是物理地址，也不是 DMA 地址。

## 6. MAP_DUMB：如何将 Buffer 映射到用户空间

CREATE_DUMB 成功后，用户态获得：

```c
handle
pitch
size
```

但此时还不能直接通过 Handle 访问 Buffer 内容。

需要继续调用：

```c
DRM_IOCTL_MODE_MAP_DUMB
```

对应 DRM Core 中的：

```c
drm_mode_mmap_dumb_ioctl()
```

它根据 GEM Handle 找到 GEM Object，并获取一个用于 mmap 的 offset。

用户态随后调用：

```c
void *map = mmap(NULL,
                 create.size,
                 PROT_READ | PROT_WRITE,
                 MAP_SHARED,
                 fd,
                 map_dumb.offset);
```

这样就可以通过 `map` 指针访问 Buffer。

整体关系：

```text
             Userspace
       +-------------------+
       |   mmap address    |
       |   0x7fxxxxxx      |
       +---------+---------+
                 |
                 | Page Mapping
                 |
       +---------v---------+
       |    GEM Object     |
       |                   |
       |  Backing Memory   |
       +---------+---------+
                 |
                 |
       +---------v---------+
       |   DMA Address     |
       |   Display HW      |
       +-------------------+
```

这里需要注意：

`mmap()` 并不是重新分配一块 Buffer，而是建立用户虚拟地址空间到已有 GEM Buffer 内存的映射。

不同 GEM 实现的 mmap 具体方式可能不同，例如 DMA GEM、CMA GEM 和 Shmem GEM 使用的辅助函数就有所区别。

## 7. ADDFB2：从 GEM Object 创建 Framebuffer

Buffer 创建并完成 mmap 后，还不能直接交给 KMS Plane 显示。

还需要调用：

```c
drmModeAddFB2()
```

对应：

```c
DRM_IOCTL_MODE_ADDFB2
```

这一步的作用是将 GEM Object 封装为 KMS 使用的 Framebuffer。

### 7.1 Driver 的 fb_create 回调

DRM Driver 通过 `drm_mode_config_funcs` 中的 `fb_create` 回调实现 Framebuffer 创建：

```c
struct drm_mode_config_funcs {
    ...

    struct drm_framebuffer *
    (*fb_create)(struct drm_device *dev,
                 struct drm_file *file_priv,
                 const struct drm_mode_fb_cmd2 *mode_cmd);

    ...
};
```

Driver 的 `fb_create()` 主要负责：

1. 根据用户态传入的 GEM Handle 查找 GEM Object。
2. 检查 Pixel Format。
3. 检查 Pitch、Offset 和 Buffer 大小。
4. 创建 `drm_framebuffer`。
5. 将 GEM Object 与 Framebuffer 关联。

简化示意：

```c
obj = drm_gem_object_lookup(file_priv,
                            mode_cmd->handles[0]);

fb = kzalloc(sizeof(*fb), GFP_KERNEL);

drm_helper_mode_fill_fb_struct(dev,
                               &fb->base,
                               mode_cmd);

fb->obj[0] = obj;

drm_framebuffer_init(dev,
                     &fb->base,
                     &funcs);
```

以上为示意代码，实际实现还需要处理 GEM Object 引用计数、错误回滚和多平面格式。

### 7.2 为什么需要 GEM Object 和 Framebuffer 两层抽象？

这是理解 DRM 内存管理与 KMS 显示架构的关键。

两者的职责不同：

| 对象 | 职责 |
|---|---|
| GEM Object | 管理实际 Buffer 内存 |
| Framebuffer | 描述 Buffer 中图像的格式、尺寸、Stride 和 Offset |

例如，同一个 GEM Object 可以被不同的 Framebuffer 描述，也可以被多个 Plane 引用。

这种设计使得内存管理与显示语义相互解耦。

## 8. Atomic Commit：Framebuffer 如何到达显示硬件

Framebuffer 创建完成后，还需要通过 KMS 的 Plane、CRTC 等对象完成显示配置。

用户态通常通过 Atomic API 提交：

```c
drmModeAtomicAddProperty(...);

drmModeAtomicCommit(fd,
                    req,
                    DRM_MODE_ATOMIC_ALLOW_MODESET,
                    NULL);
```

整个过程可以分成两个阶段。

### 8.1 Atomic Check

DRM Driver 在 Atomic Check 阶段检查新的显示状态是否合法。

例如：

- Plane 是否支持当前 Pixel Format。
- Buffer 尺寸是否满足硬件限制。
- Source / Destination Rectangle 是否合法。
- 缩放比例是否满足限制。
- Layer 资源是否足够。
- 多个 Plane 是否存在硬件资源冲突。

### 8.2 Atomic Commit

Atomic Commit 阶段将检查通过的状态提交给硬件。

以 LinlonDP 这类 DPU Driver 为例，核心工作通常包括：

1. 获取 Plane 对应的 Framebuffer。
2. 找到 Framebuffer 关联的 GEM Object。
3. 获取 Buffer 对应的 DMA 地址。
4. 配置 Layer Buffer Address。
5. 配置 Pixel Format。
6. 配置 Pitch / Stride。
7. 配置 Source / Destination。
8. 更新显示 Pipeline。

最终数据流可以理解为：

```text
GEM Object
    |
    v
Framebuffer
    |
    v
Plane State
    |
    v
Atomic Check
    |
    v
Atomic Commit
    |
    v
LinlonDP Layer Configuration
    |
    v
LPU -> CU -> DOU
    |
    v
Display Output
```

需要注意，DMA 地址的获取、pin/unpin、Buffer 生命周期管理和寄存器更新时机取决于具体 Driver 的实现，并不是 DRM Core 直接把 Buffer 地址写入显示硬件。

## 9. DRM Core 与 Driver 的职责划分

通过上面的流程，可以总结出 DRM Core 和 DRM Driver 的分工：

| 操作 | DRM Core | DRM Driver |
|---|---|---|
| CREATE_DUMB | UAPI 参数检查、流程分发 | `dumb_create` 分配 Buffer |
| GEM Object | 通用对象管理 | 提供具体内存实现 |
| GEM Handle | 管理 Handle 映射 | 创建时调用 GEM Helper |
| MAP_DUMB | 通用 mmap offset 处理 | 提供具体映射实现 |
| mmap | 通用映射入口 | 具体内存映射实现 |
| ADDFB2 | Framebuffer UAPI 处理 | `fb_create` |
| Atomic Check | 通用状态管理 | 检查硬件限制 |
| Atomic Commit | 管理提交流程 | 更新显示硬件 |
| DESTROY_DUMB | 删除 Handle、管理引用 | GEM 对象释放和内存回收 |

## 10. 结合 LinlonDP Driver 的源码阅读建议

对于基于 Linux DRM 框架开发的 DPU Driver，建议按照以下顺序阅读源码：

| 顺序 | 源码位置 / 函数 | 关注内容 |
|---|---|---|
| 1 | `drivers/gpu/drm/drm_dumb_buffers.c` | `drm_mode_create_dumb_ioctl()` |
| 2 | `drivers/gpu/drm/drm_gem_dma_helper.c` | `drm_gem_dma_dumb_create()` |
| 3 | LinlonDP Driver | `dumb_create` 实现或 GEM DMA Helper 配置 |
| 4 | LinlonDP Driver | `fb_create()` |
| 5 | `drivers/gpu/drm/drm_atomic.c` | Atomic State 管理 |
| 6 | LinlonDP Driver | Atomic Check |
| 7 | LinlonDP Driver | Atomic Update / Layer 配置 |

其中，最值得重点跟踪的是：

- Dumb Buffer 创建时，Driver 如何创建 GEM Object。
- GEM Object 如何通过 Handle 暴露给用户态。
- `MAP_DUMB` 如何建立用户空间映射。
- `fb_create()` 如何将 GEM Object 转换为 Framebuffer。
- Atomic Commit 如何从 Framebuffer 获取 DMA 地址并配置 DPU Layer。

## 11. 总结

Linux DRM 的 Dumb Buffer 处理流程体现了 DRM 框架的分层设计：

1. **Dumb Buffer** 提供用户态简单的 Buffer 创建接口。
2. **DRM Core** 负责通用 UAPI、参数检查和对象管理。
3. **GEM** 负责 Buffer 内存资源管理。
4. **Framebuffer** 负责描述图像数据。
5. **KMS** 负责显示状态管理和提交。
6. **DRM Driver** 负责将抽象的显示状态转换成具体硬件配置。

其中最重要的两个关系是：

**GEM Object 与 Framebuffer：**

GEM Object 管理内存，Framebuffer 描述图像。

**DRM Core 与 DRM Driver：**

DRM Core 提供通用框架，Driver 实现具体硬件相关操作。

正是通过这种分层设计，Linux DRM 才能够将 Buffer 内存管理与具体显示硬件解耦，使不同厂商的 DPU、GPU 和显示控制器能够共享统一的 DRM/KMS 接口。

---

### 参考资料

- Linux Kernel Documentation：DRM Internals
- Linux Kernel Documentation：DRM Memory Management
- Linux Kernel Source：`drivers/gpu/drm/drm_dumb_buffers.c`
- Linux Kernel Source：`drivers/gpu/drm/drm_gem_dma_helper.c`
- Linux Kernel Source：`include/drm/drm_gem_dma_helper.h`
- Linux Kernel Source：`include/drm/drm_framebuffer.h`

