# Research Brief — Termux DIY Windows Pipeline on Android ARM64 (Nov 2026)

Evidence tags: [D] documented primary, [CR] commonly reported, [U] UNVERIFIED.

## 1. Host setup (non-root, Android 13/14 ARM64)

- F-Droid Termux recommended. GitHub releases signed with public test key (security warning). Play Store build is limited experimental fork; do not use. Termux + plugins share `sharedUserId` — never mix sources.
- Packages: `pkg update && pkg upgrade`; `pkg install proot-distro termux-api android-tools`; `termux-setup-storage`; `termux-wake-lock`; `pkg install x11-repo`; `pkg i termux-x11-nightly`; `pkg install vulkan-tools`; `pkg install mesa-vulkan-icd-freedreno`.
- Android 12+ phantom process killer: Android 14+ Developer options → "Disable child process restrictions". Android 12L/13 non-root: `adb shell "settings put global settings_enable_monitor_phantom_procs false"`. Android 12 count limit: `device_config set_sync_disabled_for_tests persistent` then `device_config put activity_manager max_phantom_processes 2147483647` (order matters).
- Wake lock keeps CPU on, not screen; also disable battery optimization.

## 2. proot-distro and Wine approaches

Current proot-distro pulls Docker/OCI images:
```sh
proot-distro install ubuntu:24.04
proot-distro install debian:bookworm --architecture aarch64
proot-distro login ubuntu --shared-tmp --shared-x11
```
Guest env sets `PULSE_SERVER=127.0.0.1`, `container=proot-distro`. Cross-arch via QEMU user mode. Limitations: ptrace interception slows FS-heavy work, no real root/cgroups/namespaces/init/nesting, fixed `/proc/stat` etc.

Wine approaches:
| Approach | Emulates | Notes |
|---|---|---|
| ARM64 Wine + Hangover FEX/Box64 DLLs | Only Windows app; Unix/Wine native ARM64 | `libarm64ecfex.dll`, `libwow64fex.dll`, `wowbox64.dll`. Recommended where possible. |
| x86_64 Wine under Box64 | Whole Wine process | Kron4ek builds; documented fallback. |
| 32-bit x86 Wine under Box86/Box32 | 32-bit | Box32 experimental; armhf or experimental `BOX32=ON`. |
| x86 rootfs under QEMU | Everything | Slow. |

Plain ARM64 Wine cannot run x86/x64 Windows exes without WOW64/ARM64EC machinery. Termux has `hangover-wine` in x11-repo; no official `box64` package.

## 3. Component matrix / commands

- Box64: build in proot: `cmake .. -D ARM64=1 -DBAD_SIGNAL=ON -D ARM_DYNAREC=ON`, `make install`. 32-bit: `-DBOX32=ON`; WOW64 DLL `-DWOW64=ON`.
- FEX: `curl --silent https://raw.githubusercontent.com/FEX-Emu/FEX/main/Scripts/InstallFEX.py | python3`; rootfs mandatory on AArch64 via `FEXRootFSFetcher` to `$HOME/.fex-emu/RootFS/`; config `$HOME/.fex-emu/Config.json`.
- Wine: WineHQ/Kron4ek builds; staging patchset; Hangover Wine via Termux x11-repo.
- winetricks: Debian contrib `apt install winetricks`.
- Mesa Turnip: Termux `mesa-vulkan-icd-freedreno` (aarch64) ships `libvulkan_freedreno.so` + ICD JSON. Debian `mesa-vulkan-drivers` [CR].
- Vulkan loader: `vulkan-loader-generic` + `vulkan-tools`; `VK_DRIVER_FILES`/`VK_ICD_FILENAMES` absolute path.
- DXVK: copy x64 DLLs to system32, x32 to syswow64; `winecfg` native overrides for d3d8/d3d9/d3d10core/d3d11/dxgi. Debian packages `dxvk` etc.
- VKD3D-Proton: `./package-release.sh master /target --no-package`; needs DXVK 2.1+ DXGI.
- dgVoodoo2: NOT part of this pipeline (author disclaims Wine/Linux).
- Audio: PulseAudio in Termux; proot sets `PULSE_SERVER=127.0.0.1`; PipeWire-pulse [CR].
- Display: Termux:X11 recommended (`termux-x11 :1`); XSDL has no GL acceleration. Start: `termux-x11 :1 -xstartup "xfce4-session"`, then `export DISPLAY=:1`.

## 4. Environment variables

