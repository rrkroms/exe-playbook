# Research Brief — Android Windows-Gaming Frontends and Tools (Nov 2026)

## 1. Comparison matrix

| Name | Dev / maintainer | Official repo/site | License | Status/latest | Root? | Android/ABI | Bundled components | Display | Container | Notes |
|---|---|---|---|---|---|---|---|---|---|---|
| **Winlator** | BrunoSX (brunodev85); WB64 fork by winebox64 | github.com/brunodev85/winlator; winlator.org | LGPL-2.1 per live repo (older sources say MIT); WB64 Apache-2.0 | Active v11.2.0 (2026-08-19) | No | Android 9+, arm64-v8a, Snapdragon 460+/Adreno 610+/Vulkan 1.1+ | Wine arm64ec + Box64 0.4.4, Turnip v26.2.0, Vortek, DXVK/VKD3D/D8VK, CNC DDraw, Ubuntu Focal RootFS, PRoot | Native embedded XServer | PRoot | Per-container/per-shortcut settings; Box64 presets; driver import; env vars; gamepad/touch |
| **GameHub** | GameSir / Guangzhou Chicken Run Network Tech | gamehub.xiaoji.com; Play com.xiaoji.egggame | Closed source, free + IAP | Active v6.3.2 (2025-10-30) | No | Not officially stated (UNVERIFIED) | Winlator-derived + Proton, Wine/Box64, DXVK/Turnip/Zink, ImageFS | Frontend-managed | ImageFS | Steam/Epic integration, cloud profiles, in-app driver downloads; trackers/account required |
| **GameHub Lite** | Community (Producdevity) | github.com/Producdevity/gamehub-lite | Mod of proprietary APK; redistribution UNVERIFIED | v5.1.7 (2026-06-08) | No | Inherited | Patched APK: analytics/permissions removed | Same | Same | Privacy fork; trust in modder; not on Play Store |
| **DroidDeck** | Community (Droid-Deck org) | github.com/Droid-Deck/DroidDeck; droid-deck.dev | GPL-3.0 | Active v0.7.1 | No | Android 9+, arm64, Adreno 730+/8xx (6xx experimental; 710 unsupported; Mali/PowerVR experimental 0.7.0+) | ARM64 Steam client + DroidDeck-Proton (GE-Proton + Valve Wine + CachyOS fixes), FEX, gamescope, KDE Plasma, FEX Ubuntu 24.04 x86-64 rootfs, Turnip bundles, SteamRT4 | Steam Big Picture/gamescope; Wayland optional | PRoot (rootless; "run without proot" test builds) | Store integrations, cloud saves, per-game Proton, driver switching, FSR/GSR, LSFG |
| **GameNative** | Utkarsh Dalal / GameNative Inc. | github.com/utkarshdalal/GameNative; gamenative.app | GPL-3.0 | Active v1.4.0 (2026-11-05) | No | Android 8–14, arm64 | Proton 11.0-2 rebuild 9, FEX fork, gamescope default on Adreno (v1.4.0), Turnip Gen8 V38 / v27.0.0-R1, DXVK 3.1.1, vkd3d-proton 3.1, XServer, Ubuntu rootfs, ntsync-android, lsfg-vk | XServer + gamescope | Container (rootfs + prefix; proot-style UNVERIFIED) | Auto known-configs, Steam/Epic/GOG/Amazon, cloud saves, HUD, compat DB; PostHog analytics always on for launch events |
| **Mobox** | olegos2 | github.com/olegos2/mobox | No LICENSE file (UNVERIFIED) | Archived 2025-06-20; last commit 2024-11-23; community remake | No (root optional) | Android 10+ recommended, arm64, Adreno 6xx/725–740 | Termux glibc + Box64/86, Wine/wine-ge, DXVK(+ASYNC/GPLASYNC), VKD3D, D8VK, Mesa VirGL/Turnip/Zink | Termux:X11 + Input Bridge | None (Termux native/glibc prefix) | Near-native speed; menus; backup/restore; archived upstream |
| **Termux** | Termux devs | termux.dev; github.com/termux/termux-app; F-Droid | GPL-3.0-only | Active v0.118.3 stable | No | Android 7+ | Terminal + apt; no Wine by default; add-ons termux-x11, proot-distro | Optional Termux:X11 | Optional proot | Substrate, not a Windows frontend. GitHub APKs use public test key. |
| **ExaGear** | Eltechs | exagear.wiki | Proprietary/commercial | Discontinued 2019-02-28 | No | 32-bit x86 on ARMv7/v8 | Wine + proprietary x86 translator, OBB rootfs, VirGL | Custom surface | Linux container in OBB | Dead; license checks fail; security risk; 2D/strategy only |
| **Cassia** | PixelyIon + bylaws (ex-Skyline) | github.com/cassia-org | Intended closed; no LICENSE | Cancelled 2024-06; never released | No (planned) | ARM64 | Wine + FEXCore, DXVK/VKD3D, Mesa, libadrenotools, gamescope, xserver | gamescope planned | Custom runtime | Code archaeology only |
| **Pluvia** | Oxters / Ox Games | github.com/oxters168/Pluvia; gopluvia.com | GPL-3.0 | Retired; last v1.3.2 (2026-03-02) | No | arm64 | Embedded Winlator backend, Ubuntu rootfs, Wine/Box64/DXVK/Turnip | Winlator XServer | PRoot | Steam client + DepotDownloader; successor is GameNative |

