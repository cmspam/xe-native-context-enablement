This is a prototype/proof of concept for intel Xe graphics with drm_native_context. I am constantly in the process of rewriting and considering possibility to submit upstream once I feel it is solid. Feel free to play with this in the meantime.

With these patches, vulkan works well, vaapi works, opengl does not work well -- it has some issues. So it's recommended to use Zink for opengl in the meantime.
