+++
date = '2026-06-26T17:35:25+08:00'
draft = false
title = 'Drm_atomic_state如何构造'
tags = ["Linux", "DRM"]
+++

# 1. 用户态数据构造
libdrm提供了如下的编程范式：
```c
drmModeAtomicReq *req = drmModeAtomicAlloc();

drmModeAtomicAddProperty(req, conn_id, prop_conn_crtc, crtc_id);


/* crtc enable + mode */
drmModeAtomicAddProperty(req, crtc_id, prop_crtc_active, 1);
drmModeAtomicAddProperty(req, crtc_id, prop_crtc_mode, mode_blob);

/* plane setup */
drmModeAtomicAddProperty(req, plane_id, prop_plane_crtc, crtc_id);
drmModeAtomicAddProperty(req, plane_id, prop_plane_fb, buf->fb_id);

drmModeAtomicAddProperty(req, plane_id, prop_crtc_x, 0);
drmModeAtomicAddProperty(req, plane_id, prop_crtc_y, 0);
drmModeAtomicAddProperty(req, plane_id, prop_crtc_w, buf->width);
drmModeAtomicAddProperty(req, plane_id, prop_crtc_h, buf->height);

drmModeAtomicAddProperty(req, plane_id, prop_src_x, 0);
drmModeAtomicAddProperty(req, plane_id, prop_src_y, 0);
drmModeAtomicAddProperty(req, plane_id, prop_src_w, buf->width << 16);
drmModeAtomicAddProperty(req, plane_id, prop_src_h, buf->height << 16);

```

最终得到的结果是：
```c
struct _drmModeAtomicReqItem {
	uint32_t object_id;
	uint32_t property_id;
	uint64_t value;
	uint32_t cursor;
};
struct _drmModeAtomicReq {
	uint32_t cursor;
	uint32_t size_items;
	drmModeAtomicReqItemPtr items;
};
typedef struct _drmModeAtomicReq drmModeAtomicReq, *drmModeAtomicReqPtr;

Entry0:
    object = Connector0
    property = CRTC_ID
    value = CRTC0

Entry1:
    object = CRTC0
    property = ACTIVE
    value = 1

Entry2:
    object = CRTC0
    property = MODE_ID
    value = blob

Entry3:
    object = Plane0
    property = CRTC_ID
    value = CRTC0

Entry4:
    object = Plane0
    property = FB_ID
    value = fb0

...
```
到这里只有Property, 没有`drm_atomic_state` .

# 2. ioctl进入内核
```c
	int ret = drmModeAtomicCommit(fd, req, DRM_MODE_ATOMIC_ALLOW_MODESET | DRM_MODE_PAGE_FLIP_EVENT, &(vo->drm_event));
        -> ret = DRM_IOCTL(fd, DRM_IOCTL_MODE_ATOMIC, &atomic); // struct drm_mode_atomic atomic
    ---------------------------------------------------------------------

    drm_ioctl
        -> drm_mode_atomic_ioctl
```
用户层通过libdrm的接口`drmModeAtomicCommit` 进入到内核`drm_mode_atomic_ioctl`.

# 3. 内核创建`drm_atomic_state`

