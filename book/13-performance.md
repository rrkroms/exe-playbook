# 13 — Performance: Finding the Bottleneck

A translated game is slower than a native game not because of one flaw, but because every layer in the pipeline adds cost. The mistake most people make is treating "low FPS" as a single problem. It is a symptom with many possible causes, and the fix for one cause can make another worse.

This module gives you a mental model of where time goes, the tools that measure each layer, and the settings that change each cost. The goal is not to memorise every environment variable. The goal is to answer one question with evidence: **which layer is saturated right now?**

> **Scope and honesty.** Performance numbers depend on the exact Wine/Proton build, translator build, driver version, kernel, game patch and host. Treat all figures as orientation, not guarantees. Where a claim is not backed by the sources collected for this book, it is marked `[NEEDS VERIFICATION]` with the gap explained.

## 13.1 Terms you need first

| Term | Definition | Why it matters here |
|---|---|---|
| Frame time | Milliseconds between completed frames | The real unit of performance; FPS is just `1000 / frame time` |
| 1% low | The average of the slowest 1% of frames | Reveals stutter that average FPS hides |
| Bottleneck / bound | The layer that sets the ceiling for the whole pipeline | Speeding up any other layer has no effect until the bottleneck moves |
| GPU-bound | The GPU cannot finish work faster | Lower resolution usually helps |
| CPU-bound | The CPU (game logic, Wine, or translator) cannot feed the GPU | Lower resolution usually does *not* help |
| Shader / pipeline | GPU program plus its fixed state (blend, depth, vertex format) | Compiling these at the wrong time causes hitches |
| Present mode | How completed frames are handed to the display server | Controls latency, tearing and jitter |
| VRAM | GPU-local memory, separate from system RAM | Running out causes severe stutter or "out of memory" paths |
| Thermal throttling | Hardware reduces clock speed when too hot | Performance falls over time, not from the start |

A useful rule: **never diagnose from average FPS alone.** Two runs with identical FPS can have completely different 1% lows and completely different bottlenecks.

## 13.2 The layered performance model

This is a **conceptual model**, not a literal instruction-by-instruction runtime order. Real execution loops, interleaves and caches across these stages.

```text
Game logic          -> CPU work in the guest binary
  |                     (x86/x86-64 instructions)
  v
ISA translation     -> FEX / Box64 turns guest code into host code
  |                     (only on ARM or other non-x86 hosts)
  v
Wine APIs           -> Windows API calls, object management, sync
  |
  v
Graphics translation-> DXVK / VKD3D-Proton builds Vulkan commands
  |
  v
GPU driver          -> Turnip / RADV / ANV turns Vulkan into GPU work
  |
  v
Display + VRAM      -> present, composite, thermal and power limits
```

| Layer | What it costs | Which hosts pay it |
|---|---|---|
| ISA translation | Decode, JIT, block cache, memory-order barriers, flags emulation | ARM Linux, Android, Apple Silicon; absent on x86 Linux |
| Wine API / sync | Call overhead, server round trips, synchronisation primitives | All hosts using Wine/Proton |
| Graphics translation | Draw-call translation, descriptor management, shader translation | All hosts using DXVK/VKD3D |
| GPU driver | Pipeline compilation, scheduling, memory management | All hosts |
| Container / filesystem | I/O staging, path translation, overlay overhead | Strongest in proot/chroot Android setups |
| Display composition | Extra copy, queueing, vsync | All hosts; SurfaceFlinger on Android |

The practical consequence: on x86 Linux you can often fix a problem by changing only the GPU side. On ARM the CPU translation layer is always in the path, and a game can be GPU-light yet still run badly.

## 13.3 Stage 1: ISA translation cost

