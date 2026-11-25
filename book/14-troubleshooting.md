# 14 — Troubleshooting: From Symptom to Layer

Troubleshooting a translated game is an exercise in attribution. The same black screen can come from a frontend, a container, Wine, a graphics translator, a Vulkan driver, or the display compositor. Guessing wastes time. This module gives you a repeatable method: change one variable, capture a log, and point the symptom at a layer.

> **Safety and law.** Use your own legally acquired games. This module covers diagnosis only. It does not help bypass DRM or anti-cheat; see module 15. Some logging options print paths, usernames and environment values — review logs before sharing them publicly.

## 14.1 The method

1. **Write down the baseline.** Kernel version, GPU driver version, Wine/Proton version, translator version (FEX or Box64), wrapper version (DXVK/VKD3D), frontend version, game patch.
2. **Reproduce the problem twice.** Intermittent failures are usually thermal, memory or race-related, not a simple missing DLL.
3. **Record frame time and 1% lows**, not just average FPS. A crash-free game with terrible 1% lows is still a bug worth chasing.
4. **Change one variable.** One Wine version, one environment variable, one driver. Revert if there is no improvement.
5. **Capture a log for every attempt**, even the failures. Name files so you can tell them apart later.
6. **Point at a layer.** Use the table in section 14.8 before reaching for a fix.

Two habits matter more than any single tool: keeping a written experiment log, and refusing to change three things at once.

## 14.2 Log and environment knobs at a glance

These are the most useful knobs. Syntax is shown exactly where the collected sources support it; ambiguous cases are marked.

| Knob | Accepted values / syntax | Layer | Source |
|---|---|---|---|
| `WINEDEBUG` | `[[process:]class][+-]channel[,...]`; classes: `err`, `warn`, `fixme`, `trace` | Wine APIs | Wine |
| `DXVK_LOG_LEVEL` | `none`, `error`, `warn`, `info`, `debug` | D3D9–11 translation | DXVK |
| `DXVK_DEBUG` | `capture`, `hang`, `markers`, `validation` | DXVK internals | DXVK |
| `VKD3D_DEBUG` | `none`, `err`, `info`, `fixme`, `warn`, `trace` | D3D12 translation | VKD3D-Proton |
| `VK_LOADER_DEBUG` | `error,warn,info,debug,layer,driver,all` | Vulkan loader (desktop only) | Khronos |
| `MESA_DEBUG` | `silent`, `flush`, `incomplete_tex`, `incomplete_fbo`, `context` | Mesa GL | Mesa |
| `TU_DEBUG` | Common diagnostic subset (not exhaustive): `sysmem`, `gmem`, `nobin`, `forcebin`, `noubwc`, `nolrz`, `flushall`, `syncdraw`, `rast_order`, `rd` | Turnip Vulkan driver | Mesa |
| `TU_DEBUG_FILE` | Runtime toggle file for `TU_DEBUG` | Turnip | Mesa |
| `ZINK_DEBUG` | `nir`, `spirv`, `tgsi`, `validation`, `sync`, … | Zink GL-on-Vulkan | Mesa |
| `ZINK_DESCRIPTORS` | `auto`, `lazy`, `db` | Zink | Mesa |
| `BOX64_LOG` | `0`–`3` | Box64 CPU translation | Box64 |
| `BOX64_DYNAREC_LOG`, `BOX64_TRACE_FILE` | DynaRec logging and output file | Box64 | Box64 |
| FEX logging | Config keys `Logging.SilentLog`, `Logging.OutputLog`, `Logging.ProfileStats` | FEX CPU translation | FEX |
| `vulkaninfo` | `vulkaninfo --summary`, `VK_LOADER_DEBUG=error,warn vulkaninfo` | Vulkan ICD discovery | Khronos |

