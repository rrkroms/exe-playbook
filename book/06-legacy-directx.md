# 06 — Legacy DirectX: D3D7, D3D8 and D3D9 in Depth

This module is about old games. Direct3D 7, 8 and 9 predate the uniform Vulkan-based pipeline described in module 05, and they rely on hardware features that modern APIs no longer expose directly: fixed-function transform and lighting, paletted textures, 16-bit colour and DirectDraw surfaces. The tools that handle them are correspondingly specialised.

> **Safety and law.** Use **your own legally acquired copy** of every game. This book does not support piracy, cracks or circumvention of DRM or anti-cheat. Compatibility layers are legal in many jurisdictions, but details vary and you are responsible for your use. Some legacy games ship with copy protection that will not run under Wine; do not look for bypasses here.
>
> **Wine disclaimer.** [Wine](https://gitlab.winehq.org/wine/wine) is an independent reimplementation of Windows APIs, not Microsoft Windows. It is not perfect, it is not endorsed by Microsoft, and a game failing under Wine is not evidence that the game is broken. Compatibility varies by Wine version.
>
> **Closed-source warning.** Several legacy wrappers, notably [dgVoodoo2](https://www.dege.freeweb.hu/dgVoodoo2/) and [nGlide](https://www.zeussoftware.com/), are closed-source freeware. Their authors may not support Wine or ARM hosts, their redistribution terms are restricted, and their GitHub repository may be archived. Treat community reports about them as anecdotes, not guarantees.

## 6.1 What "legacy" means here

| Generation | Approx. years | Shading | Typical traits |
|---|---|---|---|
| DirectX 3–7 | 1996–1999 | Fixed function | DirectDraw for 2D, D3D immediate/retained mode, 8/16-bit colour common |
| DirectX 8 | 2000–2001 | Shader Model 1.x | First programmable shaders, still many fixed-function fallbacks |
| DirectX 9 | 2002–2013 | Shader Model 2.0/3.0 | Long tail; 32-bit EXEs; huge installed base |

Three consequences drive every decision in this module:

1. **No fixed function in Vulkan.** A translator must synthesise shaders that reproduce the old render-state formula (module 05.9).
2. **DirectDraw is part of the problem.** Many games render 3D through D3D7 but draw menus, HUDs and video through DirectDraw. A D3D-only translator is not enough.
3. **The EXEs are 32-bit.** That constrains DLLs, prefixes and CPU translation on ARM.

## 6.2 The fixed-function pipeline

In a **fixed-function pipeline**, the GPU applies a built-in sequence of operations. The game sets render states (lighting enabled, fog mode, texture blend mode, alpha test), and the hardware follows a fixed formula. There is no shader program to inspect.

```text
Vertex data -> transform + lighting -> primitive assembly -> rasterisation
                                                               |
                                                         texture stages
                                                               |
                                                    fog / alpha / stencil
                                                               |
                                                          framebuffer
```

This is a **conceptual diagram** of the D3D7 pipeline, not the literal order of calls in any game. Real D3D7 games interleave state changes, texture loads and DirectDraw blits.

Why it matters: a translator must map each render state to an equivalent shader or pipeline state. Small semantic differences — fog distance, alpha-test comparison, texture coordinate wrap — show up as wrong colours or missing geometry. This is why a D3D7 game can look correct on [WineD3D](https://gitlab.winehq.org/wine/wine) and corrupt on another translator, or vice versa.

## 6.3 DirectDraw, paletted and 16-bit colour

**DirectDraw** is the 2D part of DirectX: offscreen surfaces, page flipping, blitting and palettes. Games used it for menus, sprites and full-motion video even when 3D ran through D3D.

- **Paletted (8-bit) colour** stores a pixel as an index into a 256-entry palette. Changing the palette changes many pixels at once. Wrappers that upscale or convert the surface can break palette handling; dgVoodoo2's own documentation warns against forced resolution for 8-bit paletted DirectDraw. Even so, verify per game.
- **16-bit colour** (RGB565/RGB555) packs a pixel into two bytes. It was the performance sweet spot in 1999. [dgVoodoo2](https://www.dege.freeweb.hu/dgVoodoo2/) exposes settings such as `EnumeratedResolutionBitdepths` and dithering to preserve these modes; [DDrawCompat](https://github.com/narzoul/DDrawCompat) handles them natively but is Windows-only.
- [cnc-ddraw](https://github.com/FunkyFr3sh/cnc-ddraw) replaces **2D software DirectDraw** (typical of paletted titles) with GDI, OpenGL or D3D9. It works in Wine on Linux, macOS and Android, but it does not translate a D3D 3D path.
- [D7VK](https://github.com/WinterSnowfall/d7vk) deploys as `ddraw.dll` and expects Wine's real DirectDraw underneath; it is therefore a D3D wrapper sitting *above* DirectDraw, not a DirectDraw replacement.

## 6.4 Shader model constraints

A **shader model** is the version of the shader assembly language and its capabilities. D3D8 introduced SM1.0/1.1; D3D9 added SM2.0 and later SM3.0. Constraints that affect translation:

- SM1.x has very limited instruction counts and no dynamic branching; SM2.0 adds more registers; SM3.0 adds dynamic flow control.
- Many D3D9 games ship **both** a fixed-function path and shader paths. A translator may handle one well and the other poorly.
- DXVK 3.x uses `dxbc-spirv` for D3D8/9 shader and fixed-function compilation; upstream DXVK requires no `d3dcompiler` DLL. Any `d3dcompiler` dependency is a game/Wine-side issue, not a DXVK 3.x requirement.
- [D3D8to9](https://github.com/crosire/d3d8to9) translates D3D8 bytecode shaders exactly, but needs legacy D3DX for shader work.

The practical rule: a 1999–2002 game usually does **not** need a modern shader compiler; it needs correct fixed-function emulation and DirectDraw behaviour. Choosing a D3D12-era toolchain solves the wrong problem.

## 6.5 Choosing a legacy tool

| Tool | Best for | Avoid when |
|---|---|---|
| D7VK v2.4 | D3D7 immediate mode, Vulkan 1.4 available, 2048px surfaces acceptable | Retained-mode D3D7; Vulkan < 1.4; large surfaces |
| [DXVK-Sarek](https://github.com/pythonlover02/DXVK-Sarek) v1.14.1 | D3D3–11 on Vulkan 1.1/1.2, ARM/mobile | Fork author reports their own FOSS projects on a 1–3 month halt; check the [releases page](https://github.com/pythonlover02/DXVK-Sarek/releases) before relying on a build |
| DXVK (merged D8VK) | D3D8 games; needs `d3d8.dll` + `d3d9.dll` | Game uses D3D7 or DirectDraw only |
| D3D8to9 | Exact D3D8 → D3D9 conversion, then DXVK | VSync-sensitive games; shaders without legacy D3DX |
| [DxWrapper](https://github.com/elishacloud/dxwrapper) | DDraw/D3D1–7 redirection (`dd7to9=1`), game-specific fixes | You want a pure, minimal translator |
| dgVoodoo2 | SM1/2/3 fixed function, paletted/8/16/32-bit modes, Glide | You need supported Wine/ARM operation; closed-source is unacceptable |
| cnc-ddraw | 2D software-DirectDraw games | The game uses D3D for 3D |
| [GXVK](https://github.com/CkNoSFeRaTU/gxvk) | Games offering a 3Dfx Glide path instead of D3D | You need a mature, stable translator (it is alpha) |
| WineD3D | Universal fallback bundled with Wine | Performance matters; D3D7 geometry errors are reported |
| [Zink](https://docs.mesa3d.org/drivers/zink.html) | Providing OpenGL on Vulkan for WineD3D | You mistake it for a D3D translator |

### dgVoodoo2 configuration under Wine

dgVoodoo2 is frequently recommended for legacy games, but it needs careful handling:

1. Copy **all** 32-bit dgVoodoo2 DLLs **locally next to the game EXE**, not into the prefix's `system32`.
2. Set `WINEDLLOVERRIDES=ddraw,d3d11,dxgi=n` so those names resolve to the local native DLLs.
3. `dgVoodoo.conf` is searched in the DLL directory, then the EXE directory, then appdata. Useful fields include `OutputAPI` (`d3d12_fl12_0`, `d3d12_fl11_0`, `d3d11_fl11_0`, `d3d11_fl10_1`, `d3d11_fl10_0`, `d3d11warp`, `bestavailable`), resolution mode, filtering and FPS limit.
4. A common community setup chains dgVoodoo2 to DXVK by letting dgVoodoo2 output D3D11 and supplying DXVK's `d3d11.dll` and `dxgi.dll` underneath; neither project documents this officially.
5. **Never** override `d3d8`/`d3d9` with DXVK while dgVoodoo's own D3D8/D3D9 DLLs are active, and never stack two DirectDraw wrappers.

The author explicitly states that Wine/Linux is not a target. Community reports name different last-working builds. The PCGamingWiki file bundles v2.79.3 and v2.81.3 as Wine/Proton-compatible (a comment names v2.81.3 as the latest), while a Lutris issue calls v2.81.3 the last working build but v2.81.2 the latest viable target. Other reports name v2.79.1 or "below 2.84". Treat all such claims as unverified anecdotes. The current binary is v2.87.6 (re-released 2026-10-26), targets Windows 7+, and ships CHPE (emulated x86), ARM64EC (emulated x64) and ARM64X builds for **Windows on ARM** — not native ARM64 Linux builds. Do not use them in a Wine-on-Android prefix.

## 6.6 Case study: Project I.G.I.: I'm Going In (2000)

This is a worked example, not a universal recipe. Verify each claim against your own copy and environment.

**Verified facts from the research brief:**

| Property | Value |
|---|---|
| Title | Project I.G.I.: I'm Going In |
| Release | 2000 |
| Graphics API | Direct3D 7 (traced via `Direct3DDevice7`) |
| Executable | 32-bit |
| Pipeline | Fixed function |
| Source | [PCGamingWiki entry](https://www.pcgamingwiki.com/wiki/Project_I.G.I.%3A_I%27m_Going_In) |

Use your own legally acquired copy. IGI is a useful teaching case because it exercises almost every legacy failure mode: 32-bit EXE, D3D7 immediate mode, DirectDraw, fixed function, old runtimes and fragile vertex handling.

### 6.6.1 Shared steps (Android and ARM Linux)

1. **Create a prefix for a 32-bit game:**
   ```sh
   WINEARCH=win32 WINEPREFIX=~/igi32 wineboot -u
   ```
   A normal `win64` prefix runs 32-bit games through Wine WoW64 (`syswow64`); in Wine 11.0's new WoW64 mode a pure `WINEARCH=win32` prefix is deprecated and unsupported. The command above is retained for older Wine or a legacy wrapper that specifically needs it.
2. **Windows version:** start with Windows 7 in `winecfg`. If the game reports `DDERR_UNSUPPORTED`, try Windows XP SP3 — a community workaround recorded on the [PCGamingWiki IGI page](https://www.pcgamingwiki.com/wiki/Project_I.G.I.%3A_I%27m_Going_In), not confirmed by a primary source.
3. **Install to an ASCII-only path** (for example `C:\Games\IGI`). Non-ASCII paths break many legacy installers.
4. **Apply the Large Address Aware / 4GB patch** if the game runs out of memory on large levels; this is general 32-bit advice from PCGamingWiki, not an IGI-verified fix [NEEDS VERIFICATION: depends on build and installer].
5. **Runtimes:** install DirectX 7/8-era components and whatever Visual C++ runtime the installer needs. **No `d3dcompiler` is required** for the D3D7 fixed-function path.
6. **Cap FPS at a stable rate** (30–60 depending on build or patch). Unlimited frame rates destabilise old engines.
7. **Place the chosen 32-bit wrapper DLL next to the EXE** and set the matching override from module 05.7. Copying without overriding has no effect.

### 6.6.2 Decision tree

```text
Game crashes at launch or "Bad EXE format"
  -> 32-bit prefix? 32-bit CPU translator path? 32-bit wrapper DLLs?

Game starts, no 3D / "no 3D card"
  -> choose a D3D7 wrapper:
       Vulkan 1.4 driver available?
         yes -> D7VK v2.4 (ddraw.dll, override ddraw=native,builtin)
         no  -> DXVK-Sarek v1.14.1 (Vulkan 1.1/1.2, mobile builds)
       Need dgVoodoo2 features?
         -> dgVoodoo2 -> D3D11 FL10.1 -> DXVK d3d11/dxgi
       D3D7 path broken?
         -> DxWrapper dd7to9=1, override ddraw=n,b; optionally chain DXVK D3D9
       Game offers a Glide path?
         -> GXVK (alpha) or nGlide
       Last resort -> WineD3D (works, but geometry errors reported)

DDERR_UNSUPPORTED
  -> try Windows XP SP3 [NEEDS VERIFICATION — community report; see §6.6.1]

Vertex explosions / corrupt geometry
  -> D7VK fixes, DxWrapper options, LAA/4GB patch

Black window with audio
  -> virtual desktop / gamescope; check presentation layer

Slowdowns
  -> DxWrapper DdrawNoDrawBufferSysLock; cap FPS; check CPU governor
```

The tree is a **diagnostic aid**, not a sequence to execute top to bottom. Change one variable at a time and keep notes.

### 6.6.3 Android recipe

Android shares the Linux kernel but has a different userspace: Bionic libc, app sandboxing, SurfaceFlinger, and per-app native library loading. That changes where the driver lives.

```text
Android frontend (e.g. Winlator)
  -> bundled RootFS (proot/chroot)
  -> Wine (32-bit prefix)
  -> 32-bit x86 CPU translation (Box64/box32 or FEX)
  -> 32-bit wrapper DLL (D7VK or DXVK-Sarek)
  -> Vulkan driver (Turnip injected per app, or vendor ICD)
  -> SurfaceFlinger
```

Steps:

1. Use a frontend that provides a **32-bit-capable x86 translation path**. [Box64](https://github.com/ptitSeb/box64) handles 64-bit x86-64, while 32-bit Wine components traditionally need [Box86](https://github.com/ptitSeb/box86); experimental box32 and [FEX](https://github.com/FEX-Emu/FEX) are other routes. The bundled path varies by frontend [NEEDS VERIFICATION: check the exact frontend build].
2. Use a Vulkan 1.4 driver if you choose D7VK. On Adreno, [Turnip](https://docs.mesa3d.org/drivers/freedreno.html) (Mesa's open-source Vulkan driver) is injected **per app**; the stock Qualcomm driver should remain for the system UI. This needs no root for per-app injection.
3. Create the 32-bit prefix as in 6.6.1.
4. Put the 32-bit `ddraw.dll` next to the EXE and set `WINEDLLOVERRIDES=ddraw=n,b` for DxWrapper, or `ddraw=native,builtin` for D7VK. Do not use both.
5. Prefer D7VK v2.4 or DXVK-Sarek. **Avoid dgVoodoo2's ARM64EC/CHPE builds** — they target Windows on ARM, not Wine on Android.
6. Use the frontend's virtual display or a VNC/X server if the game opens a tiny fullscreen window.

[NEEDS VERIFICATION: exact environment-variable syntax and 32-bit support differ per frontend; confirm against the frontend's own documentation before applying.]

### 6.6.4 ARM Linux recipe

ARM Linux exposes a normal Linux userspace, so Vulkan ICDs and display servers are more accessible.

```text
Linux userspace
  -> Wine (win32 prefix)
  -> FEX or Box64/box32 (32-bit x86 path)
  -> D7VK or DXVK-Sarek
  -> Mesa driver (Turnip for Adreno, Panfrost/PanVK for Mali, V3DV for Raspberry Pi)
  -> Wayland/X11, optionally Gamescope
```

Steps:

1. Create the 32-bit prefix as in 6.6.1.
2. Confirm a Vulkan device with `vulkaninfo --summary` and check the reported driver name. Turnip needs a compatible Adreno (A6xx/A7xx); other SoCs use their own Mesa driver.
3. Select D7VK for Vulkan 1.4, DXVK-Sarek for Vulkan 1.1/1.2 or mobile-focused builds.
4. Use [Gamescope](https://github.com/ValveSoftware/gamescope) or a virtual desktop when the game assumes a small fixed resolution.
5. Monitor logs with `WINEDEBUG=+err` and the wrapper's own logging; avoid changing several variables at once.

### 6.6.5 Common fixes

| Symptom | Likely layer | First action |
|---|---|---|
| "Bad EXE format" | Prefix / CPU translation | 32-bit `WINEARCH`, 32-bit translator path |
| "No 3D card" / no 3D | Missing D3D7 wrapper | Install D7VK, DXVK-Sarek or DxWrapper with correct override |
| `DDERR_UNSUPPORTED` | Windows version reported by Wine | Try XP SP3 [NEEDS VERIFICATION — community report; see §6.6.1] |
| Vertex explosions | Fixed-function mismatch / memory | D7VK fixes, DxWrapper options, LAA/4GB patch |
| Black window, audio works | Presentation / swapchain | Virtual desktop, Gamescope, borderless window |
| Slowdowns | DDraw locking path / CPU governor | `DdrawNoDrawBufferSysLock`, cap FPS, check translation overhead |
| Corruption after driver update | Stale shader caches | Clear DXVK/Mesa caches; first run will be slow |
| Wrapper DLL ignored | Override / architecture | Check `WINEDLLOVERRIDES` and 32-vs-64-bit DLL |

## 6.7 What to remember

- Legacy DirectX is defined by fixed function, DirectDraw and 32-bit EXEs — not by a single translator.
- Fixed function must be synthesised as shaders; paletted and 16-bit content is where translations diverge.
- D7VK needs Vulkan 1.4 and Wine's DirectDraw underneath; DXVK-Sarek (v1.14.1 at the snapshot) targets older Vulkan on mobile; dgVoodoo2 has the broadest legacy feature support but is closed-source and not Wine-supported.
- For Project I.G.I.: use a 32-bit prefix, an ASCII install path, a D3D7 wrapper with the exact override, a stable FPS cap (30–60), and your own legally acquired copy.
- Never stack two DirectDraw wrappers; never use Windows-on-ARM dgVoodoo2 builds in a Wine-on-Android prefix.
- Wine is not Windows. Compatibility varies, and community workarounds are not guarantees.

Next: [07 — GPU Drivers](07-gpu-drivers.md).

## References

- Wine — https://gitlab.winehq.org/wine/wine ; release notes: https://www.winehq.org/announce/3.0 , https://www.winehq.org/announce/4.0 , https://www.winehq.org/announce/6.0
- DXVK — https://github.com/doitsujin/dxvk
- D7VK — https://github.com/WinterSnowfall/d7vk
- DXVK-Sarek — https://github.com/pythonlover02/DXVK-Sarek
- D8VK — https://github.com/AlpyneDreams/d8vk
- D3D8to9 — https://github.com/crosire/d3d8to9
- DxWrapper — https://github.com/elishacloud/dxwrapper ; IGI issue: https://github.com/elishacloud/dxwrapper/issues/453
- DDrawCompat — https://github.com/narzoul/DDrawCompat
- cnc-ddraw — https://github.com/FunkyFr3sh/cnc-ddraw
- dgVoodoo2 — https://www.dege.freeweb.hu/dgVoodoo2/ ; https://github.com/dege-diosg/dgVoodoo2
- nGlide — https://www.zeussoftware.com/
- GXVK — https://github.com/CkNoSFeRaTU/gxvk
- WineD3D (Wine) — https://gitlab.winehq.org/wine/wine
- Zink — https://docs.mesa3d.org/drivers/zink.html
- Turnip / Freedreno — https://docs.mesa3d.org/drivers/freedreno.html
- Panfrost — https://docs.mesa3d.org/drivers/panfrost.html
- V3D — https://docs.mesa3d.org/drivers/v3d.html
- Box64 — https://github.com/ptitSeb/box64 ; FEX — https://github.com/FEX-Emu/FEX
- Gamescope — https://github.com/ValveSoftware/gamescope
- Project I.G.I.: I'm Going In — https://www.pcgamingwiki.com/wiki/Project_I.G.I.%3A_I%27m_Going_In
- Mesa environment variables — https://docs.mesa3d.org/envvars.html
- DXVK-Sarek releases — https://github.com/pythonlover02/DXVK-Sarek/releases
- Box86 — https://github.com/ptitSeb/box86
- dgVoodoo2 Wine/Proton community reports — https://community.pcgamingwiki.com/files/file/3600-dgvoodoo2-v2793-v2813-wineproton-compatible/ ; https://github.com/lutris/dgvoodoo2/issues/2
