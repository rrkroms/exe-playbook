# 16 — Decision Trees and Compatibility Matrix

This module is meant to be used while configuring a game, not read linearly. Start with the host, then the game, then the failure.

## 16.1 Start with the host

```text
What host are you on?
        |
        +-- x86_64 Linux -----------------> no CPU translation
        |                                  Wine/Proton + DXVK/VKD3D
        |                                  -> module 12
        |
        +-- Apple Silicon macOS -----------> Rosetta 2 or ARM64EC Wine
        |                                  D3DMetal/DXMT/DXVK+KosmicKrisp (macOS 26+/Metal 4)
        |                                  older macOS: MoltenVK (deprecated) + older DXVK
        |                                  -> module 12
        |
        +-- Intel macOS -------------------> native x86_64 Wine, no CPU translation
        |                                  DXVK+MoltenVK only (KosmicKrisp is Apple-Silicon-only)
        |                                  -> module 12
        |
         +-- ARM Linux ---------------------> FEX / Box64 / Box86
         |                                  add muvm only on 16K-page hosts (Asahi, some Snapdragon)
         |                                  -> module 11
        |
        +-- ARM Chromebook ----------------> Crostini + VirGL/Venus, limited
        |                                  -> module 12
        |
         +-- Android ARM64 -----------------> frontend or Termux DIY
         |                                  -> modules 09, 10
```

## 16.2 What CPU translation does the game need?

| Game architecture | Host | Required approach |
|---|---|---|
| x86-64 Windows | x86_64 Linux | None |
| x86-64 Windows | ARM64 Android/Linux | FEX or Box64 (whole process or backend DLL) |
| 32-bit x86 Windows | x86_64 Linux | Wine WoW64 (no ISA translation) |
| 32-bit x86 Windows | ARM64 | FEX WoW64 DLL, Box32, Box86, or 32-bit Wine under translator |
| Windows ARM64 | ARM64 | Native; no x86 translation |
| Windows ARM64EC | ARM64 | Native ARM64EC process with FEX backend for x86 parts |
| 16-bit Windows | x86 | Wine 11.0 supports it in the new WoW64 mode; ARM64 unverified |

## 16.3 What graphics API does the game use?

```text
Game graphics API?
        |
        +-- Direct3D 7 ------> D7VK (Vulkan 1.4) | dgVoodoo2->D3D11->DXVK | WineD3D
        +-- Direct3D 8 ------> DXVK (d3d8+d3d9) | D3D8to9->DXVK | dgVoodoo2 | D8VK
        +-- Direct3D 9 ------> DXVK first | WineD3D fallback | dgVoodoo2
         +-- Direct3D 10 ------> DXVK | WineD3D
         +-- Direct3D 11 ------> DXVK | WineD3D
         +-- Direct3D 12 ------> VKD3D-Proton
        +-- OpenGL ----------> Wine opengl32 -> host GL | Zink when no native GL
        +-- Vulkan ----------> native Vulkan (rare in Windows games)
        +-- DirectDraw ------> cnc-ddraw (2D) | DDrawCompat (Windows only) | D7VK proxy
        +-- Glide -----------> GXVK | dgVoodoo2 | nGlide (Windows)
```

## 16.4 Translator selection chart

| Game API | First option | Alternative | Notes |
|---|---|---|---|
| D3D7 | D7VK | dgVoodoo2 → DXVK, WineD3D, DxWrapper | D7VK needs Vulkan 1.4 + real DDraw underneath |
| D3D8 | DXVK (d3d8+d3d9) | D3D8to9 → DXVK, dgVoodoo2, D8VK | D8VK standalone merged into DXVK |
| D3D9 | DXVK | WineD3D, dgVoodoo2 → DXVK | DXVK needs Vulkan 1.4 in 3.x |
| D3D10/11 | DXVK | WineD3D | DXVK needs d3d11/dxgi native overrides |
| D3D12 | VKD3D-Proton | — | Needs Vulkan 1.3 + descriptor indexing + robustness2 + `VK_KHR_push_descriptor` |
| OpenGL | native GL | Zink | Zink is GL on Vulkan, not a D3D translator |
| DirectDraw (2D) | cnc-ddraw | DDrawCompat (Windows only) | D7VK does not help pure DDraw titles |

## 16.5 GPU driver selection

