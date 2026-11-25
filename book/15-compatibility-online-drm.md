# 15 — Compatibility, Online Games and DRM

The hardest compatibility problems are not graphics or CPU instructions. They are **trust systems**: software whose job is to verify that the machine running it is a legitimate Windows PC. This module explains which of those systems can work under Wine/Proton, which cannot, and why.

> **Legal and ethical framing.** This book does not provide instructions for bypassing DRM, anti-cheat, licence checks, region locks or kernel driver requirements. Using a compatibility layer is not a licence to evade a game's terms of service. Anti-cheat vendors and publishers decide which platforms to support; modifying components to hide an environment from them can get accounts banned and may be unlawful in some jurisdictions. Use your own legally acquired copies, respect server rules, and do not download "bypass" tools. Where a feature is described as impossible, that is a statement of technical fact, not a challenge.

## 15.1 Terms you need first

| Term | Definition |
|---|---|
| Anti-cheat | Software that detects cheating; may run in userspace (ring 3) or require a kernel driver (ring 0) |
| Ring 0 / ring 3 | CPU privilege levels: ring 0 is kernel mode, ring 3 is ordinary application mode |
| DRM | Digital rights management: technology that restricts copying or running software |
| Launcher | A separate program that authenticates and starts the game (Steam, Epic, Ubisoft Connect, etc.) |
| Userspace component | A normal program or library running with application privileges |
| Kernel driver | Code loaded into the operating system kernel, with full machine privilege |

The distinction between userspace and kernel components is the single most important idea in this module.

## 15.2 What a compatibility layer can and cannot provide

