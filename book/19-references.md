# 19 — References: Official Repositories and Documentation

This module is the link directory for the book. Every technology name in earlier modules links to upstream documentation on first use; this table is the consolidated list with license, platform scope and notes.

**Snapshot date:** 2026-11-23. Version numbers were accurate when checked; upstream release pages are always more current.

## Compatibility layers and runtimes

| Project | Official URL | License / model | Platforms | Notes |
|---|---|---|---|---|
| Wine | https://gitlab.winehq.org/wine/wine | LGPL-2.1+ | Linux, macOS, Android via frontends | Windows API compatibility; not a CPU emulator |
| Wine 11.0 announcement | https://www.winehq.org/announce/11.0 | — | — | New WoW64, ntsync support, ARM64EC work |
| Proton | https://github.com/ValveSoftware/Proton | BSD-3 wrapper + component licenses | Linux/Steam, ARM builds | Wine + DXVK + VKD3D-Proton + patches |
| Proton-GE | https://github.com/GloriousEggroll/proton-ge-custom | Community build | Linux | Additional patches and media codecs |
| Winetricks | https://github.com/Winetricks/winetricks | Public domain / permissive | Linux/macOS | Installs DLLs and runtimes into prefixes |
| Wine Staging | https://github.com/wine-staging/wine-staging | LGPL-2.1+ | Linux/macOS | Experimental patchset on Wine development |

## CPU translation

| Project | Official URL | License | Platforms | Notes |
|---|---|---|---|---|
| FEX | https://github.com/FEX-Emu/FEX | MIT | ARM64 Linux, Android via frontends | x86/x86-64 dynamic binary translator; ARM64EC/WoW64 backend DLLs; RootFS and thunks |
| FEX wiki | https://wiki.fex-emu.com/ | — | — | Configuration and ARM64EC documentation |
| Box64 | https://github.com/ptitSeb/box64 | MIT | ARM64, RISC-V, LoongArch | x86-64 dynamic binary translator with DynaRec; wraps host libraries |
| Box86 | https://github.com/ptitSeb/box86 | MIT/GPL per repo | ARM32/armhf | 32-bit x86 translator; legacy |
| Hangover | https://github.com/AndreRH/hangover | Open source | ARM64 Linux/Android | Wine with emulator DLLs instead of whole-process emulation |
| muvm | https://github.com/AsahiLinux/muvm | Open source | Asahi Linux | 4 KiB-page microVM for x86 Steam/Wine |
| Rosetta 2 docs | https://developer.apple.com/documentation/apple-silicon/about-the-rosetta-translation-environment | Proprietary | macOS Apple Silicon | x86-64 macOS translation; no kernel extensions |

## Graphics translation

| Project | Official URL | License | Source → destination | Notes |
|---|---|---|---|---|
| DXVK | https://github.com/doitsujin/dxvk | zlib/libpng | D3D8/9/10/11 → Vulkan | v3.1.1; Vulkan 1.4 since 3.0; merged D8VK |
| VKD3D-Proton | https://github.com/HansKristian-Work/vkd3d-proton | LGPL-2.1+ | D3D12 → Vulkan | v3.1; needs Vulkan 1.3, descriptor indexing, robustness2 and `VK_KHR_push_descriptor` |
| VKD3D (upstream) | https://gitlab.winehq.org/wine/vkd3d | LGPL-2.1+ | D3D12 → Vulkan library | Shader translation library, not a drop-in DLL |
| D8VK | https://github.com/AlpyneDreams/d8vk | zlib | D3D8 → Vulkan | Merged into DXVK; standalone discontinued |
| D7VK | https://github.com/WinterSnowfall/d7vk | zlib | D3D7 → Vulkan (via D3D9) | Needs Vulkan 1.4 + underlying DirectDraw |
| DXVK-Sarek | https://github.com/pythonlover02/DXVK-Sarek | zlib | D3D3–11 → Vulkan 1.1/1.2 | Mobile/ARM focused; includes D7VK backport |
| D3D8to9 | https://github.com/crosire/d3d8to9 | BSD-2 | D3D8 → D3D9 | Pair with DXVK |
| DxWrapper | https://github.com/elishacloud/dxwrapper | zlib + others | DDraw/D3D1–9 → D3D9 etc. | Extensive legacy game options including IGI |
| DDrawCompat | https://github.com/narzoul/DDrawCompat | BSD Zero Clause | In-place DDraw fixes | Windows only; not supported in Wine |
| cnc-ddraw | https://github.com/FunkyFr3sh/cnc-ddraw | MIT | 2D DirectDraw → GDI/GL/D3D9 | Works in Wine; 2D games only |
| dgVoodoo2 | https://www.dege.freeweb.hu/dgVoodoo2/ | Closed-source freeware | Glide/DX3–9 → D3D11/12 | Windows-oriented; Wine unsupported by author |
| GXVK | https://github.com/CkNoSFeRaTU/gxvk | zlib | Glide → Vulkan | Early alpha |
| DXVK-NVAPI | https://github.com/jp7677/dxvk-nvapi | MIT | NVAPI → DXVK/VKD3D | DLSS/Reflex forwarding |
| WineD3D | https://gitlab.winehq.org/wine/wine | LGPL-2.1+ | D3D → OpenGL | Bundled with Wine |
| MoltenVK | https://github.com/KhronosGroup/MoltenVK | Apache-2.0 | Vulkan → Metal | Deprecated by Khronos; Intel Macs/iOS |
| KosmicKrisp | https://docs.mesa3d.org/drivers/kosmickrisp.html | MIT | Vulkan → Metal | Mesa's conformant Apple Silicon Vulkan driver; requires macOS 26+ / Metal 4; not available for Intel Macs |
| D3DMetal | https://developer.apple.com/games/game-porting-toolkit | Proprietary | D3D11/12 → Metal | Requires user-supplied GPTK in third-party wrappers |
| DXMT | https://github.com/3Shain/dxmt | LGPL-2.1 | D3D10/11 → Metal | Open source; no D3D12 |