```c
struct drm_atomic_state {
	struct kref ref;

	struct drm_device *dev;

	/**
	 * @allow_modeset:
	 *
	 * Allow full modeset. This is used by the ATOMIC IOCTL handler to
	 * implement the DRM_MODE_ATOMIC_ALLOW_MODESET flag. Drivers should
	 * never consult this flag, instead looking at the output of
	 * drm_atomic_crtc_needs_modeset().
	 */
	bool allow_modeset : 1;
	/**
	 * @legacy_cursor_update:
	 *
	 * Hint to enforce legacy cursor IOCTL semantics.
	 *
	 * WARNING: This is thoroughly broken and pretty much impossible to
	 * implement correctly. Drivers must ignore this and should instead
	 * implement &drm_plane_helper_funcs.atomic_async_check and
	 * &drm_plane_helper_funcs.atomic_async_commit hooks. New users of this
	 * flag are not allowed.
	 */
	bool legacy_cursor_update : 1;
	bool async_update : 1;
	/**
	 * @duplicated:
	 *
	 * Indicates whether or not this atomic state was duplicated using
	 * drm_atomic_helper_duplicate_state(). Drivers and atomic helpers
	 * should use this to fixup normal  inconsistencies in duplicated
	 * states.
	 */
	bool duplicated : 1;
	struct __drm_planes_state *planes;
	struct __drm_crtcs_state *crtcs;
	int num_connector;
	struct __drm_connnectors_state *connectors;
	int num_private_objs;
	struct __drm_private_objs_state *private_objs;

	struct drm_modeset_acquire_ctx *acquire_ctx;

	/**
	 * @fake_commit:
	 *
	 * Used for signaling unbound planes/connectors.
	 * When a connector or plane is not bound to any CRTC, it's still important
	 * to preserve linearity to prevent the atomic states from being freed to early.
	 *
	 * This commit (if set) is not bound to any CRTC, but will be completed when
	 * drm_atomic_helper_commit_hw_done() is called.
	 */
	struct drm_crtc_commit *fake_commit;

	/**
	 * @commit_work:
	 *
	 * Work item which can be used by the driver or helpers to execute the
	 * commit without blocking.
	 */
	struct work_struct commit_work;
};
```

```c
drm_mode_atomic_ioctl
    -> drm_atomic_state_alloc(dev)
        -> 创建 drm_atomic_state
```
也就是说：
```c
User
--------------------------
drmModeAtomicReq

变成

Kernel
--------------------------
drm_atomic_state
```

# 4. 用户态的Property如何转换成内核态的Atomic State?
这是整个Atomic Framework最核心的一步。内核会遍历用户提交的所有property，
```c
CRTC0:
    MODE_ID
    ACTIVE

Connector0:
    CRTC_ID

Plane0:
    FB_ID
    SRC_X
    SRC_Y
...
```

然后逐个调用`drm_atomic_set_property()`或者更具体一点，不同object会进入不同helper,
例如`drm_atomic_crtc_set_property`, `drm_atomic_plane_set_property`, `drm_atomic_connector_set_property`。

通过上面这些helper，用户态的Property慢慢会变成：
```c
drm_atomic_state

    crtc_state
        active
        mode

    plane_state
        fb
        src_x
        ...

    connector_state
        crtc

...

```