[FEX](https://github.com/FEX-Emu/FEX) and [Box64](https://github.com/ptitSeb/box64) are **CPU translators**: they decode x86/x86-64 instructions and emit host instructions. They are not Wine and they are not graphics translators. [Box86](https://github.com/ptitSeb/box86) does the same for 32-bit x86.

The dominant cost is not the translation itself once cached, but the work the translator must do to preserve guest semantics: flags, memory ordering on weakly ordered ARM, self-modifying code detection, signals, threads, and SIMD differences. This is why overhead varies wildly between games and between the two translators, and why neither is universally faster.

Key knobs and measurements:

| Tool / variable | What it shows | Notes |
|---|---|---|
| `BOX64_DYNAREC_TRACE` | DynaRec decisions | High volume; use briefly |
| `BOX64_DYNAREC_PERFMAP=1` | Map for `perf` analysis | Pair with Linux `perf` |
| FEX `Logging.ProfileStats` | Translator profile stats | Config key (`FEX_PROFILESTATS` in env form), read by supported MangoHud builds. `FEX_LOG_LEVEL` is not a generated env var in the current config schema — check your build before relying on it. |
| `BOX64_DYNACACHE` | Translation cache toggle | `0` disables, `1` enables (default), `2` uses existing files without generating new ones; `BOX64_DYNACACHE_LIMIT` sets folder size (default: 2048 MiB) |
| `BOX64_SKIPCPU` / `BOX64_MAXCPU` | CPU affinity / core skipping | Relevant on big.LITTLE designs |

**DynaRec versus interpreter.** Box64's own README reports that its dynamic recompiler is roughly 5–10× faster than its interpreter; treat that as a project-wide figure, not a per-game benchmark. Running with the interpreter enabled — or on a translator build without DynaRec for your host ISA — is a first-class performance bug. Confirm which mode your frontend actually selected; frontends often expose this as a toggle.

**Big.LITTLE caution.** On phones and Snapdragon-class laptops, cores differ. If the game's main thread lands on an efficiency core, a CPU-bound title will stutter regardless of GPU speed. One core pinned at 100% on ARM while the GPU is mostly idle points straight at this layer. [NEEDS VERIFICATION] for any specific affinity command on Android: standard `taskset` is often unavailable in the Android sandbox, and the correct approach depends on whether your frontend grants shell-level access.

## 13.4 Stage 2: Wine APIs and synchronisation

[Wine](https://gitlab.winehq.org/wine/wine) is an **API compatibility layer**. It implements Windows APIs on the host; it is not a Windows kernel and it does not translate x86 to ARM. [Proton](https://github.com/ValveSoftware/Proton) is Valve's Wine-derived build plus runtime libraries and graphics components, tuned for Steam.

The expensive part for multithreaded games is often **synchronisation**: Windows games use NT synchronisation objects (events, mutexes) heavily. Wine must emulate their exact semantics.

| Mechanism | Status in this book's sources | Practical note |
|---|---|---|
| esync | Obsolete in Proton 11.0 | Do not spend time tuning it on modern Proton |
| fsync | Uses Linux futex; auto-disabled if `FUTEX_WAIT_MULTIPLE` is unavailable | Kernel support matters |
| ntsync | Kernel char device `/dev/ntsync`; complete in mainline Linux 6.14; initial support reported in Wine 10.15, headline feature in Wine 11.0 | Proton 11.0 enables it automatically when the device can be opened |

Check ntsync with `lsof /dev/ntsync` on a desktop Linux host. Proton 11.0 documents only the opt-out `PROTON_NO_NTSYNC`; there is no `PROTON_NTSYNC=1` enable switch in the official docs. Sources: the [kernel ntsync documentation](https://docs.kernel.org/userspace-api/ntsync.html), the [Wine 11.0 announcement](https://www.winehq.org/announce/11.0), the [Debian ntsync HOWTO](https://wiki.debian.org/Wine/NtsyncHowto), and the [Proton 11.0 README](https://github.com/ValveSoftware/Proton/tree/proton_11.0), which notes that the older enable switch is obsolete.

The kernel documentation's key point is conceptual: userspace cannot match Windows-equivalent performance while faithfully emulating these semantics. That is why making ntsync available is a platform-level improvement, not a game setting. On Android, where you have a Linux kernel but a restricted userspace, availability depends on whether the device kernel includes the driver and whether the container can reach the device node — `[NEEDS VERIFICATION]` per device.

## 13.5 Stage 3: Graphics translation cost

The game's graphics API selects the translator:

| Game API | Translator | HUD / profiling hook |
|---|---|---|
| D3D9/10/11 | [DXVK](https://github.com/doitsujin/dxvk) | `DXVK_HUD`, `DXVK_LOG_LEVEL` |
| D3D12 | [VKD3D-Proton](https://github.com/HansKristian-Work/vkd3d-proton) | `VKD3D_PROFILE_PATH`, `VKD3D_DEBUG` |
| D3D8 | [DXVK](https://github.com/doitsujin/dxvk) via `d3d8.dll` + `d3d9.dll` | `DXVK_HUD`; standalone D8VK not required (merged into DXVK 2.4) |
| D3D7 | [D7VK](https://github.com/WinterSnowfall/d7vk) | DXVK-derived HUD |
| OpenGL | native GL, [Zink](https://docs.mesa3d.org/drivers/zink.html), or WineD3D | `ZINK_DEBUG`, Mesa env vars |
| Legacy fixed function | [WineD3D](https://gitlab.winehq.org/wine/wine) | Wine debug channels |

DXVK and VKD3D-Proton are **graphics translators**: they convert D3D calls into Vulkan. They do not execute x86 code and they do not implement Windows APIs.

Useful HUD fields for this layer:

```text
DXVK_HUD=drawcalls,submissions,pipelines,descriptors,compiler
```

- **drawcalls / submissions**: a spike here points at CPU-side translation or game engine overhead, not the GPU.
- **pipelines / compiler**: active compilation means you are watching shader cost, not steady-state rendering.
- `VKD3D_PROFILE_PATH` enables VKD3D-Proton's profiling output in profiling-enabled builds; see its repository documentation for the exact format and analysis tooling.

Version floors matter: DXVK 3.x requires Vulkan 1.4; VKD3D-Proton needs Vulkan 1.3 plus descriptor indexing with at least 1,000,000 UpdateAfterBind descriptors, `VK_EXT_robustness2`, and `VK_KHR_push_descriptor`. If a game fails only on one wrapper build, check the driver's exposed Vulkan version and features first.

## 13.6 Stage 4: The GPU and its driver

The translator emits Vulkan; the **GPU driver** turns Vulkan into hardware work. [Turnip](https://docs.mesa3d.org/drivers/freedreno.html) is Mesa's open-source Vulkan driver for Adreno — a driver, not a translator. Other examples are RADV (AMD), ANV (Intel), PanVK (Mali), V3DV (Raspberry Pi) and vendor proprietary stacks.

On Android, respect the kernel/userspace boundary. Android uses the Linux kernel, but its userspace is Bionic, its display stack is SurfaceFlinger, and GPU vendors ship their own userspace driver (UMD) alongside a kernel driver such as KGSL. An injected Turnip library is a **per-app userspace replacement**, not a new kernel driver, and it does not change what the Android compositor uses for the system UI. This is why the UI can look correct while the game renders incorrectly.

Measurement cautions:

- DXVK's `gpuload` HUD field can be inaccurate; do not make decisions from it alone.
- MangoHud fields such as `gpu_stats`, `gpu_power`, `gpu_temp` and `throttling_status` are more informative where supported, but availability varies by driver.
- GPU at 100% usually means you are finally GPU-bound, which is good news on a translated stack. GPU low and CPU high means look earlier in the pipeline.

## 13.7 Stage 5: Memory and VRAM

VRAM exhaustion behaves like stutter, not like a clean error. DXVK's `memory,allocations` HUD fields and MangoHud's `vram,proc_vram` help separate "game is heavy" from "translator is wasting memory".

Relevant levers from the brief:

| Lever | Effect |
|---|---|
| DXVK 3.x `dxbc-spirv` | Saves around 1 GiB of system memory in some games (e.g. Overwatch, God of War) |
| `VKD3D_CONFIG=no_upload_hvv` | Avoids one upload-heavy path in some D3D12 workloads; may free VRAM at the cost of lower GPU performance |
| `DXVK_SHADER_CACHE_PATH` | Moves the DXVK cache, useful when the prefix disk is small |

Do not confuse system RAM pressure with VRAM pressure. On Android, "memory" also includes pressure from the compositor and other apps; the kernel may reclaim pages and hurt the game even when the game itself is well behaved.

## 13.8 Stage 6: Storage, caches and first-run cost

Shader and pipeline caches live on disk. A cold cache produces minutes of stutter that disappears on the second run. This is expected behaviour, not a bug.

Sources of cache growth include DXVK, VKD3D-Proton, Mesa and Proton crash reports, which the brief notes can fill a disk. Keep an eye on free space before concluding that a driver "got slower" after an update: driver or wrapper updates routinely invalidate caches.

## 13.9 Stage 7: Thermal and power

Thermal limits are the layer people forget because they do not appear in a log. A game can start at 60 FPS and settle at 35 as the SoC heats up. This is **throttling**, not a software regression.

MangoHud can display `cpu_temp`, `gpu_temp`, `throttling_status` and `battery_watt`. On x86, RAPL-based power readings may report 0 W on some hardware — treat that as "unsupported", not "no power used". On Android, thermal state is reported through system services and is strongly device-specific. `[NEEDS VERIFICATION]` for any claim that a specific frontend can disable thermal limits safely; do not advise users to defeat thermal protection.

## 13.10 Measurement toolkit

| Tool | Layer | First command / hook |
|---|---|---|
| DXVK HUD | Graphics + memory + compile | `DXVK_HUD=devinfo,fps,frametimes,drawcalls,pipelines,compiler,memory` |
| MangoHud | Whole-system overlay | `mangohud %command%`; for 1% lows use `MANGOHUD_CONFIG=gpu_stats,cpu_temp,fps_metrics=avg+0.01` (or escape the comma: `fps_metrics=avg\,0.01`) |
| VKD3D-Proton profiling | D3D12 translation | `VKD3D_PROFILE_PATH=/path` |
| Box64 stats | ISA translation | `BOX64_DYNAREC_PERFMAP=1`, `BOX64_DYNAREC_TRACE` |
| FEX stats | ISA translation | `ProfileStats` config key |
| `perf` | Host CPU | `perf record`/`perf report` on Linux; pair with translator maps |
| Android `dumpsys` | Android framework | `adb shell dumpsys gfxinfo <PKG> framestats` |
| Perfetto | Whole-device tracing | https://perfetto.dev/ |
| Simpleperf | Android CPU sampling | https://developer.android.com/ndk/guides/simpleperf |

Official references: [MangoHud](https://github.com/flightlessmango/MangoHud), [Vulkan Loader debugging](https://github.com/KhronosGroup/Vulkan-Loader/blob/main/docs/LoaderDebugging.md), [Android dumpsys](https://developer.android.com/tools/dumpsys), [Android performance measurement](https://developer.android.com/topic/performance/measuring-performance). `VK_LOADER_DEBUG` is a desktop-loader facility and does not work with Android's loader.

A disciplined measurement session records: kernel version, driver version, Wine/Proton version, translator version, wrapper version, resolution, and whether the cache was warm. Without those, two runs are not comparable.

## 13.11 Shader compilation and caches

Modern D3D games do not ship ready-to-run GPU machine code. They ship shaders in a high-level or intermediate form, and the driver compiles them into a pipeline for the current hardware. Compilation is expensive; doing it during gameplay causes hitches.

| Component | Cache / mechanism | Notes |
|---|---|---|
| DXVK 2.x+ | Graphics Pipeline Library (`VK_EXT_graphics_pipeline_library`) | Reduces compile stutter; games that preload shaders in menus still stall. Watch with `DXVK_HUD=compiler` |
| DXVK 3.x | `dxbc-spirv` plus `DXVK_SHADER_CACHE_PATH` | Default cache location is under AppData/Local in the prefix; the legacy state cache was removed in 2.7 |
| VKD3D-Proton | `vkd3d-proton.cache` in the working directory; `VKD3D_SHADER_CACHE_PATH` | `VKD3D_CONFIG=pipeline_library_app_cache` relates to app-managed pipeline caches |
| Mesa | `MESA_SHADER_CACHE_DIR`, `MESA_SHADER_CACHE_MAX_SIZE` (default 1 GiB) | `MESA_DISK_CACHE_SINGLE_FILE=1` uses a Fossilize database, which ignores the size cap |
| Fossilize | `VK_LAYER_fossilize`, `.foz` files, `fossilize-replay` | Records and replays Vulkan pipelines for reproducible warm-up |

[Fossilize](https://github.com/ValveSoftware/Fossilize) is a Vulkan pipeline caching and replay tool from Valve, not a game-level settings panel.

### About `DXVK_ASYNC`

There is **no official `DXVK_ASYNC`** build in the upstream DXVK project. `dxvk-async` was an archived third-party patch; its successor is [dxvk-gplasync](https://gitlab.com/Ph42oN/dxvk-gplasync), another third-party fork (the original `Sporif/dxvk-async` repo is archived). dxvk-gplasync is not upstream, and its project warns about anti-cheat risk. These forks trade correctness for fewer compile stalls and are not safe for online games; changing rendering behaviour in multiplayer can violate anti-cheat rules (see module 15). Prefer warming caches through normal gameplay.

## 13.12 Upscaling and frame generation

**Upscaling** renders at a lower internal resolution and reconstructs a higher-resolution image. **Frame generation** creates additional frames between real ones. They are different technologies with different costs.

| Technology | Type | Requirement / caveat |
|---|---|---|
| FSR1 | Spatial upscaler | Can be applied externally by [gamescope](https://github.com/ValveSoftware/gamescope) with `-F fsr`; also NIS and SGSR |
| FSR2 / FSR3 | Temporal upscalers | Need engine integration; use game settings where available |
| FSR3 frame generation | Frame generation | Increases queued frames and latency; does not fix a CPU-bound translated game |
| AMD FSR Upscaling 4.1.2 (formerly "FSR 4") | ML-based upscaling | RX 9000/7000/6000 and supported Ryzen AI APUs; otherwise [AMD's FSR API](https://gpuopen.com/amd-fsr-sdk/) selects FSR 3.1.6 analytical (the SDK also bundles FSR 2.3.4). Requires engine-integrated signed DLLs and D3D12 or Vulkan 1.2; new APU/Vulkan paths need Adrenalin 26.10.1+. Not a gamescope-style external filter. |
| [vkBasalt](https://github.com/DadSchoorse/vkBasalt) | Vulkan post-process layer | Can apply FSR1/CAS where engine support is absent |

Frame generation is not free performance. It adds latency and can look worse when the underlying frame rate is already unstable — which is common under ISA translation. If your real frame time is 40 ms, generating extra frames does not make the game respond faster. Fix the bottleneck first.

## 13.13 Present modes and pacing

A **present mode** is the rule the driver and compositor use to put a finished frame on screen. It affects latency, tearing and smoothness.

| Knob | Purpose |
|---|---|
| MangoHud `vsync=0..3`, `vulkan_present_mode` | Overlay-driven pacing controls |
| `VKD3D_SWAPCHAIN_PRESENT_MODE` | D3D12 swapchain present mode |
| `MESA_VK_WSI_PRESENT_MODE` | Mesa Vulkan WSI present mode |
| `DXVK_CONFIG="dxgi.syncInterval = 0"` | Disable DXVK's DXGI sync interval |
| gamescope `-r <fps>` | Frame-rate limit inside gamescope |

Present modes do not change how fast the game renders; they change when completed frames are shown. Confusing these two leads to people "fixing" GPU load by changing vsync, which only moves the symptom.

Gamescope is a Wayland compositor from Valve that can run a game in a nested session and apply upscaling, frame limiting and HDR handling. On Android the equivalent role is played by SurfaceFlinger, which you generally cannot reconfigure from inside a translated game.

## 13.14 A worked diagnostic sketch

Suppose a game reports 45 FPS with a 1% low of 18.

1. Open a HUD with frametimes, draw calls, compiler and VRAM.
2. If `compiler` is active and 1% lows recover after a second run, it was shader compilation. Warm the cache.
3. If one ARM core is at 100% and GPU load is low, suspect ISA translation or sync; check translator mode and ntsync availability.
4. If draw calls spike with low GPU use, suspect the graphics translation or engine CPU work.
5. If VRAM is exhausted, reduce textures or move/increase the cache.
6. If performance decays over 20 minutes, check thermal fields before changing any software setting.

Change **one** variable between measurements. Module 14 expands this into a full troubleshooting method.

## 13.15 What to remember

- Measure frame time and 1% lows, not average FPS alone.
- Identify the saturated layer before changing settings; the layers are ISA translation, Wine API/sync, graphics translation, GPU, memory, storage/cache, and thermal.
- FEX/Box64 translate CPU instructions; DXVK/VKD3D translate graphics APIs; Wine/Proton provide Windows API compatibility; Turnip is a GPU driver.
- On ARM hosts, CPU translation and synchronisation are often the ceiling even when the GPU is idle.
- Shader caches are normal: the first run is slow, and driver changes invalidate caches.
- There is no official `DXVK_ASYNC`; async forks are unsafe for online games.
- Frame generation hides latency problems rather than solving CPU-bound translation.
- Respect thermal limits; never "fix" performance by disabling them.

Next: [14 — Troubleshooting](14-troubleshooting.md).

## References

- Wine — https://gitlab.winehq.org/wine/wine ; Wine 11.0 — https://www.winehq.org/announce/11.0
- Proton — https://github.com/ValveSoftware/Proton
- FEX — https://github.com/FEX-Emu/FEX
- Box64 — https://github.com/ptitSeb/box64 ; usage docs — https://github.com/ptitSeb/box64/blob/main/docs/USAGE.md
- Box86 — https://github.com/ptitSeb/box86
- DXVK — https://github.com/doitsujin/dxvk
- VKD3D-Proton — https://github.com/HansKristian-Work/vkd3d-proton
- D8VK — https://github.com/AlpyneDreams/d8vk (merged into DXVK 2.4; standalone build discontinued) ; D7VK — https://github.com/WinterSnowfall/d7vk
- Mesa environment variables — https://docs.mesa3d.org/envvars.html ; Freedreno/Turnip — https://docs.mesa3d.org/drivers/freedreno.html ; Zink — https://docs.mesa3d.org/drivers/zink.html
- ntsync kernel docs — https://docs.kernel.org/userspace-api/ntsync.html ; Debian HOWTO — https://wiki.debian.org/Wine/NtsyncHowto
- MangoHud — https://github.com/flightlessmango/MangoHud
- gamescope — https://github.com/ValveSoftware/gamescope
- Fossilize — https://github.com/ValveSoftware/Fossilize
- Vulkan Loader debugging — https://github.com/KhronosGroup/Vulkan-Loader/blob/main/docs/LoaderDebugging.md
- vkBasalt — https://github.com/DadSchoorse/vkBasalt
- AMD FSR — https://gpuopen.com/amd-fsr-sdk/
- Android dumpsys — https://developer.android.com/tools/dumpsys ; performance measurement — https://developer.android.com/topic/performance/measuring-performance
- Perfetto — https://perfetto.dev/ ; Simpleperf — https://developer.android.com/ndk/guides/simpleperf
- dxvk-gplasync — https://gitlab.com/Ph42oN/dxvk-gplasync
- MangoHud config — https://github.com/flightlessmango/MangoHud/blob/master/data/MangoHud.conf
- AMD FSR upscaling — https://gpuopen.com/amd-fsr-upscaling ; FSR SDK manual — https://gpuopen.com/manuals/fsr_sdk/techniques/super-resolution-ml/
