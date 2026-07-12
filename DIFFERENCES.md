# Xe native context vs i915 native context: differences and rationale

A technical reference for the Xe `drm_native_context` patches: what each change
does, and why it differs from the existing i915 native-context implementation.
It covers the three patches in this repository: the virglrenderer host backend,
the Mesa guest layer, and the intel-media-driver guest shim.

## Background

DRM native context lets a guest issue the real kernel-driver ioctls of the host
GPU. The guest virtio-gpu driver carries those ioctls to the host, where
virglrenderer replays them against the real kernel driver. An i915 backend for
this already exists (Mesa `virtio-intel`, virglrenderer i915 renderer). These
patches add an equivalent backend for the **Xe** kernel driver, which is the
driver modern Intel GPUs use instead of i915.

The Xe files were structured after the existing i915 files and changed only where
the Xe kernel UAPI differs from the i915 UAPI. Almost every difference in this
document traces back to that UAPI difference, so it is described first.

## 1. Xe kernel UAPI vs i915 kernel UAPI

i915 and Xe drive the same class of hardware through very different interfaces:

| Concept | i915 UAPI | Xe UAPI |
|---|---|---|
| Address space | Implicit per-context GTT; softpin addresses passed at execbuf | Explicit **VM object** (`DRM_IOCTL_XE_VM_CREATE`) |
| Mapping BOs into GPU VA | Implicit in `execbuffer2` (softpin) | Explicit **`DRM_IOCTL_XE_VM_BIND`** ops (MAP/UNMAP) |
| Submission context | GEM **context** (`GEM_CONTEXT_CREATE`) | **exec_queue** (`EXEC_QUEUE_CREATE`), bound to a VM plus engine instances |
| Submit | `GEM_EXECBUFFER2` (buffer list + relocations + submit in one ioctl) | `DRM_IOCTL_XE_EXEC` (a batch address plus a sync array; BOs already VM-bound) |
| Fencing | inline fences in `execbuffer2` (`rsvd2`, sync_file fds) | explicit **`drm_xe_sync`** array (DRM syncobj / timeline syncobj / user fence) |
| BO CPU caching | selected indirectly (mmap mode, PAT) | explicit **`cpu_caching`** (WB/WC) at `GEM_CREATE`; PAT index at VM_BIND |
| Hang detection | `GET_RESET_STATS` on the context | `EXEC_QUEUE_GET_PROPERTY` with the BAN property |

Because the guest issues real Xe ioctls, the native-context layers relay the Xe
submission model (VM, VM_BIND, exec_queue, sync arrays), not the i915 one.

## 2. virglrenderer host backend (`src/drm/xe/`)

A new backend registered in `drm_renderer.c` under a new context type
`VIRTGPU_DRM_CONTEXT_XE = 6`, built with `-Ddrm-renderers=xe-experimental`.

### 2.1 Command set

The Xe backend has 8 ccmds:
`DEVICE_QUERY, GEM_CREATE, VM_CREATE, VM_DESTROY, VM_BIND,
EXEC_QUEUE_CREATE, EXEC_QUEUE_DESTROY, EXEC`. The i915 backend has 9. The mapping:

| i915 ccmd | Xe ccmd | Difference |
|---|---|---|
| `IOCTL_SIMPLE` (generic passthrough for reset_stats, context/vm create+destroy, get/setparam, tiling, set_domain) | none | Xe has no generic passthrough ccmd; every operation is a typed ccmd. |
| `GETPARAM`, `QUERYPARAM` | `DEVICE_QUERY` | The Xe UAPI exposes all device introspection through one ioctl, `DRM_IOCTL_XE_DEVICE_QUERY`. |
| `GEM_CREATE`, `GEM_CREATE_EXT` | `GEM_CREATE` | Xe takes caching and placement directly on create, so there is no `_EXT` variant or extension parsing. |
| `GEM_CONTEXT_CREATE` | `EXEC_QUEUE_CREATE`, `EXEC_QUEUE_DESTROY` | The Xe submission entity is an exec_queue with its own create and destroy ioctls. |
| (implicit in execbuffer2) | `VM_CREATE`, `VM_DESTROY`, `VM_BIND` | Xe requires an explicit VM and explicit binds; i915 has no VM object and binds implicitly via softpin. |
| `GEM_EXECBUFFER` | `EXEC` | Xe `EXEC` carries only a batch address and a sync array; residency is already established by VM_BIND. |
| `GEM_SET_MMAP_MODE` | none | Caching is fixed at create time (see 2.2). |
| `GEM_BUSY` | none | No busy-query ioctl in the Xe path. |

`ccmd_alignment` is 8 for Xe versus 4 for i915. The Xe request and response
structs contain `uint64_t` fields (VM addresses, sizes), and
`XE_STATIC_ASSERT_SIZE` enforces `sizeof % 8 == 0`, so 8-byte packing keeps those
fields naturally aligned.

### 2.2 GEM create and CPU caching (`xe_object.c`)

