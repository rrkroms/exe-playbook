# Research Brief — Legacy DirectX and Graphics Translators

**Scope:** running legacy Windows D3D/Glide titles on non-Windows hosts, verified 2026-11-23. "Latest" = newest release visible on the primary release page at that date.

## 1. Tool matrix

| Tool | Source API → destination API | Official repo / docs | License | Supported hosts | Status / latest (Nov 2026) | Important notes |
|---|---|---|---|---|---|---|
| **DXVK** | D3D8, 9, 10, 11 → Vulkan | github.com/doitsujin/dxvk; wiki: Driver-support, Feature-support | zlib/libpng | Linux/Wine primary; Windows works but unsupported; Android/ARM via Wine front-ends; macOS via forks/MoltenVK; ChromeOS via Crostini | **v3.1.1 (15 Sep 2026)**, active | D3D8 support merged from D8VK. Requires **Vulkan 1.4** since 3.0; `VK_KHR_maintenance5` since 2.7. D3D11 up to FL 12_1. New `dxbc-spirv` compiler (3.0). Fixed-function as ubershaders. D3D8 install needs `d3d8.dll` + `d3d9.dll`. |
| **VKD3D-Proton** | D3D12 → Vulkan | github.com/HansKristian-Work/vkd3d-proton | LGPL-2.1-or-later | Linux/Wine/Proton; Windows dev testing only | **v3.1 (28 Oct 2026)**, active | Needs Vulkan 1.3, descriptor indexing, `VK_EXT_robustness2`, `VK_KHR_push_descriptor`. Shares **DXGI with DXVK 2.1+**. Native `d3d12.dll`/`d3d12core.dll`. |
| **VKD3D (upstream)** | D3D12 → Vulkan; shader translation library | gitlab.winehq.org/wine/vkd3d | LGPL-2.1+ | Linux, Windows | **v1.15 (19 Feb 2025)** latest verified | `libvkd3d-shader` translates HLSL/DXIL/DXBC/SM1–3 → SPIR-V/GLSL/MSL. Library, not a drop-in game DLL. |
| **D8VK** | D3D8 → DXVK D3D9 backend → Vulkan | github.com/AlpyneDreams/d8vk | zlib | Same as DXVK | **Merged into upstream DXVK**; final standalone v1.0 (10 May 2026); discontinued | Standalone obsolete; use current DXVK's `d3d8.dll`. |
| **D7VK** | D3D7/6/5/3 immediate mode → DXVK D3D9 backend → Vulkan | github.com/WinterSnowfall/d7vk | zlib | Wine/Linux primary; Android/ARM via Wine; Vulkan **1.4** | **v2.4 (28 Oct 2026), maintenance mode** | Deploys as `ddraw.dll` with `native,builtin` override; requires real DDraw below. No retained-mode. 2048-pixel surface limits. |
| **DXVK-Sarek** | D3D3/5/6/7/8/9/10/11 → Vulkan 1.1/1.2 | github.com/pythonlover02/DXVK-Sarek | zlib | Linux/Wine/Proton, ARM/mobile/Box64/FEX/Mali/Adreno | **v1.13.0** (date UNVERIFIED) | Includes D7VK v2.1 + mobile fixes; ddraw/D7VK built x86-only. Proton-Sarek discontinued; Proton-CachyOS `PROTON_DXVK_SAREK=1`. |
| **dgVoodoo2** | Glide 2.11/2.45/3.1/Napalm + DirectDraw/D3D3–7 + D3D8.1 + D3D9 → D3D11/D3D12; WARP | dege.freeweb.hu/dgVoodoo2; github.com/dege-diosg/dgVoodoo2 | **Closed-source freeware** | Windows 7+ only; CHPE/ARM64EC builds for Windows on ARM. Author says Wine/Linux is not a target | **v2.87.6 (25/26 Oct 2026)**; GitHub repo archived | Output FL 12.0/11.0/10.1/10.0 (D3D11) or D3D12 FL12/11. Full SM1/2/3 FF, paletted textures, 8/16/32-bit modes. |
| **D3D8to9** | D3D8 API + bytecode shaders → D3D9 | github.com/crosire/d3d8to9 | BSD-2-Clause | Windows/Wine | Active source; no formal release verified | Exact translation only; VSync may differ. Needs legacy D3DX for shaders. |
| **dxwrapper (DxWrapper)** | DDraw/D3D1–7→D3D9, D3D8→D3D9, D3D9 hooking, DInput→8, DSound hooking | github.com/elishacloud/dxwrapper | zlib + BSD-2 + others | Windows 10/11 primary, Wine/Proton | **v1.9.8700.25 (12 Oct 2026)** | Drop-in stub DLL + INI. For DX7 use `ddraw` stub with `dd7to9=1`, override `ddraw=n,b`; can chain to DXVK. Extensive IGI-specific options. |
| **DDrawCompat** | No conversion — in-place DDraw/D3D1–7 + GDI fixes | github.com/narzoul/DDrawCompat | BSD Zero Clause (binaries from v0.3.0) | **Windows Vista–11 only**; unsupported in Wine/VMs | **v0.7.2 pre-release (14 Oct 2026)** | Needs SSE2 CPU + SM3 GPU. Not usable for Android/ARM-Linux recipe. |
| **cnc-ddraw** | DirectDraw (2D software) → GDI / OpenGL / D3D9 | github.com/FunkyFr3sh/cnc-ddraw | MIT | Windows ME–11; **Wine on Linux/macOS/Android** | **v7.1.0.0 (28 Dec; year UNVERIFIED)** | 2D/paletted software-DDraw games only; not D3D games. Override `ddraw` or run `cnc-ddraw config.exe`. |
| **WineD3D** | DDraw/D3D1–11 → OpenGL (experimental Vulkan renderer) | gitlab.winehq.org/wine/wine | LGPL-2.1+ | Every Wine host | Bundled with Wine release train | CSMT default since Wine 3.2/4.0; `renderer=gl|gdi|vulkan` (Vulkan renderer SM4/5, D3D10/11 only, since Wine 6.0). Often slow/buggy for D3D7. |
| **Zink** | OpenGL/ES → Vulkan (Mesa Gallium) | docs.mesa3d.org/drivers/zink.html | MIT | Linux, Android, ChromeOS; macOS experimental | Mesa release train | GL 2.1 minimum → 4.6. Supplies OpenGL to WineD3D when no native GL. |
| **VirGL / Venus** | Guest OpenGL → host OpenGL; guest Vulkan → host Vulkan | gitlab.freedesktop.org/virgl/virglrenderer; docs.mesa3d.org/drivers/virgl | MIT (UNVERIFIED for GitLab) | Linux host + Linux guest under QEMU | virglrenderer 1.0+ (Venus) | GL 4.3 default, 4.6 with blob. Double shader translation; serialized command stream. |
| **DXVK-NVAPI** | NVAPI/NVOFAPI → DXVK/VKD3D | github.com/jp7677/dxvk-nvapi | MIT | Linux/Wine/Proton; ARM64 build | **v0.9.3 (30 Oct 2026)** | DLSS/Reflex/OMM. Needs `DXVK_ENABLE_NVAPI=1`. |
| **GXVK** | 3Dfx Glide 2.x/3.x → Vulkan | github.com/CkNoSFeRaTU/gxvk | zlib | Wine/Linux; Windows untested | Early alpha v0.1 (2026) | For games with Glide path. |

