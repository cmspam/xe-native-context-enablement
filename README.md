
<img width="2069" height="1286" alt="Screenshot_20260403_042820" src="https://github.com/user-attachments/assets/9f9682c0-eb4c-46df-8c20-3a2b45dc1080" />

This enables intel Xe graphics with drm_native_context.

I do not believe the code is close to production quality, so please consider it "getting it to work" project and not "getting it to work properly."

I started with i915 drivers, and then troubleshoot issues as they came up to get to this point. I would like to rewrite it more cleanly, so I may do this when I have time.

With these patches, vulkan works, vaapi works (you will need to patch the intel media driver, and force i915 for LIBVA driver), OpenGL is probably not implemented right and has performance issues with 2D. So it's recommended to use Zink for opengl for now. This can be done by applying the mesa-xe-zink patch to mesa, instead of the mesa-xe-native-context.patch, to enable zink by default.

Patches:
- intel-media-driver-xe-native-context.patch (Apply this to intel-media-driver to get VAAPI working in the VM)
- mesa-xe-native-context.patch (Don't apply this at all, use mesa-xe-zink instead, unless you want to have a bad time with opengl)
- mesa-xe-zink.patch (Apply this to mesa on guest instead of mesa-xe-native-context to enable zink for opengl and have a good time)
- virglrenderer-xe-native-context.patch (Apply this to virglrenderer on host)


Important qemu settings -- Use very recent qemu from git:

```
    -accel kvm,honor-guest-pat=on
    -device virtio-vga-gl,blob=on,hostmem=4G,drm_native_context=on
```
