# 05 — Graphics Translators: From Direct3D to Vulkan

This module maps the graphics layer of the pipeline. It covers the APIs games call, the translators that convert those calls, and the rules that decide which DLL actually runs. Module 06 goes deeper on legacy Direct3D 7/8/9; module 07 covers the GPU drivers underneath everything here.

Read module 01 first if you have not. The key idea is that **an API, a translator, and a driver are three different things**:

| Thing | What it is | What it does *not* do |
|---|---|---|
| Graphics API (Direct3D, [OpenGL](https://www.opengl.org/), [Vulkan](https://www.vulkan.org/)) | A specification: a set of calls and guarantees a program may make | It does not execute anything by itself |
| Graphics translator ([DXVK](https://github.com/doitsujin/dxvk), [VKD3D-Proton](https://github.com/HansKristian-Work/vkd3d-proton), [WineD3D](https://gitlab.winehq.org/wine/wine)) | Software that implements one API in terms of another | It is not a GPU driver and does not talk to hardware directly |
| GPU driver ([Turnip](https://docs.mesa3d.org/drivers/freedreno.html), RADV, Qualcomm UMD) | Software that turns Vulkan/OpenGL into GPU commands | It does not know what Direct3D is |

A typical frame travels through all three. Keeping them separate is what lets you debug: a missing Vulkan extension is a driver problem, a shader that fails to compile is a translator problem, and a call the game should never have made is an API/compatibility problem.

## 5.1 The APIs and their generations

### Direct3D

[DirectX](https://learn.microsoft.com/en-us/windows/win32/directx) is Microsoft's family of multimedia APIs; **Direct3D** is its 3D graphics component. Each version added capabilities:

| API | Era | Shading model | Key idea |
|---|---|---|---|
| Direct3D 5/6/7 | 1997–1999 | Fixed function only | Game sets render states; GPU applies a built-in transform/lighting/texture formula |
| Direct3D 8 | 2000 | Shader Model 1.0/1.1 | Programmable vertex and pixel shaders arrive |
| Direct3D 9 | 2002 | Shader Model 2.0, later 3.0 | Long-lived standard; hundreds of games use it |
| Direct3D 10 | 2006 | Shader Model 4.0 | New resource model, mandatory unified shaders |
| Direct3D 11 | 2009 | Shader Model 5.0 | Tessellation, compute, multithreaded rendering |
| Direct3D 12 | 2015 | SM 5.0/5.1/6.x | Low-level, explicit command lists and descriptor management |

**Feature level** and **shader model** are different axes. A *feature level* (for example `11_0` or `12_1`) is a declared capability tier: an application requests one and receives the subset of the API that tier guarantees. A *shader model* is the version of the shader language and its instruction set. A game can use D3D11 calls while requesting feature level `10_0`, and a translator can expose a higher feature level than the physical GPU supports by emulating some features. See Microsoft's [feature levels reference](https://learn.microsoft.com/en-us/windows/win32/direct3d11/feature-levels) and [shader models reference](https://learn.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl-models).

### OpenGL, OpenGL ES and Vulkan

- [OpenGL](https://www.opengl.org/) is a cross-platform, driver-managed API. You set state; the driver decides how to execute. Desktop Linux exposes full OpenGL.
- [OpenGL ES](https://www.khronos.org/opengles/) is the mobile variant. Android primarily exposes GLES, not desktop OpenGL. They are related but not identical.
- [Vulkan](https://www.vulkan.org/) is a low-overhead, explicit API. The application manages synchronization, memory and command buffers. It is the main destination for modern translation layers and is documented by the [Khronos Vulkan docs](https://docs.vulkan.org/).

On Android, Vulkan does not come from a "GL driver" you install. It comes from the device's vendor userspace, exposed through the Vulkan loader and an ICD manifest. Module 07 explains that boundary.

## 5.2 What a translator actually does

A translator is a user-space DLL that the game loads instead of (or on top of) Microsoft's. It intercepts API calls and rebuilds equivalent work in the destination API.

```text
Game calls: IDirect3DDevice9::DrawIndexedPrimitive(...)
        |
        v
+------------------------------------------------------+
| Translator (e.g. DXVK d3d9.dll)                       |
|  - map D3D state -> Vulkan pipeline state             |
|  - translate HLSL bytecode -> SPIR-V                  |
|  - allocate buffers/images in Vulkan memory           |
|  - record Vulkan command buffers                      |
+------------------------------------------------------+
        |
        v
Vulkan loader -> ICD -> GPU driver -> GPU
```

Concrete responsibilities:

1. **State translation.** D3D exposes a large global state block (render states, texture stages, samplers). Vulkan requires most of that to be baked into pipeline objects and descriptor sets. The translator detects state changes and rebuilds only what changed.
2. **Shader translation.** HLSL bytecode (DXBC) is translated to SPIR-V, which the GPU driver compiles to machine code. This is where most first-launch stutter originates.
3. **Resource translation.** D3D textures and buffers become Vulkan images and buffers, with their own memory allocation and synchronization.
4. **Submission and presentation.** The translator records command buffers, submits them to queues, and presents swapchain images to the host compositor.
5. **Feature emulation.** Where the host lacks a feature, the translator may emulate it in shaders or extra passes.

The diagram above is a **conceptual call path**, not literal runtime order. In practice Wine, CPU translation, the game's worker threads and the GPU driver run concurrently; several layers may live in one process, and a single frame can cross the diagram many times. Use the diagram to place blame, not to read a stack trace.

## 5.3 The translator matrix

Versions below are the snapshot from the research brief, verified 2026-11-23. "*Latest*" means newest on the primary release page at that date.

| Tool | Source → destination | Latest (Nov 2026) | Requirements / notes | License |
|---|---|---|---|---|
| [DXVK](https://github.com/doitsujin/dxvk) | D3D8/9/10/11 → Vulkan | v3.1.1 (15 Sep 2026) | Vulkan 1.4 since 3.0; `VK_KHR_maintenance5` since 2.7; D3D11 up to FL 12_1; `dxbc-spirv` compiler since 3.0 | zlib/libpng |
| [VKD3D-Proton](https://github.com/HansKristian-Work/vkd3d-proton) | D3D12 → Vulkan | v3.1 (28 Oct 2026) | Vulkan 1.3 + descriptor indexing + `VK_EXT_robustness2` + `VK_KHR_push_descriptor`; shares DXGI with DXVK 2.1+ | LGPL-2.1-or-later |
| [VKD3D](https://gitlab.winehq.org/wine/vkd3d) (upstream) | D3D12 → Vulkan; shader library | v1.15 (19 Feb 2025) | `libvkd3d-shader` handles HLSL/DXIL/DXBC/SM1–3 → SPIR-V/GLSL/MSL; a library, not a game DLL | LGPL-2.1+ |
| [D8VK](https://github.com/AlpyneDreams/d8vk) | D3D8 → DXVK D3D9 backend → Vulkan | Merged into DXVK; final standalone v1.0 (10 May 2026) | Standalone discontinued; use current DXVK's `d3d8.dll` | zlib |
| [D7VK](https://github.com/WinterSnowfall/d7vk) | D3D7/6/5/3 immediate mode → DXVK D3D9 backend → Vulkan | v2.4 (28 Oct 2026), maintenance mode | Vulkan 1.4; deploys as `ddraw.dll`; needs real DDraw underneath; no retained mode; 2048-pixel surface limit | zlib |
| [DXVK-Sarek](https://github.com/pythonlover02/DXVK-Sarek) | D3D3–11 → Vulkan 1.1/1.2 | v1.14.1 (12 Oct 2026; [releases](https://github.com/pythonlover02/DXVK-Sarek/releases)) | Includes D7VK v2.2 + mobile fixes; DDraw/D7VK built x86-only; fork author reports their projects on a 1–3 month halt | zlib |
| [dgVoodoo2](https://www.dege.freeweb.hu/dgVoodoo2/) | Glide, DDraw/D3D3–7, D3D8.1, D3D9 → D3D11/D3D12 or WARP | v2.87.6 (25/26 Oct 2026) | Closed-source freeware; GitHub repo [archived](https://github.com/dege-diosg/dgVoodoo2); author does not target Wine/Linux | proprietary |
| [D3D8to9](https://github.com/crosire/d3d8to9) | D3D8 API + bytecode shaders → D3D9 | Active source; no formal release verified | Exact translation only; VSync may differ; needs legacy D3DX for shaders | BSD-2-Clause |
| [DxWrapper](https://github.com/elishacloud/dxwrapper) | DDraw/D3D1–7 → D3D9, D3D8 → D3D9, DInput/DSound hooks | v1.9.8700.25 (12 Oct 2026) | Drop-in stub DLL + INI; can chain to DXVK | zlib/BSD-2/others |
| [DDrawCompat](https://github.com/narzoul/DDrawCompat) | In-place DDraw/D3D1–7 + GDI fixes (no API conversion) | v0.7.2 pre-release (14 Oct 2026) | Windows Vista–11 only; unsupported in Wine/VMs; needs SSE2 + SM3 GPU | BSD Zero Clause |
| [cnc-ddraw](https://github.com/FunkyFr3sh/cnc-ddraw) | 2D software DirectDraw → GDI/OpenGL/D3D9 | v7.1.0.0 (28 Dec 2024) | 2D/paletted software-DDraw only; works in Wine on Linux/macOS/Android | MIT |
| [WineD3D](https://gitlab.winehq.org/wine/wine) | DDraw/D3D1–11 → OpenGL (experimental Vulkan renderer) | Bundled with Wine | CSMT default since Wine 3.2/4.0; often slow/buggy for D3D7 | LGPL-2.1+ |
| [Zink](https://docs.mesa3d.org/drivers/zink.html) | OpenGL/ES → Vulkan (Mesa Gallium) | Mesa release train | GL 2.1 → 4.6; supplies OpenGL to WineD3D where native GL is absent | MIT |
| [VirGL](https://docs.mesa3d.org/drivers/virgl/) / [Venus](https://docs.mesa3d.org/drivers/venus.html) | Guest OpenGL → host OpenGL; guest Vulkan → host Vulkan | virglrenderer 1.0+ | Linux guest under QEMU; GL 4.3 default, 4.6 with blob; double shader translation | MIT (upstream virglrenderer LICENSE; Mesa's VirGL/Venus drivers are also MIT) |
| [GXVK](https://github.com/CkNoSFeRaTU/gxvk) | 3Dfx Glide 2.x/3.x → Vulkan | Early alpha v0.1 (2026) | Wine/Linux; Windows untested; for games with a Glide path | zlib |
| [DXVK-NVAPI](https://github.com/jp7677/dxvk-nvapi) | NVAPI/NVOFAPI → DXVK/VKD3D | v0.9.3 (30 Oct 2026) | DLSS/Reflex/OMM; enable with `DXVK_ENABLE_NVAPI=1` | MIT |

Two clarifications that prevent common mistakes:

- **VirGL and Venus are not alternatives to DXVK.** They translate *guest* OpenGL/Vulkan to *host* OpenGL/Vulkan inside a virtual machine. DXVK runs in the guest and produces Vulkan; Venus then carries that Vulkan to the host. They compose rather than compete.
- **Zink is not a D3D translator.** It implements OpenGL on top of Vulkan. Its relevance here is indirect: it can provide the OpenGL that WineD3D needs on hosts with no usable native OpenGL.

## 5.4 Modern paths in detail

### DXVK

DXVK is the default choice for D3D8/9/10/11. Since version 3.0 it requires **Vulkan 1.4** and uses a new `dxbc-spirv` compiler that translates DXBC directly to SPIR-V. Version 2.7 removed the legacy state cache; current versions use Vulkan graphics pipeline libraries and a shader cache.

Important deployment facts:

- D3D8 support was merged from D8VK. A D3D8 game needs **both `d3d8.dll` and `d3d9.dll`** from DXVK.
- D3D11 is supported up to feature level `12_1`.
- Fixed-function D3D9 work is handled by generated ubershaders since 3.0, so **no `d3dcompiler` DLL is needed** for those paths.
- Windows is officially unsupported as a host; Linux/Wine is the primary target. Android and ARM work through Wine frontends and their bundled Vulkan driver.
- DXGI is shared with VKD3D-Proton 2.1+, so pair compatible `dxgi.dll` versions.

### VKD3D-Proton

[VKD3D-Proton](https://github.com/HansKristian-Work/vkd3d-proton) implements D3D12 on Vulkan and is the tool for D3D12 games. It requires Vulkan 1.3 plus descriptor indexing, `VK_EXT_robustness2` and `VK_KHR_push_descriptor` — a notably higher bar than "a Vulkan driver exists." It installs the native `d3d12.dll` / `d3d12core.dll` pair. Windows builds are for development testing, not production use.

Do not confuse it with upstream [VKD3D](https://gitlab.winehq.org/wine/vkd3d), which is a library (including the `libvkd3d-shader` translator) used by Wine and other projects, not a drop-in pair of game DLLs.

### DXVK-NVAPI

[DXVK-NVAPI](https://github.com/jp7677/dxvk-nvapi) implements NVIDIA's NVAPI/NVOFAPI on top of DXVK/VKD3D, exposing features such as DLSS, Reflex and OMM to games running on non-NVIDIA or non-Windows stacks. It is enabled with `DXVK_ENABLE_NVAPI=1` and is a companion to the graphics translator, not a replacement for it. An ARM64 build exists.

## 5.5 Legacy paths in detail

The legacy tools exist because older games use fixed-function hardware and DirectDraw in ways modern DXVK does not directly emulate. Module 06 covers when to choose each.

- **D7VK** targets D3D7 immediate mode by translating to DXVK's D3D9 backend. It deploys as `ddraw.dll` with a `native,builtin` override and relies on Wine's real DirectDraw implementation underneath. It rejects retained-mode D3D7 and has a 2048-pixel surface limit. Requires Vulkan 1.4. Now in maintenance mode.
- **D8VK** was the original D3D8 → DXVK bridge. It is merged into upstream DXVK; use DXVK's `d3d8.dll` rather than the discontinued standalone release.
- **D3D8to9** converts D3D8 calls and bytecode shaders to D3D9 with exact translation, then DXVK handles D3D9. VSync behaviour may differ, and legacy D3DX is needed for shaders.
- **DxWrapper** is a configurable stub DLL plus INI file. For D3D7 it can redirect `ddraw` to D3D9 (`dd7to9=1`) and then chain to DXVK's D3D9. It includes game-specific options, including some for Project I.G.I.
- **dgVoodoo2** wraps Glide, DirectDraw/D3D3–7, D3D8.1 and D3D9 and outputs D3D11/D3D12 (or WARP). It supports full SM1/2/3 fixed function, paletted textures and 8/16/32-bit modes — exactly the features legacy games need. It is closed-source freeware targeting Windows 7+, and its author does not support Wine/Linux. Under Wine it must be installed **locally next to the EXE**, never in `system32`.
- **cnc-ddraw** handles 2D software-DirectDraw games (including paletted titles), replacing the surface with GDI/OpenGL/D3D9. It supports Wine on Linux, macOS and Android. It does **not** help a game that uses Direct3D for 3D.
- **DDrawCompat** makes in-place fixes to DirectDraw/GDI without converting APIs, but it is Windows-only and unsupported in Wine or VMs, so it is not usable in an Android or ARM-Linux pipeline.
- **GXVK** targets the Glide path that some 1990s games offer instead of Direct3D. It is early alpha.

## 5.6 Selection chart by game API

| Game uses | First choice | Alternative | Notes |
|---|---|---|---|
| D3D12 | VKD3D-Proton | — | Check Vulkan 1.3 + descriptor indexing + robustness2 + push descriptor |
| D3D11 | DXVK | WineD3D | DXVK up to FL 12_1 |
| D3D10 | DXVK | WineD3D | — |
| D3D9 | DXVK | WineD3D, dgVoodoo2 | dgVoodoo2 outputs D3D11 → chain to DXVK |
| D3D8 | DXVK (`d3d8.dll` + `d3d9.dll`) | D3D8to9 + DXVK, DxWrapper | D8VK standalone is obsolete |
| D3D7 / D3D6 / D3D5 | D7VK (Vulkan 1.4) | DxWrapper `dd7to9`, dgVoodoo2, DXVK-Sarek | See module 06 |
| D3D3 / D3D2 / D3D1 | dgVoodoo2, DXVK-Sarek | DxWrapper | — |
| DirectDraw (2D/paletted) | cnc-ddraw | DxWrapper, dgVoodoo2 | DDrawCompat is Windows-only |
| Glide 2.x/3.x | GXVK | nGlide (where supported) | Alpha software |
| OpenGL | Native OpenGL, or Zink → Vulkan | — | WineD3D is for D3D, not a general GL replacement |
| Vulkan | Native Vulkan | Venus (in a VM) | Rare in Windows games |

## 5.7 DLL override rules

Wine decides which DLL satisfies an import using overrides (`WINEDLLOVERRIDES`, or the prefix's registry). The two values are **native** (`n`, a real Windows DLL placed in the prefix) and **builtin** (`b`, Wine's implementation). `n,b` means "try native first, then builtin"; `b,n` reverses that.

Rules that matter in practice:

| Translator | Required override / placement |
|---|---|
| DXVK (D3D8/9/10/11) | `native` for `d3d8`, `d3d9`, `d3d10core`, `d3d11`, `dxgi`; copy the 32-bit or 64-bit DLLs beside the EXE or into the right prefix `syswow64`/`system32` |
| D7VK | `ddraw=native,builtin` — the order matters; it needs Wine's real DDraw as fallback |
| DxWrapper (D3D7) | `ddraw=n,b` |
| dgVoodoo2 | Copy **all** its DLLs locally next to the EXE; `WINEDLLOVERRIDES=ddraw,d3d11,dxgi=n`; **never** in `system32` |

Three rules prevent most "it does nothing" bugs:

1. **Copying a DLL into the prefix without an override does nothing.** The loader will not prefer it.
2. **Match the architecture.** A 32-bit game needs 32-bit translator DLLs. A 64-bit DLL in a 32-bit prefix will not load.
3. **Chain, do not stack.** dgVoodoo2 → D3D11 → DXVK is a valid chain; overriding `d3d8`/`d3d9` with DXVK while dgVoodoo's own D3D8/D3D9 DLLs are active is not. Never place two DirectDraw wrappers over the same game.

## 5.8 Shader caches and first-launch stutter

Shader translation is expensive, so every layer caches its work:

| Cache | Where / how | Control |
|---|---|---|
| DXVK | Graphics Pipeline Library (2.x+) + `DXVK_SHADER_CACHE_PATH` (default AppData/Local); legacy state cache removed in 2.7 | `DXVK_HUD=compiler` to watch compilation |
| VKD3D-Proton | `vkd3d-proton.cache` in the working directory; pipeline library app cache option | `VKD3D_SHADER_CACHE_PATH`, `VKD3D_CONFIG=pipeline_library_app_cache` |
| Mesa | Disk shader cache, default 1 GiB | `MESA_SHADER_CACHE_DIR`, `MESA_SHADER_CACHE_MAX_SIZE`, `MESA_DISK_CACHE_SINGLE_FILE=1` |
| Fossilize | `.foz` capture/replay of pipeline creation | `VK_LAYER_fossilize`, `fossilize-replay` |

Practical guidance:

- Games that "load shaders" in a menu still stall on first run even with GPL. This is expected; let the cache warm.
- **Clear caches after changing a wrapper, translator or driver version.** Stale pipeline state is a common cause of corruption and crashes.
- There is **no official `DXVK_ASYNC`**. The original dxvk-async project is archived; a third-party successor (`dxvk-gplasync`) exists. Async compilation can render missing geometry temporarily and is **not safe for online games**.
- Do not confuse the translator's cache with the driver's cache. Both may need clearing.

## 5.9 Fixed-function constraints

**Fixed function** means the GPU applies a built-in formula — transform, lighting, texture blending — determined by render states rather than by a shader program. D3D7 is entirely fixed function; D3D8/9 mix fixed function with programmable shaders (SM1–3).

This matters because Vulkan has no fixed-function pipeline. Translators must express it as shaders:

- DXVK generates ubershaders for D3D9 fixed function (since 3.0).
- D7VK bridges D3D7 fixed function to DXVK's D3D9 fixed-function path.
- dgVoodoo2 generates shaders for SM1/2/3 fixed function and supports paletted textures and 8/16/32-bit modes.

Consequences for users:

- Features that depend on exact fixed-function behaviour (paletted textures, 16-bit colour, specific fog or alpha-test semantics) are where legacy games break. Prefer a translator that explicitly supports them.
- Paletted 8-bit DirectDraw can be upscaled with nearest-neighbour or bilinear shaders; cnc-ddraw restricts bicubic/lanczos filters to 16/32-bit games, and some titles have per-game upscaling issues — test per title.
- Fixed-function paths usually do not need `d3dcompiler`; shader-based D3D9/D3D11 paths may need a working `d3dcompiler_*` DLL depending on the translator version.

## 5.10 32-bit versus 64-bit

The game's PE architecture decides everything downstream:

- A **32-bit game** loads 32-bit DLLs (`syswow64`) from a 64-bit prefix under Wine 11.0's new WoW64. Pure `WINEARCH=win32` prefixes are deprecated and unsupported in that mode, though older Wine or a legacy wrapper may still need one. Almost every game from the D3D7/8 era is 32-bit.
- A **64-bit game** needs 64-bit DLLs and a 64-bit prefix.
- On ARM64 hosts, CPU translation must cover the game's architecture. [FEX](https://github.com/FEX-Emu/FEX) supports both 32-bit and 64-bit x86 binaries; [Box64](https://github.com/ptitSeb/box64) targets x86-64 and [Box86](https://github.com/ptitSeb/box86) targets 32-bit x86. Frontends such as Winlator bundle Wine with Box86/Box64 and select per container — verify the exact build.
- DXVK-Sarek's DDraw/D7VK components are built x86-only, which suits 32-bit legacy games but not 64-bit ones.
- Antivirus false positives on 32-bit DXVK DLLs were acknowledged by the DXVK maintainer in September 2026; this is primarily a Windows-host AV problem, but the same 32-bit DLLs have also triggered ClamAV false positives on Linux (for example Bottles/Flatpak). It is not Android-specific ([DXVK issue #5762](https://github.com/doitsujin/dxvk/issues/5762)).

## 5.11 What to remember

- API, translator and driver are separate layers; identify which one is failing.
- Feature level and shader model are different axes of capability.
- DXVK handles D3D8–11 on Vulkan 1.4; VKD3D-Proton handles D3D12 on Vulkan 1.3+ with specific extensions.
- Legacy games need legacy translators: D7VK, D8VK/D3D8to9, DxWrapper, dgVoodoo2, cnc-ddraw.
- Zink provides OpenGL on Vulkan; VirGL/Venus carry guest graphics over a VM; neither replaces a D3D translator.
- DLL override order and architecture must match the game, or the translator never loads.
- Clear shader caches after changing wrappers, translators or drivers.
- Fixed function is emulated in shaders; that is why paletted and 16-bit legacy content is fragile.

Next: [06 — Legacy DirectX](06-legacy-directx.md).

## References

- DXVK — https://github.com/doitsujin/dxvk ; wiki: https://github.com/doitsujin/dxvk/wiki
- VKD3D-Proton — https://github.com/HansKristian-Work/vkd3d-proton
- VKD3D (upstream) — https://gitlab.winehq.org/wine/vkd3d
- D8VK — https://github.com/AlpyneDreams/d8vk
- D7VK — https://github.com/WinterSnowfall/d7vk
- DXVK-Sarek — https://github.com/pythonlover02/DXVK-Sarek
- dgVoodoo2 — https://www.dege.freeweb.hu/dgVoodoo2/ ; https://github.com/dege-diosg/dgVoodoo2
- D3D8to9 — https://github.com/crosire/d3d8to9
- DxWrapper — https://github.com/elishacloud/dxwrapper
- DDrawCompat — https://github.com/narzoul/DDrawCompat
- cnc-ddraw — https://github.com/FunkyFr3sh/cnc-ddraw
- WineD3D (Wine) — https://gitlab.winehq.org/wine/wine ; CSMT notes: https://www.winehq.org/announce/3.0 , https://www.winehq.org/announce/4.0
- Zink — https://docs.mesa3d.org/drivers/zink.html
- VirGL — https://docs.mesa3d.org/drivers/virgl/ ; Venus — https://docs.mesa3d.org/drivers/venus.html ; virglrenderer — https://gitlab.freedesktop.org/virgl/virglrenderer
- GXVK — https://github.com/CkNoSFeRaTU/gxvk
- DXVK-NVAPI — https://github.com/jp7677/dxvk-nvapi
- DirectX documentation — https://learn.microsoft.com/en-us/windows/win32/directx
- Feature levels — https://learn.microsoft.com/en-us/windows/win32/direct3d11/feature-levels
- Shader models — https://learn.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl-models
- OpenGL — https://www.opengl.org/ ; OpenGL ES — https://www.khronos.org/opengles/ ; Vulkan — https://www.vulkan.org/ ; Vulkan docs — https://docs.vulkan.org/
- FEX — https://github.com/FEX-Emu/FEX ; Box64 — https://github.com/ptitSeb/box64 ; Box86 — https://github.com/ptitSeb/box86
- Mesa environment variables — https://docs.mesa3d.org/envvars.html ; Fossilize — https://github.com/ValveSoftware/Fossilize
