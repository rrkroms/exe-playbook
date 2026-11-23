# Research Brief — Non-Android Hosts: ARM Linux, x86 Linux, macOS, ChromeOS (Nov 2026)

## 1. ARM Linux summary

| Platform | CPU/GPU | Graphics driver | CPU translation | Wine/Proton | Steam | Notes |
|---|---|---|---|---|---|---|
| **Asahi (Apple Silicon)** | M1/M2/M3, AGX | Upstream AGX Mesa + HoneyKrisp Vulkan; 16K pages; Linux 7.3 | FEX primary, Box64 fallback, inside muvm | Wine ARM64EC mature; FEX as WoW64/ARM64EC backend | Fedora `dnf install steam` wrapper via muvm | muvm ~20% penalty, no Wayland, slow I/O; x86 overlay immutable no root; M3 3D young |
| **Raspberry Pi 4/5** | A72/A76, V3D 4.2/7.1 | v3d GL + v3dv Vulkan 1.3 conformant | Box64 (RPI4/5 flags), Box86 for 32-bit | Box64 Wine64/Proton | No native ARM Steam | 4K pages (no muvm); GPU low bandwidth; Pi≤3 2G/2G split incompatible |
| **Snapdragon X Elite** | Oryon, Adreno msm | Freedreno/Turnip | FEX (Linaro/TUXEDO demo), Box64 | Wine possible | No native ARM Steam | Kernel 6.15 supports many laptops; firmware extraction from Windows; Phoronix found GPU init failures |
| **ARM Chromebooks** | Mali | Panfrost/PanVK, often Venus virtio path | Box64/FEX in Crostini (unsupported) | Wine possible | Borealis beta ended 1 Jan 2026 | Google deprecated graphics acceleration in Linux-on-ChromeOS |
| **Other SBCs** | Mali/Vivante/PowerVR | Panfrost/PanVK, Lima, Etnaviv, PVR Vulkan | Box64/Box86, FEX | Same | None | PanVK conformance only G610 |

## 2. FEX / Box64 / muvm

- **FEX** (MIT): usermode x86/x86_64 emulator, custom IR → optimizing JIT, syscall translation, rootfs, thunks (GL/Vulkan/SDL). Can build as WoW64/ARM64EC backend DLL (`libarm64ecfex.dll`, `libwow64fex.dll`). Monthly releases (FEX-2611.1, 11 Nov 2026). Requires ARMv8.0-a with FP+CRC32. Valve-sponsored; bundled in Proton. Gaps: debug regs, ptrace, RDPID/RDTSC (fixed in 2611 pending kernel).
- **Box64** (MIT): x86_64 emulator with DynaRec (ARM64/RISC-V/LoongArch), wraps native host libs, 5–10× faster than interpreter. v0.4.2 (Apr 2026); v0.4.4 referenced unverified. Supports Wine64/Proton. 32-bit needs Box86; Wine WOW64 under Box64 experimental.
- **Box86**: 32-bit x86, requires armhf chroot + multilib; legacy.
- **muvm**: microVM (libkrun) giving x86 binaries 4K pages; guest shares host rootfs; `--emu=fex|box`; DRM Native Context for GPU. Asahi ~20% penalty; DRM Native Context upstream in virglrenderer 1.x/Mesa 25.2+.

Asahi chain: Steam wrapper → muvm (4K guest) → binfmt → FEX → x86 rootfs + Mesa overlay → Wine/Proton → DXVK/VKD3D → AGX.

## 3. x86_64 Linux (reference platform)

No CPU translation. Wine 11.0 stable (13 Jan 2026), dev 11.22. New WoW64 supported; ntsync kernel when available (6.14+); vkd3d 1.18. Proton 11.0-1 stable; 11.0-2/2e; Experimental current. DXVK 3.1.1 (Vulkan 1.4). vkd3d-proton 3.1 (28 Oct 2026). Tools: Lutris (GPL-3.0), Heroic (GPLv3), Bottles (GPL-3.0), ProtonUp-Qt (GPL-3.0), MangoHud (MIT), Gamescope, GameMode (BSD-3). Steam Play enable in client.

