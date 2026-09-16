# RiveRenderStream

A standalone Windows executable that renders Rive (`.riv`) content as a
[Disguise RenderStream](https://www.disguise.one/en/products/renderstream/)
workload, with **bidirectional texture support**. It is the Rive analogue of the
`SpoutRenderstream` (DX11 branch) tool: RenderStream drives it, it renders on the
GPU, and it can both send and receive textures.

- **Output (Rive → Disguise):** each RenderStream *scene* is one artboard in the
  `.riv`. Rive renders it into a D3D11 texture that is `sendFrame`'d directly to
  Disguise (`RS_FRAMETYPE_DX11_TEXTURE`) - no CPU readback.
- **Input (Disguise → Rive):** every view-model **image** property on an artboard
  is exposed as an `RS_PARAMETER_IMAGE`. The incoming texture is written straight
  into a Rive **GPU Canvas** (`rive::gpu::RenderCanvas`); the canvas's live,
  GPU-sampled image is bound to that view-model property. No CPU round-trip.

## Layout

| File | Role |
|------|------|
| `src/Main.cpp` | RenderStream loop: schema, `awaitFrameData`, receive/send. |
| `src/RiveRSDevice.hpp` | D3D11 device + Rive `RenderContextD3DImpl`; per-stream targets and per-input GPU canvases. |
| `src/RiveRSScene.hpp` | `.riv` load, artboard/state-machine/view-model instances, draw + image binding. |
| `include/` | RenderStream C++ API glue (`d3renderstream.h`, `renderstream.hpp`, `d3helpers.hpp`), vendored from the Disguise SDK. |
| `scripts/build_rive.bat` | Clones + builds the Rive runtime static libs (canvas + scripting). |

The single shared D3D11 device is handed to both Rive (`RenderContextD3DImpl`)
and `rs_initialiseGpGpuWithDX11Device`, which is what makes the zero-copy
send/receive possible. The Rive rendering recipe follows the
[TDRive](https://github.com/medcelerate/TDRive) plugin's D3D11 backend.

## Building

Windows only (Disguise RenderStream + D3D11). From a *Developer Command Prompt
for VS 2022* with `premake5.exe` and Git for Windows on `PATH`:

```bat
scripts\build_rive.bat release
cmake -B build -G "Visual Studio 17 2022" -A x64
cmake --build build --config Release
```

`build_rive.bat` clones `rive-app/rive-runtime` into `third_party/` and builds it
`--with_rive_canvas --with_rive_scripting` (GPU Canvas + Luau). Output:
`build\Release\RiveRenderStream.exe`.

The build defines `RIVE_CANVAS` / `RIVE_ORE` so the `makeRenderCanvas` API is
visible and its object layout matches the `--with_rive_canvas` runtime libs.
Prebuilt binaries are attached to each [GitHub Release](../../releases).

## Setup in Disguise (`.riv` as a RenderStream asset)

RiveRenderStream registers as the handler for `.riv` files, so you select a `.riv`
directly in Disguise rather than passing it on a command line. One-time setup on
each render (rx) machine:

1. **Enable the extension.** In your *RenderStream Projects* folder (path from
   `HKEY_CURRENT_USER\Software\d3 Technologies\d3 Production Suite\RenderStream
   Projects Folder`), create/append `permitted_custom_extensions.txt` with one
   line:
   ```
   riv
   ```
2. **Associate `.riv` with the exe.** Set Windows' default app for `.riv` to
   `RiveRenderStream.exe` (Open with → Choose another app → Always). `d3service`
   launches custom-extension assets with their registered default application.
3. **Drop your `.riv`** files into the RenderStream Projects folder. `d3service`
   detects each as a RenderStream Asset (and picks up an `rs_<name>.json` schema
   cache next to it, which the exe writes on first launch).

Then, in a **RenderStream layer**, pick the `.riv` asset. Disguise launches
`RiveRenderStream.exe` with the asset path as the first argument automatically -
no `-f` needed. Each artboard shows up as a selectable scene; view-model image
properties appear as image inputs for bidirectional textures.

## Sequenceable parameters (per scene)

Each scene exposes these RenderStream parameters, controllable and sequenceable
from Disguise (no command line needed):

| Parameter | Type | Notes |
|-----------|------|-------|
| **Fit** | dropdown | Fill · Contain · Cover · Fit width · Fit height · None · Scale down · Layout |
| **Alignment** | dropdown | Top/Center/Bottom × Left/Center/Right |
| **&lt;image properties&gt;** | image | One per view-model image property on the artboard (bidirectional input). |

The `--fit` / `--align` command-line values only set each dropdown's **default**;
the live parameter drives them at runtime.

> **Graphics adapter is not a parameter.** The D3D11 device (shared with
> RenderStream) is created once at startup, before any parameter can be read, so
> the adapter can't be switched live. Set it with the `--graphics-adapter` / `-g`
> workload argument (or leave `-1` to use RenderStream's default adapter).

## Command-line arguments

The asset `.riv` is the first positional argument (Disguise supplies it). Extra
options can be set in the RenderStream layer's *workload arguments*; for manual
runs outside Disguise, pass them yourself:

```
RiveRenderStream.exe path\to\scene.riv [options]
```

| Argument | Default | Meaning |
|------|---------|---------|
| *(positional)* / `--file`, `-f` | *(from Disguise)* | `.riv` asset to render. `--file` is an explicit override for manual runs. |
| `--fit` | `1` (contain) | *Default* for the Fit parameter: 0 fill · 1 contain · 2 cover · 3 fitWidth · 4 fitHeight · 5 none · 6 scaleDown · 7 layout. Overridden live by the Fit dropdown. |
| `--align` | `4` (center) | *Default* for the Alignment parameter: 0-8 (topLeft … bottomRight). Overridden live by the Alignment dropdown. |
| `--graphics-adapter`, `-g` | `-1` | DXGI adapter ordinal (`-1` = first). Startup-only (not a live parameter). |
| `--timeout-limit` | `5000` | `awaitFrameData` timeout, ms. |
| `--no-input` | *(off)* | Disable the image-input parameters (output only). |

Unknown flags are ignored so Disguise-appended arguments never abort startup.

## Hardware-verification notes

This tool compiles against the canvas-enabled Rive runtime but the RenderStream
round-trip can only be exercised on a Windows machine with the Disguise
RenderStream runtime installed. Items to confirm on hardware:

1. **Input pixel-channel order.** The GPU-canvas backing is created in the
   RenderStream-native format (`toDxgiFormat`). If Disguise delivers inputs as
   BGRA8 while the Rive image sampler expects RGBA, colors will swap; if so, force
   the RenderStream side to RGBA8 or add a swizzle blit into the canvas.
2. **`getFrameImage2` texture requirements.** The receive texture is created
   `DEFAULT` usage with RTV/SRV/UAV binds (no `MISC_SHARED`). SpoutRenderstream's
   input texture used `MISC_SHARED`; confirm RenderStream accepts our texture, and
   add the flag if required.
3. **Schema hashes.** Scene `hash` is left 0 at build time (as SpoutRenderstream
   does); confirm `getFrameParameters` resolves image parameters after
   `setSchema`.