Other 2026 tools: WinNative (GPL-3.0), Bannerlator (GPL-3.0 v3.1.8), MiceWine (MIT, archived), Vectras VM (closed app, QEMU, slow full-system emulation).

## 2. Frontend vs compatibility layer

Frontends orchestrate: container lifecycle, shortcut settings, input mapping, driver/component selection, store login, logs. They bundle Wine/Proton (Windows APIs), Box64/FEX (CPU), DXVK/VKD3D (graphics), Mesa Turnip/VirGL (drivers), PRoot (filesystem), XServer/gamescope (display). Users override env vars, driver bundles, Proton versions, per-game configs.

## 3. Turnip / Adreno 730 specifics

- Winlator: in-app driver picker, AdrenoTools injection, `TU_DEBUG`, `FD_DEV_FEATURES`, `MESA_EXTENSION_MAX_YEAR`.
- DroidDeck: weekly Mesa driver bundles; `TU_DEBUG=sysmem` default except A710/A720; separate runtime/display driver replacement.
- GameNative: manifest-pinned Turnip Gen8 V38 + v27.0.0-R1; gamescope default on Adreno (breaking change).
- For Adreno 730 / SD 8+ Gen 1: community lists Turnip 26.0.0 R8 and Qualcomm v615.77 as starting points; Mobox README says enable a7xx flicker fix; do not apply A740 UBWC hint hack to A730; avoid extracted Qualcomm v744/v757/v777/v786 (crash reports).
- Runtime driver (guest Vulkan ICD) vs display/compositor (SurfaceFlinger/HWC) are different roles; UBWC mismatch causes flicker.

## 4. Building from source

- Winlator: Android Studio/Gradle, NDK 29, CMake, submodules; ImageFS/RootFS packaging.
- GameNative: `git submodule update --init --recursive`, Android Studio; ubuntufs manifests.
- DroidDeck: `tools/build_local.sh`; Docker, Java 17, SDK/NDK, Rust, Go, MinGW-w64 x86_64+i686 DWARF-2, GitHub CLI, Python, zstd; binfmt for arm64 on x86_64.
- Termux: Gradle + bootstrap from termux-packages.
- Common failures: uninitialized submodules, NDK/AGP drift, OBB signature mismatch, phantom process killer.

## 5. Choosing a frontend

- Adreno 7xx/8xx: DroidDeck, GameNative, Winlator, GameHub, Bannerlator. Adreno 6xx: Winlator/Mobox/MiceWine. Mali/PowerVR: Winlator (VirGL/Vortek), MiceWine; lower perf.
- Root not required. Prefer open source/auditable for privacy. Closed: GameHub, Vectras app, ExaGear.
- Easiest: GameHub, GameNative, DroidDeck. Most tinkering: Winlator, Mobox, Bannerlator. Avoid dead: ExaGear, Cassia, Pluvia, upstream Mobox.

## 6. Security warnings

Only trust official repos/sites (winlator.org is original; fake winlator.com/.dev exist). Termux GitHub APKs use public test key — prefer F-Droid; never mix sources. GameHub requires account, broad permissions, trackers. GameNative discloses PostHog analytics. Steam login inside third-party clients — use QR login. Mod APKs are untrusted. Licensing: respect GPL-3.0, LGPL-2.1, MIT, proprietary.

## 7. Citations (verified)

- https://github.com/brunodev85/winlator
- https://github.com/winebox64/winlator
- https://winlator.org/
- https://github.com/utkarshdalal/GameNative
- https://gamenative.app/
- https://github.com/Droid-Deck/DroidDeck
- https://droid-deck.dev
- https://github.com/Droid-Deck/Drivers
- https://github.com/olegos2/mobox
- https://github.com/termux/termux-app
- https://github.com/termux/termux-x11
- https://f-droid.org/en/packages/com.termux
- https://github.com/ptitSeb/box64
- https://github.com/K11MCH1/AdrenoToolsDrivers/releases
- https://github.com/The412Banner/Banners-Turnip
- https://developer.android.com/ndk/guides/cmake
