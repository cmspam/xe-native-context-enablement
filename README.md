# Xe Native Context Enablement for virtio-gpu

Patches to enable Intel Xe GPU acceleration in QEMU/KVM virtual machines via `drm_native_context`. This allows OpenGL, Vulkan, and VA-API (hardware video decode/encode) to work inside a guest VM without full GPU passthrough.

## What This Is

The Linux `virtio-gpu` driver supports a mechanism called **native context** (`drm_native_context`) that allows a guest VM to submit GPU commands directly to the host kernel driver, with the host-side `virglrenderer` acting as a thin relay rather than doing full GPU command translation. This is already supported for i915 (legacy Intel) and other drivers.

These patches extend native context support to the **Intel Xe kernel driver**, which is the modern replacement for i915 on recent Intel GPUs (Arc, Meteor Lake, Lunar Lake, etc.).

## Status

**Functional and tested.** OpenGL (via iris/Gallium), Vulkan (via ANV), and VA-API (hardware video acceleration) are all working.

- **Tested hardware:** Intel Arc B390 (iGPU on Core X7 358H / Lunar Lake)
- **Testers welcome** for other Xe-based Intel GPUs (Arc A-series, Meteor Lake, Arrow Lake, Battlemage, etc.) -- please open an issue with your results.

Performance varies by workload and rendering path. Some paths work very well; others may show overhead compared to bare metal. This is experimental.

## Patches

| Patch | Apply To | Where |
|-------|----------|-------|
| `virglrenderer-xe-native-context.patch` | [virglrenderer](https://gitlab.freedesktop.org/virgl/virglrenderer) | **Host** |
| `mesa-01-xe-native-context-plus-iris-upload-fix.patch` | [Mesa](https://gitlab.freedesktop.org/mesa/mesa) | **Guest** and **Host** |
| `intel-media-driver-xe-native-context.patch` | [intel-media-driver](https://github.com/intel/media-driver) | **Guest** (only needed for VA-API) |

### virglrenderer (host)

Adds an Xe renderer backend to virglrenderer. This is the host-side component that receives Xe DRM commands from the guest over the virtio-gpu transport and replays them against the real Xe kernel driver. Build with `-Ddrm-renderers=xe-experimental` (or add `xe-experimental` to your existing list).

### Mesa (guest and host)

Adds Xe virtio ccmd support to Mesa's shared Intel device/virtio layer. This plumbing is used by both the iris Gallium driver (OpenGL) and ANV (Vulkan) to issue Xe ioctls over the vdrm protocol instead of talking to a real kernel driver. Also includes a targeted fix for upload buffer caching on virtio Xe (forces WC for staging BOs that exhibit CPU-write stalls under WB caching).

The Mesa patch must be applied on both the **host** (where virglrenderer links against Mesa utilities) and the **guest** (where the actual GPU drivers run).

### intel-media-driver (guest, optional)

A self-contained vdrm client shim for Intel's iHD media driver. This transparently intercepts Xe DRM ioctls from the media driver and routes them through the virtio-gpu vdrm protocol. Only needed if you want hardware video decode/encode (VA-API) in the guest.

## How to Apply

These patches were developed against upstream git as of early April 2026. They may need minor rebasing if the upstream projects have changed significantly since then.

```bash
# virglrenderer (host)
cd virglrenderer
git apply virglrenderer-xe-native-context.patch

# Mesa (guest and host)
cd mesa
git apply mesa-01-xe-native-context-plus-iris-upload-fix.patch

# intel-media-driver (guest, only for VA-API)
cd media-driver
git apply intel-media-driver-xe-native-context.patch
```

## QEMU Configuration

You need a **recent QEMU build from git** (the required virtio-gpu features are not available in older releases).

Required QEMU flags:

```
-accel kvm,honor-guest-pat=on
-device virtio-vga-gl,blob=on,hostmem=4G,drm_native_context=on
```

Key requirements:
- `drm_native_context=on` -- enables the native context path (without this, you get software rendering or virgl translation, not Xe passthrough)
- `blob=on` -- enables blob resources for zero-copy buffer sharing
- `hostmem=4G` -- host memory window size for blob resources (adjust as needed)
- `honor-guest-pat=on` -- required for correct guest memory caching behavior on Intel

The host must be running the patched virglrenderer. The guest kernel must have `virtio-gpu` and `drm` support enabled.

## Building

### virglrenderer (host)

```bash
meson setup build -Ddrm-renderers=xe-experimental
ninja -C build
sudo ninja -C build install
```

If you also use i915 native context, combine them: `-Ddrm-renderers=i915-experimental,xe-experimental`.

### Mesa (guest and host)

Build Mesa as normal for your distribution. The patch integrates into the existing `intel/dev/virtio/` subsystem and the iris Gallium driver -- no special build flags are needed beyond what a normal Mesa build already uses.

### intel-media-driver (guest)

Build as normal per the [upstream instructions](https://github.com/intel/media-driver#build). The virtio shim is compiled automatically when the patched source is built. In the guest, set:

```bash
export LIBVA_DRIVER_NAME=iHD
```

## Architecture Overview

```
Guest VM                              Host
+--------------------------+          +---------------------------+
| Application              |          |                           |
|   |                      |          |                           |
| iris/ANV (Mesa)          |          |                           |
|   |                      |          |                           |
| xe virtio ccmd layer     |  virtio  | virglrenderer             |
|   (vdrm protocol)       <----------->  xe renderer backend     |
|   |                      |   gpu    |   |                       |
| virtio-gpu kernel driver |          | Xe kernel driver (real)   |
+--------------------------+          +---------------------------+
                                              |
                                         Intel Xe GPU
```

## Known Limitations

- The `banned_exec_queue_mask` in shared memory tracks up to 64 exec queues (one bit per queue in a `uint64_t`). This is sufficient for normal usage.
- VM binds are synchronous on the host side (the host waits for each bind to complete before returning to the guest). This is by design to guarantee mapping visibility before dependent submissions.
- The intel-media-driver shim uses a single-instance global vdrm connection, matching iHD's one-device-per-process model.

## License

The modified files in these patches are MIT/X11 licensed (see SPDX headers in each patched file).