`WINEPREFIX`, `WINEARCH=win32` only for pure 32-bit prefix, `WINEDLLOVERRIDES="d3d9=n,dxgi=n"`, `DXVK_HUD=devinfo,fps`, `DXVK_LOG_LEVEL`, `VK_DRIVER_FILES` (absolute), `VK_LOADER_DEBUG=all`, `MESA_VK_WSI_DEBUG=sw`, `MESA_GL_VERSION_OVERRIDE=3.2`, `TU_DEBUG=sysmem|gmem`, `ZINK_DESCRIPTORS`, `DISPLAY=:1`, `PULSE_SERVER=127.0.0.1`, `BOX64_DYNAREC=1`, `BOX64_DYNAREC_BIGBLOCK`, `BOX64_DYNAREC_STRONGMEM`, `BOX64_SKIPCPU/MAXCPU`, `FEX_ROOTFS`, `FEX_THUNK*`, `HODLL64=libarm64ecfex.dll`, `HODLL=libwow64fex.dll`.

Example launch:
```sh
export DISPLAY=:1 WINEPREFIX="$HOME/.wine-adreno" WINEARCH=win64 PULSE_SERVER=127.0.0.1
export VK_DRIVER_FILES=/usr/share/vulkan/icd.d/freedreno_icd.aarch64.json
export DXVK_HUD=devinfo,fps DXVK_LOG_LEVEL=warn
export BOX64_DYNAREC=1 BOX64_DYNAREC_BIGBLOCK=3 BOX64_DYNACACHE=1
hangover-wine app.exe
```

## 5. GPU driver reality

Turnip = Mesa Vulkan driver for Adreno 6xx/7xx/8xx (Vulkan 1.3 A6xx, 1.4 A7xx); no a5xx. Freedreno GL up to GL 4.5/GLES 3.2. Zink = GL on Vulkan. ICD packaging from Termux `mesa-vulkan-icd-freedreno`. Root not required for per-app injection; `/dev/kgsl-3d0` must be accessible. Qualcomm system driver used by Android; wrapping it as ICD failed — use generic loader + Turnip. Verify with `vulkaninfo --summary`. D3D12 via VKD3D least reliable tier.

## 6. Performance caveats

proot ptrace slows FS; no root/cgroups; fixed /proc stats; phantom killer; off-screen CPU throttling (use sharedUid Termux:X11); big.LITTLE via BOX64_SKIPCPU/MAXCPU; thermal; keep RootFS under `$HOME`; 32-bit via WOW64/FEX DLLs; Android 15+ 16KB pages need NDK r28+/max-page-size flags (impact on prebuilt .so UNVERIFIED); shader compilation stalls via GPL.

## 7. Minimal test path

Stage 0: Termux setup + phantom killer disabled + Termux:X11 + vulkan-tools + Turnip.
Stage 1: `vulkaninfo --summary` shows Adreno/Turnip.
Stage 2: `termux-x11 :1 &`, `proot-distro login debian --shared-tmp --shared-x11`, `export DISPLAY=:1 PULSE_SERVER=127.0.0.1`.
Stage 3: 64-bit console EXE: `HODLL64=libarm64ecfex.dll hangover-wine hello64.exe` or `box64 wine64 hello64.exe`.
Stage 4: 32-bit: `HODLL=libwow64fex.dll hangover-wine hello32.exe`.
Stage 5: D3D9 test: copy DXVK x32/x64 DLLs, `WINEDLLOVERRIDES="d3d9=n,dxgi=n" DXVK_HUD=devinfo,fps wine d3d9-test.exe`. HUD devinfo shows Turnip.
Stage 6: D3D11: `WINEDLLOVERRIDES="d3d11=n,dxgi=n"`.

## 8. Citations (verified)

- https://f-droid.org/packages/com.termux/
- https://raw.githubusercontent.com/termux/termux-app/master/README.md
- https://github.com/termux/proot-distro
- https://raw.githubusercontent.com/termux/termux-packages/master/x11-packages/hangover-wine/build.sh
- https://github.com/ptitSeb/box64
- https://raw.githubusercontent.com/ptitSeb/box64/main/docs/WINE.md
- https://github.com/FEX-Emu/FEX
- https://wiki.fex-emu.com/index.php/Development:Configuration
- https://raw.githubusercontent.com/AndreRH/hangover/master/README.md
- https://raw.githubusercontent.com/doitsujin/dxvk/master/README.md
- https://raw.githubusercontent.com/HansKristian-Work/vkd3d-proton/master/README.md
- https://docs.mesa3d.org/drivers/freedreno.html
- https://docs.mesa3d.org/drivers/zink.html
- https://github.com/KhronosGroup/Vulkan-Loader/tree/main/docs
- https://raw.githubusercontent.com/termux/termux-x11/master/README.md
- https://developer.android.com/guide/practices/page-sizes