`xe_ccmd_gem_create` passes the guest's `cpu_caching` value into the host
`drm_xe_gem_create`. `xe_object_create` records the corresponding `map_info`:
`VIRGL_RENDERER_MAP_CACHE_WC` for `DRM_XE_GEM_CPU_CACHING_WC`, and
`VIRGL_RENDERER_MAP_CACHE_CACHED` for WB. `get_blob` returns that `map_info` so
the guest maps the blob with the matching caching mode.

The i915 UAPI has no `cpu_caching` argument on GEM create; caching is chosen later
through `GEM_CREATE_EXT` or the mmap-mode ioctl. Because Xe fixes caching at
create time, the Xe object needs no deferred mmap-mode state.

### 2.3 Synchronous VM_BIND (`xe_ccmd_vm_bind`)

Each VM_BIND creates a temporary syncobj, submits `DRM_IOCTL_XE_VM_BIND` with that
syncobj as a signal fence, waits for it to signal, then destroys it. The patch
comment states the reason: the bind is performed synchronously on the host so that
mappings are visible before any dependent exec submission. The Xe VM_BIND ioctl is
asynchronous by nature, so the host waits on the bind fence to give the guest a
completed mapping on return.

### 2.4 Exec and fencing (`xe_ccmd_exec`, `xe_renderer.c`)

- Each exec_queue has a `drm_timeline`, created in `EXEC_QUEUE_CREATE` and stored
  in a hash table keyed by `exec_queue_id + 1`.
- `EXEC` creates an out syncobj as a signal fence, optionally imports the guest's
  in-fence fd as an in syncobj, runs `DRM_IOCTL_XE_EXEC`, then exports the out
  syncobj to an fd and hands it to the queue's timeline.
- `submit_fence` maps virtio `ring_idx` to an exec_queue. Per the patch comment,
  `ring_idx 0` is reserved for the host CPU timeline; otherwise
  `ring_idx == exec_queue_id + 1`.

i915 carries in and out fences inline in `execbuffer2` through the `rsvd2` field.
Xe has no inline fence field on `DRM_IOCTL_XE_EXEC`, so fences are DRM syncobjs
(`drm_xe_sync`), which is why the exec path creates, imports, exports, and destroys
syncobjs around each submission. The per-exec_queue timeline lives in a hash table
rather than a fixed array because exec_queue ids are assigned by the kernel and are
not a small dense index.

### 2.5 banned_exec_queue_mask (`xe_proto.h`, `xe_ccmd.c`)

`xe_shmem` adds a `uint64_t banned_exec_queue_mask` after the base `vdrm_shmem`.
When `DRM_IOCTL_XE_EXEC` returns `EIO`, the host sets the bit for that
`exec_queue_id`. The guest reads it back through `EXEC_QUEUE_GET_PROPERTY` with the
BAN property, which the handler answers from the mask instead of forwarding to the
host. Per the patch comment, this lets the guest detect GPU hangs; the mask is a
`uint64_t`, so tracking is limited to the first 64 exec_queue ids.

The i915 backend has the same construct as `banned_ctx_mask`, keyed on the i915
context id.

### 2.6 Capset struct (`drm_hw.h`)

Adds `VIRTGPU_DRM_CONTEXT_XE 6` and a `u.xe` struct carrying the PCI identifiers,
filled in by `xe_renderer_probe`. `wire_format_version` is 1 and the DRM capset
`max_ver` moves from 0 to 1. The guest defines the same `u.xe` struct, so the
layout is a shared contract between guest and host.

## 3. Mesa guest layer (`src/intel/dev/virtio/`)

The shared Intel virtio layer used by both iris (OpenGL) and ANV (Vulkan). New
files `xe_virtio_ccmd.c` and `xe_virtio_proto.h`; the proto structs match
virglrenderer's `xe_proto.h`.

### 3.1 ioctl dispatch (`intel_virtio_ioctl`)

An `if (dev->is_xe)` block routes `DRM_IOCTL_XE_*` ioctls to the `xe_virtio_*`
handlers. The SYNCOBJ and PRIME ioctls fall through to the same shared handlers
the i915 path uses. `DRM_IOCTL_XE_WAIT_USER_FENCE` is a no-op returning success;
the patch comment states the reason: synchronization is handled by the
syncobj/timeline flow through vdrm exec submission, not by local user-fence waits.

The i915 and Xe ioctl-number namespaces are disjoint, and Xe has no
tiling, domain, relocation, or legacy-context ioctls, so a separate dispatch block
is required.

### 3.2 Device detection (`intel_virtio_device.c`, `intel_kmd.c`)

`intel_virtio_init_fd` connects `VIRTGPU_DRM_CONTEXT_XE` first, sets `dev->is_xe`
on success, and falls back to `VIRTGPU_DRM_CONTEXT_I915`. `is_intel_virtio_xe`
exposes the flag, and `intel_kmd.c` resolves the KMD type from it instead of
assuming i915 for every virtio fd. The guest cannot read the host KMD name (the
DRM version name is `virtio_gpu`), so it probes by capset context type.

