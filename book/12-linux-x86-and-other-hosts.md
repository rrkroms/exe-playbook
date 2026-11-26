# 12 — x86_64 Linux and Other Non-ARM Hosts

This module covers the hosts where the pipeline from modules 02–07 is either simplest or most unusual. **x86_64 Linux** is the reference case: the CPU already executes the game's instructions, so the CPU-translation box disappears. **macOS** and **ChromeOS** keep the same logical pipeline but replace the graphics path with Metal or with virtualized GPU devices.

Keep the four categories separate throughout:

| Category | Question it answers | Examples in this module |
|---|---|---|
| API compatibility | "Where is `CreateFileW`?" | [Wine](https://gitlab.winehq.org/wine/wine), [Proton](https://github.com/ValveSoftware/Proton) |
| CPU translation | "How does an x86 instruction run on this CPU?" | [Rosetta 2](https://developer.apple.com/documentation/apple-silicon/about-the-rosetta-translation-environment) (macOS only); none on x86_64 |
| Graphics translation | "How does Direct3D become Vulkan or Metal?" | [DXVK](https://github.com/doitsujin/dxvk), [VKD3D-Proton](https://github.com/HansKristian-Work/vkd3d-proton), D3DMetal (inside [Game Porting Toolkit](https://developer.apple.com/games/game-porting-toolkit)), [DXMT](https://github.com/3Shain/dxmt) |
| GPU driver | "How does Vulkan/OpenGL become GPU commands?" | [RADV](https://docs.mesa3d.org/drivers/radv.html), [ANV](https://docs.mesa3d.org/drivers/anv.html), NVIDIA, MoltenVK/KosmicKrisp |

## 12.1 x86_64 Linux: the baseline with no CPU translation

**x86-64** (also called AMD64) is the 64-bit extension of the x86 instruction set used by most desktop and laptop PCs. A Windows game compiled for x86-64 therefore runs its own instructions natively on an x86-64 Linux host; a 32-bit x86 game runs natively too, using the CPU's compatibility mode. No [FEX](https://github.com/FEX-Emu/FEX) or [Box64](https://github.com/ptitSeb/box64) is needed. This removes one entire layer of overhead and one entire class of bugs.

What remains is API compatibility (Wine/Proton), graphics translation (DXVK/VKD3D-Proton), and the GPU driver.

```text
Windows game (x86 or x86-64 PE)
        |
        v
  Wine / Proton                 <-- Windows API compatibility
        |
        v
  (no CPU translation)          <-- CPU executes guest code natively
        |
        v
  DXVK / VKD3D-Proton           <-- D3D -> Vulkan
        |
        v
  RADV / ANV / NVIDIA driver    <-- Vulkan -> GPU commands
        |
        v
  Wayland / X11
```

This is a **conceptual diagram**: a real game has many threads, and the driver submits work asynchronously while Wine's server process coordinates state. The picture shows *which layer owns which job*, not a literal instruction-by-instruction sequence.

**Version snapshot (Nov 2026).** Versions matter because "Wine" alone is not a reproducible description.

| Component | Version in the briefs | Role |
|---|---|---|
| [Wine](https://www.winehq.org/announce/11.0) | 11.0 stable (13 Jan 2026); development 11.22 | Windows API compatibility layer; includes new WoW64 and ntsync support |
| [Proton](https://partner.steamgames.com/doc/steamhardware/proton) | 11.0-1 stable; 11.0-2/2e; Experimental current | Valve's curated Wine distribution for [Steam Play](https://partner.steamgames.com/doc/steamhardware/proton) |
| [DXVK](https://github.com/doitsujin/dxvk) | 3.1.1 (15 Sep 2026) | D3D8/9/10/11 → Vulkan; requires Vulkan 1.4 since 3.0 |
| [VKD3D-Proton](https://github.com/HansKristian-Work/vkd3d-proton) | 3.1 (28 Oct 2026) | D3D12 → Vulkan; needs Vulkan 1.3 plus specific extensions |
| [VKD3D](https://gitlab.winehq.org/wine/vkd3d) (upstream) | 1.18 | Shader library used by Wine and CrossOver; not a drop-in game DLL |
| [Mesa](https://www.mesa3d.org/) | 26.2.7 (11 Nov 2026) | Open-source OpenGL/Vulkan drivers |

Proton is not a separate engine: it is Wine plus patches, bundled runtimes and Steam integration. Proton-GE is another *distribution* of Wine ([Proton-GE](https://github.com/GloriousEggroll/proton-ge-custom)).

## 12.2 The driver choice on x86 Linux

The GPU driver is a separate layer from both DXVK and Wine. Your GPU family decides which Vulkan ICD the loader will find.

| GPU | Vulkan driver | OpenGL driver | Notes |
|---|---|---|---|
| AMD | [RADV](https://docs.mesa3d.org/drivers/radv.html) | RadeonSI | Default on Steam Deck; Vulkan 1.4 conformant |
| Intel | [ANV](https://docs.mesa3d.org/drivers/anv.html) | Iris | Intel's official Linux driver path; newer parts need GuC firmware |
| NVIDIA | [Proprietary ICD](https://developer.nvidia.com/vulkan-driver) | GLVND / proprietary | Closed userspace; NVK + [Zink](https://docs.mesa3d.org/drivers/zink.html) is the open alternative with reclocking caveats ([NVK docs](https://docs.mesa3d.org/drivers/nvk.html)) |

Verify the driver rather than assuming it:

```sh
vulkaninfo --summary | grep -i driverName
VK_LOADER_DEBUG=error,warn vulkaninfo --summary
```

`vulkaninfo` comes from [Vulkan-Tools](https://github.com/KhronosGroup/Vulkan-Tools). If no device appears, no amount of DXVK tuning will help. Note that VKD3D-Proton's requirement is stricter than "Vulkan exists": it needs descriptor indexing, `VK_EXT_robustness2` and `VK_KHR_push_descriptor`.

Legacy games still need legacy graphics tools. D3D9 goes through DXVK; D3D8 needs DXVK's `d3d8.dll` plus `d3d9.dll`; D3D7 immediate mode can use [D7VK](https://github.com/WinterSnowfall/d7vk) (which needs Vulkan 1.4 and Wine's real DirectDraw underneath); 2D DirectDraw titles can use [cnc-ddraw](https://github.com/FunkyFr3sh/cnc-ddraw). [WineD3D](https://gitlab.winehq.org/wine/wine) on OpenGL is the universal fallback, with the fixed-function caveats from module 06. These choices are identical on x86 and ARM Linux — only the CPU layer differs.

## 12.3 Frontends and tools

A **frontend** launches games and selects components. It is not the compatibility layer: a frontend bug can look like a Wine bug, and vice versa (module 02).

| Tool | What it does | What it does **not** do |
|---|---|---|
| [Steam Play](https://partner.steamgames.com/doc/steamhardware/proton) | Enables non-Windows games in Steam using Proton | Not a general launcher for games bought elsewhere |
| [Lutris](https://lutris.net/) | Launcher with per-game scripts and runners | Does not implement Win32 APIs itself |
| [Bottles](https://github.com/bottlesdevs/Bottles) | Manages Wine prefixes ("bottles") with per-bottle settings | Not a CPU translator or driver |
| [Heroic Games Launcher](https://heroicgameslauncher.com/) | Launcher for storefronts such as GOG and Epic | Not a replacement for Wine/Proton |
| [ProtonUp-Qt](https://github.com/DavidoTek/ProtonUp-Qt) | Installs and updates Proton/Proton-GE builds | Does not change graphics drivers |
| [MangoHud](https://github.com/flightlessmango/MangoHud) | On-screen performance overlay | Does not improve performance |
| [Gamescope](https://github.com/ValveSoftware/gamescope) | Nested Wayland compositor; fixes fullscreen, scaling and per-game display settings | Not a frame-rate multiplier |
| [GameMode](https://github.com/FeralInteractive/gamemode) | Daemon that applies CPU governor and process niceness for a game | Not a driver or translator |

Practical notes:

- Enable Steam Play for all titles in the Steam client if you want Proton outside explicitly whitelisted games; select a Proton version per game.
- Use one prefix or bottle per game. Install VC++/.NET/Gecko and D3DX/XAudio helper DLLs there, as described in module 03.
- Use ProtonUp-Qt to install an extra Proton build rather than hand-copying Wine directories into Steam's compatibility-tools folder.
- Record the frontend version and the exact Wine/Proton, DXVK and driver versions when reporting a bug. "It doesn't work with Proton" is not actionable.

### 12.3.1 What Proton configures for you

Proton sets many environment variables that you would otherwise set by hand. Reusing them with a different Wine build can produce confusing results, so know the main ones:

| Variable | Effect |
|---|---|
| `PROTON_NO_NTSYNC` | Disables the ntsync kernel path; there is no `PROTON_NTSYNC=1` |
| `PROTON_NO_FSYNC` | Disables the futex-based synchronization path |
| `DXVK_HUD=compiler` | Shows DXVK shader compilation, useful for first-launch stutter |
| `VK_DRIVER_FILES` | Selects a specific Vulkan ICD manifest instead of loader discovery |
| `MESA_SHADER_CACHE_DIR` | Moves Mesa's shader cache (default size 1 GiB) |

These are launch-time knobs, not components: changing one does not install a driver or a graphics translator.

## 12.4 Step-by-step notes for x86_64 Linux

1. Confirm the host: `uname -m` should print `x86_64`. If it prints `aarch64`, you are on module 11's territory and need CPU translation.
2. Check the kernel for ntsync support. [ntsync](https://docs.kernel.org/userspace-api/ntsync.html) is complete in mainline Linux 6.14; Proton 11.0 enables it automatically when `/dev/ntsync` can be opened. Check with `lsof /dev/ntsync`. There is no `PROTON_NTSYNC=1`; the opt-out is `PROTON_NO_NTSYNC`.
3. Verify a Vulkan device with `vulkaninfo --summary` and note the driver name.
4. Launch through Steam/Proton for Steam games, or through Lutris/Bottles/Heroic with an explicit Wine/Proton build for other stores.
5. If graphics initialization fails, check the DXVK/VKD3D-Proton logs and the Vulkan device before touching DLL overrides.
6. If the game starts but stutters at first, let shader caches warm; clear DXVK, VKD3D-Proton and Mesa caches together after changing any of those components (module 05).
7. Add MangoHud for measurement, Gamescope for display problems, and GameMode for scheduler settings — one at a time, so you know which one changed behaviour.

**Remaining limitations.** Kernel-level anti-cheat is still a hard blocker on Linux: it expects Windows and will not run under Wine, Proton or a compatibility layer. A Linux v2 of [Easy Anti-Cheat](https://onlineservices.epicgames.com/news/introducing-easy-anti-cheat-linux-v2) was announced with signed kernel/eBPF requirements; Epic publishes a trusted-signing key list and names Steam Hardware as a known gap, so coverage depends on the device's boot configuration. DRM that depends on a Windows kernel component fails for the same reason.

### 12.4.1 32-bit games and WoW64 on x86_64

A 32-bit Windows game normally runs in a 64-bit prefix through WoW64, with 32-bit Windows DLLs in `syswow64`; a pure `WINEARCH=win32` prefix is deprecated and unsupported under Wine 11.0's new WoW64 mode. On an x86-64 host the CPU executes the 32-bit instructions natively — there is no translation. [WoW64](https://learn.microsoft.com/en-us/windows/win32/winprog64/running-32-bit-applications) is only the API/ABI thunk that converts 32-bit arguments and structures into their 64-bit forms before they reach the host. Calling WoW64 an emulator is wrong on every host, and especially wrong here.

"Multilib" means having both 64-bit and 32-bit host libraries installed. A 64-bit-only Wine installation will fail to start a 32-bit game even though the CPU is capable; the distribution's `wine32` or `lib32` packages are required, not a translator.

| Symptom | Likely cause | First check |
|---|---|---|
| "Bad EXE format" for an old game | 32-bit support missing | 32-bit Wine packages / WoW64 `syswow64` DLLs; `WINEARCH=win32` only on older Wine |
| 32-bit DLL fails to load | Architecture mismatch | 32-bit translator DLLs in `syswow64`, not `system32` |
| Game runs, then stalls at startup | Missing directx helper DLLs | Install D3DX/XAudio/XInput helpers (module 03) |
| DXVK DLL ignored | Override not set | `WINEDLLOVERRIDES` with `n` for `d3d9`/`dxgi` (module 05) |

## 12.5 macOS: three translations, not one

macOS is a non-Windows host with a different kernel, ABI and graphics stack. The concepts from module 03 still apply, but three *different* translations are easy to conflate:

1. **CPU translation** — [Rosetta 2](https://developer.apple.com/documentation/apple-silicon/about-the-rosetta-translation-environment) executes x86-64 **macOS** binaries on Apple Silicon. It does not implement Windows APIs and does not help a Windows PE on its own. It does not support AVX-512 or virtualize x86-64. Apple documents availability through macOS 27, with a reduced subset afterwards.
2. **API compatibility** — Wine implements Win32/NT. Native macOS Wine builds exist; an x86-64 Windows game needs either an x86-64 Wine process under Rosetta (Intel-style path) or an ARM64EC-aware Wine with a backend such as FEX.
3. **Graphics translation** — D3DMetal or DXMT converts Direct3D to Metal; DXVK converts Direct3D to Vulkan, which is then carried to Metal by [MoltenVK](https://github.com/KhronosGroup/MoltenVK) or KosmicKrisp ([Mesa](https://gitlab.freedesktop.org/mesa/mesa)). WineD3D on OpenGL is legacy-only here.

> **Apple OpenGL is frozen at 4.1.** That makes the WineD3D/OpenGL route unsuitable for modern games on macOS; prefer a Metal-backed path ([Apple developer documentation](https://developer.apple.com/)).

### macOS component map

| Component | Source → destination | Notes |
|---|---|---|
| [CrossOver](https://www.codeweavers.com/crossover) | Commercial Wine distribution | CrossOver 26 shipped Wine 11.0, D3DMetal 3.0, DXMT 0.72 and vkd3d 1.18; roughly $74. Proprietary |
| [Game Porting Toolkit](https://developer.apple.com/games/game-porting-toolkit) | Developer evaluation environment; includes D3DMetal and Metal Shader Converter | Closed source; GPTK 4 in the briefs. Third-party wrappers generally require you to supply toolkit components yourself |
| D3DMetal | D3D11/12 → Metal, inside GPTK | Best Direct3D 12 route on macOS; license-gated through GPTK |
| [DXMT](https://github.com/3Shain/dxmt) | D3D10/11 → Metal; no D3D12 | LGPL; community reports, including from DXMT's author, often find it faster than DXVK+MoltenVK per game; no controlled multi-game benchmark |
| [DXVK](https://github.com/doitsujin/dxvk) + [MoltenVK](https://github.com/KhronosGroup/MoltenVK) | D3D → Vulkan → Metal | MoltenVK is deprecated by Khronos; prefer KosmicKrisp on Apple Silicon |
| [KosmicKrisp](https://docs.mesa3d.org/drivers/kosmickrisp.html) | Vulkan → Metal on Apple Silicon | Mesa's conformant Apple Silicon Vulkan driver; **requires macOS 26+ and Metal 4** (MIT license) |
| [Whisky](https://github.com/frankea/Whisky) | Frontend | Original project discontinued; the frankea fork is active (GPL-3.0, Wine 11.19, DXMT + DXVK) |

**Version gate:** KosmicKrisp requires macOS 26+ and Metal 4. On older Apple Silicon macOS the only Vulkan-to-Metal layer is the now-deprecated MoltenVK, which may need an older DXVK. For this reason "DXVK + KosmicKrisp" is not a universal Apple Silicon recommendation.

[CrossOver 27](https://www.codeweavers.com/blog/mjohnson/2026/6/11/whats-in-and-whats-out-for-crossover-27) is announced as Apple-Silicon-only on macOS Sonoma+, with ARM64 builds for Mac and Linux and 32-bit bottles dropped; as of November 2026 it is Preview only, with GA expected early 2027. Intel Macs' last version is CrossOver 26. Intel Macs need no CPU translation for x86-64 Windows code, but still need Wine and a Vulkan-to-Metal path (MoltenVK); KosmicKrisp does not support Intel Macs. Apple Silicon Macs need all three translations.

### 12.5.1 Choosing a macOS graphics route

Only one Direct3D implementation should service a game at a time. D3DMetal, DXMT and DXVK each supply their own `d3d11.dll`/`dxgi.dll` style components; stacking two of them produces the same class of override conflict described in module 05.

| Game API | Preferred macOS route | Fallback | Constraint |
|---|---|---|---|
| D3D12 | D3DMetal (via GPTK) | None equivalent | Requires user-supplied GPTK components in third-party wrappers |
| D3D11 | DXMT | D3DMetal, DXVK + KosmicKrisp | DXMT has no D3D12 |
| D3D9 / D3D8 | DXVK + KosmicKrisp (or MoltenVK) | WineD3D | OpenGL is capped at 4.1, so WineD3D is legacy-only |
| D3D7 / DirectDraw | D7VK or cnc-ddraw | WineD3D | 32-bit game paths involve WoW64 and, on Apple Silicon, 32-bit translation |

Note the consequence for legacy titles: an old 32-bit D3D7 game on Apple Silicon needs 32-bit x86 translation as well as a DirectDraw wrapper. Rosetta 2 covers 64-bit macOS binaries only, so this is the hardest combination on macOS.

### Step-by-step notes for macOS

1. Identify your CPU with `uname -m`: `x86_64` (Intel) or `arm64` (Apple Silicon). This decides whether Rosetta or an ARM64EC path is involved.
2. Pick a frontend: CrossOver for a supported commercial product, or a Whisky fork plus a Wine build for a community setup.
3. Decide the D3D route: D3DMetal for D3D12 (requires GPTK components), DXMT for D3D10/11 without D3D12, DXVK + KosmicKrisp where a Vulkan path is wanted.
4. Do **not** supply GPTK's D3DMetal binaries to a wrapper unless the wrapper's documentation tells you to; licensing and component pairing matter.
5. For a 32-bit legacy game, remember that Rosetta 2 handles only x86-64 macOS code. A 32-bit Windows game needs Wine's WoW64 plus a 32-bit translation path — the same problem as on ARM Linux ([Section 11.9.3 — 32-bit support at a glance](11-linux-arm.md#1193-32-bit-support-at-a-glance)), with fewer ready-made recipes on macOS.
6. Expect kernel anti-cheat to fail, as on every non-Windows host.

## 12.6 ChromeOS

ChromeOS is a Linux-kernel-based operating system whose desktop is the Chrome browser. Games reach it through virtual machines and containers, not through a native Wine desktop.

- **[Crostini](https://chromeos.dev/en/linux/linux-on-chromeos-deep-dive)** runs Linux apps in a VM (using [crosvm](https://github.com/google/crosvm) over [KVM](https://www.linux-kvm.org/)) plus a container. Wayland and X11 reach the host display through Sommelier, ChromeOS's display bridge. Google describes graphics acceleration as **no longer supported** and Baguette as the default for **new installs from M147**, controlled by `#containerless-crostini` ([ChromeOS developer news](https://developers.google.com/chromeos/app-development/develop/news)).
- **Borealis** was Google's managed Steam environment: Steam in an Arch VM with the [Venus](https://docs.mesa3d.org/drivers/venus.html) virtual Vulkan driver for near-native performance. Its managed beta ended on **1 January 2026**; DIY Crostini is the remaining route ([ChromeOS Steam announcement](https://chromeos.dev/en/posts/bringing-steam-to-chromeos)).
- **[VirGL](https://docs.mesa3d.org/drivers/virgl/)** translates guest OpenGL to host OpenGL; **Venus** carries guest Vulkan to the host Vulkan driver. They are *virtualization/paravirtualization* layers, not alternatives to DXVK. A typical chain is: DXVK produces Vulkan in the guest, Venus transports it to the host ChromeOS driver.
- **Steam for Chromebook** had a hardware floor of an i3/Ryzen 3 class CPU, 8 GB RAM and 128 GB storage, on a limited set of devices ([Google support](https://support.google.com/chromebook/answer/14220699)); Google still labels it Beta with an explicit, changing supported-device list.

### Step-by-step notes for ChromeOS

1. Enable the Linux development environment and confirm whether GPU acceleration is actually exposed; without it, Vulkan-heavy games are impractical.
2. Install a distribution in Crostini with Wine/Proton, plus FEX or Box64 if the host CPU is ARM.
3. Check which Vulkan ICD the *container* sees. A device visible to ChromeOS is not automatically visible inside the VM/container.
4. Use DXVK for D3D8–11 and VKD3D-Proton for D3D12 as usual; Venus carries the resulting Vulkan to the host.
5. Treat Crostini-based Windows gaming as unsupported and device-specific: a ChromeOS update can change the graphics path.

### 12.6.1 VirGL versus Venus

These two are often mentioned together, but they are different layers and neither replaces a Direct3D translator.

| Layer | Traffic it carries | Where it runs | Trade-off |
|---|---|---|---|
| VirGL | Guest OpenGL → host OpenGL | Mesa virtio_gpu / virglrenderer in the VM | Double shader compilation; GL-only |
| Venus | Guest Vulkan → host Vulkan | Mesa Venus driver + validated host driver | Production-capable; needs a compatible host Vulkan driver |
| DRM Native Context | Native driver commands passed through | Same GPU driver family on both sides | Lower overhead; limited to matching driver families |

A typical chain therefore composes rather than competes: the game calls D3D11, DXVK emits Vulkan in the guest, Venus transports that Vulkan to the host, and the host driver executes it. If the host driver is incompatible or acceleration is disabled, DXVK will report no usable Vulkan device — the fix belongs in the ChromeOS/VM configuration, not in DXVK.

## 12.7 Decision matrix by host

| Host | CPU translation | API compatibility | Graphics path | GPU driver | Relative difficulty |
|---|---|---|---|---|---|
| x86_64 Linux | None | Wine / Proton | DXVK / VKD3D-Proton → Vulkan | RADV, ANV, NVIDIA | Lowest |
| Intel macOS | None for x86-64 code | Wine / CrossOver / Whisky | D3DMetal, DXMT, DXVK→MoltenVK | Metal via Apple drivers | Moderate; OpenGL 4.1 cap |
| Apple Silicon macOS | Rosetta 2 or ARM64EC+FEX | Wine / CrossOver / Whisky | D3DMetal, DXMT, DXVK→KosmicKrisp | Metal via Apple drivers | Moderate–high; license-gated DX12 |
| ChromeOS (x86) | None; VM/container instead | Wine / Proton in Crostini | DXVK/VKD3D → Venus/VirGL | Host ChromeOS driver | High; Borealis ended |
| ChromeOS (ARM) | FEX/Box64 in Crostini | Wine / Proton in Crostini | DXVK/VKD3D → Venus | Host ChromeOS driver | Highest; unsupported |
| ARM Linux (module 11) | FEX / Box64, muvm on 16K | Wine / Proton | DXVK / VKD3D-Proton → Vulkan | Mesa AGX, V3D, Freedreno, Panfrost | Moderate–high |
| Android (modules 09–10) | FEX / Box64 in a container | Wine bundled by a frontend | DXVK / VKD3D → Vulkan | Turnip or vendor ICD | High; app sandbox |

Reading the matrix:

- **CPU translation is needed on all ARM hosts and on Apple Silicon macOS — not on x86 Linux or Intel macOS.**
- **The most direct D3D12 path is macOS D3DMetal**, but it is license-gated through GPTK. The most open path is DXVK/VKD3D-Proton → Vulkan on Linux.
- **The best GPU story is x86 Linux**, because drivers are mature and conformant; among ARM, Asahi AGX and Raspberry Pi V3D/V3DV lead.
- **Anti-cheat is the worst on ARM Linux and blocked everywhere off Windows.** Plan around single-player titles.
- **A VM/container is not a CPU translator.** Crostini and Borealis virtualize or isolate the environment; Venus/VirGL translate graphics traffic across that boundary.

## 12.8 What to remember

- x86_64 Linux removes the CPU-translation layer entirely; the remaining layers are Wine/Proton, DXVK/VKD3D-Proton and the GPU driver.
- Wine 11.0 and Proton 11.0 are distributions and API layers, not separate translation engines; ntsync is automatic on kernel 6.14+ and disabled with `PROTON_NO_NTSYNC`.
- Frontends (Lutris, Bottles, Heroic, Steam Play) orchestrate components; MangoHud, Gamescope and GameMode measure, present and schedule — none replaces Wine or a driver.
- On macOS, keep three translations apart: Rosetta 2 (CPU, x86-64 macOS binaries), Wine (Windows APIs), D3DMetal/DXMT/DXVK (graphics). Apple OpenGL 4.1 makes WineD3D legacy-only.
- D3DMetal is the strongest D3D12 route on macOS but needs GPTK components; DXMT has no D3D12; MoltenVK is deprecated and KosmicKrisp is preferred on Apple Silicon.
- ChromeOS runs games through Crostini's VM/container; Borealis ended 1 January 2026. Venus and VirGL move guest graphics to the host and do not replace DXVK.
- Verify the Vulkan device with `vulkaninfo --summary` before debugging translators, and match D3D12's extra Vulkan requirements.
- Kernel anti-cheat remains a blocker on every non-Windows host.

Next: [13 — Performance](13-performance.md).

## References

- Wine: https://gitlab.winehq.org/wine/wine ; Wine 11.0 announcement: https://www.winehq.org/announce/11.0
- Proton: https://github.com/ValveSoftware/Proton ; Steam Proton docs: https://partner.steamgames.com/doc/steamhardware/proton
- Proton-GE: https://github.com/GloriousEggroll/proton-ge-custom
- Linux kernel ntsync docs: https://docs.kernel.org/userspace-api/ntsync.html
- DXVK: https://github.com/doitsujin/dxvk
- VKD3D-Proton: https://github.com/HansKristian-Work/vkd3d-proton
- VKD3D (upstream): https://gitlab.winehq.org/wine/vkd3d
- D7VK: https://github.com/WinterSnowfall/d7vk ; cnc-ddraw: https://github.com/FunkyFr3sh/cnc-ddraw
- Mesa: https://www.mesa3d.org/ ; RADV: https://docs.mesa3d.org/drivers/radv.html ; ANV: https://docs.mesa3d.org/drivers/anv.html ; NVK: https://docs.mesa3d.org/drivers/nvk.html ; Zink: https://docs.mesa3d.org/drivers/zink.html
- NVIDIA Vulkan driver: https://developer.nvidia.com/vulkan-driver
- Vulkan-Tools (`vulkaninfo`): https://github.com/KhronosGroup/Vulkan-Tools ; Vulkan loader docs: https://docs.vulkan.org/guide/latest/loader.html
- Steam Play / Proton docs: https://partner.steamgames.com/doc/steamhardware/proton
- Lutris: https://lutris.net/ ; Bottles: https://github.com/bottlesdevs/Bottles ; Heroic: https://heroicgameslauncher.com/ ; ProtonUp-Qt: https://github.com/DavidoTek/ProtonUp-Qt
- MangoHud: https://github.com/flightlessmango/MangoHud ; Gamescope: https://github.com/ValveSoftware/gamescope ; GameMode: https://github.com/FeralInteractive/gamemode
- Apple Rosetta 2: https://developer.apple.com/documentation/apple-silicon/about-the-rosetta-translation-environment ; Apple developer docs: https://developer.apple.com/
- Apple Game Porting Toolkit: https://developer.apple.com/games/game-porting-toolkit
- Mesa source (KosmicKrisp upstream): https://gitlab.freedesktop.org/mesa/mesa
- Microsoft WoW64 documentation: https://learn.microsoft.com/en-us/windows/win32/winprog64/running-32-bit-applications
- CrossOver: https://www.codeweavers.com/crossover
- DXMT: https://github.com/3Shain/dxmt ; MoltenVK: https://github.com/KhronosGroup/MoltenVK ; Whisky (frankea fork): https://github.com/frankea/Whisky
- ChromeOS Linux deep dive: https://chromeos.dev/en/linux/linux-on-chromeos-deep-dive ; Steam on ChromeOS: https://chromeos.dev/en/posts/bringing-steam-to-chromeos
- crosvm: https://github.com/google/crosvm ; KVM: https://www.linux-kvm.org/
- Venus: https://docs.mesa3d.org/drivers/venus.html ; VirGL: https://docs.mesa3d.org/drivers/virgl/
- FEX: https://github.com/FEX-Emu/FEX ; Box64: https://github.com/ptitSeb/box64
