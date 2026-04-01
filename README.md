# Intel Xe Native Context Enablement for virtio-gpu

Full GPU acceleration for Intel Xe GPUs (Arc, Battlemage, Panther Lake, etc.) in QEMU/KVM virtual machines using virtio-gpu native context.

This enables **Vulkan**, **OpenGL**, and **VA-API hardware video decode/encode** to work natively in the guest, with the host GPU doing the actual rendering. No GPU passthrough required.

## What this is

Three patches that together enable Intel Xe GPU acceleration over virtio-gpu:

| Patch | Target | What it does |
|-------|--------|-------------|
| `virglrenderer-xe.patch` | [virglrenderer](https://gitlab.freedesktop.org/virgl/virglrenderer) (host) | Xe native context backend — receives guest GPU commands and executes them on the host GPU |
| `mesa-xe-virtio.patch` | [mesa](https://gitlab.freedesktop.org/mesa/mesa) (guest) | iris (OpenGL) and ANV (Vulkan) support over Xe native context |
| `intel-media-driver-virtio.patch` | [intel-media-driver](https://github.com/intel/media-driver) (guest) | iHD VA-API driver with embedded virtio transport for hardware video decode/encode |

## Features

- **Vulkan**: Full Intel ANV driver — games, compute, everything
- **OpenGL 4.6**: Intel iris driver with native GPU acceleration
- **VA-API**: Hardware H.264, H.265/HEVC, VP9, AV1, VVC decode and encode
- **No GPU passthrough**: Host keeps full use of the GPU
- **Multiple VMs**: Several VMs can share one GPU simultaneously

## Architecture

```
Guest VM                              Host
---------                             ----
Vulkan app -> ANV ------\
                         |-> vdrm ccmd -> virtio-gpu -> virglrenderer -> Intel Xe GPU
GL app -> iris ---------/              (kernel)         (Xe backend)

VA-API app -> iHD -> embedded vdrm -> virtio-gpu -> virglrenderer -> Intel Xe GPU
```

The guest drivers translate GPU ioctls into compact command messages (ccmds) sent through virtio-gpu's native context mechanism. The host virglrenderer Xe backend receives these and executes them on the real GPU.

## Requirements

- **Host**: Intel Xe GPU (Arc A-series, Battlemage, Panther Lake, or newer), Linux kernel with Xe driver
- **Guest**: Linux with virtio-gpu DRM driver
- **QEMU**: With virtio-gpu-pci and virglrenderer support

## Building

### Host: virglrenderer

```bash
git clone https://gitlab.freedesktop.org/virgl/virglrenderer.git
cd virglrenderer
git apply /path/to/virglrenderer-xe.patch
meson setup builddir -Ddrm-renderers=xe-experimental
ninja -C builddir
sudo ninja -C builddir install
```

### Guest: mesa

```bash
git clone https://gitlab.freedesktop.org/mesa/mesa.git
cd mesa
git apply /path/to/mesa-xe-virtio.patch
meson setup _build \
    -Dintel-virtio-experimental=true \
    -Dvulkan-drivers=intel \
    -Dgallium-drivers=iris,virgl,zink \
    -Dplatforms=wayland,x11
ninja -C _build
sudo ninja -C _build install
```

### Guest: intel-media-driver (VA-API)

```bash
git clone https://github.com/intel/media-driver.git
cd media-driver
git apply /path/to/intel-media-driver-virtio.patch
mkdir build && cd build
cmake .. -DCMAKE_INSTALL_PREFIX=/usr -DCMAKE_INSTALL_LIBDIR=lib \
    -DCMAKE_BUILD_TYPE=Release -DCMAKE_POLICY_VERSION_MINIMUM=3.5
make -j$(nproc)
sudo make install
```

**Note**: The VA-API driver requires `LIBVA_DRIVER_NAME=iHD` to be set, since the kernel reports `virtio_gpu` as the driver name. Add to `/etc/environment` or `/etc/profile.d/`:

```bash
# Only needed on virtio-gpu systems
export LIBVA_DRIVER_NAME=iHD
```

## QEMU Configuration

Example QEMU command line flags:

```
-device virtio-gpu-pci,virgl=on,blob=true,hostmem=4G,context_init=true
```

The `context_init=true` flag is required for native context support. QEMU must be built with virglrenderer that includes the Xe backend.

## How it works

The patches implement a **cross-call command (ccmd) protocol** between guest and host:

1. **DEVICE_QUERY** — guest queries GPU capabilities (engines, memory regions, etc.)
2. **GEM_CREATE** — allocate GPU memory objects
3. **VM_CREATE/DESTROY** — manage GPU virtual address spaces
4. **VM_BIND** — map memory into GPU address space
5. **EXEC_QUEUE_CREATE/DESTROY** — create GPU execution queues
6. **EXEC** — submit GPU command buffers for execution

The mesa patch includes a critical fence optimization: exec completion fences use `VIRTGPU_EXECBUF_FENCE_FD_OUT` to get a fence fd directly from the virtio-gpu kernel, then import it into the caller's syncobjs. This bypasses the host's fence retirement thread and provides low-latency frame presentation.

## Status

Working and tested:
- glxgears, glmark2, vkmark, vkcube
- mpv, VLC, ffmpeg hardware video decode
- ffmpeg hardware H.264, HEVC, AV1 encode
- SuperTuxKart, Extreme Tux Racer

## License

All patches are MIT licensed, matching their respective upstream projects.