## GPU drivers and graphics APIs

| Project | Official URL | License | Notes |
|---|---|---|---|
| Mesa | https://www.mesa3d.org/ | MIT (mostly) | OpenGL/Vulkan drivers; stable 26.2.7 at snapshot |
| Mesa env vars | https://docs.mesa3d.org/envvars.html | — | TU_DEBUG, MESA_SHADER_CACHE_*, ZINK_* |
| Freedreno / Turnip docs | https://docs.mesa3d.org/drivers/freedreno.html | MIT | Adreno GL + Vulkan driver |
| Panfrost docs | https://docs.mesa3d.org/drivers/panfrost.html | MIT | Mali GL + Vulkan |
| PowerVR docs | https://docs.mesa3d.org/drivers/powervr.html | MIT | Vulkan-only; GL via Zink |
| Asahi docs | https://docs.mesa3d.org/drivers/asahi.html | MIT | Apple AGX GL; HoneyKrisp Vulkan |
| V3D/V3DV docs | https://docs.mesa3d.org/drivers/v3d.html | MIT | Raspberry Pi GL/Vulkan |
| RADV docs | https://docs.mesa3d.org/drivers/radv.html | MIT | AMD Vulkan |
| ANV docs | https://docs.mesa3d.org/drivers/anv.html | MIT | Intel Vulkan |
| NVK docs | https://docs.mesa3d.org/drivers/nvk.html | MIT | NVIDIA Vulkan |
| Zink docs | https://docs.mesa3d.org/drivers/zink.html | MIT | OpenGL on Vulkan |
| Venus docs | https://docs.mesa3d.org/drivers/venus.html | Open | Virtual Vulkan over virtio-gpu |
| VirGL docs | https://docs.mesa3d.org/drivers/virgl/ | Open | Virtual OpenGL over virtio-gpu |
| Vulkan loader | https://github.com/KhronosGroup/Vulkan-Loader | Apache-2.0 | ICD discovery, layers, VK_DRIVER_FILES |
| Vulkan loader docs | https://docs.vulkan.org/guide/latest/loader.html | — | Loader architecture and debugging |
| vulkaninfo | https://github.com/KhronosGroup/Vulkan-Tools | Apache-2.0 | Verify active GPU/driver |
| Khronos conformant products | https://www.khronos.org/conformance/adopters/conformant-products/vulkan | — | Authoritative conformance list |
| Android graphics docs | https://source.android.com/docs/core/graphics | — | SurfaceFlinger, HWC, Vulkan on Android |
| Mesa on Android | https://docs.mesa3d.org/android.html | — | Building Mesa/KGSL for Android |

## Android frontends and environment

| Project | Official URL | License | Notes |
|---|---|---|---|
| Winlator | https://github.com/brunodev85/winlator | LGPL-2.1 (repo) | Box64, Wine, Turnip, PRoot, native XServer |
| Winlator (WB64 fork) | https://github.com/winebox64/winlator | Apache-2.0 | Separate fork line |
| Winlator site | https://winlator.org/ | — | Official site; other domains are mirrors/fakes |
| GameNative | https://github.com/utkarshdalal/GameNative | GPL-3.0 | Proton + FEX + gamescope |
| DroidDeck | https://github.com/Droid-Deck/DroidDeck | GPL-3.0 | ARM64 Steam, Proton, FEX, drivers |
| GameHub | https://gamehub.xiaoji.com/ | Proprietary | Closed source; account and analytics |
| Mobox | https://github.com/olegos2/mobox | No LICENSE file | Archived; Termux-native |
| Pluvia | https://github.com/oxters168/Pluvia | GPL-3.0 | Retired; succeeded by GameNative |
| Termux | https://github.com/termux/termux-app | GPL-3.0 | Use F-Droid build; never mix signing sources |
| Termux:X11 | https://github.com/termux/termux-x11 | GPL-3.0 | X server companion |
| proot-distro | https://github.com/termux/proot-distro | MIT | Linux distributions under Termux |
| AdrenoTools drivers | https://github.com/K11MCH1/AdrenoToolsDrivers/releases | Community | Turnip/Qualcomm driver bundles |
| Bannerlator | https://github.com/The412Banner/Bannerlator | GPL-3.0 | Winlator fork with driver feeds |
| WinNative | https://github.com/WinNative-Emu/WinNative | GPL-3.0 | Winlator/Pluvia lineage |
| Vectras VM | https://www.getvectras.com/ | Proprietary app | Full-system QEMU emulation; slow |