### 3.3 Capset PCI plumbing (`intel_virtio_device.c`, `iris_drm_winsys.c`)

`intel_virtio_get_pci_device_info` reads `caps.u.xe.*` when `dev->is_xe`, and
`iris_drm_probe_nctx` accepts `VIRTGPU_DRM_CONTEXT_XE` and reads
`caps->u.xe.pci_device_id`. The capset layout and the context-type value are a
memory-layout contract with the host virglrenderer.

### 3.4 Exec path (`xe_virtio_exec`)

Splits the `drm_xe_sync` array into in and out `drm_virtgpu_execbuffer_syncobj`
arrays (signal vs wait, with the timeline point for timeline syncobjs), sets
`ring_idx = exec_queue_id + 1`, and submits through `vdrm_execbuf`. No per-exec
buffer list is sent, because the buffers are already resident from VM_BIND, and
there are no relocations in the Xe UAPI. The i915 path sends the full buffer-handle
list every submit.

### 3.5 VM_BIND (`xe_virtio_vm_bind`)

Marshals the bind ops (translating each object handle to a resource id) and sends
the ccmd synchronously. After it returns, the guest signals any signal syncs
locally, because the host completes the bind before responding (see 2.3). i915 has
no VM_BIND, so this path has no i915 counterpart.

### 3.6 SYSMEM size clamp (`intel/dev/xe/intel_device_info.c`)

`xe_clamp_virtio_sysmem_size` clamps the SYSMEM memory-region size and free values
to the guest's physical RAM when running on a virtio fd. `DRM_XE_DEVICE_QUERY` runs
against the host device and reports host-sized system memory; the clamp keeps the
guest allocator's view bounded by guest RAM. It applies only on virtio, so
bare-metal behavior is unchanged.

### 3.7 iris upload-buffer caching (`iris/xe/iris_kmd_backend.c`)

For upload/staging BOs (`BO_ALLOC_CACHED_COHERENT | BO_ALLOC_SMEM |
BO_ALLOC_NO_SUBALLOC`) on virtio Xe, `cpu_caching` is forced to
`DRM_XE_GEM_CPU_CACHING_WC`. The patch comment states the reason: on virtio Xe,
upload/staging BOs with WB caching cause severe CPU-write stalls, and WC restores
streaming upload performance. It is gated on `is_intel_virtio_xe`, so bare-metal Xe
keeps its normal choice. This is the change named in the patch filename.

### 3.8 ANV (Vulkan)

Intel KMD ioctls funnel through `intel_ioctl` into `intel_virtio_ioctl`, so ANV
uses the same Xe dispatch and handlers as iris through the shared virtio layer. The
two iris-specific changes in this patch are the winsys probe (3.3) and the
upload-buffer caching (3.7); there is no ANV-specific code.

## 4. intel-media-driver guest shim (`mos_xe_virtio.c/.h`)

The media driver does not link against Mesa, so it cannot reuse Mesa's vdrm
client. This shim is a self-contained vdrm client and Xe ccmd translation layer;
the file comment records that it was adapted from Mesa's `src/virtio/vdrm/` and
`src/intel/dev/virtio/xe_virtio_ccmd.c`, and its ccmd structs match
virglrenderer's `xe_proto.h`. There is no i915 counterpart to compare against.

Integration is mechanical: `mos_bufmgr_xe.c` and `mos_synchronization_xe.c` route
their Xe and PRIME ioctls through `mos_xe_ioctl`, which forwards to vdrm when the
virtio proxy is active and calls the real ioctl otherwise. The shim is only needed
for VA-API hardware video.

Points recorded in the file's own comments:
- One global vdrm connection (`g_vdrm`), matching the media driver's
  one-device-per-process model.
- A reopened-fd path when a context already exists on the fd (for example when
  Mesa also uses it): the shim reopens the fd to get its own context for the shmem
  mapping, and in that case waits on an exported fence before signalling the
  caller's syncobjs, because those syncobj handles belong to the caller's fd.
- `GEM_MMAP_OFFSET` cannot use the Xe mmap path on virtio, so `mos_bo_map_xe`
  calls `mos_xe_gem_mmap` (VIRTGPU_MAP) instead.

## 5. Known limitations

- **VM binds are synchronous on the host** (2.3, 3.5). The host waits for each bind
  to complete before returning, which adds a round-trip on the mapping path.
- **`banned_exec_queue_mask` tracks 64 exec_queues** (2.5), one bit per queue in a
  `uint64_t`.
- **The intel-media-driver shim uses a single global vdrm connection** (4),
  matching iHD's one-device-per-process model.

## Related upstream work

- Mesa i915 native context: merge request `!29870`.
- virglrenderer i915 backend: merge request `!1384`.
- QEMU support: the "Support virtio-gpu DRM native context" series.
