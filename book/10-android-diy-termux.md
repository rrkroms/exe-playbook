# 10 — Android DIY: The Termux Pipeline

A frontend ([module 09](09-android-frontends.md)) hides the seven layers behind a settings UI. Building the pipeline yourself means assembling those same layers by hand, one command at a time. You gain control over exact component versions, driver selection and environment variables — and you inherit every failure mode that the frontend used to hide.

This module builds a **non-root** stack on an ARM64 Android device: [Termux](https://termux.dev/) as the host shell, a [proot-distro](https://github.com/termux/proot-distro) Linux userspace, Wine with either a CPU translator or an ARM64EC bridge, Mesa [Turnip](https://docs.mesa3d.org/drivers/freedreno.html) as the Vulkan driver, [Termux:X11](https://github.com/termux/termux-x11) for display, and PulseAudio for sound. The reference device is a Snapdragon 8+ Gen 1 / Adreno 730 / Android 13 tablet (preface). Every command below is run as a normal Android app user; no `su` is involved.

## 10.1 The pipeline you are assembling

```text
Android kernel (/dev/kgsl-3d0, Binder, SurfaceFlinger)   <-- host boundary
  |
  +-- Termux app (Bionic libc, apt packages)
  |     +-- PulseAudio server, Termux:X11 server, Vulkan loader + Turnip ICD
  |
  +-- proot-distro guest (Debian/Ubuntu userspace, glibc)
        +-- Wine prefix (Windows APIs, drive letters, registry)
              +-- game .exe (x86 / x86-64 / 32-bit)
                    |
                    +-- CPU path: FEX or Box64/Box86  (if guest is x86)
                    +-- GFX path: DXVK / VKD3D-Proton / D8VK / WineD3D
                          -> Vulkan (Turnip) or OpenGL (Freedreno/Zink)
                                -> X11 socket -> SurfaceFlinger -> screen
```

This is the same logical pipeline as [module 02 §2.10 Variant B](02-pipeline.md), drawn as a build order. It is a *conceptual* dependency map: the X server, audio server and GPU driver are separate host processes, and the translator and graphics translator run alongside the game rather than in strict sequence.

The Android kernel/userspace boundary matters here. Android provides the Linux kernel and the GPU device node, but its userspace is Bionic, not glibc, and its display stack is SurfaceFlinger, not X11. Termux runs on the Android side of that boundary; proot-distro adds a glibc userspace on top. proot is a `ptrace`-based fake container, not a kernel namespace sandbox ([module 08](08-runtime-prefixes.md)).

| Layer | Component you install | Built earlier in |
|---|---|---|
| Windows API compatibility | Wine (or Hangover Wine) | module 03 |
| ABI bridge / CPU translation | FEX + Hangover DLLs, or Box64/Box86 | module 04 |
| Graphics translation | DXVK, VKD3D-Proton, D8VK/D7VK, WineD3D | modules 05–06 |
| GPU driver | Mesa Turnip / Freedreno / Zink | module 07 |
| RootFS + prefix | proot-distro + Wine prefix | module 08 |

## 10.2 Stage 0 — Install Termux from a trusted source

Install Termux from [F-Droid](https://f-droid.org/en/packages/com.termux/) ([F-Droid itself](https://f-droid.org/)), not from the Play Store (a limited, abandoned fork) and preferably not from a [GitHub release](https://github.com/termux/termux-app/releases): GitHub APKs are signed with a public **test key**, so anyone can sign an APK with the same key. Termux and its plugins share a `sharedUserId`; mixing F-Droid, GitHub and Play Store builds breaks the whole installation and can let an impostor app read Termux's data. Pick one source and stay with it.

Keep plugins from the same source too. You will need [Termux:X11](https://github.com/termux/termux-x11), [termux-api](https://github.com/termux/termux-api) and the packages from [termux-packages](https://github.com/termux/termux-packages). Install the Termux:X11 *companion app* from the same source as Termux, or its `sharedUserId` will not match.

## 10.3 Stage 1 — Keep Android from killing your session

Android 12+ monitors apps that spawn many child processes and kills them to save power. A Wine session (`wineserver` plus game processes) matches that pattern exactly, so the game can die after minutes with no error of its own ([module 08 §8.10](08-runtime-prefixes.md)).

Mitigations, least to most invasive. The `adb` commands use the [Android Debug Bridge](https://developer.android.com/tools/adb) from a computer or from Termux's `android-tools` package.

1. Android 14+: Developer options → **Disable child process restrictions**.
2. Android 12L/13 without root:
   ```sh
   adb shell "settings put global settings_enable_monitor_phantom_procs false"
   ```
3. Android 12 older count limit (order matters):
   ```sh
   adb shell device_config set_sync_disabled_for_tests persistent
   adb shell device_config put activity_manager max_phantom_processes 2147483647
   ```

[The AOSP keys exist, but OEM builds may hide or rename the developer option; the adb/device_config commands are community-documented, not officially supported. Verify with `adb shell settings list global`.]

Then, in Termux:

```sh
termux-wake-lock          # CPU on; does NOT keep the screen on
```

Also disable battery optimisation for Termux and Termux:X11 in Android Settings. Keep Termux:X11 in the same app family so off-screen rendering is not throttled differently from the shell.

## 10.4 Stage 2 — Host packages and storage

Run this once in Termux (the host side, before entering proot):

```sh
pkg update && pkg upgrade
pkg install proot-distro termux-api android-tools
termux-setup-storage
termux-wake-lock
pkg install x11-repo
pkg i termux-x11-nightly
pkg install vulkan-loader-generic vulkan-tools
pkg install mesa-vulkan-icd-freedreno
```

What each provides:

| Package | Purpose |
|---|---|
| `proot-distro` | Installs and launches a glibc Linux userspace |
| `termux-api` / `android-tools` | Android API helpers and `adb`/`settings` from the device |
| `x11-repo` → `termux-x11-nightly` | X11 server that presents into Android |
| `vulkan-loader-generic` + `vulkan-tools` | `libvulkan.so.1` and `vulkaninfo` |
| `mesa-vulkan-icd-freedreno` | Turnip `libvulkan_freedreno.so` plus its ICD JSON manifest |

`termux-setup-storage` grants access to public storage via `~/storage`; keep the RootFS and Wine prefixes under `$HOME` (app-private storage) because Android's scoped storage makes other locations awkward. On a device without Adreno, `mesa-vulkan-icd-freedreno` is the wrong driver — substitute your GPU's Mesa driver (module 07).

## 10.5 Stage 3 — Display and audio

**Display.** [Termux:X11](https://github.com/termux/termux-x11) is the recommended X server because it can use GPU acceleration through the Android side. Start it, then point Wine at it:

```sh
termux-x11 :1 -xstartup "xfce4-session" &
export DISPLAY=:1
```

`xfce4-session` is the session manager of the [XFCE](https://xfce.org/) desktop; any lightweight X session (or none) works.

You can omit `-xstartup` and let Wine create windows on an empty root window. [XServer XSDL](https://github.com/pelya/xserver-xsdl) is an older alternative but has no GL acceleration, so it is unsuitable for translated 3D.

**Audio.** Wine implements Windows audio APIs (XAudio2, DirectSound, WASAPI) and forwards them to a host audio server ([module 08 §8.7](08-runtime-prefixes.md)). In Termux that server is the Termux-patched [PulseAudio](https://www.freedesktop.org/wiki/Software/PulseAudio/) with AAudio/OpenSL ES sinks, started in TCP mode on the host; [PipeWire](https://pipewire.org/) is packaged but `pipewire-pulse` is not a drop-in replacement (no Android sinks; open upstream issue). Verify sinks with `pactl list short sinks`. Inside the guest, set:

```sh
export PULSE_SERVER=127.0.0.1
```

Wine then connects to the PulseAudio server running in Termux over localhost. If you hear nothing, verify the server process is alive before changing Wine's audio driver.

## 10.6 Stage 4 — The Linux userspace (RootFS)

Current proot-distro installs [Docker](https://www.docker.com/)/OCI images:

```sh
proot-distro install debian:bookworm --architecture aarch64
# or
proot-distro install ubuntu:24.04

proot-distro login debian --shared-tmp --shared-x11
```

`--shared-tmp` and `--shared-x11` expose the host's `/tmp` and X11 socket to the guest — required for `DISPLAY=:1` to work. Inside the guest, set the environment:

```sh
export PULSE_SERVER=127.0.0.1 container=proot-distro DISPLAY=:1
```

proot limitations to expect (from module 08 and the proot-distro project): ptrace interception slows filesystem-heavy work, there is no real root, no cgroups or namespaces and no init system, and values such as `/proc/stat` are synthetic. Tools that detect containers may refuse to run. Cross-architecture guest images can be run through [QEMU](https://www.qemu.org/) user mode, but that adds a second translation layer on top of Box64/FEX — avoid it unless you have no alternative.

## 10.7 Stage 5 — Choose a Wine architecture strategy

This is the most important decision in the DIY build. The table compares what actually gets translated.

| Approach | What is translated | What runs natively | Notes |
|---|---|---|---|
| **ARM64 Wine + Hangover FEX/Box64 DLLs** | Only the Windows game's x86/x86-64 code | Wine, wineserver and Unix libraries | Recommended where available. Uses `libarm64ecfex.dll`, `libwow64fex.dll` or `wowbox64.dll` as ABI bridges. |
| **x86-64 Wine under Box64** | The entire Wine process plus the game | Nothing in the guest | Documented fallback; [Kron4ek](https://github.com/Kron4ek/Wine-Builds) x86-64 builds are commonly used. More overhead. |
| **32-bit x86 Wine under Box86/Box32** | 32-bit Wine and game | Host libraries | Box32 support is experimental (armhf or `BOX32=ON`). |
| **x86 RootFS under QEMU** | Everything, including the kernel-visible ABI | Only the host kernel | Slowest; last resort. |

**Why ARM64 Wine is different.** Wine is an API-compatibility layer: it implements Windows APIs but does not itself translate x86 instructions to ARM64. A plain ARM64 Wine cannot run an x86 `.exe` on its own. [Hangover](https://github.com/AndreRH/hangover) supplies DLLs that exploit Microsoft's [ARM64EC](https://learn.microsoft.com/en-us/windows/arm/arm64ec) and [WoW64](https://learn.microsoft.com/en-us/windows/win32/winprog64/running-32-bit-applications) ABIs: Wine and most DLLs execute as native ARM64, while only the game's incompatible code is handed to FEX or Box64. This is an **ABI bridge plus CPU translation**, not Windows API emulation — keep those roles separate in your head.

Termux publishes `hangover-wine` in its `x11-repo`, built against Termux's Bionic prefixes. There is no official `box64` package in Termux, so Box64 must be built yourself (§10.8). The reference launch path uses Hangover Wine; the Box64 path is the fallback if a game fails under FEX.

**Namespace caveat:** Termux's `hangover-wine` is a Termux-host package, not a Debian/Ubuntu glibc guest package. The staged tests in §10.12 must either run from the Termux host with a prefix in a shared location, or use a Wine build installed inside the glibc guest. Do not assume the Termux binary can simply be invoked after `proot-distro login`.

## 10.8 Stage 6 — Install the CPU translator

### FEX (for Hangover / ARM64EC)

FEX needs a 32-bit-compatible userspace rootfs on AArch64. The upstream installer below pipes a remote script into Python — a supply-chain risk; download and read it before running it, and prefer a signed distribution package where available. Then fetch the rootfs:

```sh
curl --silent https://raw.githubusercontent.com/FEX-Emu/FEX/main/Scripts/InstallFEX.py | python3
FEXRootFSFetcher
```

FEX stores its rootfs under `$HOME/.fex-emu/RootFS/` and reads configuration from `$HOME/.fex-emu/Config.json` ([FEX configuration wiki](https://wiki.fex-emu.com/index.php/Development:Configuration)). Set `FEX_ROOTFS` if you use a non-default location.

[Piping a remote script into `python3` is a supply-chain risk; the [FEX README](https://github.com/FEX-Emu/FEX/blob/main/Readme.md) itself shows only the pipe-to-python command. Download and read `InstallFEX.py` before running it: use `curl -O <URL>`, inspect it, then run `python3 InstallFEX.py`.]

### Box64 (fallback, and for x86-64 Wine)

Build Box64 inside the proot guest (Android 9+ build tools, CMake):

```sh
git clone https://github.com/ptitSeb/box64
cd box64 && mkdir build && cd build
cmake .. -D ARM64=1 -DBAD_SIGNAL=ON -D ARM_DYNAREC=ON
make -j$(nproc) && make install
```

Useful build flags from Box64's [Wine documentation](https://github.com/ptitSeb/box64/blob/main/docs/WINE.md):

| Flag | Effect |
|---|---|
| `-DBOX32=ON` | Experimental 32-bit support (Box32) |
| `-DWOW64=ON` | Build the WOW64 bridge DLL (`wowbox64.dll`) |
| `-D ARM_DYNAREC=ON` | Enable the dynamic recompiler; without it, translation is interpretive and much slower |
| `-DBAD_SIGNAL=ON` | Signal-handling workaround used in this environment |

[Box86](https://github.com/ptitSeb/box86) is the 32-bit counterpart for older games; its build is analogous. Box64 is the primary translator for 64-bit titles.

## 10.9 Stage 7 — GPU driver: Turnip and the Vulkan loader

[Mesa](https://gitlab.freedesktop.org/mesa/mesa)'s Turnip driver ships in Termux as `mesa-vulkan-icd-freedreno`: it provides `libvulkan_freedreno.so` and an ICD JSON manifest. The exact path is under Termux's `$PREFIX`, for example `$PREFIX/share/vulkan/icd.d/freedreno_icd.aarch64.json`. The Vulkan loader and the ICD manifest must be visible in the **same filesystem namespace**: if you run the loader from a proot guest, install an ICD inside that guest or point `VK_DRIVER_FILES` at the Termux-prefix path as mounted into proot. The example below uses a generic absolute path:

```sh
export VK_DRIVER_FILES=/absolute/path/to/freedreno_icd.aarch64.json
export VK_LOADER_DEBUG=error,warn      # optional: diagnose discovery failures
vulkaninfo --summary
```

Use an **absolute** path. `VK_ICD_FILENAMES` is a deprecated alias; `VK_ADD_DRIVER_FILES` is additive. Verify the output names your Adreno GPU and the Turnip driver — a failure here means every later graphics test is meaningless.

**Why `VK_DRIVER_FILES` works here but not in a native Android app.** Inside Termux/proot, Wine is a Linux ELF process using the desktop [Vulkan loader](https://github.com/KhronosGroup/Vulkan-Loader) ([loader docs](https://docs.vulkan.org/guide/latest/loader.html)), which discovers drivers through ICD manifests. A native Android app instead uses Android's platform Vulkan library and `VK_KHR_android_surface`, and gets its driver from the system; ICD-file overrides do not apply there — per-app injection is the route (module 07). This distinction is why a frontend's Turnip picker and your Termux `VK_DRIVER_FILES` are different mechanisms.

Turnip supports Adreno 6xx/7xx/8xx (Vulkan 1.3 on A6xx, 1.4 on A7xx; no A5xx). [Freedreno](https://docs.mesa3d.org/drivers/freedreno.html) supplies OpenGL/OpenGL ES; [Zink](https://docs.mesa3d.org/drivers/zink.html) implements OpenGL on top of Vulkan. Mesa documents Freedreno at desktop OpenGL 4.5 / GLES 3.2; Mesa core implements the 4.6 API but per-driver support varies — check `glxinfo` on your build. Keep the stock Qualcomm driver for Android's UI and use Turnip only in this guest; `/dev/kgsl-3d0` must be accessible, but no root is required. Wrapping the extracted Qualcomm system driver as an ICD has been reported to fail — use the generic loader plus Turnip. Direct3D 12 through VKD3D-Proton is the least reliable tier on this stack.

For Adreno 730 specifics — stable Mesa builds, plain “A6xx/A7xx Standard” variants, and the “do not apply the A740 UBWC hint” rule — see [module 09 §9.5](09-android-frontends.md).

## 10.10 Stage 8 — Graphics translators

### DXVK (D3D8/9/10/11 → Vulkan)

Install [DXVK](https://github.com/doitsujin/dxvk) into the prefix: copy the x64 DLLs into `system32` and the x32 DLLs into `syswow64`, then set native overrides in `winecfg` (Libraries tab) or with `WINEDLLOVERRIDES`:

```sh
export WINEDLLOVERRIDES="d3d8=n,d3d9=n,d3d10core=n,d3d11=n,dxgi=n"
export DXVK_HUD=devinfo,fps
export DXVK_LOG_LEVEL=warn
```

The `devinfo` HUD should report Turnip — this is your end-to-end confirmation that DXVK is reaching the driver. Override only the DLLs the game and translator actually use; adding overrides “just in case” is a common self-inflicted failure ([module 08 §8.5](08-runtime-prefixes.md)).

### VKD3D-Proton (D3D12 → Vulkan)

Build [VKD3D-Proton](https://github.com/HansKristian-Work/vkd3d-proton) from source:

```sh
./package-release.sh master /target --no-package
```

It needs DXVK 2.1+ DXGI alongside it; D3D12 is the least mature path on Adreno, so test D3D9/D3D11 first.

### Legacy D3D7/8/9 (module 06)

Not every old game fits DXVK:

| Game API | Tool in this stack |
|---|---|
| Direct3D 9 | DXVK, or WineD3D as an OpenGL fallback |
| Direct3D 8 | [D8VK](https://github.com/AlpyneDreams/d8vk) (standalone discontinued and merged into DXVK — module 05), or [D3D8to9](https://github.com/crosire/d3d8to9) + DXVK, or WineD3D |
| Direct3D 7 / DirectDraw | [D7VK](https://github.com/WinterSnowfall/d7vk), WineD3D, or a DirectDraw wrapper |
| OpenGL (Windows game) | Freedreno native GL, or Zink |

[WineD3D](https://gitlab.winehq.org/wine/wine) is Wine's built-in Direct3D-to-OpenGL implementation; it is the fallback when a Vulkan path is unavailable, though it is generally older and slower than DXVK. [dgVoodoo2](https://www.dege.freeweb.hu/dgVoodoo2/) is **not** part of this pipeline — its author disclaims Wine/Linux support, so do not copy Windows dgVoodoo2 recipes into a Termux prefix.

## 10.11 Environment variables reference

| Variable | Meaning |
|---|---|
| `DISPLAY=:1` | X11 display served by Termux:X11 |
| `PULSE_SERVER=127.0.0.1` | PulseAudio server address inside proot |
| `WINEPREFIX` | Absolute path to the Wine prefix |
| `WINEARCH=win32` | Deprecated and unsupported under Wine 11.0's new WoW64; use only with older Wine or a legacy wrapper needing a pure 32-bit prefix |
| `WINEDLLOVERRIDES` | Built-in vs native DLL choice, e.g. `d3d9=n,dxgi=n` |
| `VK_DRIVER_FILES` | Absolute path to the Turnip ICD JSON manifest |
| `VK_LOADER_DEBUG=all` | Verbose Vulkan loader diagnostics |
| `DXVK_HUD=devinfo,fps` | On-screen driver/FPS overlay |
| `DXVK_LOG_LEVEL` | DXVK logging level (`warn`, `info`, `debug`) |
| `TU_DEBUG=sysmem`, `TU_DEBUG=gmem` | Turnip render-path selection |
| `MESA_VK_WSI_DEBUG=sw`, `ZINK_DESCRIPTORS` | Mesa/Zink debugging knobs |
| `BOX64_DYNAREC=1`, `BOX64_DYNAREC_BIGBLOCK=3`, `BOX64_DYNAREC_STRONGMEM=1` | Box64 recompiler settings |
| `BOX64_SKIPCPU`, `BOX64_MAXCPU` | Restrict Box64 to specific CPU cores (big.LITTLE) |
| `FEX_ROOTFS`, `FEX_THUNK*` | FEX rootfs path and thunk configuration |
| `HODLL64=libarm64ecfex.dll`, `HODLL=libwow64fex.dll` | Hangover ABI-bridge DLLs for 64-bit and 32-bit games |

## 10.12 Minimal test path — prove each layer before launching a game

Work in this order; stop at the first failure and fix that layer.

**Stage 0 — host ready.** Termux from F-Droid, phantom-process mitigations applied, `termux-wake-lock` held, Termux:X11 companion installed, `vulkaninfo` available.

**Stage 1 — driver visible.**

```sh
vulkaninfo --summary | grep -i -E 'adreno|driverName|turnip'
```

You should see your Adreno GPU and the Turnip driver. If not, recheck `VK_DRIVER_FILES` on an absolute path.

**Stage 2 — display and guest.**

```sh
termux-x11 :1 &
proot-distro login debian --shared-tmp --shared-x11
export DISPLAY=:1 PULSE_SERVER=127.0.0.1
```

**Stage 3 — a 64-bit console EXE.** No graphics yet — this isolates CPU translation and Wine:

```sh
HODLL64=libarm64ecfex.dll hangover-wine hello64.exe
# fallback: box64 wine64 hello64.exe
```

**Stage 4 — a 32-bit EXE** (tests the WoW64 bridge):

```sh
HODLL=libwow64fex.dll hangover-wine hello32.exe
```

**Stage 5 — a D3D9 test** (tests graphics translation and driver):

```sh
# DXVK x64 DLLs in system32, x32 in syswow64 (§10.10)
WINEDLLOVERRIDES="d3d9=n,dxgi=n" DXVK_HUD=devinfo,fps wine d3d9-test.exe
```

The HUD should show Turnip as the device. A black screen with working audio usually means presentation/swapchain, not driver compilation.

**Stage 6 — a D3D11 test, then D3D12:**

```sh
WINEDLLOVERRIDES="d3d11=n,dxgi=n" wine d3d11-test.exe
# D3D12 uses VKD3D-Proton (§10.10); expect the weakest results
```

## 10.13 Performance caveats

| Constraint | Effect | Mitigation |
|---|---|---|
| proot `ptrace` interception | Slows filesystem-heavy work; no real cgroups/namespaces/init | Keep assets under `$HOME`; expect lower I/O than native |
| Phantom process killer | Game dies after minutes | §10.3 mitigations |
| big.LITTLE scheduling | Translator threads land on slow cores | `BOX64_SKIPCPU` / `BOX64_MAXCPU` to restrict cores, or set affinity |
| Thermal limits | Sustained FPS falls after minutes; battery health degrades | Cap FPS, lower resolution, accept a stable 30 FPS |
| Shader compilation | First run stutters; driver/translator updates invalidate caches | Warm up shaders; do not mistake first-launch stutter for a regression |
| 16 KB pages (Android 15+) | Packaged NDK `.so` files need 16 KB ELF alignment (NDK r28+ by default; linker flags on r27 and lower); a backcompat mode exists | Play enforcement from 2027-02-01; verify your specific libraries inside proot |
| Off-screen throttling | Rendering slows when the screen is off | Keep Termux:X11 in the same app family; wake lock keeps CPU on, not screen on |

The GPU is often not the bottleneck: a game can be CPU-translation-bound with low GPU usage, or GPU-bound at high resolution. Measure before optimising (module 13).

## 10.14 A launch script

```sh
#!/bin/sh
# Run inside the proot guest, after termux-x11 :1 is running on the host
export DISPLAY=:1
export PULSE_SERVER=127.0.0.1
export WINEPREFIX="$HOME/.wine-adreno"
# WINEARCH is chosen when the prefix is created (wineboot), not per launch.
# export WINEARCH=win64
export VK_DRIVER_FILES=/absolute/path/to/freedreno_icd.aarch64.json
export WINEDLLOVERRIDES="d3d9=n,dxgi=n"
export DXVK_HUD=devinfo,fps
export DXVK_LOG_LEVEL=warn
export BOX64_DYNAREC=1
export BOX64_DYNAREC_BIGBLOCK=3
export HODLL64=libarm64ecfex.dll

exec hangover-wine "$HOME/Games/Game/game.exe"
```

Change one variable at a time, keep a copy of any working script as a template, and record the exact Turnip, Wine and translator versions alongside it ([module 08 §8.11](08-runtime-prefixes.md)).

## 10.15 Non-Android hosts

Everything from §10.7 onward applies to ARM Linux too, but the Android-specific stages disappear: there is no Termux, proot or `sharedUserId`, and no SurfaceFlinger — you use the host's glibc userspace, X11/Wayland and Mesa driver directly, with `VK_DRIVER_FILES` or the normal ICD discovery path. That simpler variant is covered in [module 11](11-linux-arm.md). On x86 Linux the CPU-translation stage disappears entirely because the host already executes x86-64 ([module 12](12-linux-x86-and-other-hosts.md)). The layered diagnostic method is unchanged.

## 10.16 What to remember

- Termux is a substrate, not a Windows frontend; you add Wine, a translator, graphics libraries and a display yourself.
- Install Termux from one trusted source (F-Droid); never mix sources or use mod APKs.
- Defeat the phantom process killer and hold a wake lock before debugging games.
- Prefer ARM64 Wine plus Hangover's ARM64EC/WoW64 DLLs; fall back to x86-64 Wine under Box64.
- Wine provides Windows APIs; FEX/Box64 translate CPU instructions; DXVK/VKD3D translate graphics; Turnip is the driver — keep these roles distinct.
- Verify with `vulkaninfo`, then a console EXE, then D3D9, then D3D11/D3D12; fix one layer at a time.
- Old games may need D8VK/D7VK/WineD3D rather than DXVK; dgVoodoo2 is not part of this pipeline.

Next: [11 — ARM Linux](11-linux-arm.md).

## References

- Termux: https://termux.dev/ ; repo https://github.com/termux/termux-app ; F-Droid https://f-droid.org/en/packages/com.termux/ ; packages https://github.com/termux/termux-packages ; termux-api https://github.com/termux/termux-api
- Display: Termux:X11 https://github.com/termux/termux-x11 ; XServer XSDL https://github.com/pelya/xserver-xsdl ; Android graphics https://source.android.com/docs/core/graphics
- Container: proot https://github.com/PRoot/proot ; proot-distro https://github.com/termux/proot-distro
- Wine and ABI bridges: Wine https://gitlab.winehq.org/wine/wine ; Hangover https://github.com/AndreRH/hangover ; Hangover Wine package https://github.com/termux/termux-packages/blob/master/x11-packages/hangover-wine/build.sh ; ARM64EC https://learn.microsoft.com/en-us/windows/arm/arm64ec ; WoW64 https://learn.microsoft.com/en-us/windows/win32/winprog64/running-32-bit-applications ; Wine builds https://github.com/Kron4ek/Wine-Builds
- CPU translation: FEX https://github.com/FEX-Emu/FEX ; FEX config https://wiki.fex-emu.com/index.php/Development:Configuration ; Box64 https://github.com/ptitSeb/box64 ; Box64 Wine notes https://github.com/ptitSeb/box64/blob/main/docs/WINE.md ; Box86 https://github.com/ptitSeb/box86 ; QEMU https://www.qemu.org/
- Graphics translation: DXVK https://github.com/doitsujin/dxvk ; VKD3D-Proton https://github.com/HansKristian-Work/vkd3d-proton ; D8VK https://github.com/AlpyneDreams/d8vk ; D7VK https://github.com/WinterSnowfall/d7vk ; D3D8to9 https://github.com/crosire/d3d8to9 ; dgVoodoo2 https://www.dege.freeweb.hu/dgVoodoo2/ ; WineD3D https://gitlab.winehq.org/wine/wine
- Drivers: Freedreno/Turnip https://docs.mesa3d.org/drivers/freedreno.html ; Zink https://docs.mesa3d.org/drivers/zink.html ; Mesa https://gitlab.freedesktop.org/mesa/mesa ; Mesa env vars https://docs.mesa3d.org/envvars.html
- Vulkan loader: https://github.com/KhronosGroup/Vulkan-Loader ; loader guide https://docs.vulkan.org/guide/latest/loader.html ; Vulkan-Tools https://github.com/KhronosGroup/Vulkan-Tools
- Audio: PulseAudio https://www.freedesktop.org/wiki/Software/PulseAudio/ ; PipeWire https://pipewire.org/
- Page sizes: https://developer.android.com/guide/practices/page-sizes