## 2. Direct3D 7/8/9 specifics

- **D3D7**: D7VK, DXVK-Sarek, dgVoodoo2, dxwrapper `Dd7to9`, WineD3D. D7VK needs Vulkan 1.4 + underlying DDraw; rejects retained-mode.
- **D3D8**: DXVK (merged D8VK), D3D8to9, dxwrapper, dgVoodoo2, WineD3D. DXVK D3D8 = `d3d8.dll` + `d3d9.dll`.
- **D3D9**: DXVK, WineD3D, dgVoodoo2, Wine-Nine/Gallium Nine. SM1–3 fixed function.
- **Fixed-function**: DXVK ubershaders since 3.0; dgVoodoo2 shader-generated FF; D7VK bridged to D3D9 FF. No `d3dcompiler` needed.
- **DirectDraw/2D**: cnc-ddraw (2D only), DDrawCompat (Windows only), D7VK proxies DDraw.
- **16-bit colour**: dgVoodoo2 `EnumeratedResolutionBitdepths`, Dithering; DDrawCompat native.
- **8-bit paletted**: dgVoodoo2 supports except some virtual cards; **do not force resolution upscaling on 8-bit paletted DDraw**.

## 3. dgVoodoo2 specifics

- Wraps Glide, DDraw/D3D3–7, D3D8.1, D3D9 → D3D11/D3D12 or WARP. Closed-source freeware; restricted redistribution; v2.87.6; repo archived.
- Config: `dgVoodoo.conf` searched DLL dir → EXE dir → appdata. `OutputAPI` = d3d12_fl12_0, d3d12_fl11_0, d3d11_fl11_0, d3d11_fl10_1, d3d11_fl10_0, d3d11_warp. Resolution static/dynamic/custom. Filtering, watermark, Fast video memory access, FPSLimit, VendorID spoofing.
- Under Wine: copy **all 32-bit DLLs locally next to EXE**, not system32. Set `WINEDLLOVERRIDES=ddraw,d3d11,dxgi=n`. KnownDLLs/OLE redirection may be needed. Author disclaims Wine support; community claims v2.81.3 last Wine-friendly (UNVERIFIED).
- Combining with DXVK: dgVoodoo D3D11/DXGI output serviced by DXVK `d3d11.dll`+`dxgi.dll`. Never override d3d8/d3d9 with DXVK while dgVoodoo's are active; never stack two DDraw wrappers.

