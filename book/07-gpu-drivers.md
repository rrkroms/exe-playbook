# 07 — GPU Drivers: From Vulkan Commands to Pixels

Modules 05 and 06 covered graphics APIs and the translators that turn Direct3D into Vulkan or OpenGL. This module covers the layer *below* the translator: the GPU driver. A translator can be perfect and still produce a black screen if no driver is loaded, if the wrong driver is loaded, or if the driver that renders the game is not the driver that presents the frame.

This is the layer most often misnamed “the graphics driver” in forum advice. We will be precise, because the distinctions explain otherwise contradictory symptoms.

## 7.1 What a GPU driver is — and what it is not

A **GPU driver** is the software that turns a graphics API command stream into work the GPU can execute. It has two halves:

| Half | Runs where | Name in this book | Example |
|---|---|---|---|
| Userspace module driver (UMD) | In the game/translator process | ICD or UMD | `libvulkan_freedreno.so` ([Turnip](https://docs.mesa3d.org/drivers/freedreno.html)) |
| Kernel module driver (KMD) | In the host kernel | KMD | `drm/msm`, `panfrost`, `amdgpu` |

The **ICD** (Installable Client Driver) is the userspace Vulkan driver plus the JSON manifest that describes it to the loader. The KMD owns the hardware: memory management, command submission and scheduling. [OpenGL](https://www.opengl.org/) drivers have a similar userspace/kernel split but use a different loader ([GLVND](https://gitlab.freedesktop.org/glvnd/libglvnd) on Linux).

What the driver is **not**:

- It is not the graphics API. [Vulkan](https://www.vulkan.org/) is a specification; the driver implements it.
- It is not the graphics translator. [DXVK](https://github.com/doitsujin/dxvk) turns D3D9/10/11 into Vulkan; it does not talk to the GPU itself.
- It is not the CPU translator. [Box64](https://github.com/ptitSeb/box64) or [FEX](https://github.com/FEX-Emu/FEX) execute x86 code; they do not compile shaders.
- On Android, it is not necessarily the driver drawing your status bar. See §7.4.

A conceptual view of the path:

```text
DXVK / VKD3D-Proton / D8VK / WineD3D
              |
              |  Vulkan or OpenGL calls
              v
      +-------------------+
      | Vulkan loader     |   libvulkan.so.1 + ICD manifests
      +---------+---------+
                |
                v
      +-------------------+
      | ICD / UMD         |   Turnip, Qualcomm, RADV, ANV, ...
      +---------+---------+
                |  ioctls / kernel ABI
                v
      +-------------------+
      | KMD               |   drm/msm, panfrost, amdgpu, i915, pvr
      +---------+---------+
                |
                v
       GPU  ->  display / compositor
```

This is a *conceptual* diagram, not a literal runtime order. The loader, ICD and translator are libraries inside the game process and run on the CPU concurrently with the game. The KMD runs in kernel space; the compositor is a separate process. Several boxes may come from one package (for example, [Mesa](https://gitlab.freedesktop.org/mesa/mesa) supplies both the ICD and much of the OpenGL driver).

## 7.2 The Vulkan loader and ICD model

The [Vulkan loader](https://github.com/KhronosGroup/Vulkan-Loader) is a small library, normally `libvulkan.so.1`, that applications link against. It does not implement Vulkan. It reads **ICD manifests** — small JSON files — from directories such as `/usr/share/vulkan/icd.d/` and `~/.local/share/vulkan/icd.d/`, then forwards `vk*` calls to the selected driver. See the [Khronos loader guide](https://docs.vulkan.org/guide/latest/loader.html).

An illustrative manifest (exact fields vary by loader version):

```json
{
  "file_format_version": "1.0.0",
  "ICD": {
    "library_path": "/usr/lib/aarch64-linux-gnu/libvulkan_freedreno.so",
    "api_version": "1.3.0",
    "library_arch": "64"
  }
}
```

The important fields:

| Field | Meaning |
|---|---|
| `ICD.library_path` | Path to the driver `.so`. A wrong path is the most common cause of “no Vulkan device”. |
| `api_version` | Vulkan API version the ICD advertises. A game needing Vulkan 1.3 will fail on a 1.1-only ICD. |
| `library_arch` | Whether the library is 32- or 64-bit; must match the process. |

Because the loader reads manifests, you can install several drivers and choose per process without uninstalling anything.

Legacy note: a game that uses Direct3D 7/8/9 may reach this layer through very different paths. [D7VK](https://github.com/WinterSnowfall/d7vk) and [D8VK](https://github.com/AlpyneDreams/d8vk) target Vulkan, [DXVK](https://github.com/doitsujin/dxvk) targets Vulkan for D3D9–11, while WineD3D (part of [Wine](https://gitlab.winehq.org/wine/wine)) can use an OpenGL backend. [dgVoodoo2](https://www.dege.freeweb.hu/dgVoodoo2/) wraps Glide, DirectDraw and D3D3–9 into **D3D11 or D3D12** (or WARP). If your path is WineD3D on its OpenGL backend, the Vulkan ICD is irrelevant; the OpenGL driver choices in §7.5 still matter. This is why “install the newest Turnip” fixes some old games and does nothing for others.

## 7.3 Choosing an ICD with environment variables

On desktop Linux, three variables control discovery (see the [Vulkan-Loader documentation](https://github.com/KhronosGroup/Vulkan-Loader/tree/main/docs)):

| Variable | Effect | Use |
|---|---|---|
| `VK_DRIVER_FILES` | **Replaces** normal discovery with a colon-separated list of manifest paths | Force one specific ICD for this process |
| `VK_ICD_FILENAMES` | Deprecated alias of `VK_DRIVER_FILES` | Older frontends and scripts |
| `VK_ADD_DRIVER_FILES` | **Adds** manifests to the discovered set | Add a driver without hiding others |

Use absolute paths. Relative paths and symlinks into app sandboxes are a frequent source of “the variable is set but ignored”.

```sh
# Force Mesa Turnip for one launch
export VK_DRIVER_FILES=/usr/lib/aarch64-linux-gnu/share/vulkan/icd.d/freedreno_icd.aarch64.json
vulkaninfo --summary
```

Two important behaviours:

1. The loader ignores ICD and layer environment variables under elevated privileges. If selection “does nothing” under `sudo` or a privileged helper, that is expected, not a driver bug ([Khronos loader architecture](https://github.com/KhronosGroup/Vulkan-Loader/blob/main/docs/LoaderInterfaceArchitecture.md#elevated-privilege-caveats)).
2. Android is different. A normal Android app cannot set `VK_DRIVER_FILES` to change its Vulkan driver; per-app injection is the route (§7.4).

**Verify, do not assume.** [vulkaninfo](https://github.com/KhronosGroup/Vulkan-Tools) is the authoritative check:

```sh
vulkaninfo --summary | grep -i driverName
VK_LOADER_DEBUG=error,warn vulkaninfo
```

`VK_LOADER_DEBUG` (values `error`, `warn`, `info`, `debug`, `layer`, `driver`, `all`) shows which manifests were found and which were rejected. Run it on the *desktop* loader; Android builds may not honour it. If `vulkaninfo` shows the expected driver but the game does not, the problem is probably in the graphics translator or WSI presentation, not the ICD.

A concrete example from the Termux pipeline:

```sh
export VK_DRIVER_FILES=/usr/share/vulkan/icd.d/freedreno_icd.aarch64.json
export DXVK_HUD=devinfo,fps
wine game.exe        # HUD 'devinfo' should name the Turnip device
```

`DXVK_HUD=devinfo` is useful because it reports the device DXVK actually got, not the device you intended to give it.

## 7.4 Runtime driver vs display/compositor driver

This distinction matters almost only on Android, but it is the source of some of the most confusing bugs.

- The **runtime driver** is the UMD loaded by your compatibility stack to render the game — for example Turnip injected per app.
- The **display driver** is what [SurfaceFlinger and Hardware Composer (HWC)](https://source.android.com/docs/core/graphics) use to compose the desktop, navigation bar and other apps. On an Adreno phone this is normally Qualcomm's system driver.

They can be different drivers using different memory layouts. Android WSI is exposed through `VK_KHR_android_surface`; the loader hands the ICD an Android native window/BufferQueue, and the resulting buffers are composited by the system. See [Android's Vulkan implementation guide](https://source.android.com/docs/core/graphics/implement-vulkan).

Practical consequences:

| Situation | Likely cause |
|---|---|
| Game renders but UI is fine; flicker appears only in fullscreen or on present | Runtime ICD and system compositor disagree on buffer format/compression |
| A740 devices show UI flicker or black frames during Android compositing (community driver-maintainer reports) | A UBWC (Universal Bandwidth Compression) flag mismatch with the system driver is suspected; do **not** apply the “A740 UBWC hint” blindly. Treat as a device/firmware-specific lead, not an upstream rule. |
| Injecting a system-wide driver breaks the launcher or banking apps | You replaced the display driver, not just the runtime driver |

**Per-app injection** (no root) is the safe model: keep the vendor driver for the UI and load the alternative ICD only inside the game process, typically with a front-end or a tool such as [AdrenoTools](https://github.com/K11MCH1/AdrenoToolsDrivers/releases) [community, not Qualcomm; actively maintained as of 2026-11-08, with 2026 releases]. **System-wide replacement** requires an unlocked bootloader, a remounted `/system` or `/vendor`, and fixing the library SONAME so existing callers still find it. It voids warranties and can break DRM apps; see the preface safety warning.

sysmem and gmem are two Adreno render paths: **sysmem** renders to system memory; **gmem** uses the GPU's on-chip tile buffer. Select with `TU_DEBUG=sysmem` or `TU_DEBUG=gmem`. Mesa also documents an official autotuner with `TU_AUTOTUNE_ALGO` (`bandwidth`, `profiled`, `profiled_imm`, `prefer_sysmem`, `prefer_gmem`) and `TU_AUTOTUNE_FLAGS` ([Freedreno docs](https://docs.mesa3d.org/drivers/freedreno.html#autotune)); the forced `TU_DEBUG` modes are separate. If a scene corrupts only under gmem, forcing sysmem is a standard diagnostic step, at a bandwidth cost.

## 7.5 Driver matrix by GPU family

The table below reflects a 2026-11-25 snapshot (Mesa 26.3.1; 26.2.7 was the final 26.2.x maintenance release). Treat version numbers as a snapshot, not guarantees. “GL” means a Mesa OpenGL driver; “Vulkan 1.x” is the advertised API version.

| GPU family | Driver(s) | APIs | Kernel path | License | Status / limits |
|---|---|---|---|---|---|
| Adreno 6xx/7xx | [Freedreno GL + Turnip Vulkan](https://docs.mesa3d.org/drivers/freedreno.html) (`libvulkan_freedreno.so`) | GL 4.5 / GLES 3.2; Vulkan 1.3 A6xx, 1.4 A7xx | mainline `drm/msm`; Android KGSL (`/dev/kgsl-3d0`) | MIT | Active. No A5xx. Build choice depends on KGSL vs msm. Community flicker reports concern A740 (8 Gen 2) specifically, not all A7xx; no upstream Mesa warning found. |
| Adreno (any) | [Qualcomm proprietary UMD](https://www.qualcomm.com/products/technology/gpus/adreno) | GLES 3.2, Vulkan 1.1–1.4 | vendor BSP | Proprietary | Default on devices; per-device builds. Extracted packages can crash on a different GPU |
| Mali | [Panfrost + PanVK](https://docs.mesa3d.org/drivers/panfrost.html) | GLES 3.1 / GL 3.1; Vulkan 1.3 Bifrost/Valhall, 1.4 5th gen | `panfrost` KMD | MIT | GLES conformant on G52/G57/G610; Vulkan on G610. PanVK still gated by `PAN_I_WANT_A_BROKEN_VULKAN_DRIVER=1` outside supported parts; no Vulkan for V9 Valhall |
| Mali (stock) | [Arm Mali DDK](https://developer.arm.com/Processors/Mali-Gpus) | GLES / Vulkan per DDK | vendor BSP | Proprietary | Stock Android driver; safest for UI |
| PowerVR | [Mesa PVR](https://docs.mesa3d.org/drivers/powervr.html) Vulkan + [Zink](https://docs.mesa3d.org/drivers/zink.html) for GL | Vulkan 1.2; GL via Zink | `pvr` KMD (Linux 6.16+) | MIT | Active; no native Mesa OpenGL driver; KMSRO needed; pre-Series6 not supported |
| Apple AGX | [Asahi GL + HoneyKrisp Vulkan](https://asahilinux.org/) | GL 4.6 / GLES 3.2; Vulkan 1.3/1.4 | Asahi DRM | MIT | Upstream in Mesa 25.2 / Linux 6.16. 16K host pages; M3 3D support young; reverse-engineered |
| Raspberry Pi | [V3D + V3DV](https://docs.mesa3d.org/drivers/v3d.html) | GLES 3.1 conformant; GL 3.1 non-conformant; Vulkan 1.3 conformant | `vc4` display + `v3d` render | MIT | Active (Igalia); low limits and bandwidth |
| Intel | [ANV + Iris](https://docs.mesa3d.org/drivers/anv.html) (HasVK for older parts) | Vulkan 1.4 conformant; GL 4.6 | `i915` / `xe` | MIT | Intel's official Linux driver; newer parts need GuC firmware |
| AMD | [RADV + RadeonSI](https://docs.mesa3d.org/drivers/radv.html) | Vulkan 1.4 conformant; GL 4.6 | `amdgpu` | MIT | Default on Steam Deck; older GCN may need KMD parameter tuning |
| NVIDIA | [Proprietary ICD](https://developer.nvidia.com/vulkan-driver) or [NVK](https://docs.mesa3d.org/drivers/nvk.html)+Zink | Vulkan 1.4; GL via GLVND/Zink | NVIDIA KMS / `nouveau` | Proprietary / MIT | Closed userspace for the ICD; NVK has performance and reclocking caveats |
| Virtual GPU | [VirGL](https://gitlab.freedesktop.org/virgl/virglrenderer) (GL), [Venus](https://docs.mesa3d.org/drivers/venus.html) (Vulkan), DRM Native Context | GL 4.3–4.6; Vulkan 1.4; native pass-through | virtio-gpu host side | Open | Venus production-capable; needs host blob support and validated host drivers. VirGL compiles shaders twice |

Two cross-cutting notes:

- Newer graphics translators raise the driver bar. DXVK 3.x expects Vulkan 1.4; [VKD3D-Proton](https://github.com/HansKristian-Work/vkd3d-proton) expects Vulkan 1.3 plus descriptor indexing, `robustness2` and push descriptors. A driver can be “working” and still be too old for the translator you chose.
- macOS is not a Vulkan-ICD host in this sense. Its games reach Metal through [D3DMetal](https://developer.apple.com/games/game-porting-toolkit), [DXMT](https://github.com/3Shain/dxmt) or Vulkan-to-Metal layers such as [MoltenVK](https://github.com/KhronosGroup/MoltenVK) / KosmicKrisp. Apple's OpenGL is frozen at 4.1, so WineD3D's legacy path is weak there. ChromeOS VMs typically use VirGL or Venus instead of a native ICD on the guest.

## 7.6 Turnip in depth, with safe guidance for Adreno 730

Turnip is Mesa's Vulkan driver for Adreno, shipped as part of [Freedreno](https://docs.mesa3d.org/drivers/freedreno.html). It is a real driver, not an emulator.

**Version labels.** Mesa uses `YY.N` releases (for example 26.3). Community builds add their own revision schemes such as “26.0.0 R8”. These are *not* comparable across builders. Always read the commit or tag a build names; a number alone is not enough to know what code you are running.

**Chip support.** A6xx parts 640/650/660 are mature; A7xx parts 710/720/722/725/730/732/735/740/750 have Vulkan 1.4; A8xx OpenGL arrived in Mesa 26.0 with Vulkan still experimental. There is no A5xx support.

**Kernel split.** Mainline Linux uses `drm/msm`; stock Android uses KGSL `/dev/kgsl-3d0`. A Turnip build is compiled against one kernel ABI, so a build for mainline will not drive a KGSL device and vice versa. Per-app use needs no root, but the device node must be accessible to the app's process. System-wide replacement needs unlocked-bootloader-level changes and a SONAME fix, as described in §7.4.

**TU_DEBUG and properties.** Useful values from the [Mesa environment variable reference](https://docs.mesa3d.org/envvars.html) and Freedreno docs include `sysmem`, `gmem`, `nobin`/`forcebin`, `noubwc`, `nolrz`, `flushall`, `syncdraw`, `rd`. On Mesa for Android these map to system properties: `debug.*` works unrooted, `vendor.*` needs root.

**Safe recipe for the reference tablet (Adreno 730):**

1. Keep the stock Qualcomm driver for the Android UI; inject Turnip only into the game process.
2. Use a stable upstream Mesa release (26.3.1 at the 2026-11-25 snapshot; 26.2.7 was the final 26.2.x maintenance release) or a community build that explicitly names its upstream commit. Winlator 11.2 uses a Mesa Turnip 26.2.0 branch build rather than stock upstream Mesa.
3. Prefer plain “A6xx/A7xx Standard” builds over specialised variants.
4. Do **not** copy A740-specific UBWC hints to a 730 without testing: community driver builders document `FD_DEV_FEATURES=enable_tp_ubwc_flag_hint=1` for One UI / A740 UI corruption, and warn it can cause the glitch on other 8 Gen 2 firmwares. No upstream Mesa or Qualcomm confirmation exists.
5. Avoid random re-uploaded ZIPs with no commit hash; a mislabelled build for a different Adreno generation is common.
6. Community driver lists cite Turnip 26.0.0 R8 (published 2026-01-08) and Qualcomm v615.77 for A730 as anecdotal starting points, not upstream recommendations; newer community Turnip builds exist (for example v27.0.0 in November 2026).

If a specific game regresses, step back one known-good build rather than chasing “latest”.

## 7.7 Choosing a driver per host

| Host | Recommended runtime driver | Notes |
|---|---|---|
| Android / Adreno | Stock Qualcomm for UI; Turnip per app for emulation | No-root per-app injection; see §7.6 |
| Android / Mali | Stock Arm DDK | PanVK is upstream Mesa and buildable for Android with the NDK; on phones the stock Arm DDK remains default and third-party injection is community territory |
| ARM Linux / Qualcomm SoC | Freedreno + Turnip (`drm/msm`) | Snapdragon X laptops need firmware extraction |
| ARM Linux / Mali | Panfrost + PanVK where supported | Check the exact GPU model against PanVK support |
| ARM Linux / Raspberry Pi | V3D + V3DV | 4K pages; low GPU bandwidth limits resolution |
| ARM Linux / Apple Silicon (Asahi) | Asahi GL + HoneyKrisp | 16K pages; use muvm/FEX for x86 code (module 12) |
| x86 Linux / AMD | RADV | Default on Steam Deck |
| x86 Linux / Intel | ANV | Check GuC firmware |
| x86 Linux / NVIDIA | Proprietary ICD; NVK+Zink as open alternative | Reclocking caveats on NVK |
| VM / guest | VirGL for light GL; Venus for guest Vulkan; Zink-on-Venus; DRM Native Context when host and guest share a driver family | Venus needs a validated host driver |
| macOS / ChromeOS | See §7.5; these hosts do not use a desktop Vulkan ICD path in the same way | License- and device-dependent |

Rollback procedure: record the exact ICD path and version before changing anything; to roll back, point `VK_DRIVER_FILES` at the previous manifest (or restore the previous package) and clear shader caches (§7.8). Never change translator *and* driver in the same test.

## 7.8 Shader caches

Shaders are compiled twice in this stack: by the translator from D3D bytecode to SPIR-V, and by the driver from SPIR-V to GPU machine code. Both stages cache results.

| Cache | Location / knob | Note |
|---|---|---|
| Mesa | `MESA_SHADER_CACHE_DIR`; default max size 1 GiB | `MESA_DISK_CACHE_SINGLE_FILE=1` uses a single [Fossilize](https://github.com/ValveSoftware/Fossilize) database |
| DXVK | `DXVK_SHADER_CACHE_PATH` (default under `AppData/Local` in the prefix) | Graphics Pipeline Library in DXVK 2.x+ reduces compile stutter; 3.x uses `dxbc-spirv` |
| VKD3D-Proton | `vkd3d-proton.cache` in the working directory, or `VKD3D_SHADER_CACHE_PATH` | `VKD3D_CONFIG=pipeline_library_app_cache` helps |
| Translator-side async options | `dxvk-gplasync` (third-party fork) | No official `DXVK_ASYNC`; not safe for online games |

Driver or ABI changes invalidate caches. Expect the first launch after an update to stutter while shaders recompile — that is normal, not a regression. Conversely, graphical artifacts that appear immediately after a driver change often disappear when caches are cleared. Clearing a cache makes the next run slow; do not interpret that slowdown as a failure.

## 7.9 Failure modes and first checks

| Symptom | Likely layer | First check |
|---|---|---|
| `vkCreateInstance` fails / “no Vulkan device” | ICD discovery | `VK_DRIVER_FILES` path, manifest permissions, `vulkaninfo --summary`, `VK_LOADER_DEBUG` |
| Driver selection variable ignored | Loader security or Android | Elevated process restrictions; Android requires per-app injection |
| Game starts, then device lost | Driver / KMD | `TU_DEBUG=sysmem,nolrz,flushall`, `dmesg`, kernel logs |
| Flicker only in fullscreen | UBWC / present mode | Compare runtime vs display driver; try `noubwc`, do not force A740 hint on other chips |
| Missing textures or artifacts | Translator/driver mismatch or stale cache | Update translator, clear shader cache, change one variable |
| Slow first launch after update | Cache invalidation | Wait one run; check cache directory is writable |
| Black screen, audio fine | WSI / presentation | Fullscreen vs windowed, swapchain, compositor, runtime vs display driver |
| DXVK/VKD3D refuses to start | Driver too old | Check advertised `api_version` against Vulkan 1.4 / 1.3 + required extensions |

A general rule: confirm the driver with `vulkaninfo` *before* debugging the game. Many “game bugs” are a driver that was never loaded.

## 7.10 What to remember

- The Vulkan loader reads ICD manifests; the ICD is the userspace driver, the KMD owns the hardware.
- `VK_DRIVER_FILES` replaces discovery, `VK_ICD_FILENAMES` is its deprecated alias, and `VK_ADD_DRIVER_FILES` adds; use absolute paths and verify with `vulkaninfo`.
- On Android the game's runtime driver can differ from SurfaceFlinger/HWC's display driver; per-app injection is the safe route.
- Choose the driver by GPU family *and* host kernel (KGSL vs `drm/msm`), and check the Vulkan version your translator requires.
- Turnip is a real Adreno Vulkan driver; for Adreno 730, keep stock for UI, inject a commit-labelled stable build per app.
- Shader caches make updates look slow and changes look broken; clear them deliberately.
- Diagnose by layer: ICD discovery, translator, driver, presentation.

Next: [08 — Runtimes, Containers and Wine Prefixes](08-runtime-prefixes.md).

## References

- Vulkan Loader: https://github.com/KhronosGroup/Vulkan-Loader ; loader docs https://github.com/KhronosGroup/Vulkan-Loader/tree/main/docs ; guide https://docs.vulkan.org/guide/latest/loader.html
- Vulkan Tools (`vulkaninfo`): https://github.com/KhronosGroup/Vulkan-Tools
- Mesa driver docs: freedreno/Turnip https://docs.mesa3d.org/drivers/freedreno.html ; panfrost https://docs.mesa3d.org/drivers/panfrost.html ; powervr https://docs.mesa3d.org/drivers/powervr.html ; asahi https://docs.mesa3d.org/drivers/asahi.html ; v3d https://docs.mesa3d.org/drivers/v3d.html ; anv https://docs.mesa3d.org/drivers/anv.html ; radv https://docs.mesa3d.org/drivers/radv.html ; nvk https://docs.mesa3d.org/drivers/nvk.html ; zink https://docs.mesa3d.org/drivers/zink.html ; venus https://docs.mesa3d.org/drivers/venus.html ; envvars https://docs.mesa3d.org/envvars.html
- Mesa source: https://gitlab.freedesktop.org/mesa/mesa
- Qualcomm Adreno: https://www.qualcomm.com/products/technology/gpus/adreno ; Arm Mali: https://developer.arm.com/Processors/Mali-Gpus
- Asahi Linux: https://asahilinux.org/
- NVIDIA Vulkan driver: https://developer.nvidia.com/vulkan-driver
- GLVND: https://gitlab.freedesktop.org/glvnd/libglvnd
- VirGL renderer: https://gitlab.freedesktop.org/virgl/virglrenderer
- Android graphics: https://source.android.com/docs/core/graphics ; Vulkan implementation https://source.android.com/docs/core/graphics/implement-vulkan
- Graphics translators: DXVK https://github.com/doitsujin/dxvk ; VKD3D-Proton https://github.com/HansKristian-Work/vkd3d-proton ; D8VK https://github.com/AlpyneDreams/d8vk ; D7VK https://github.com/WinterSnowfall/d7vk
- Wine: https://gitlab.winehq.org/wine/wine ; WineD3D is part of Wine
- dgVoodoo2: https://www.dege.freeweb.hu/dgVoodoo2/
- Fossilize: https://github.com/ValveSoftware/Fossilize
- AdrenoTools (community): https://github.com/K11MCH1/AdrenoToolsDrivers/releases
- macOS graphics: Game Porting Toolkit https://developer.apple.com/games/game-porting-toolkit ; DXMT https://github.com/3Shain/dxmt ; MoltenVK https://github.com/KhronosGroup/MoltenVK
