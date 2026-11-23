# Research Brief — GPU Driver Stacks Across Hosts (Nov 2026)

Status: Mesa stable 26.2.7 (2026-11-11); Mesa core MIT. [V]=primary, [C]=community, [U]=unverified.

## 1. Driver matrix

| GPU family | Driver(s) | API | Docs | License | Hosts | Status | Limits |
|---|---|---|---|---|---|---|---|
| Adreno | Freedreno GL + Turnip Vulkan (`libvulkan_freedreno.so`) | GL 4.6/GLES 3.2; Vulkan 1.4 A7xx, 1.3 A6xx | docs.mesa3d.org/drivers/freedreno.html | MIT | Mainline `drm/msm`, Android KGSL, ChromeOS, Termux/PRoot | Active; A6xx newly Vulkan 1.3 conformant, A7xx 1.4 | No a5xx; KGSL vs msm build choice; A7xx flicker [C] |
| Adreno | Qualcomm proprietary UMD | GLES 3.2, Vulkan 1.1–1.4 | vendor BSP | Proprietary | Android, WoA, downstream Linux | Default on devices | Per-device builds; extracted packages crash on wrong GPU |
| Mali | Panfrost + PanVK | GLES 3.1/GL 3.1; Vulkan 1.3 Bifrost/Valhall, 1.4 5th gen | docs.mesa3d.org/drivers/panfrost.html | MIT | Linux ARM panfrost KMD; Android community [C] | Conformant GLES G52/G57/G610; Vulkan G610 | Experimental PanVK gated by `PAN_I_WANT_A_BROKEN_VULKAN_DRIVER=1`; no V9 Valhall Vulkan |
| PowerVR | Mesa PVR (Vulkan) + Zink (GL) | Vulkan 1.2; GL via Zink | docs.mesa3d.org/drivers/powervr.html | MIT | Linux 6.16+ `pvr` KMD, RISC-V | Active, Imagination-employed | No native Mesa GL; KMSRO needed; pre-Series6 impossible |
| Apple AGX | Asahi GL + HoneyKrisp Vulkan | GL 4.6/GLES 3.2, Vulkan 1.3/1.4 | docs.mesa3d.org/drivers/asahi.html; asahilinux.org | MIT | Apple Silicon Linux (M1/M2; M3 early) | Fully upstream Mesa 25.2/Linux 6.16 | 16K pages; M3 3D young; reverse-engineered |
| Raspberry Pi | V3D + V3DV | GLES 3.1 conformant; GL 3.1 non-conformant; Vulkan 1.3 conformant | docs.mesa3d.org/drivers/v3d.html | MIT | Pi 4/5 | Active (Igalia) | vc4 display + v3d render; low limits |
| Intel | ANV + Iris; HasVK | Vulkan 1.4 conformant; GL 4.6 | docs.mesa3d.org/drivers/anv.html | MIT | Linux i915/xe | Intel official Linux driver | GuC firmware required on newer parts |
| AMD | RADV + RadeonSI | Vulkan 1.4 conformant; GL 4.6 | docs.mesa3d.org/drivers/radv.html | MIT | Linux amdgpu | Default on Steam Deck | KMD parameter tuning on older GCN |
| NVIDIA | Proprietary ICD; NVK+Zink | Vulkan 1.4; GL via GLVND/Zink | developer.nvidia.com/vulkan-driver; docs.mesa3d.org/drivers/nvk.html | Proprietary/MIT | Linux x86_64/aarch64 | Active | Closed userspace; NVK perf/reclocking caveats |
| VM | VirGL (GL), Venus (Vulkan), DRM Native Context | GL 4.3/4.6; Vulkan 1.4; native | docs.mesa3d.org/drivers/venus.html; qemu.org | Open | Guest Linux VM | Venus production-capable | VirGL double shader compile; Venus needs blob + validated host drivers |

## 2. Vulkan loader / ICD