First-mention links: [Wine](https://gitlab.winehq.org/wine/wine), [Proton](https://github.com/ValveSoftware/Proton), [DXVK](https://github.com/doitsujin/dxvk), [VKD3D-Proton](https://github.com/HansKristian-Work/vkd3d-proton), [Box64](https://github.com/ptitSeb/box64), [FEX](https://github.com/FEX-Emu/FEX), [Mesa](https://docs.mesa3d.org/), [Turnip](https://docs.mesa3d.org/drivers/freedreno.html), [Zink](https://docs.mesa3d.org/drivers/zink.html), [Vulkan Loader](https://github.com/KhronosGroup/Vulkan-Loader/blob/main/docs/LoaderDebugging.md), [vulkaninfo](https://github.com/KhronosGroup/Vulkan-Tools).

## 14.3 Reading Wine logs

`WINEDEBUG` selects which message classes and channels are printed. The three classes you will use most:

- `err` — something failed.
- `warn` — suspicious, but the program continued.
- `fixme` — Wine knows the behaviour is not fully implemented.

Examples:

```bash
WINEDEBUG=fixme-all,warn+cursor,+relay game.exe   # broad relay trace, noisy
WINEDEBUG=-all game.exe                            # silence everything
```

`+relay` logs every DLL call and is enormous; use it for seconds, not minutes. `WINEDEBUG=-all` is for performance testing, never for diagnosis.

A common beginner mistake is to read `fixme:` lines as errors. Many games print hundreds of harmless ones. Prioritise `err:` lines near the moment of failure, then look at surrounding `warn:` lines.

## 14.4 Reading graphics translator logs

DXVK's `DXVK_LOG_LEVEL=debug` plus `DXVK_DEBUG` modes are your first stop for D3D9/10/11 failures. `validation` surfaces API misuse, `hang` helps when the game freezes, `capture` and `markers` are for deeper tracing.

For D3D12, use `VKD3D_DEBUG=warn` or `VKD3D_DEBUG=trace`. `VKD3D_PROFILE_PATH` writes profiling data only in a VKD3D-Proton build configured with `-Denable_profiling=true` (as noted in module 13); a stock build ignores it.

If the game crashes at graphics initialisation, check the driver side too: a translator may report a Vulkan error that is actually a driver bug exposed by a specific feature.

## 14.5 Reading the Vulkan loader and driver

The [Vulkan loader](https://github.com/KhronosGroup/Vulkan-Loader/blob/main/docs/LoaderDebugging.md) discovers **ICDs** (Installable Client Drivers) through JSON manifests. If the wrong driver is being loaded, no amount of game tweaking helps.

```bash
vulkaninfo --summary | grep driverName
VK_LOADER_DEBUG=error,warn vulkaninfo
```

Important limitation: `VK_LOADER_DEBUG` is a **desktop** debugging facility. On Android the loader is part of the platform and environment-variable debugging is not available in the same way; per-app driver selection is done through injection instead. `[NEEDS VERIFICATION]` for any specific Android loader-debug command — the collected sources do not document one.

Mesa-side knobs are covered in the [Mesa environment variable reference](https://docs.mesa3d.org/envvars.html). Two driver-specific examples:

- `TU_DEBUG=sysmem,nolrz,flushall` is a documented example for narrowing down suspected Turnip GMEM/LRZ device-lost cases; treat it as a diagnostic experiment, not a recommendation.
- `ZINK_DEBUG=validation` dumps Vulkan validation-layer output; Zink requests `VK_LAYER_KHRONOS_validation` itself when that layer is installed and discoverable. `ZINK_DESCRIPTORS=db` changes descriptor handling.
- D7VK v1.3+ uses `D7VK_LOG_LEVEL` and `D7VK_LOG_PATH`; v1.2 and earlier still use the DXVK variables. Check the bundled D7VK version before choosing a log knob.

On Android, Mesa maps environment-variable names to system properties in the order `debug.*`, `vendor.*`, then the bare name; `debug.*` works without root, while `vendor.*` requires root ([Mesa Android System Properties](https://docs.mesa3d.org/android.html)). The exact property name for a given option is inferred from that generic mapping. See module 07 for driver selection details.

## 14.6 Reading CPU translator logs

Box64 logging:

```bash
BOX64_LOG=2 BOX64_DYNAREC_LOG=1 BOX64_TRACE_FILE=/tmp/box64.log game.exe
BOX64_DYNAREC_PERFMAP=1 game.exe    # emit a map for perf
```

Use these when a crash only happens under translation, or when one core is pegged. `BOX64_LOG` levels run 0–3; higher is noisier.

FEX uses a configuration model rather than a single log-level variable. The documented keys include `Logging.SilentLog`, `Logging.OutputLog` and `Logging.ProfileStats`; each becomes an environment override of the form `FEX_` plus the uppercase key (for example `FEX_PROFILESTATS=1`). `FEX_LOG_LEVEL` is not a generated env var in the current config schema; check the [FEX repository](https://github.com/FEX-Emu/FEX) documentation for your build rather than copying that variable from forum posts.

Remember what these tools do: Box64 and FEX translate **CPU instructions**. They cannot tell you why a shader failed to compile.

## 14.7 Debuggers

[winedbg](https://wiki.winehq.org/Winedbg) is Wine's debugger. Useful commands include `info process`, `info thread`, `info wnd` and `info exception`. With Wine symbols installed, Wine's GDB proxy mode (`winedbg --gdb`) is useful when you need to inspect a native crash.

For driver-level or host crashes, standard Linux tools apply: `gdb`, `perf`, `dmesg` and `strace`. On Android, use `adb logcat`, tombstone files and kernel logs where accessible.

Debugging across layers means knowing which debugger sees what:

| Problem | Right tool |
|---|---|
| Wine DLL throws an exception | `winedbg` |
| Translator crashes on an instruction | Box64/FEX logging, host `gdb` |
| Vulkan validation error | `VK_LOADER_DEBUG`, wrapper `validation` modes |
| Driver hang or device lost | `dmesg`, driver debug flags |
| Android app process death | `logcat`, tombstone |

## 14.8 Symptom-to-layer table

| Symptom | Most likely layer | First check |
|---|---|---|
| Nothing happens on launch | Frontend, container, Wine loader | Frontend logs, mounts, prefix |
| `Bad EXE format` | Architecture mismatch or 16-bit installer | File architecture, 32/64-bit Wine |
| Wine error about missing DLL | Windows API / dependency layer | DLL overrides, VC++ / .NET / DirectX runtime |
| Game starts, then crashes at graphics init | Graphics translator or driver | DXVK/VKD3D logs, Vulkan device creation |
| First-run stutter, gone on second run | Shader compilation | Warm the cache; see module 13 |
| High CPU, low GPU, many Windows threads | NT synchronisation | ntsync availability, sync settings |
| One ARM core at 100%, GPU idle | ISA translation | Translator mode, DynaRec, affinity |
| Draw-call spikes, low GPU use | Graphics translation or engine CPU work | `DXVK_HUD=drawcalls` |
| GPU at 100% | GPU / VRAM | Resolution, effects, VRAM HUD fields |
| Black screen, audio works | Presentation / WSI / ICD swapchain | Present mode, fullscreen, compositor |
| Device lost | GPU driver | Driver logs, `dmesg`, `TU_DEBUG` options |
| No sound | Wine multimedia | Audio server, Wine audio driver, dependencies |
| .NET installation dialog | Mono / Gecko / VC++ runtime | Prefix components |
| Android UI jank but game fine | HWUI / SurfaceFlinger compositor | Android performance tools, not game settings |
| Crash after minutes | Thermal, memory or driver instability | Temps, RAM/VRAM, logs |

Use this table as a starting hypothesis, never as a final verdict.

## 14.9 Crash triage workflow

1. **Capture the full log from launch to crash.** Enable one verbose channel set at a time.
2. **Find the last successful operation.** The line before the first `err:` often identifies the failing API.
3. **Identify the layer.** Use section 14.8.
4. **Reduce the case.** Lower resolution, disable effects, try a different wrapper, try a different Wine build — one at a time.
5. **Check for known incompatibilities.** Anti-cheat and DRM are common causes and are covered in module 15.
6. **If reproducible, report upstream** with versions, logs and a minimal description.

Do not report a bug to the DXVK project when the log shows a Wine loader failure, and vice versa. Layer attribution saves maintainers time.

## 14.10 Stale caches and driver rollback

After a driver, Wine or wrapper update, behaviour can change because old caches are invalid. Symptoms include new artifacts, new stutter and occasional crashes. Clear the relevant caches and accept a slow first run. Cache locations are listed in module 13: DXVK, VKD3D-Proton, Mesa/Fossilize and Proton crash reports.

Driver rollback is legitimate diagnosis. If a game worked on driver version A and fails on version B, reinstalling A confirms the regression. Prefer a consistent driver userspace stack (loader, ICD and Mesa libraries) from one version; mixing versions can cause hard-to-diagnose failures. On Android, per-app Turnip injection is safer than replacing the system driver used by SurfaceFlinger — a bad system-wide driver can break the whole UI.

## 14.11 Android-specific diagnostics

Android shares the Linux kernel with desktop Linux but has a different userspace: Bionic libc, scoped storage, the Android linker namespace model, and SurfaceFlinger composition. That changes which diagnostic tools exist.

| Tool | What it gives you |
|---|---|
| `adb logcat` | System and app logs, including native crashes |
| `adb shell dumpsys gfxinfo <PKG> framestats` | Frame timing data for a package |
| Perfetto | Whole-device tracing, including CPU and graphics |
| Simpleperf | CPU sampling of native code |
| Tombstone files | Native process crash backtraces |
| Kernel logs (`dmesg`, where readable) | Driver and OOM messages |
| [Android GPU Inspector (AGI)](https://developer.android.com/agi) | GPU counter and frame profiling where supported (Adreno, Mali, PowerVR); device validation required; Android 12+ expected; no emulators |

Specific Android failure patterns:

- **App dies immediately**: check `logcat` for linker errors — a missing `.so` or a namespace restriction is common in bundled RootFS setups.
- **UI fine, game black**: the injected game driver differs from the one SurfaceFlinger uses. This is a WSI/driver-boundary issue, not a missing texture.
- **Stutter only after minutes**: thermal throttling or memory pressure; Perfetto is the right tool.
- **Permissions denied on shared storage**: Android scoped storage, not Wine. Frontends that request broad storage access change this behaviour.
- **Vulkan device creation fails**: confirm which driver the app actually loaded. There is no desktop-style `VK_LOADER_DEBUG` here; per-app injection configuration is the control point.

## 14.12 A worked example

Symptom: D3D12 game launches, shows a menu, then the device is lost on entering gameplay.

1. Record versions (example values): kernel 6.14, Mesa/Turnip build, VKD3D-Proton 3.1, Proton 11.0.
2. Run with `VKD3D_DEBUG=warn` and capture stderr to a file.
3. Check the Android log or `dmesg` for driver messages at the same timestamp.
4. Try `vulkaninfo --summary` (desktop) or verify injected driver selection (Android) to confirm which ICD is active.
5. Clear the shader cache and retest; a corrupt cache can mimic a driver bug.
6. If still failing, try a different driver version and a different VKD3D-Proton version separately.
7. Report upstream with the exact failing operation and all versions.

Notice that no step changes game graphics settings. The failure is at an API/driver boundary until proven otherwise.

## 14.13 What to remember

- Change one variable per experiment and keep a version record.
- `err:` lines matter most; `fixme:` lines are usually noise.
- `WINEDEBUG`, wrapper logs, `VK_LOADER_DEBUG`, Mesa driver flags and translator logs each cover a different layer.
- `VK_LOADER_DEBUG` is a desktop facility; Android needs per-app driver verification instead.
- Box64 and FEX log CPU translation, not graphics.
- Use 1% lows and frame times to distinguish stutter from sustained slowdown.
- Clear caches after driver or wrapper changes; roll back drivers to confirm regressions.
- On Android, start with `logcat`, `dumpsys`, Perfetto and tombstones before touching game settings.

Next: [15 — Compatibility, Online Games and DRM](15-compatibility-online-drm.md).

## References

- Wine — https://gitlab.winehq.org/wine/wine ; winedbg — https://wiki.winehq.org/Winedbg
- Proton — https://github.com/ValveSoftware/Proton
- DXVK — https://github.com/doitsujin/dxvk
- VKD3D-Proton — https://github.com/HansKristian-Work/vkd3d-proton
- Box64 — https://github.com/ptitSeb/box64 ; usage docs — https://github.com/ptitSeb/box64/blob/main/docs/USAGE.md
- FEX — https://github.com/FEX-Emu/FEX
- Mesa environment variables — https://docs.mesa3d.org/envvars.html ; Zink — https://docs.mesa3d.org/drivers/zink.html ; Freedreno/Turnip — https://docs.mesa3d.org/drivers/freedreno.html
- Vulkan Loader debugging — https://github.com/KhronosGroup/Vulkan-Loader/blob/main/docs/LoaderDebugging.md ; Vulkan Tools (`vulkaninfo`) — https://github.com/KhronosGroup/Vulkan-Tools
- Android `logcat` — https://developer.android.com/tools/logcat ; `dumpsys` — https://developer.android.com/tools/dumpsys ; `adb` — https://developer.android.com/tools/adb
- Perfetto — https://perfetto.dev/ ; Simpleperf — https://developer.android.com/ndk/guides/simpleperf
- Mesa Android System Properties — https://docs.mesa3d.org/android.html
- Android GPU Inspector — https://developer.android.com/agi ; supported devices — https://developer.android.com/agi/supported-devices