| Host / GPU | First choice | When to switch |
|---|---|---|
| Android / Adreno | Stock Qualcomm for UI; Turnip injected per app | Compatibility or speed; use pinned community build with known commit |
| Android / Mali | Stock Arm DDK | Experimental PanVK only for testing |
| ARM Linux / Adreno | Freedreno GL + Turnip Vulkan (`drm/msm`) | KGSL build only on Android |
| ARM Linux / Mali | Panfrost/PanVK | Zink if only Vulkan exists |
| ARM Linux / Apple | Asahi GL + HoneyKrisp Vulkan | muvm for 4K pages |
| ARM Linux / RPi | V3D + V3DV | — |
| x86 Linux / AMD | RADV | NVIDIA/Intel different stack |
| x86 Linux / Intel | ANV | — |
| x86 Linux / NVIDIA | Proprietary ICD | NVK+Zink to avoid closed userspace |
| VM | Venus (Vulkan), VirGL (GL) | DRM Native Context when guest/host driver families match |

## 16.6 Symptom → layer quick map

| Symptom | Layer | First evidence |
|---|---|---|
| Nothing launches | Frontend/container/Wine loader | frontend logs, prefix, mounts |
| "Bad EXE format" | Architecture/32-bit support | file arch, Wine arch, translator |
| Missing DLL | Dependency layer | overrides, VC++/.NET/DirectX runtime |
| Crash at graphics init | Translator or driver | DXVK/VKD3D logs, vulkaninfo |
| Black screen, audio works | Presentation/WSI | fullscreen mode, present mode, compositor |
| Low FPS, low GPU usage | CPU translation or sync | HUD CPU, Box64/FEX stats, ntsync |
| Low FPS, high GPU usage | GPU/resolution | resolution, effects, driver |
| First-run stutter, then fine | Shader compilation | DXVK_HUD=compiler |
| Controller/input missing | Input layer | XInput/DInput mapping |
| Audio missing | Audio layer | PulseAudio/PipeWire, Wine audio |
| Crash after minutes | Thermal/memory/driver | temperatures, RAM, logs |

## 16.7 Per-game profile template

Copy this into a note for each game:

```text
GAME:
  Name / store version:
  Architecture (32/64-bit):
  Graphics API:
  Launcher / DRM / anti-cheat:

HOST:
  Device / OS build:
  Kernel (Linux):
  GPU / driver name + version:
  Vulkan device + API version (vulkaninfo):

CONTAINER:
  Frontend / version:
  RootFS / container type:
  Wine/Proton build:
  Prefix path / WINEARCH:
  Windows version:

TRANSLATORS:
  CPU: FEX / Box64 / Box86 / none
  Graphics: DXVK / VKD3D-Proton / D7VK / WineD3D / ...
  Translator version:

DLLs / RUNTIMES:
  Native overrides:
  Installed runtimes (VC++, .NET, DirectX, XAudio, ...):

ENV:
  DXVK_HUD / LOG_LEVEL:
  VK_DRIVER_FILES:
  TU_DEBUG / MESA_*:
  BOX64_* / FEX_*:

RESULTS:
  Resolution / settings:
  Average FPS / 1% low:
  CPU/GPU load:
  Temperatures:
  Known issues:
```

## 16.8 Checklists

### Before installing a game
- [ ] Identify host, GPU and driver.
- [ ] Identify game architecture and graphics API.
- [ ] Check anti-cheat/DRM viability (module 15).
- [ ] Choose frontend or DIY path.
- [ ] Prepare a fresh prefix.

### After first launch
- [ ] Verify Vulkan device with `vulkaninfo`.
- [ ] Verify DXVK/VKD3D is active (HUD `devinfo`).
- [ ] Install missing runtimes one at a time.
- [ ] Set correct DLL overrides.
- [ ] Record baseline FPS/frame times.

### When debugging
- [ ] Change one variable only.
- [ ] Clear stale caches after driver/translator changes.
- [ ] Capture logs with the correct knobs (module 14).
- [ ] Compare against a known-good profile.
- [ ] Roll back to previous driver/translator if needed.

## 16.9 Legacy DirectX quick path

For a pre-2006 game:
1. Confirm 32-bit exe and Direct3D version (often D3D7/8/9).
2. Fresh 64-bit prefix; in Wine 11.0's new WoW64 mode, pure `WINEARCH=win32` prefixes are deprecated and unsupported, so use one only with older Wine or a wrapper that requires it.
3. Windows 7, then XP SP3 if needed.
4. D3D7: D7VK or dgVoodoo2 → D3D11 → DXVK.
5. D3D8: DXVK d3d8+d3d9 or D3D8to9 → DXVK.
6. D3D9: DXVK or WineD3D.
7. Cap FPS at a stable rate (30–60 for old D3D games).
8. See module 06 for IGI 1 walkthrough.

## 16.10 What to remember

- Host first, game second, symptom third.
- The graphics API determines the translator; the translator determines which driver features matter.
- Never stack two wrappers for the same interface.
- Keep a known-good per-game profile; record versions.
- Measure layers, not just FPS.

Next: [17 — Lab: Legion Tab Gen 2](17-lab-legion-tab.md).
