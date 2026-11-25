# Research Brief — Performance, Troubleshooting, Compatibility & Anti-Cheat (Nov 2026)

> **2026-11-25 correction note.** Chapters 13–15 supersede this brief on verified points. Key corrections: FEX uses `Logging.ProfileStats`/`FEX_PROFILESTATS`, not `FEX_LOG_LEVEL`; `BOX64_DYNACACHE_LIMIT` has the 2048 MiB default; Mesa 26.3.1 was current at the snapshot; D7VK v1.3+ uses `D7VK_` log variables; EAC Linux v2 loads no kernel module; Wine 10.0 required 4K pages but Wine 11.0 simulates 4K on larger-page hosts; Wine 11.0 restored 16-bit support; dxvk-gplasync and RICOCHET details are in chapters 13 and 15. This brief remains a historical research snapshot.

[V]=primary source. "Impossible locally" = impossible inside Wine/Proton/emulator userspace model without vendor-supported Linux component; no bypass guidance.

## 1. Bottleneck analysis

| Layer | Measurement | Knobs |
|---|---|---|
| ISA translation | `BOX64_DYNAREC_TRACE`, `BOX64_DYNAREC_PERFMAP=1`+perf; FEX `ProfileStats` | DynaRec 5–10× faster than interpreter; `BOX64_DYNACACHE` default 2048 MiB |
| Wine API / sync | `WINEDEBUG=+sync,+ntdll,+server`; `lsof /dev/ntsync`; perf | ntsync kernel docs: userspace cannot match Windows performance with accurate semantics |
| Graphics translation | `DXVK_HUD=drawcalls,submissions,pipelines,descriptors,compiler`; `VKD3D_PROFILE_PATH` | DXVK 3.x needs Vulkan 1.4; vkd3d needs 1.3 + descriptor indexing + robustness2 + push_descriptor |
| GPU | DXVK_HUD `gpuload` (may be inaccurate), MangoHud `gpu_stats,gpu_power` | `MESA_VK_TRACE`, `INTEL_MEASURE` |
| Memory/VRAM | DXVK_HUD `memory,allocations`; MangoHud `vram,proc_vram` | DXVK 3.0 dxbc-spirv saves ~1 GiB RAM; `VKD3D_CONFIG=no_upload_hvv` |
| Storage/cache | `MESA_SHADER_CACHE_SHOW_STATS`, `DXVK_SHADER_CACHE_PATH` | Proton crash reports can fill disk |
| Thermal | MangoHud `cpu_temp,gpu_temp,throttling_status,battery_watt` | RAPL may read 0 W |

Rule: decide GPU-bound vs CPU-bound with HUD data, not FPS alone. Android: `adb shell dumpsys gfxinfo <PKG> framestats`, Perfetto, Simpleperf.

## 2. Shader caches

- DXVK 2.x+ Graphics Pipeline Library (`VK_EXT_graphics_pipeline_library`) reduces compile stutter; games loading shaders in menus still stall. `DXVK_HUD=compiler`.
- DXVK 3.x `dxbc-spirv` + `DXVK_SHADER_CACHE_PATH` (default AppData/Local). Legacy state cache removed in 2.7.
- vkd3d-proton cache `vkd3d-proton.cache` in CWD; `VKD3D_SHADER_CACHE_PATH`; `VKD3D_CONFIG=pipeline_library_app_cache`.
- Mesa `MESA_SHADER_CACHE_DIR`, `MESA_SHADER_CACHE_MAX_SIZE` default 1 GiB; `MESA_DISK_CACHE_SINGLE_FILE=1` Fossilize DB.
- Fossilize `VK_LAYER_fossilize`, `.foz` files, `fossilize-replay` for warmer/repro.
- **No official `DXVK_ASYNC`**; dxvk-async archived; successor third-party `dxvk-gplasync` fork. Not safe for online games.
- Clear caches after driver/wrapper changes; first run after clear is intentionally slow.

## 3. Sync and CPU features

- esync obsoleted in Proton 11.0. fsync via futex; auto-disabled without FUTEX_WAIT_MULTIPLE.
- ntsync: kernel char device `/dev/ntsync`, complete in mainline 6.14; Wine 10.15 initial, Wine 11.0 headline; Proton 11.0 automatic when device openable; disable with `PROTON_NO_NTSYNC` (no `PROTON_NTSYNC=1`). Check with `lsof /dev/ntsync`.
- CPU: `BOX64_SKIPCPU` for big.LITTLE, `BOX64_MAXCPU`; performance governor/taskset common practice [U].

## 4. Upscaling / pacing