[Wine](https://gitlab.winehq.org/wine/wine) implements Windows **APIs** in userspace. It is not a Windows kernel. It does not implement `NtLoadDriver` in a way that loads a Windows `.sys` file into the host kernel, and it cannot make the host kernel execute Windows kernel code. [Proton](https://github.com/ValveSoftware/Proton) is a Wine-derived distribution with extra libraries and components, not a Windows kernel either.

This produces a hard boundary:

| Request from a game | Can Wine/Proton satisfy it? | Why |
|---|---|---|
| Call Win32 APIs | Yes, mostly | That is Wine's purpose |
| Translate x86 to ARM | Via [FEX](https://github.com/FEX-Emu/FEX) or [Box64](https://github.com/ptitSeb/box64), not via Wine | Separate CPU translation layer |
| Translate D3D to Vulkan | Via [DXVK](https://github.com/doitsujin/dxvk)/[VKD3D-Proton](https://github.com/HansKristian-Work/vkd3d-proton) in Proton, or Wine's own opt-in Vulkan Direct3D renderer (`renderer=vulkan`); none of these involves a Windows kernel | Separate graphics translation layer |
| Load a Windows kernel driver | **No** | No Windows kernel; cannot run `.sys` in host ring 0 |
| Run a vendor-supplied Linux anti-cheat client | Sometimes | Only if the vendor ships and supports one |

A workaround that fakes the *result* of a kernel driver without providing its actual function is a bypass, and is outside this book's scope.

Here is the **trust boundary** as a conceptual diagram. It is a logical picture of where components live, not a literal runtime call order:

```text
+---------------------------------------------------+
| Windows game process (ring 3)                     |
|   calls Win32 APIs, XAudio2, XInput, D3D          |
+------------------------+--------------------------+
                         |
      Wine/Proton implement these APIs (userspace)
                         |
+---------------------------------------------------+
| Host Linux/Android userspace                      |
|   Wine, DXVK/VKD3D, FEX/Box64, audio server       |
+------------------------+--------------------------+
                         |
                  host kernel syscall boundary
                         |
+---------------------------------------------------+
| Host kernel (ring 0)                              |
|   Linux drivers, filesystems, scheduler           |
+---------------------------------------------------+
```

A Windows kernel driver would need to run *inside* the bottom box. Wine lives in the middle box. There is no Windows kernel in the bottom box, so a `.sys` file has nowhere to go. This is a structural limit, not a missing feature that a future Wine release can simply add.

## 15.3 Anti-cheat: the three categories

| Category | Examples | Status on non-Windows hosts |
|---|---|---|
| Opt-in userspace anti-cheat with Linux support | [Easy Anti-Cheat](https://www.easyanticheat.net/) (EAC), [BattlEye](https://www.battleye.com/) | Possible per game, when the developer enables it |
| Kernel-mode anti-cheat | Vanguard, RICOCHET (Call of Duty) | Not possible locally; requires a vendor-supported Windows kernel driver |
| Game-specific or server-side checks | Various | Varies; may work if no Windows-only client component is required |

"Possible" is not "guaranteed". It means the vendor offers a supported path that a developer can choose to enable for a specific title.

## 15.4 Easy Anti-Cheat

EAC is the clearest example of the opt-in model. According to the collected sources, it is possible under Wine/Proton **per game**, not globally:

- The developer must enable the Linux client for that title.
- In Valve's Kamu EAC example, the vendor-supplied Linux library (`libeasyanticheat.so`) is renamed `easyanticheat_x64.so` and shipped beside `EasyAntiCheat_x64.dll`; Epic Online Services EAC uses its own Linux client-module/bootstrapper setup.
- The developer must publish that build.

If any of those is missing, no client-side configuration will make the game work. The player cannot enable EAC Linux support for a title whose developer has not done so. Valve's documented example uses the Kamu EAC variant; Epic Online Services EAC has its own setup instructions.

Epic announced [EAC Linux v2](https://onlineservices.epicgames.com/news/introducing-easy-anti-cheat-linux-v2) on 2026-10-14. It deliberately does **not** install or load an anti-cheat kernel module; instead it relies on signed kernels and modules, TPM 2.0, Secure Boot and measured boot, kernel lockdown, IOMMU support on non-handhelds, disabled kexec and its own eBPF logic. Epic says that the Steam Deck does not support the required features, and that the security features are not available on every distribution and may require configuration changes. Check the announcement and Valve's [Proton documentation for developers](https://partner.steamgames.com/doc/steamhardware/proton) before drawing conclusions about a specific device.

This is an important nuance: "kernel component" in a Linux-native anti-cheat design is not the same as loading a Windows `.sys` file through Wine. If such support exists, it is a **native Linux component provided by the vendor**, used directly — not something Wine translates.

## 15.5 BattlEye

BattlEye is similarly opt-in per title. The brief records that support is enabled through manual configuration between Valve and BattlEye for specific games. As with EAC, the practical question for a player is simply whether the publisher has enabled it; if not, the game will not launch under Proton, and there is no legitimate client-side fix.

Do not trust community lists as authority. Anti-cheat support changes with game patches, and a title that worked last month can stop working after an update.

## 15.6 Vanguard, RICOCHET and kernel-mode anti-cheat

Kernel-mode anti-cheat runs with ring-0 privileges on Windows. Wine cannot host it because there is no Windows kernel to load it into, and Proton cannot grant a Windows driver access to the Linux kernel. Valve states that kernel-space anti-cheat solutions are not currently supported for Steam Play, and [Activision confirms](https://support.activision.com/articles/ricochet-overview) that RICOCHET uses a kernel-level driver on PC. Vanguard and RICOCHET are therefore impossible locally.

Practical consequences:

- The game will usually refuse to start when it cannot initialise its anti-cheat driver.
- Running under proot/chroot, a different Wine build, or a different CPU translator does not change this.
- Attempting to deceive the anti-cheat is both a terms-of-service violation and outside this book's scope.

The same boundary applies on macOS, ChromeOS and ARM Linux. A native Linux build of a game with kernel-mode anti-cheat is a different matter — that is the vendor's own supported product, not a translated Windows binary.

## 15.7 Kernel drivers generally

Legacy games and utilities sometimes install a kernel driver for copy protection, hardware access or low-level anti-cheat. Wine cannot load Windows kernel drivers. There is no general bridge from a Windows `.sys` file to a Linux kernel module.

If a piece of hardware needs a driver, the correct solution is a **native Linux driver** from the vendor or the open-source community. For graphics, that means Mesa or a vendor Linux driver — see module 07. The driver operates below Wine and works with the host kernel; Wine merely sees the resulting device.

## 15.8 Legacy copy protection: SafeDisc, SecuROM, Denuvo

| Protection | Mechanism described in the brief | Status |
|---|---|---|
| SafeDisc | Uses `secdrv.sys`, a kernel driver in older versions | No general support; historically version-specific userspace emulation existed for very old SafeDisc 1.x |
| SecuROM | Ring 3; uses disc subchannel signatures (up to v4.76) or DPM (v5.00+; discs in the v4.76–v5.00 range may use either) | Generally does not work |
| Denuvo | Commercial DRM with no documented general Wine implementation | No general solution documented; outcomes are per title |

These systems predate modern storefront DRM and were designed for physical media and Windows kernel services. Vendor documentation is legacy or archived rather than actively maintained, and current practical detail comes from community references. Outcomes are version-specific, so treat the table as a summary, not a per-title guarantee.

[Denuvo](https://denuvo.com/) is still commercially active, but a "runs under Wine" result for one game says nothing about another. Do not generalise from a single compatibility report.

## 15.9 Online launchers, .NET, WPF and Media Foundation

Modern PC games often depend on a stack of Microsoft components. These are compatibility problems in the ordinary sense — Wine and Proton can often supply them — but they are common sources of failure.

| Component | Notes |
|---|---|
| Separate launchers | Valve recommends skipping separate launchers where possible; use the Steam-provided build when available |
| [.NET Framework](https://dotnet.microsoft.com/) | Known problem area for Proton; [Mono](https://www.mono-project.com/) may be used in some prefixes |
| [WPF](https://learn.microsoft.com/en-us/dotnet/desktop/wpf/) | Windows Presentation Foundation; part of the .NET desktop stack, often a sticking point |
| Wine Gecko | HTML rendering component; see the [Wine Gecko page](https://wiki.winehq.org/Gecko) |
| Visual C++ redistributables | Frequently needed DLLs such as `msvcp140.dll` |
| [Media Foundation](https://learn.microsoft.com/en-us/windows/win32/medfound/microsoft-media-foundation-sdk) | Incomplete in Wine; Proton has `nomfdxgiman`. Wine 10 added an opt-in experimental FFmpeg backend, enabled by setting `DisableGstByteStreamHandler=1` in `HKCU\\Software\\Wine\\MediaFoundation` (current stable is Wine 11.0) |
| FFmpeg | Open-source multimedia framework used by the Wine backend ([FFmpeg](https://ffmpeg.org/)) |

Valve recommends standalone VP9 or AV1 codecs to reduce bandwidth and disk use; this is a developer recommendation, not a claim that Wine's Media Foundation implementation "works best" with them. If a game's cutscenes fail while gameplay works, suspect the multimedia stack rather than the GPU.

First-run installers for these components are also a storage and cache concern; see module 13.

## 15.10 Audio and input APIs

[XAudio2](https://learn.microsoft.com/en-us/windows/win32/xaudio2/xaudio2-introduction) and [XInput](https://learn.microsoft.com/en-us/windows/win32/xinput/xinput-versions) are Windows APIs for game audio and controller input. Wine implements them on top of host audio and input systems; a missing native `xaudio2_7.dll` override is a classic cause of silence.

This is **API compatibility**, not driver emulation. Wine does not emulate an Xbox controller; it maps XInput calls to the host's input devices, and controller mapping is usually supplied by the frontend. On Android, touch controls are a frontend feature layered on top.

For diagnosis, see the symptom table in module 14.

## 15.11 16-bit applications

Some installers and very old games are 16-bit PE or NE executables. Wine 11.0 restored full 16-bit application support in the new WoW64 mode; the experimental Wine 9.0–10.x WoW64 mode lacked it. Do not assume 16-bit works on ARM64 hosts — no primary source confirms that case. If a launcher fails with `Bad EXE format`, check the executable architecture before blaming the prefix.

A 16-bit installer can also be the reason a modern 64-bit game's setup fails, even when the game binary itself is fine. Running the game through Proton's setup steps or using a native installer is often the practical route.

## 15.12 ARM64 and page-size constraints

ARM hosts add a constraint that has nothing to do with Windows APIs: **memory page size**.

| Constraint | Detail from the brief |
|---|---|
| Wine ARM64EC | Initial build support in Wine 9.0; full support with feature parity in Wine 10.0 ([announcement](https://www.winehq.org/announce/10.0)); current stable is Wine 11.0 |
| FEX | Implements the 64-bit x86 interface and can act as a WoW64/ARM64EC backend |
| Host page size | Wine 10.0 required 4K; Wine 11.0 simulates 4K on 16K/64K hosts for simple applications, but a native 4K kernel is strongly recommended |
| Proton ARM64 builds | Cannot simply be used inside an x86 Steam installation via FEX |
| Asahi Linux | 16K host pages; projects such as [muvm](https://github.com/AsahiLinux/muvm) provide a 4K guest microVM |
| Other ARM hosts | Snapdragon-class devices and SBCs vary; see modules 02 and 07 |

The reason is concrete: x86 Windows software assumes 4K pages. Wine 10.0 required a 4K host kernel. Wine 11.0 can simulate 4K pages on 16K/64K hosts for simple applications, but the difference cannot be fully hidden and more demanding programs may still fail. A native 4K kernel is strongly recommended. The robust route on a 16K host such as Asahi is a small 4K guest VM via [muvm](https://github.com/AsahiLinux/muvm).

This is **virtualization or memory-layout emulation**, not CPU translation alone. FEX still translates instructions; the 4K environment is a separate requirement.

## 15.13 A practical pre-flight checklist

Before spending hours tuning graphics settings, answer these questions in order. They move from hard blockers to adjustable details.

| # | Question | If the answer is no |
|---|---|---|
| 1 | Is the game's anti-cheat supported on Linux for this title? | Stop. No local configuration changes this. |
| 2 | Does the game need a Windows kernel driver or legacy disc copy protection? | Stop. Wine cannot load it. |
| 3 | Are your CPU architecture and page size compatible with the chosen Wine/translator setup? | Prefer a 4K-page host; on 16K/64K hosts, try Wine 11.0's 4K simulation or a microVM such as muvm. |
| 4 | Is the game 32-bit, 64-bit or 16-bit? | Match the prefix and Wine architecture to the game; Wine 11.0 supports 16-bit in the new WoW64 mode on x86, but ARM64 is unverified. |
| 5 | Do you have the required runtimes (VC++, .NET, DirectX components)? | Install them into the prefix before blaming the renderer. |
| 6 | Is a separate launcher required? | Prefer the store build that starts the game directly. |
| 7 | Do you have a legally acquired copy and the right to play it on this platform? | Do not proceed. |

Only after all seven answers are acceptable should you move to graphics and performance work in modules 13 and 14. This ordering saves time because the first two items are absolute blockers, while the later items are tunable.

A second habit: after any game update, re-check items 1 and 2. Publishers can change anti-cheat requirements in a patch, which makes a previously working configuration stop overnight. That looks like a graphics regression but is not one.

## 15.14 What to remember

- Wine and Proton provide Windows API compatibility in userspace; they are not Windows kernels.
- CPU translation (FEX/Box64) and graphics translation (DXVK/VKD3D-Proton) are separate concerns.
- EAC and BattlEye are possible only when the developer has enabled them for the specific title.
- Kernel-mode anti-cheat such as Vanguard and RICOCHET cannot run locally on non-Windows hosts; there is no legitimate bypass.
- Windows kernel drivers generally cannot be loaded through Wine; use native Linux drivers instead.
- SafeDisc, SecuROM and Denuvo have no general solution; outcomes are per title.
- .NET, WPF, launchers and Media Foundation are ordinary compatibility problems, but frequent ones.
- Wine 11.0 restored 16-bit support in the new WoW64 mode; page-size mismatches still constrain ARM64 setups.
- Respect terms of service and do not use bypass tools.

Next: return to [01 — Architecture](01-architecture.md) or continue with the host-specific modules.

## References

- Wine — https://gitlab.winehq.org/wine/wine ; Wine 10.0 — https://www.winehq.org/announce/10.0 ; Wine 11.0 — https://www.winehq.org/announce/11.0 ; Gecko — https://wiki.winehq.org/Gecko
- Proton — https://github.com/ValveSoftware/Proton ; Valve Proton developer documentation — https://partner.steamgames.com/doc/steamhardware/proton
- FEX — https://github.com/FEX-Emu/FEX ; Box64 — https://github.com/ptitSeb/box64
- DXVK — https://github.com/doitsujin/dxvk ; VKD3D-Proton — https://github.com/HansKristian-Work/vkd3d-proton
- Easy Anti-Cheat — https://www.easyanticheat.net/ ; Epic EAC Linux v2 announcement — https://onlineservices.epicgames.com/news/introducing-easy-anti-cheat-linux-v2
- BattlEye — https://www.battleye.com/ ; Activision RICOCHET — https://support.activision.com/articles/ricochet-overview
- Denuvo — https://denuvo.com/ ; SecuROM legacy FAQ — https://support.securom.com/faq_general.html
- .NET — https://dotnet.microsoft.com/ ; Mono — https://www.mono-project.com/ ; WPF — https://learn.microsoft.com/en-us/dotnet/desktop/wpf/
- Media Foundation — https://learn.microsoft.com/en-us/windows/win32/medfound/microsoft-media-foundation-sdk ; FFmpeg — https://ffmpeg.org/
- XAudio2 — https://learn.microsoft.com/en-us/windows/win32/xaudio2/xaudio2-introduction ; XInput — https://learn.microsoft.com/en-us/windows/win32/xinput/xinput-versions
- muvm — https://github.com/AsahiLinux/muvm ; Asahi Linux — https://asahilinux.org/