# 5. 为什么这样设计
因为用户态不知道内核态数据结构，例如kernel下的`drm_crtc_state`, 用户根本无法构造。
另一个方面，**Property**(ACTIVE, MODE_ID, FB_ID, CRTC_ID)是ABI, 可以一直保持兼容;
而`drm_atomic_state`,`drm_crtc_state`, `drm_plane_state`完全可以随着 kernel 演进不断增加字段。
所以：
>Property 是用户态与内核之间稳定的 ABI，而 drm_atomic_state 是内核内部对这些 Property 的组织和表达。
```c
struct drm_crtc_state {
	/** @crtc: backpointer to the CRTC */
	struct drm_crtc *crtc;

	/**
	 * @enable: Whether the CRTC should be enabled, gates all other state.
	 * This controls reservations of shared resources. Actual hardware state
	 * is controlled by @active.
	 */
	bool enable;

	/**
	 * @active: Whether the CRTC is actively displaying (used for DPMS).
	 * Implies that @enable is set. The driver must not release any shared
	 * resources if @active is set to false but @enable still true, because
	 * userspace expects that a DPMS ON always succeeds.
	 *
	 * Hence drivers must not consult @active in their various
	 * &drm_mode_config_funcs.atomic_check callback to reject an atomic
	 * commit. They can consult it to aid in the computation of derived
	 * hardware state, since even in the DPMS OFF state the display hardware
	 * should be as much powered down as when the CRTC is completely
	 * disabled through setting @enable to false.
	 */
	bool active;

	/**
	 * @planes_changed: Planes on this crtc are updated. Used by the atomic
	 * helpers and drivers to steer the atomic commit control flow.
	 */
	bool planes_changed : 1;

	/**
	 * @mode_changed: @mode or @enable has been changed. Used by the atomic
	 * helpers and drivers to steer the atomic commit control flow. See also
	 * drm_atomic_crtc_needs_modeset().
	 *
	 * Drivers are supposed to set this for any CRTC state changes that
	 * require a full modeset. They can also reset it to false if e.g. a
	 * @mode change can be done without a full modeset by only changing
	 * scaler settings.
	 */
	bool mode_changed : 1;

	/**
	 * @active_changed: @active has been toggled. Used by the atomic
	 * helpers and drivers to steer the atomic commit control flow. See also
	 * drm_atomic_crtc_needs_modeset().
	 */
	bool active_changed : 1;

	/**
	 * @connectors_changed: Connectors to this crtc have been updated,
	 * either in their state or routing. Used by the atomic
	 * helpers and drivers to steer the atomic commit control flow. See also
	 * drm_atomic_crtc_needs_modeset().
	 *
	 * Drivers are supposed to set this as-needed from their own atomic
	 * check code, e.g. from &drm_encoder_helper_funcs.atomic_check
	 */
	bool connectors_changed : 1;
	/**
	 * @zpos_changed: zpos values of planes on this crtc have been updated.
	 * Used by the atomic helpers and drivers to steer the atomic commit
	 * control flow.
	 */
	bool zpos_changed : 1;
	/**
	 * @color_mgmt_changed: Color management properties have changed
	 * (@gamma_lut, @degamma_lut or @ctm). Used by the atomic helpers and
	 * drivers to steer the atomic commit control flow.
	 */
	bool color_mgmt_changed : 1;

	/**
	 * @no_vblank:
	 *
	 * Reflects the ability of a CRTC to send VBLANK events. This state
	 * usually depends on the pipeline configuration. If set to true, DRM
	 * atomic helpers will send out a fake VBLANK event during display
	 * updates after all hardware changes have been committed. This is
	 * implemented in drm_atomic_helper_fake_vblank().
	 *
	 * One usage is for drivers and/or hardware without support for VBLANK
	 * interrupts. Such drivers typically do not initialize vblanking
	 * (i.e., call drm_vblank_init() with the number of CRTCs). For CRTCs
	 * without initialized vblanking, this field is set to true in
	 * drm_atomic_helper_check_modeset(), and a fake VBLANK event will be
	 * send out on each update of the display pipeline by
	 * drm_atomic_helper_fake_vblank().
	 *
	 * Another usage is CRTCs feeding a writeback connector operating in
	 * oneshot mode. In this case the fake VBLANK event is only generated
	 * when a job is queued to the writeback connector, and we want the
	 * core to fake VBLANK events when this part of the pipeline hasn't
	 * changed but others had or when the CRTC and connectors are being
	 * disabled.
	 *
	 * __drm_atomic_helper_crtc_duplicate_state() will not reset the value
	 * from the current state, the CRTC driver is then responsible for
	 * updating this field when needed.
	 *
	 * Note that the combination of &drm_crtc_state.event == NULL and
	 * &drm_crtc_state.no_blank == true is valid and usually used when the
	 * writeback connector attached to the CRTC has a new job queued. In
	 * this case the driver will send the VBLANK event on its own when the
	 * writeback job is complete.
	 */
	bool no_vblank : 1;

	/**
	 * @plane_mask: Bitmask of drm_plane_mask(plane) of planes attached to
	 * this CRTC.
	 */
	u32 plane_mask;

	/**
	 * @connector_mask: Bitmask of drm_connector_mask(connector) of
	 * connectors attached to this CRTC.
	 */
	u32 connector_mask;

	/**
	 * @encoder_mask: Bitmask of drm_encoder_mask(encoder) of encoders
	 * attached to this CRTC.
	 */
	u32 encoder_mask;

	/**
	 * @adjusted_mode:
	 *
	 * Internal display timings which can be used by the driver to handle
	 * differences between the mode requested by userspace in @mode and what
	 * is actually programmed into the hardware.
	 *
	 * For drivers using &drm_bridge, this stores hardware display timings
	 * used between the CRTC and the first bridge. For other drivers, the
	 * meaning of the adjusted_mode field is purely driver implementation
	 * defined information, and will usually be used to store the hardware
	 * display timings used between the CRTC and encoder blocks.
	 */
	struct drm_display_mode adjusted_mode;

	/**
	 * @mode:
	 *
	 * Display timings requested by userspace. The driver should try to
	 * match the refresh rate as close as possible (but note that it's
	 * undefined what exactly is close enough, e.g. some of the HDMI modes
	 * only differ in less than 1% of the refresh rate). The active width
	 * and height as observed by userspace for positioning planes must match
	 * exactly.
	 *
	 * For external connectors where the sink isn't fixed (like with a
	 * built-in panel), this mode here should match the physical mode on the
	 * wire to the last details (i.e. including sync polarities and
	 * everything).
	 */
	struct drm_display_mode mode;

	/**
	 * @mode_blob: &drm_property_blob for @mode, for exposing the mode to
	 * atomic userspace.
	 */
	struct drm_property_blob *mode_blob;

	/**
	 * @degamma_lut:
	 *
	 * Lookup table for converting framebuffer pixel data before apply the
	 * color conversion matrix @ctm. See drm_crtc_enable_color_mgmt(). The
	 * blob (if not NULL) is an array of &struct drm_color_lut.
	 */
	struct drm_property_blob *degamma_lut;

	/**
	 * @ctm:
	 *
	 * Color transformation matrix. See drm_crtc_enable_color_mgmt(). The
	 * blob (if not NULL) is a &struct drm_color_ctm.
	 */
	struct drm_property_blob *ctm;

	/**
	 * @gamma_lut:
	 *
	 * Lookup table for converting pixel data after the color conversion
	 * matrix @ctm.  See drm_crtc_enable_color_mgmt(). The blob (if not
	 * NULL) is an array of &struct drm_color_lut.
	 *
	 * Note that for mostly historical reasons stemming from Xorg heritage,
	 * this is also used to store the color map (also sometimes color lut,
	 * CLUT or color palette) for indexed formats like DRM_FORMAT_C8.
	 */
	struct drm_property_blob *gamma_lut;

	/**
	 * @target_vblank:
	 *
	 * Target vertical blank period when a page flip
	 * should take effect.
	 */
	u32 target_vblank;

	/**
	 * @async_flip:
	 *
	 * This is set when DRM_MODE_PAGE_FLIP_ASYNC is set in the legacy
	 * PAGE_FLIP IOCTL. It's not wired up for the atomic IOCTL itself yet.
	 */
	bool async_flip;

	/**
	 * @vrr_enabled:
	 *
	 * Indicates if variable refresh rate should be enabled for the CRTC.
	 * Support for the requested vrr state will depend on driver and
	 * hardware capabiltiy - lacking support is not treated as failure.
	 */
	bool vrr_enabled;

	/**
	 * @self_refresh_active:
	 *
	 * Used by the self refresh helpers to denote when a self refresh
	 * transition is occurring. This will be set on enable/disable callbacks
	 * when self refresh is being enabled or disabled. In some cases, it may
	 * not be desirable to fully shut off the crtc during self refresh.
	 * CRTC's can inspect this flag and determine the best course of action.
	 */
	bool self_refresh_active;

	/**
	 * @scaling_filter:
	 *
	 * Scaling filter to be applied
	 */
	enum drm_scaling_filter scaling_filter;

	/**
	 * @event:
	 *
	 * Optional pointer to a DRM event to signal upon completion of the
	 * state update. The driver must send out the event when the atomic
	 * commit operation completes. There are two cases:
	 *
	 *  - The event is for a CRTC which is being disabled through this
	 *    atomic commit. In that case the event can be send out any time
	 *    after the hardware has stopped scanning out the current
	 *    framebuffers. It should contain the timestamp and counter for the
	 *    last vblank before the display pipeline was shut off. The simplest
	 *    way to achieve that is calling drm_crtc_send_vblank_event()
	 *    somewhen after drm_crtc_vblank_off() has been called.
	 *
	 *  - For a CRTC which is enabled at the end of the commit (even when it
	 *    undergoes an full modeset) the vblank timestamp and counter must
	 *    be for the vblank right before the first frame that scans out the
	 *    new set of buffers. Again the event can only be sent out after the
	 *    hardware has stopped scanning out the old buffers.
	 *
	 *  - Events for disabled CRTCs are not allowed, and drivers can ignore
	 *    that case.
	 *
	 * For very simple hardware without VBLANK interrupt, enabling
	 * &struct drm_crtc_state.no_vblank makes DRM's atomic commit helpers
	 * send a fake VBLANK event at the end of the display update after all
	 * hardware changes have been applied. See
	 * drm_atomic_helper_fake_vblank().
	 *
	 * For more complex hardware this
	 * can be handled by the drm_crtc_send_vblank_event() function,
	 * which the driver should call on the provided event upon completion of
	 * the atomic commit. Note that if the driver supports vblank signalling
	 * and timestamping the vblank counters and timestamps must agree with
	 * the ones returned from page flip events. With the current vblank
	 * helper infrastructure this can be achieved by holding a vblank
	 * reference while the page flip is pending, acquired through
	 * drm_crtc_vblank_get() and released with drm_crtc_vblank_put().
	 * Drivers are free to implement their own vblank counter and timestamp
	 * tracking though, e.g. if they have accurate timestamp registers in
	 * hardware.
	 *
	 * For hardware which supports some means to synchronize vblank
	 * interrupt delivery with committing display state there's also
	 * drm_crtc_arm_vblank_event(). See the documentation of that function
	 * for a detailed discussion of the constraints it needs to be used
	 * safely.
	 *
	 * If the device can't notify of flip completion in a race-free way
	 * at all, then the event should be armed just after the page flip is
	 * committed. In the worst case the driver will send the event to
	 * userspace one frame too late. This doesn't allow for a real atomic
	 * update, but it should avoid tearing.
	 */
	struct drm_pending_vblank_event *event;

	/**
	 * @commit:
	 *
	 * This tracks how the commit for this update proceeds through the
	 * various phases. This is never cleared, except when we destroy the
	 * state, so that subsequent commits can synchronize with previous ones.
	 */
	struct drm_crtc_commit *commit;

	/** @state: backpointer to global drm_atomic_state */
	struct drm_atomic_state *state;
};
```

# 7. 总结
```c
User Space
────────────────────────────────────────────

drmModeAtomicAlloc()

        │
        ▼

drmModeAtomicReq
    │
    ├── (CRTC, ACTIVE, 1)
    ├── (CRTC, MODE_ID, blob)
    ├── (Connector, CRTC_ID, crtc0)
    ├── (Plane, FB_ID, fb0)
    └── ...

        │
        │ drmModeAtomicCommit()
        ▼

DRM_IOCTL_MODE_ATOMIC

────────────────────────────────────────────
Kernel Space

drm_mode_atomic_ioctl()

        │
        ▼

drm_atomic_state_alloc()

        │
        ▼

遍历所有 Property

        │
        ▼

drm_atomic_set_property()

        │
        ▼

drm_atomic_state

    ├── drm_crtc_state
    ├── drm_plane_state
    ├── drm_connector_state
    └── ...

        │
        ▼

drm_atomic_check()

        │
        ▼

drm_atomic_commit()

        │
        ▼

mode_set / enable / flush
```