- gamescope `-F fsr` = FSR1 spatial; also NIS, SGSR. FSR2/3 temporal needs engine integration; FSR4/4.1.2 ML only on RX 9000/7000/6000 + selected Ryzen AI, otherwise falls back to FSR3/2 analytical.
- Frame generation adds latency/queues; does not fix CPU-bound translation.
- Present modes: MangoHud `vsync=0..3` / `vulkan_present_mode`; `VKD3D_SWAPCHAIN_PRESENT_MODE`; `MESA_VK_WSI_PRESENT_MODE`; `DXVK_CONFIG="dxgi.syncInterval = 0"`; gamescope `-r` FPS limit.

## 5. Troubleshooting

Method: change one variable; record kernel/driver/Wine versions; capture 1% lows; grep logs for `err:, warn:, fixme:`.

| Knob | Syntax |
|---|---|
| WINEDEBUG | `[class][+-]channel[,...]`, classes err/warn/fixme/trace; `WINEDEBUG=fixme-all,warn+cursor,+relay`, `WINEDEBUG=-all` |
| DXVK_LOG_LEVEL | none/error/warn/info/debug; DXVK_DEBUG=capture/hang/markers/validation |
| VKD3D_DEBUG | none/err/info/fixme/warn/trace |
| VK_LOADER_DEBUG | error,warn,info,debug,layer,driver,all (desktop only) |
| MESA_DEBUG | silent,flush,incomplete_tex,incomplete_fbo,context |
| TU_DEBUG | sysmem,gmem,nobin,forcebin,noubwc,nolrz,flushall,syncdraw,rast_order,rd; TU_DEBUG_FILE runtime toggle |
| ZINK_DEBUG | nir,spirv,tgsi,validation,sync,...; ZINK_DESCRIPTORS=auto/lazy/db |
| BOX64_LOG | 0..3; BOX64_DYNAREC_LOG; BOX64_TRACE_FILE |
| FEX | config keys `Logging.SilentLog`, `Logging.OutputLog`, `Logging.ProfileStats`; env `FEX_` + uppercase key. `FEX_LOG_LEVEL` NOT in current schema [U] |
| winedbg | `info process/thread/wnd/exception`; GDB mode with Wine symbols |

Symptom → layer: first-run stutter → shaders; high CPU+low GPU Windows-thread-heavy → NT sync; one core 100% on ARM → ISA translation; drawcall spikes → graphics translation; GPU 100% → GPU/VRAM; black screen → ICD/WSI; device lost → driver; audio-only → Wine multimedia; .NET dialog → Mono/Gecko/VC++; Android UI jank → HWUI/compositor.

## 6. Compatibility limits

- EAC: possible, **opt-in per game**; developer must enable Linux client, ship `libeasyanticheat.so` renamed `easyanticheat_x64.so` beside `EasyAntiCheat_x64.dll`, publish build.
- BattlEye: possible, opt-in per title via Valve/BattlEye manual configuration.
- Vanguard/RICO, kernel anti-cheat: **impossible locally** — requires Windows kernel driver; Valve states kernel-space solutions not supported.
- Kernel drivers generally: impossible (no `NtLoadDriver` to ring-0 host).
- SafeDisc: generally impossible (`secdrv.sys`). SecuROM: ring-3 but disc subchannel/DPM auth not implemented. Denuvo: no general Wine implementation; case-by-case.
- Online launchers: Valve recommends skipping separate launchers; .NET/WPF known Proton issue; Media Foundation incomplete (Proton `nomfdxgiman`, Wine 10 FFmpeg backend); MF needs VP9/AV1 preferred.
- 16-bit apps: effectively unsupported in current Wine (exact version UNVERIFIED).
- ARM64: Wine 10 ARM64EC fully supported, FEX implements 64-bit x86 interface; 4K host pages required (16K/64K unsupported). Proton ARM64 builds cannot be used in x86 Steam via FEX.
- DXVK README: manipulating D3D DLLs in multiplayer may be considered cheating and get banned. vkBasalt FAQ: maybe.

## 7. Citations (verified)

- https://github.com/doitsujin/dxvk
- https://github.com/HansKristian-Work/vkd3d-proton
- https://github.com/ValveSoftware/Proton
- https://github.com/flightlessmango/MangoHud
- https://github.com/ValveSoftware/gamescope
- https://github.com/DadSchoorse/vkBasalt
- https://github.com/ValveSoftware/Fossilize
- https://docs.kernel.org/userspace-api/ntsync.html
- https://www.winehq.org/announce/11.0
- https://wiki.debian.org/Wine/NtsyncHowto
- https://github.com/FEX-Emu/FEX
- https://github.com/ptitSeb/box64/blob/main/docs/USAGE.md
- https://developer.android.com/tools/dumpsys
- https://developer.android.com/topic/performance/measuring-performance
- https://docs.mesa3d.org/envvars.html
- https://github.com/KhronosGroup/Vulkan-Loader/blob/main/docs/LoaderDebugging.md
- https://gpuopen.com/amd-fsr-sdk
- https://partner.steamgames.com/doc/steamhardware/proton
- https://onlineservices.epicgames.com/news/introducing-easy-anti-cheat-linux-v2
