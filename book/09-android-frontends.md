# 09 — Android Frontends

An Android Windows-gaming **frontend** is an app that assembles the seven-layer pipeline from modules 02–08 into one installable package: a launcher, a settings UI, a packaged Linux userspace, and a curated set of translators and drivers. It is **not** itself a Windows compatibility layer, a CPU translator, a graphics translator, or a GPU driver. Confusing the frontend with the components it bundles is the single biggest source of “it works in app X but not app Y” confusion.

Recall from [module 01](01-architecture.md) that Android is Linux-*kernel*-based but not a normal Linux desktop: it uses Bionic libc, [SurfaceFlinger](https://source.android.com/docs/core/graphics) for composition, scoped storage, SELinux and a restricted linker namespace. None of that changes the contracts a Windows `.exe` expects, so all seven layers are still required. A frontend simply ships them and wires them together.

The table below is a **snapshot dated 2026-11-23**. Cells backed by a primary repository are identified as such; cells marked `[NEEDS VERIFICATION]` are brief-reported only. This field moves quickly: check the upstream release page before making decisions.

## 9.1 Frontend versus compatibility layer

| Role | What it does | What it is NOT | Android examples |
|---|---|---|---|
| **Frontend** (this module) | Orchestrates containers, shortcuts, env vars, input mapping, store login, driver selection, logs | Not a Windows API implementation; not a translator; not a driver | Winlator, DroidDeck, GameHub, GameNative |
| **API compatibility** | Implements Windows APIs on a POSIX/Linux userspace | Does not translate CPU instructions | [Wine](https://gitlab.winehq.org/wine/wine), [Proton](https://github.com/ValveSoftware/Proton) |
| **ABI bridge** | Lets x86-64 and ARM64 code coexist in one process | Not a CPU translator by itself | ARM64EC, WoW64, Hangover DLLs |
| **CPU translation** | Decodes guest x86/x86-64 instructions and emits ARM64 | Does not implement Windows APIs | [Box64](https://github.com/ptitSeb/box64), [Box86](https://github.com/ptitSeb/box86), [FEX](https://github.com/FEX-Emu/FEX) |
| **Graphics translation** | Converts Direct3D/OpenGL calls to Vulkan or OpenGL | Not a GPU driver; not a Windows API layer | [DXVK](https://github.com/doitsujin/dxvk), [VKD3D-Proton](https://github.com/HansKristian-Work/vkd3d-proton), [D8VK](https://github.com/AlpyneDreams/d8vk), [D7VK](https://github.com/WinterSnowfall/d7vk), [WineD3D](https://gitlab.winehq.org/wine/wine) |
| **GPU driver** | Compiles Vulkan/GL to GPU machine code and presents frames | Not a translator | [Turnip / Freedreno](https://docs.mesa3d.org/drivers/freedreno.html), Qualcomm proprietary driver, [VirGL](https://docs.mesa3d.org/drivers/virgl.html), [Zink](https://docs.mesa3d.org/drivers/zink.html) |
| **Container / RootFS** | Provides a Linux userspace with libc, loaders and libraries | Not a VM; no separate kernel | [proot](https://github.com/PRoot/proot), [proot-distro](https://github.com/termux/proot-distro), ImageFS |
| **Display** | Presents the guest's X11/Wayland surface through Android | Not part of Wine | Embedded XServer, [gamescope](https://github.com/ValveSoftware/gamescope) |

The practical consequence: when a game crashes, the fault can be in the frontend's configuration, the bundled Wine version, the translator build, the graphics translator, or the driver. Record the frontend version *and* the exact component versions it bundled before reporting a bug. A frontend update that changes its bundled Wine can “fix” or break a game without any frontend code changing.

```text
 Android app (frontend)  <-- orchestration only
   +-- RootFS / container ......... proot, ImageFS
   +-- Windows API compat ......... Wine or Proton
   +-- ABI bridge ................. ARM64EC / WoW64 / Hangover
   +-- CPU translation ............ Box64 or FEX
   +-- Graphics translation ....... DXVK / VKD3D-Proton / D8VK / WineD3D
   +-- GPU driver ................. Turnip / Qualcomm (runtime)
   +-- Display .................... XServer / gamescope -> SurfaceFlinger
```

This is a conceptual map of responsibilities, not a literal process tree — several roles are provided by one process or one package, and they run concurrently.

## 9.2 Comparison matrix

“Root” means *device root access required for normal use*. All of these apps can render without root; system-wide GPU driver replacement is a separate, root-requiring operation that none of them require.

| Frontend | License | Status (2026-11-23) | Root | Key bundled components | Display | Container |
|---|---|---|---|---|---|---|
| **Winlator** | LGPL-2.1 per live repo (older sources say MIT) | Active, v11.2.0 | No | Wine ARM64EC + Box64, Turnip, Vortek, DXVK/VKD3D/D8VK, DDraw wrappers, Ubuntu Focal RootFS | Embedded XServer | proot |
| **GameHub** | Closed source, free + IAP | Active; last verified release v6.3.2 (2025-10-30) | No | Winlator-derived + Proton, Wine/Box64, DXVK/Turnip/Zink, ImageFS | Frontend-managed | ImageFS |
| **GameHub Lite** | Mod of a proprietary APK; redistribution rights [NEEDS VERIFICATION] | v5.1.7 | No | Same as GameHub, minus analytics/permissions | Same | Same |
| **DroidDeck** | GPL-3.0 | Active, v0.7.1 | No | ARM64 Steam + DroidDeck-Proton (GE-Proton + Valve Wine + CachyOS fixes), FEX, gamescope, KDE Plasma, FEX Ubuntu 24.04 x86-64 RootFS, Turnip bundles, SteamRT4 | gamescope / Steam Big Picture (Wayland optional) | proot (rootless; “run without proot” test builds) |
| **GameNative** | GPL-3.0 | Active, v1.4.0 | No | Proton rebuild, FEX fork, gamescope, Turnip bundles, DXVK, VKD3D-Proton, XServer, Ubuntu RootFS, ntsync-android, lsfg-vk | XServer + gamescope | RootFS + prefix, proot-style [NEEDS VERIFICATION] |
| **Mobox** | No LICENSE file [NEEDS VERIFICATION] | Archived 2025-06-20; last commit 2024-11-23 | No (root optional) | Termux glibc + Box64/Box86, Wine/wine-ge, DXVK variants, VKD3D, D8VK, Mesa VirGL/Turnip/Zink | Termux:X11 + Input Bridge | None (Termux native/glibc prefix) |
| **Termux** | GPL-3.0-only | Active, v0.118.3 stable | No | Terminal + apt only; no Wine by default; add-ons supply the rest | Optional Termux:X11 | Optional proot |
| **ExaGear** | Proprietary / commercial | Discontinued 2019-02-28 | No | Wine + proprietary x86 translator, OBB RootFS, VirGL | Custom surface | Linux container in OBB |
| **Cassia** | Intended closed; no LICENSE | Cancelled 2024-06; never released | No (planned) | Wine + FEXCore, DXVK/VKD3D, Mesa, libadrenotools, gamescope, XServer | gamescope (planned) | Custom runtime |
| **Pluvia** | GPL-3.0 | Retired; last v1.3.2 | No | Embedded Winlator backend, Ubuntu RootFS, Wine/Box64/DXVK/Turnip | Winlator XServer | proot |
| **WinNative** | GPL-3.0 [NEEDS VERIFICATION — official upstream URL not in supplied research] | [NEEDS VERIFICATION] | — | — | — | — |
| **Bannerlator** | GPL-3.0 (v3.1.8) [NEEDS VERIFICATION — official upstream URL not in supplied research] | Active-ish [NEEDS VERIFICATION] | No | Winlator fork; Banners-Turnip driver builds | Winlator XServer | proot |
| **MiceWine** | MIT | Archived [NEEDS VERIFICATION — official upstream URL not in supplied research] | No | Wine/Box64 binaries | [NEEDS VERIFICATION] | [NEEDS VERIFICATION] |
| **Vectras VM** | Closed app | Status unverified | No | [QEMU](https://www.qemu.org/) full-system emulation | Emulated display | Full VM |

Some components named above (WinNative, MiceWine, Vectras VM, Vortek, ImageFS, ntsync-android, lsfg-vk and libadrenotools) lack a verified primary URL in the sources used here; treat them as unverified until checked upstream. Bannerlator's confirmed link is its driver repo, [Banners-Turnip](https://github.com/The412Banner/Banners-Turnip), not the frontend itself.

## 9.3 The maintained frontends in detail

### Winlator

[Winlator](https://github.com/brunodev85/winlator) ([site](https://winlator.org/)) is the most configurable general-purpose frontend and the reference point for most others; a community fork, [Winlator WB64](https://github.com/winebox64/winlator), tracks Wine/Box64 changes. It targets Android 9+ ARM64 devices with Adreno 610+ and Vulkan 1.1+ (information from the project repository). It bundles Wine in ARM64EC form with Box64 as the CPU translator, DXVK/VKD3D/D8VK and DirectDraw wrappers for graphics, and an Ubuntu Focal RootFS under proot. Settings are per-container and per-shortcut: Box64 presets, driver import, environment variables, touch controls and gamepad mapping. Its embedded XServer presents directly through Android.

The winlator.org domain is the original project site; similarly named domains are not necessarily affiliated (§9.7).

### DroidDeck

[DroidDeck](https://github.com/Droid-Deck/DroidDeck) ([site](https://droid-deck.dev)) is [Steam](https://store.steampowered.com/)-first: it runs an ARM64 Steam client with a custom Proton build (“DroidDeck-Proton”, derived from [GE-Proton](https://github.com/GloriousEggroll/proton-ge-custom), Valve Wine and [CachyOS](https://cachyos.org/) fixes), [FEX](https://github.com/FEX-Emu/FEX) for CPU translation, gamescope and [KDE Plasma](https://kde.org/plasma-desktop/) for a Big-Picture-style desktop, and an FEX-provided Ubuntu 24.04 x86-64 RootFS. It targets Adreno 7xx/8xx (6xx experimental, 710 unsupported; Mali/PowerVR experimental in 0.7.0+). Drivers are published separately in [Droid-Deck/Drivers](https://github.com/Droid-Deck/Drivers) as weekly Mesa bundles, with separate runtime and display driver replacement. Build recipes use `tools/build_local.sh` (§9.6).

### GameNative

[GameNative](https://github.com/utkarshdalal/GameNative) ([site](https://gamenative.app/)) is the successor to Pluvia (§9.4). It uses a Proton rebuild plus an [FEX](https://github.com/FEX-Emu/FEX) fork, with gamescope enabled by default on Adreno as of v1.4.0 — a deliberate breaking change from XServer-only presentation. It ships manifest-pinned Turnip builds, DXVK 3.1.1 and VKD3D-Proton 3.1, an Ubuntu RootFS, an Android ntsync implementation and an lsfg-vk frame-generation component. It supports Steam, Epic, GOG and Amazon store integrations, cloud saves, a HUD and a compatibility database. Privacy note: launch events are sent to [PostHog](https://posthog.com/) analytics (§9.7).

### GameHub

[GameHub](https://gamehub.xiaoji.com) is a closed-source, free-with-IAP frontend from GameSir / Guangzhou Chicken Run Network Tech with Steam and Epic integration, cloud profiles and in-app driver downloads. It is Winlator-derived and bundles Proton/Wine with [Box64](https://github.com/ptitSeb/box64), DXVK and Turnip/Zink. The research brief does not state official Android/ABI requirements — [NEEDS VERIFICATION — check the store listing]. It requires an account, broad permissions and ships trackers; see §9.7. [GameHub Lite](https://github.com/Producdevity/gamehub-lite) is a community-modified APK with analytics and permissions stripped; its redistribution rights are [NEEDS VERIFICATION], and it is not on the Play Store.

### Mobox

[Mobox](https://github.com/olegos2/mobox) is a Termux-based environment rather than a conventional APK frontend: it runs a Termux glibc prefix with Box64/Box86, Wine/wine-ge, DXVK variants, VKD3D, D8VK and Mesa VirGL/Turnip/Zink, presenting through Termux:X11 plus an Input Bridge. It was archived on 2025-06-20 (last commit 2024-11-23) and has a community remake. The repository has no LICENSE file — [NEEDS VERIFICATION — do not assume a license]. It targets Android 10+ ARM64, Adreno 6xx and 725–740. Treat it as a historical reference for the DIY approach in [module 10](10-android-diy-termux.md), not a current recommendation.

### Termux

[Termux](https://termux.dev/) is not a Windows frontend at all; it is a terminal emulator and Linux-package environment built on Android's userspace ([repo](https://github.com/termux/termux-app), [F-Droid](https://f-droid.org/en/packages/com.termux/)). You add Wine, a translator, graphics libraries and a display server yourself. That is the subject of module 10.

## 9.4 Dead, cancelled and retired projects

| Project | Fate | Why it matters |
|---|---|---|
| [ExaGear](https://exagear.wiki/) | Discontinued 2019-02-28 | Proprietary 32-bit x86-on-ARM translator with Wine, an OBB RootFS and VirGL. License checks now fail and old installers are a security risk. Historically 2D/strategy games only. Do not install. |
| [Cassia](https://github.com/cassia-org) | Cancelled 2024-06, never released | Planned Wine + FEXCore + DXVK/VKD3D + Mesa + libadrenotools + gamescope stack from ex-Skyline developers. Code archaeology only — no supported download exists. |
| [Pluvia](https://github.com/oxters168/Pluvia) ([site](https://gopluvia.com)) | Retired, last v1.3.2 | GPL-3.0 Steam client using an embedded Winlator backend. Its successor is GameNative. |

A frontend that still has a downloadable APK is not necessarily maintained. Check the *last commit and release date*, not the store rating.

## 9.5 Turnip, drivers and Adreno 730 guidance

[Turnip](https://docs.mesa3d.org/drivers/freedreno.html) is Mesa's open-source Vulkan driver for Adreno 6xx/7xx/8xx (Vulkan 1.3 on A6xx, 1.4 on A7xx; no A5xx). [Freedreno](https://docs.mesa3d.org/drivers/freedreno.html) is its OpenGL companion; [Zink](https://docs.mesa3d.org/drivers/zink.html) implements OpenGL on top of Vulkan. These are *drivers*, not translators — they consume the Vulkan/GL commands that DXVK or WineD3D produce. The reference device in this book is a Snapdragon 8+ Gen 1 with an **Adreno 730**; the following guidance applies to that generation.

**Runtime driver versus display driver.** The game's userspace driver (e.g. Turnip injected via [AdrenoTools](https://github.com/K11MCH1/AdrenoToolsDrivers/releases)) is not the driver Android uses to draw the UI (SurfaceFlinger/Hardware Composer with the stock Qualcomm driver). Keep the stock Qualcomm driver for the system UI and inject Turnip only into the game. A UBWC (tile-buffer compression) format mismatch between the two is a reported cause of flicker [NEEDS VERIFICATION — community report; exact flag varies by build].

**Frontend-specific driver handling:**

- Winlator: in-app driver picker, AdrenoTools injection, and knobs such as `TU_DEBUG`, `FD_DEV_FEATURES` and `MESA_EXTENSION_MAX_YEAR`.
- DroidDeck: weekly Mesa bundles; `TU_DEBUG=sysmem` by default except on A710/A720; separate runtime and display driver replacement.
- GameNative: manifest-pinned Turnip Gen8 V38 and v27.0.0-R1; gamescope default on Adreno.

**Adreno 730 recipe (conservative):**

1. Keep stock Qualcomm for Android; inject Turnip per app, do not replace the system driver system-wide.
2. Use a stable upstream Mesa release (26.2.x at the snapshot date) or a community build that explicitly names its upstream commit.
3. Prefer plain “A6xx/A7xx Standard” builds over specialised variants.
4. Do **not** apply the A740 UBWC hint to a 730 — different GPU, and a known flicker trigger on A740 [NEEDS VERIFICATION — community report].
5. Avoid re-uploaded ZIPs with no commit hash; mislabelled builds for a different Adreno generation are common.
6. Community reports favour Turnip 26.0.0 R8 and Qualcomm v615.77 [NEEDS VERIFICATION — anecdotal preference, not a primary-source recommendation].

Roll back one known-good build rather than chasing “latest”, and change either the translator or the driver in a given test — never both.

### Legacy Direct3D 7/8/9 in frontends

Modern DXVK covers D3D9–11 and VKD3D-Proton covers D3D12, but older games are a different decision tree (module 06). Winlator bundles [D8VK](https://github.com/AlpyneDreams/d8vk) for D3D8, [D7VK](https://github.com/WinterSnowfall/d7vk) and DirectDraw wrappers for D3D7/DirectDraw-era titles, and WineD3D (Wine's OpenGL backend) remains a fallback when a Vulkan path is unavailable. When a 1999–2002 game fails in a frontend, check *which* of these the bundle actually provides — an older DXVK build without its D3D8 component cannot translate D3D8 — then check the bundled DXVK version and whether `d3d8.dll` is present, since current DXVK includes D8VK support. The same override logic applies inside [module 10](10-android-diy-termux.md)'s DIY stack.

## 9.6 Building from source

Building a frontend is mostly a packaging problem: native libraries for the guest architecture, a RootFS image, and the Android app shell.

| Project | Build path | Notable requirements |
|---|---|---|
| Winlator | Android Studio / Gradle | NDK 29, CMake, git submodules; ImageFS/RootFS packaging |
| GameNative | Android Studio | `git submodule update --init --recursive`; ubuntufs manifests |
| DroidDeck | `tools/build_local.sh` | Docker, Java 17, SDK/NDK, Rust, Go, MinGW-w64 x86_64+i686 DWARF-2, GitHub CLI, Python, zstd; binfmt for arm64 on x86_64 |
| Termux | Gradle + bootstrap from [termux-packages](https://github.com/termux/termux-packages) | Android NDK/toolchain |

Common failures: uninitialised submodules, NDK/Android Gradle Plugin version drift, RootFS/OBB signature mismatch between app and image, and (at runtime) Android's phantom process killer ([module 08 §8.10](08-runtime-prefixes.md)). Official NDK/CMake documentation: https://developer.android.com/ndk/guides/cmake. Build-from-source is optional; for most users an official signed release is safer than a self-signed local build (§9.7).

## 9.7 Security warnings

- **Fake APKs.** Download only from primary repositories or official sites. The original Winlator site is winlator.org; fake `winlator.com`, `winlator.dev` and similar domains have been reported. Treat any non-primary domain as untrusted. A malicious APK has full access to your storage and accounts.
- **Termux signing key.** Termux GitHub release APKs use a public *test key*. Prefer the [F-Droid build](https://f-droid.org/en/packages/com.termux/), and never mix F-Droid, GitHub and Play Store sources: Termux and its plugins share a `sharedUserId`, so signature mismatches break the installation and can be exploited by an impostor app.
- **Mod APKs.** GameHub Lite and any other patched APK are untrusted third-party binaries. You are trusting the modifier not to add code; redistribution rights are often unclear.
- **Analytics and accounts.** GameHub requires an account, broad permissions and ships trackers. GameNative discloses PostHog launch analytics. Read the privacy policy before logging in.
- **Store credentials.** If a frontend asks you to log in to Steam/Epic/GOG, prefer QR-code login rather than typing a password into a third-party client. Enable two-factor authentication on the store account.
- **Legal.** Respect the licences: GPL-3.0 (DroidDeck, GameNative, Pluvia, Termux), LGPL-2.1 (Winlator, per live repo), MIT (MiceWine), and proprietary (GameHub, ExaGear, Vectras). Do not use these tools for piracy or anti-cheat circumvention.

## 9.8 How to choose

| Situation | Sensible first choice | Reason |
|---|---|---|
| Adreno 7xx/8xx, want a working setup fast | GameNative or DroidDeck | FEX + gamescope path, pinned drivers, store integration |
| Adreno 7xx/8xx, want maximum control | Winlator | Per-shortcut settings, driver picker, large community |
| Adreno 6xx / 725–740 | Winlator; Mobox as historical reference | Mature A6xx support |
| Mali / PowerVR | Winlator (VirGL/Vortek), MiceWine | Lower performance; Vulkan driver support is weaker |
| Want to understand or customise every layer | Termux + [module 10](10-android-diy-termux.md) | Full control, more failure modes |
| Want privacy / auditable code | Open-source frontends (GPL/LGPL) | Closed-source apps add trackers and accounts |
| Encounter ExaGear, Cassia or Pluvia | Avoid | Dead, cancelled or retired |

Root is not required for any current frontend. Judge a project by its last release date, licence, and whether it names the exact Wine/translator/driver versions it ships — not by a screenshot of one game running.

**Non-Android hosts.** Frontends are an Android packaging solution. On ARM Linux or x86 Linux the same roles are played by Steam/Proton, Lutris, Bottles or Heroic, which manage prefixes and runtimes directly on a normal Linux userspace with no proot or SurfaceFlinger. On macOS, CrossOver/Whisky fill the equivalent role. See [module 11](11-linux-arm.md) and [module 12](12-linux-x86-and-other-hosts.md).

## 9.9 What happens when you press Launch

A frontend's “play” button does not start a single program. It runs the same staged sequence as [module 02](02-pipeline.md), with the frontend providing the glue between stages:

| Stage | Frontend action | Typical failure |
|---|---|---|
| 1. Frontend | Reads the shortcut's container, Wine version, translator, driver and env vars | Wrong profile selected; a setting not saved |
| 2. Container | Enters the RootFS/ImageFS; prepares the X11 socket and storage | Missing mount, permission denied, storage not granted |
| 3. Wine | Creates or opens the prefix, starts `wineserver`, applies DLL overrides | Corrupt prefix, missing runtime, wrong architecture |
| 4. CPU translation | Starts Box64/FEX, or loads the ARM64EC/WoW64 bridge DLLs | Unsupported instruction, translator bug |
| 5. Graphics | Loads DXVK/VKD3D/D8VK/WineD3D and creates a Vulkan device | Wrong override, missing shader compiler, driver mismatch |
| 6. Driver | Turnip or Qualcomm compiles pipelines and presents | Missing extension, UBWC flicker, device lost |
| 7. Display | XServer/gamescope hands the frame to SurfaceFlinger | Black screen with working audio |

Record these seven things before reporting a bug: frontend version; bundled Wine/Proton version; translator name and version; graphics translator and version; GPU driver build or commit; Android version and device; and the exact game build. A report that says “game X doesn't work on frontend Y” cannot be acted on — the same game may work with a different bundled Wine.

### Emulation, virtualization and translation — a terminology check

Vectras VM (§9.2) is a different category from the other entries: it uses [QEMU](https://www.qemu.org/) full-system emulation, which reproduces a complete machine including CPU and devices. That is **emulation**, not API compatibility. A container such as proot shares the host kernel and fakes a filesystem view; a hardware-assisted VM (KVM) is **virtualization**. Neither is what Wine does: Wine is an **API compatibility layer**, and Box64/FEX are **CPU translators**. ABI compatibility (ARM64EC, WoW64) is a fourth, narrower idea: it lets two instruction sets share one process. Confusing these categories leads to wrong fixes — installing a guest Windows image or a “BIOS” cannot repair a missing `d3dx9_43.dll`.

## 9.10 What to remember

- A frontend orchestrates; Wine/Proton provide Windows APIs, Box64/FEX translate CPU instructions, DXVK/VKD3D translate graphics, and Turnip/Qualcomm are the drivers.
- The comparison matrix is a snapshot; verify licence, status and bundled versions upstream.
- Root is not needed for current frontends; system-wide driver replacement is a separate, riskier operation.
- For Adreno 730, keep the stock driver for UI, inject a stable Turnip per app, and do not copy A740-specific hacks.
- Old games may need D8VK/D7VK/WineD3D, not just DXVK — check what the bundle contains.
- Avoid dead projects, fake APKs, mod APKs and mixed Termux sources; prefer auditable open-source releases.

Next: [10 — Android DIY: The Termux Pipeline](10-android-diy-termux.md).

## References

- Winlator: https://github.com/brunodev85/winlator ; https://winlator.org/ ; WB64 fork https://github.com/winebox64/winlator
- GameHub: https://gamehub.xiaoji.com ; GameHub Lite https://github.com/Producdevity/gamehub-lite
- DroidDeck: https://github.com/Droid-Deck/DroidDeck ; https://droid-deck.dev ; drivers https://github.com/Droid-Deck/Drivers
- GameNative: https://github.com/utkarshdalal/GameNative ; https://gamenative.app/ ; analytics https://posthog.com/
- Mobox: https://github.com/olegos2/mobox
- Termux: https://termux.dev/ ; https://github.com/termux/termux-app ; F-Droid https://f-droid.org/en/packages/com.termux/ ; Termux:X11 https://github.com/termux/termux-x11
- ExaGear: https://exagear.wiki/ ; Cassia https://github.com/cassia-org ; Pluvia https://github.com/oxters168/Pluvia ; https://gopluvia.com
- Banners-Turnip: https://github.com/The412Banner/Banners-Turnip
- Wine: https://gitlab.winehq.org/wine/wine ; Proton https://github.com/ValveSoftware/Proton
- CPU translation: Box64 https://github.com/ptitSeb/box64 ; Box86 https://github.com/ptitSeb/box86 ; FEX https://github.com/FEX-Emu/FEX
- Graphics translation: DXVK https://github.com/doitsujin/dxvk ; VKD3D-Proton https://github.com/HansKristian-Work/vkd3d-proton ; D8VK https://github.com/AlpyneDreams/d8vk ; D7VK https://github.com/WinterSnowfall/d7vk
- Drivers: Freedreno/Turnip https://docs.mesa3d.org/drivers/freedreno.html ; VirGL https://docs.mesa3d.org/drivers/virgl.html ; Zink https://docs.mesa3d.org/drivers/zink.html ; AdrenoTools https://github.com/K11MCH1/AdrenoToolsDrivers/releases
- Android graphics: https://source.android.com/docs/core/graphics
- Container/display: proot https://github.com/PRoot/proot ; proot-distro https://github.com/termux/proot-distro ; gamescope https://github.com/ValveSoftware/gamescope ; QEMU https://www.qemu.org/
- Build tooling: Android NDK/CMake https://developer.android.com/ndk/guides/cmake ; termux-packages https://github.com/termux/termux-packages
- Non-Android frontends: Lutris https://lutris.net/ ; Bottles https://github.com/bottlesdevs/Bottles ; Heroic https://heroicgameslauncher.com/