- Loader `libvulkan.so.1` + JSON manifests in `icd.d` dirs. Fields: `ICD.library_path`, `api_version`, `library_arch`.
- `VK_DRIVER_FILES` replaces discovery (colon-separated, absolute paths preferred); `VK_ICD_FILENAMES` deprecated alias; `VK_ADD_DRIVER_FILES` additive. Ignored under sudo/setuid.
- Verify: `vulkaninfo --summary | grep driverName`; `VK_LOADER_DEBUG=error,warn vulkaninfo`.
- Runtime vs display driver: on Android, app's UMD (e.g. Turnip via AdrenoTools) vs SurfaceFlinger/HWC using system GLES/Vulkan. WSI implemented in loader via `VK_KHR_android_surface`; no `VK_DRIVER_FILES` on Android — per-app injection is the route. A740 UBWC flag mismatch causes flicker [C].
- sysmem/gmem are renderpaths, `TU_DEBUG=sysmem|gmem`; autotuner available.

## 3. Turnip specifics

- Mesa versions YY.N (26.2 etc.); community builds have own revision schemes (e.g. 26.0.0 R8) — read commit/tag.
- Chips: A6xx 640/650/660 mature; A7xx 710/720/722/725/730/732/735/740/750; A8xx GL in Mesa 26.0, Vulkan experimental.
- Mainline uses `drm/msm`; Android uses KGSL `/dev/kgsl-3d0`. Per-app use needs no root. System-wide replacement needs unlocked bootloader/remount + SONAME fix.
- Mesa Android maps `TU_DEBUG` to system properties (`debug.*` unrooted, `vendor.*` needs root).
- For Adreno 730: use stable Mesa (26.2.x) or community build naming upstream commit; keep Qualcomm system driver for UI; inject Turnip per app; prefer plain "A6xx/A7xx Standard" builds; do NOT apply A740 UBWC hint; avoid random re-uploaded ZIPs. Community favorites: Turnip 26.0.0 R8, Qualcomm v615.77 [C].

## 4. Choosing a driver

- Android/Adreno: stock Qualcomm for UI; Turnip per-app for emulation. Android/Mali: stock Arm DDK.
- ARM SBC: Mali→Panfrost/PanVK; RPi→V3D/V3DV; Qualcomm→Freedreno/Turnip; PowerVR→Mesa PVR+Zink.
- x86 Linux: AMD→RADV, Intel→ANV, NVIDIA→proprietary; NVK+Zink alternative.
- VMs: VirGL for light GL; Venus for guest Vulkan; Zink-on-Venus; DRM Native Context for same driver family.

## 5. Caches and failures

- Mesa shader cache `MESA_SHADER_CACHE_DIR`, default 1 GiB; `MESA_DISK_CACHE_SINGLE_FILE=1` Fossilize DB. Driver/ABI changes invalidate caches.
- Failures: no Vulkan device → check ICD path/permissions; override ignored → elevated privileges; device lost → `TU_DEBUG=sysmem,nolrz,flushall`, dmesg; flicker → UBWC/present mode; artifacts → clear caches; first launch slow after update → cache invalidation.

## 6. Citations (verified)

- https://docs.mesa3d.org/drivers/freedreno.html
- https://docs.mesa3d.org/drivers/panfrost.html
- https://docs.mesa3d.org/drivers/powervr.html
- https://docs.mesa3d.org/drivers/asahi.html
- https://docs.mesa3d.org/drivers/v3d.html
- https://docs.mesa3d.org/drivers/radv.html
- https://docs.mesa3d.org/drivers/anv.html
- https://docs.mesa3d.org/drivers/nvk.html
- https://docs.mesa3d.org/drivers/zink.html
- https://docs.mesa3d.org/drivers/venus.html
- https://docs.mesa3d.org/envvars.html
- https://github.com/KhronosGroup/Vulkan-Loader
- https://docs.vulkan.org/guide/latest/loader.html
- https://source.android.com/docs/core/graphics
- https://source.android.com/docs/core/graphics/implement-vulkan
- https://asahilinux.org/
