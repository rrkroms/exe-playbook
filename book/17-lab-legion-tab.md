# 17 — Lab: Legion Tab Gen 2

This lab applies the whole book to the reference device: a Lenovo Legion Tab Gen 2 with Snapdragon 8+ Gen 1 (ARM64), Adreno 730, 12 GB RAM, Android 13.

The same steps generalise: replace the device, GPU and driver bundle, keep the layer model.

## 17.1 Lab goals

1. Establish a working Android Windows-gaming environment.
2. Run a legacy Direct3D 7 game (*Project I.G.I.: I'm Going In*, 2000) using the correct legacy path.
3. Run a modern Direct3D 9/11 title to verify the DXVK path.
4. Measure CPU/GPU load, frame time and thermals.
5. Diagnose a deliberate failure and fix it by changing one variable.

Use only games you legally own or free titles. No pirated content or anti-cheat circumvention.

## 17.2 Device preparation

**Warning:** changing system drivers or rooting may void warranty and weaken security. This lab uses non-root paths only.

- Update Android to the latest available build.
- Enable Developer options.
- Android 14+: Developer options → **Disable child process restrictions**. On Android 13, use the `settings put global settings_enable_monitor_phantom_procs false` adb command from module 10 if container processes are killed.
- Disable battery optimisation for the frontend/Termux.
- Keep the device cool; a stable 30 FPS is better than an unstable 60 FPS.
- Verify Vulkan support: install a frontend with a GPU driver manager, or run `vulkaninfo` in Termux. Expect an Adreno (TM) 730 device with Turnip or Qualcomm Vulkan.

## 17.3 Path A — frontend setup (recommended for most users)

Three good starting points for this GPU:

| Frontend | Why |
|---|---|
| **Winlator** | Mature, lots of community documentation, per-game settings, Turnip injection |
| **GameNative** | Proton-based, FEX, gamescope on Adreno, auto configs |
| **DroidDeck** | ARM64 Steam client, FEX, per-game Proton, weekly driver bundles |

Install only from the official sources listed in module 09 and the [references](19-references.md). Avoid mirrored APKs.

### Winlator steps
1. Install from the official GitHub/winlator.org source.
2. Create a container: x86_64 Wine under Box64, Ubuntu RootFS, Turnip driver.
3. Enable the A7xx flicker fix if the UI flickers; do not apply A740-specific UBWC hacks to Adreno 730.
4. Set Box64 preset to a stable default; do not tune multiple variables at once.
5. Mount your game folder and add a shortcut.
6. For legacy D3D7, see module 06: install a 32-bit Wine prefix and place the correct `ddraw.dll` wrapper next to the EXE.

### GameNative / DroidDeck steps
1. Install the official APK and allow the one-time runtime/RootFS download (~several GB).
2. Keep the bundled FEX and Turnip versions; record their numbers.
3. Add your game, let the compatibility database apply a known config, then inspect what changed.
4. Note: GameNative 1.4.0 enables gamescope by default on Adreno — if presentation breaks, disable it as the first variable.

## 17.4 Path B — Termux DIY (for learning the pipeline)

Follow module 10 in full. Summary:

1. Install Termux from F-Droid (never mix sources).
2. Disable the phantom process killer; grant storage permission; acquire wake lock.
3. Install `proot-distro`, `termux-x11-nightly`, `vulkan-tools`, `mesa-vulkan-icd-freedreno`.
4. Verify with `vulkaninfo --summary` that Turnip reports Adreno 730.
5. Create a Debian/Ubuntu rootfs and start Termux:X11.
6. Choose ARM64 Wine + Hangover FEX DLLs (preferred) or x86_64 Wine under Box64.
7. Install DXVK/VKD3D, set native DLL overrides, and run staged tests (console EXE → D3D9 → D3D11).

This path gives maximum control and maximum opportunity to break things. Keep notes.

## 17.5 Lab 1 — Legacy D3D7: IGI 1

### Legal premise
Use your own legally acquired copy. This lab provides no game files, cracks or keys.

### Verified facts about the game
- Direct3D 7 immediate mode, fixed-function, 32-bit executable.
- Original requirement: DirectX 7.0a, 8–16 MB VRAM.
- No `d3dcompiler` needed.

### Steps
1. Create a fresh 32-bit prefix:
   ```sh
   WINEARCH=win32 WINEPREFIX=~/igi32 wineboot -u
   ```
   In a frontend, use its "new 32-bit container" option instead.
2. Set Windows version to Windows 7; if the game reports `DDERR_UNSUPPORTED`, try Windows XP SP3.
3. Install the game to a simple ASCII path; keep a backup of original DLLs.
4. Apply the LAA/4GB patch to `IGI.exe` if the executable needs it.
5. Choose the graphics path:
   - **Vulkan 1.4 + native DDraw available** → D7VK `ddraw.dll` next to the EXE, `WINEDLLOVERRIDES=ddraw=n,b`.
   - **Vulkan 1.1/1.2 or mobile GPU** → DXVK-Sarek with its x86-only DDraw/D7VK module.
   - **Need dgVoodoo2 features** → dgVoodoo2 v2.87.6 local DLLs → D3D11 FL10.1 → DXVK `d3d11.dll`/`dxgi.dll`. Never stack D7VK and dgVoodoo2 DDraw in one process.
6. Install only needed runtimes (VC++ from your installer; DirectX 7/8-era components if a launcher demands them).
7. Cap FPS at a stable rate (30–60 for old D3D games). Old D3D games break at high refresh rates.
8. Launch and capture logs if it fails.

### Common failures
| Symptom | Fix |
|---|---|
| "No 3D card" | Correct wrapper + `ddraw` override |
| `DDERR_UNSUPPORTED` | Windows XP SP3 mode |
| Vertex explosions | D7VK 2.3/2.4, dxwrapper options, LAA patch |
| Black/short window | Virtual desktop / gamescope / `ddraw.backBufferResize` last resort |
| Vertex-buffer lock slowdowns | dxwrapper `DdrawNoDrawBufferSysLock`, `ForceSystemMemVertexCache` |

## 17.6 Lab 2 — Modern D3D9/D3D11

Use a free, legal **single-player/offline** Windows title. *Trackmania Nations Forever* (D3D9) is a reasonable example — check each game's requirements. Avoid multiplayer/VAC-protected titles such as *Team Fortress 2* for the DLL-override exercise below: upstream DXVK warns that manipulating D3D DLLs in multiplayer may be treated as cheating and can get an account banned.

### Steps
1. Create a 64-bit prefix or container.
2. Install DXVK: copy `x64/*.dll` to `system32` and `x32/*.dll` to `syswow64`.
3. In `winecfg` or `WINEDLLOVERRIDES`, set `d3d9`, `d3d11`, `dxgi` (and `d3d10core` if needed) to **native**.
4. Start with Turnip; if the GPU driver menu allows it, keep the system driver for Android UI.
5. Launch with `DXVK_HUD=devinfo,fps`. The device name should show Adreno/Turnip — if not, DXVK is not active and WineD3D is being used.
6. Record: resolution, average FPS, 1% low, `DXVK_HUD=gpuload`, temperatures after 15 minutes.

### Deliberate failure exercise
Change **one** variable: set `d3d11` to builtin (remove the override). Relaunch. You should see WineD3D behaviour (often lower FPS or different rendering). Restore the override. This proves which layer mattered.

## 17.7 Benchmarking on a thermal-limited device

1. Let the device cool to a consistent starting temperature.
2. Run each configuration for at least 10–15 minutes.
3. Record average FPS, 1% low, frame time, CPU/GPU load and temperature.
4. Watch for throttling: a falling clock speed with stable settings means heat, not drivers.
5. Compare with the same scene, same resolution, same shader cache warmed.
6. Do not benchmark the first run after a driver or translator change — caches are cold.

## 17.8 Lab notes template

```text
DATE:
DEVICE / OS BUILD:
FRONTEND / VERSION:
ROOTFS / PREFIX:
WINE/PROTON:
CPU TRANSLATOR + VERSION:
GRAPHICS TRANSLATOR + VERSION:
GPU DRIVER (runtime):
GPU DRIVER (display/system):

GAME:
  API:
  Architecture:
  Store/version:
  Settings:

COMMANDS / OVERRIDES:
ENV VARS:

RESULT:
  FPS avg / 1% low:
  CPU load:
  GPU load:
  Temp after 15 min:
  Verdict:

WHAT CHANGED FROM LAST RUN:
```

## 17.9 What to remember

- Start with one working configuration and change one variable at a time.
- The reference device is Adreno 730: keep system Qualcomm driver for UI, inject Turnip per app.
- Legacy games need the legacy path (module 06); modern games use DXVK/VKD3D.
- Verify the active layer with HUDs and logs, not assumptions.
- Thermals decide sustained performance.

Next: [18 — Glossary](18-glossary.md).