## 4. Legally safe recipe — Project I.G.I.: I'm Going In (2000)

Use your own legally acquired copy. Verified API: **Direct3D 7** (PCGamingWiki, traced via `Direct3DDevice7`), 32-bit exe, fixed-function.

Shared steps:
1. `WINEARCH=win32 WINEPREFIX=~/igi32 wineboot -u`
2. Windows version: start Windows 7; if `DDERR_UNSUPPORTED`, try XP SP3 (community, UNVERIFIED).
3. Install to ASCII path; apply LAA/4GB patch if needed.
4. Runtimes: DirectX 7/8-era components, VC++ used by installer; **no d3dcompiler**.
5. Cap FPS at 60.

Decision tree:
- Vulkan 1.4 → D7VK v2.4 + Wine DDraw.
- Vulkan 1.1/1.2, ARM/mobile → DXVK-Sarek 1.13.0.
- Need dgVoodoo2 features → dgVoodoo2 → D3D11 FL10.1 → DXVK d3d11/dxgi.
- D3D7 broken → dxwrapper `dd7to9=1` + options, optionally chain DXVK D3D9.
- Glide path → GXVK or nGlide.
- Retail exe D3D8 → DXVK d3d8+d3d9 or D3D8to9 + DXVK.
- WineD3D fallback: works but geometry errors reported.

Android: Wine frontend with 32-bit-capable x86 translator (Box64/box32 or FEX) + Vulkan 1.4 driver; prefer D7VK v2.4 or DXVK-Sarek; put 32-bit `ddraw.dll` next to exe; `WINEDLLOVERRIDES=ddraw=n,b`; avoid dgVoodoo ARM64 builds (Windows-on-ARM only).

ARM Linux: same with Box64/FEX + Mesa Turnip/Panfrost; D7VK or DXVK-Sarek; Gamescope/virtual desktop for small windows.

Common fixes: no 3D card → wrapper + ddraw override; DDERR_UNSUPPORTED → XP SP3; vertex explosions → D7VK fixes/dxwrapper options/LAA; black window → virtual desktop; slowdowns → dxwrapper DdrawNoDrawBufferSysLock.

## 5. Known pitfalls

- DXVK overrides: native for d3d8/d3d9/d3d10core/d3d11/dxgi; D7VK: `ddraw=native,builtin` (exact order); dxwrapper D3D7: `ddraw=n,b`.
- Copying DLLs to system32 without override does nothing. dgVoodoo must be local.
- DXVK ≥3.0 `dxbc-spirv` needs no d3dcompiler for D3D9/8 FF.
- IGI is 32-bit → 32-bit DLLs. AV false positives on DXVK 32-bit binaries.
- dgVoodoo + DXVK is a chain, not competing DLLs.
- Zink adds a layer; VirGL double shader translation.
- WineD3D CSMT default since 3.2/4.0; disable via registry `csmt=0` if needed.
- DXVK 2.7 removed legacy state cache; GPU pipeline libraries + `DXVK_SHADER_CACHE_PATH`. vkd3d cache `vkd3d-proton.cache`. Clear caches after wrapper/driver changes.

## 6. Citations (verified URLs)

- https://github.com/doitsujin/dxvk
- https://github.com/doitsujin/dxvk/wiki
- https://github.com/HansKristian-Work/vkd3d-proton
- https://gitlab.winehq.org/wine/vkd3d
- https://github.com/AlpyneDreams/d8vk
- https://github.com/WinterSnowfall/d7vk
- https://github.com/pythonlover02/DXVK-Sarek
- https://github.com/crosire/d3d8to9
- https://github.com/elishacloud/dxwrapper
- https://github.com/narzoul/DDrawCompat
- https://github.com/FunkyFr3sh/cnc-ddraw
- https://github.com/jp7677/dxvk-nvapi
- https://github.com/CkNoSFeRaTU/gxvk
- https://www.dege.freeweb.hu/dgVoodoo2/
- https://github.com/dege-diosg/dgVoodoo2
- https://docs.mesa3d.org/drivers/zink.html
- https://docs.mesa3d.org/drivers/virgl/
- https://www.winehq.org/announce/3.0
- https://www.winehq.org/announce/4.0
- https://www.winehq.org/announce/6.0
- https://www.pcgamingwiki.com/wiki/Project_I.G.I.%3A_I%27m_Going_In
- https://github.com/elishacloud/dxwrapper/issues/453