## Platforms and tooling

| Project | Official URL | License | Notes |
|---|---|---|---|
| Asahi Linux | https://asahilinux.org/ | Project code moving to GPL | Apple Silicon Linux; progress reports |
| Fedora Asahi Remix x86 support | https://docs.fedoraproject.org/ms/fedora-asahi-remix/x86-support/ | — | muvm + FEX + Steam wrapper |
| Raspberry Pi | https://www.raspberrypi.com/ | — | V3D/V3DV graphics docs |
| Qualcomm Snapdragon X Linux | https://www.qualcomm.com/developer/blog/2024/05/upstreaming-linux-kernel-support-for-the-snapdragon-x-elite | — | Kernel enablement |
| ChromeOS Linux deep dive | https://chromeos.dev/en/linux/linux-on-chromeos-deep-dive | — | Crostini architecture |
| ChromeOS Borealis / Venus | https://chromeos.dev/en/posts/bringing-steam-to-chromeos | — | Managed Steam path ended 2026 |
| Steam on Linux / Proton | https://partner.steamgames.com/doc/steamhardware/proton | — | Valve's Proton and anti-cheat documentation |
| Lutris | https://lutris.net/ | GPL-3.0 | Game launchers and install scripts |
| Bottles | https://github.com/bottlesdevs/Bottles | GPL-3.0 | Wine/Proton bottle manager |
| Heroic Games Launcher | https://heroicgameslauncher.com/ | GPLv3 | Epic/GOG/Amazon launcher |
| ProtonUp-Qt | https://github.com/DavidoTek/ProtonUp-Qt | GPL-3.0 | Installs Proton-GE/Wine-GE |
| MangoHud | https://github.com/flightlessmango/MangoHud | MIT | Performance overlay |
| Gamescope | https://github.com/ValveSoftware/gamescope | Open source | Micro-compositor, FSR, frame limiting |
| GameMode | https://github.com/FeralInteractive/gamemode | BSD-3 | CPU governor and I/O priority |
| vkBasalt | https://github.com/DadSchoorse/vkBasalt | MIT | Vulkan post-processing layer |
| Fossilize | https://github.com/ValveSoftware/Fossilize | Open source | Pipeline cache and replay |
| CrossOver | https://www.codeweavers.com/crossover | Proprietary | Commercial Wine for Mac/Linux |
| Game Porting Toolkit | https://developer.apple.com/games/game-porting-toolkit | Apple terms | D3DMetal and Metal shader converter |
| Whisky (frankea fork) | https://github.com/frankea/Whisky | GPL-3.0 | Active Apple Silicon Wine wrapper |
| Epic EAC Linux v2 | https://onlineservices.epicgames.com/news/introducing-easy-anti-cheat-linux-v2 | — | Opt-in per-title Linux support |

## Performance and diagnostics

| Project | Official URL | Notes |
|---|---|---|
| DXVK HUD docs | https://github.com/doitsujin/dxvk#hud | `DXVK_HUD` options |
| MangoHud docs | https://github.com/flightlessmango/MangoHud#configuration | `MANGOHUD_CONFIG` |
| ntsync kernel docs | https://docs.kernel.org/userspace-api/ntsync.html | Kernel 6.14+ |
| ntsync Debian howto | https://wiki.debian.org/Wine/NtsyncHowto | Checking `/dev/ntsync` |
| Vulkan loader debugging | https://github.com/KhronosGroup/Vulkan-Loader/blob/main/docs/LoaderDebugging.md | `VK_LOADER_DEBUG` |
| Android dumpsys | https://developer.android.com/tools/dumpsys | `gfxinfo`, `framestats` |
| Android performance docs | https://developer.android.com/topic/performance/measuring-performance | Perfetto, Simpleperf |
| Mesa env vars | https://docs.mesa3d.org/envvars.html | Debug and cache variables |
| AMD FSR SDK | https://gpuopen.com/amd-fsr-sdk | FSR versions and integration |
| Wine debugging guide | https://www.codeweavers.com/blog/aeikum/2019/1/15/working-on-wine-part-4-debugging-wine | WINEDEBUG, logs |

## How to cite a source in this book

When contributing, add the upstream URL next to the claim, record the version/date observed, and state whether the claim is official documentation, a release note, or a community report. Mark uncertain claims `[NEEDS VERIFICATION]` rather than presenting them as fact.

Next: return to the [README](../README.md).