## 4. macOS

- Rosetta 2 translates x86_64 macOS apps; no AVX-512; cannot virtualize x86_64. Available through macOS 27 (subset after).
- Wine native macOS builds; Apple Silicon x86 exes need Rosetta or ARM64EC Wine.
- CrossOver 26 ships Wine 11.0, D3DMetal 3.0, DXMT 0.72, vkd3d 1.18; proprietary ~$74. CrossOver 27 announced ARM64, Apple Silicon only.
- Game Porting Toolkit 4: developer evaluation environment + Metal Shader Converter; D3DMetal inside (closed source).
- D3DMetal: D3D11/12 → Metal, best DX12 route; requires user-supplied GPTK in third-party wrappers.
- DXMT (LGPL): D3D10/11 → Metal, no DX12, often faster than DXVK.
- DXVK + KosmicKrisp/MoltenVK: Vulkan→Metal; MoltenVK deprecated by Khronos; KosmicKrisp recommended on Apple Silicon.
- Whisky original discontinued; frankea fork active (GPL-3.0, Wine 11.19, DXMT+DXVK).
- Apple OpenGL frozen at 4.1 → WineD3D legacy only.

## 5. ChromeOS

- Crostini: Linux apps in VM (crosvm/KVM) + container; Wayland/X via Sommelier; GPU accel device-dependent. "Baguette" containerless from M147; graphics acceleration listed no-longer-supported.
- Borealis: Steam in Arch VM with Venus virtual Vulkan for near-native; managed beta ended 1 Jan 2026; DIY Crostini only.
- VirGL vs Venus: VirGL GL paravirtualization; Venus Vulkan streaming. ARM Chromebooks typically Venus path.
- Steam for Chromebook hardware floor: i3/Ryzen 3, 8GB, 128GB; limited devices.

## 6. Decision matrix

- Easiest overall: x86_64 Linux. Easiest ARM: Raspberry Pi 4/5 or Asahi with Fedora Asahi Remix.
- CPU translation needed: all ARM Linux, Apple Silicon macOS, ARM Chromebooks; NOT x86 Linux, NOT Intel macOS.
- 4K-page workaround: Asahi and 16K Snapdragon (muvm/Wine emulation/P3S).
- Best GPU story: x86 Linux; among ARM: Asahi AGX, RPi V3D/V3DV, then Freedreno/Turnip, Panfrost/PanVK.
- Most direct DX12: macOS D3DMetal (license-gated); most open: DXVK/VKD3D→Vulkan on Linux.
- Worst anti-cheat: ARM Linux (especially Asahi); kernel anti-cheat blocked on all non-Windows. EAC Linux v2 announced 14 Oct 2026 with signed kernel/eBPF; Steam Deck lacks required features.

## 7. Citations (verified)

- https://asahilinux.org/2026/11/progress-report-7-3/
- https://docs.fedoraproject.org/ms/fedora-asahi-remix/x86-support/
- https://github.com/FEX-Emu/FEX
- https://github.com/ptitseb/box64
- https://github.com/ptitseb/box86
- https://github.com/AsahiLinux/muvm
- https://www.winehq.org/announce/11.0
- https://github.com/ValveSoftware/Proton
- https://github.com/doitsujin/dxvk
- https://github.com/HansKristian-Work/vkd3d-proton
- https://lutris.net/
- https://heroicgameslauncher.com/
- https://github.com/bottlesdevs/Bottles
- https://github.com/DavidoTek/ProtonUp-Qt
- https://github.com/flightlessmango/MangoHud
- https://github.com/ValveSoftware/gamescope
- https://github.com/FeralInteractive/gamemode
- https://developer.apple.com/documentation/apple-silicon/about-the-rosetta-translation-environment
- https://developer.apple.com/games/game-porting-toolkit
- https://www.codeweavers.com/crossover
- https://github.com/frankea/Whisky
- https://github.com/KhronosGroup/MoltenVK
- https://chromeos.dev/en/linux/linux-on-chromeos-deep-dive
- https://chromeos.dev/en/posts/bringing-steam-to-chromeos
- https://partner.steamgames.com/doc/steamhardware/proton
