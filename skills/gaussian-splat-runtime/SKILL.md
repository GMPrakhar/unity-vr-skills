---
name: gaussian-splat-runtime
description: Load 3D Gaussian splat scans (.ply from Polycam, Scaniverse, Luma or COLMAP-based training) at runtime in Unity and render them, rather than importing them in the editor. Use when players supply their own scans, when splats must stream in at runtime, or when adapting the aras-p UnityGaussianSplatting package for runtime ingestion.
---

# Runtime Gaussian splat ingestion in Unity

The standard Unity splat renderer, `aras-p/UnityGaussianSplatting` (MIT), imports
`.ply` files **in the editor** and bakes them into `TextAsset` blobs. That is
fine for authored content and useless the moment the *player* supplies the scan.
This skill covers building the runtime path.

## Decide first: do you actually need runtime loading?

If scans ship with the game, use the editor importer. It is better tested, and it
gives you compressed formats for free. Build the runtime path only when the scan
arrives after the build: user captures, downloads, or user-generated content.

## The 3DGS `.ply` format

Binary little-endian, 62 float32 properties per splat:

```
x, y, z, nx, ny, nz, f_dc_0..2, f_rest_0..44, opacity, scale_0..2, rot_0..3
```

Points that are easy to get wrong:

- **`rot_0` is `w`**, not `x`. The quaternion is stored `w, x, y, z`.
- **`f_rest` is channel-major**: 15 coefficients for red, then 15 for green, then
  15 for blue. It must be interleaved into `float3` per band before use.
- **`nx, ny, nz` are usually meaningless.** Most trainers write zeros. Do not
  build lighting on them.
- Values are stored **unactivated** and must be linearised:

```csharp
color   = dc * 0.2820948f + 0.5f;   // SH band 0 to linear RGB
opacity = 1f / (1f + exp(-v));      // sigmoid
scale   = exp(v);                   // log-scale to metres
rotation = normalize(q);            // then swizzle to the renderer's convention
```

Parse the header generically by property name and offset rather than assuming the
canonical order. Exporters do reorder and add fields, and a silent offset error
produces a plausible-looking but subtly wrong cloud, which is painful to debug.

Reject or explicitly handle ASCII `.ply`; most scanners emit binary, and a
misdetected format reads garbage.

## Coordinate conventions

Scanners disagree about handedness and up-axis. The upstream importer applies
**no** flip, which means a scan may load mirrored or upside down.

Do not scatter axis flips through the reader. Expose one explicit convention
enum, apply it as a single rigid pre-rotation, and default to "as authored":

```csharp
public enum SourceConvention { AsAuthored, ThreeDGS_YDown }
```

A mirrored scan is the tell for a handedness flip; an upside-down scan is a
Y-axis flip. They need different fixes, so keep them distinguishable rather than
combining them into one "fix orientation" toggle.

Validate against a real capture from your target scanner before trusting any
convention. Synthetic test data almost always uses the convention you wrote it
in, so it cannot catch this class of bug.

## Building an asset at runtime

`GaussianSplatAsset` is built around editor-imported `TextAsset` blobs. Add a
parallel runtime path rather than replacing the existing one, so upstream rebases
stay tractable:

- Add `NativeArray` payload fields alongside the `TextAsset` fields, plus an
  `isRuntime` flag.
- Add accessors (`GetPosData<T>()`, `GetOtherData<T>()`, ...) and size properties
  that switch on that flag.
- Change the renderer to go through the accessors instead of touching
  `TextAsset` directly.
- Make the URP/HDRP render feature `public` if you need to add it from a script.

Keep every change confined to those seams and document them, or the next upstream
merge becomes archaeology.

### The chunk-data constraint

The renderer supports chunkless assets **only when all four formats are
lossless** (`Float32` position, `Float32` scale, `Float32x4` colour, `Float32`
SH). Any compressed format requires chunk data, produced by the equivalent of
`CalcChunkDataJob`.

The simplest working runtime builder therefore uses lossless formats — and that
choice is expensive:

| | Lossless | Typical compressed |
| --- | --- | --- |
| Bytes per splat | ~236 | ~25–30 |
| 700k splats (a house) | ~163 MB | ~20 MB |

SH alone is 192 B/splat at `Float32`. **163 MB for one scene will not fit a
standalone headset's budget.** Treat lossless as a bring-up step, then implement
compressed encoding (Norm11 position/scale, Norm8x4 colour, Norm6 clustered SH)
plus chunk generation before shipping to mobile VR.

If SH dominates your budget and the scene is diffuse, dropping to SH band 0 is a
much bigger win than any other single change.

## Performance and memory expectations

Measure and report these separately, and never conflate them:

- **Load time** scales roughly linearly with splat count. Expect on the order of
  a few seconds for ~700k splats including parse, Morton reorder and encode.
- **Frame time** on a desktop GPU is not predictive of a mobile headset. Sorting
  cost grows with splat count and with how much of the cloud is on screen.
- Parsing a large `.ply` needs the file plus the decoded arrays in memory at once.
  Offer a `maxSplats` subsampling cap so huge captures degrade instead of
  crashing.

Do the parse and encode off the main thread, or the app will hitch hard on load.

## Rendering in VR

Upstream VR support is minimal — the renderer consults only
`XRSettings.eyeTextureWidth`. **Multi-pass stereo is required**; single-pass
instanced will break. See the `unity-xr-simulator-testing` skill for verifying
this without hardware.

Splats are sorted per view. Two eyes means two sorts, so budget for it.

## Verifying the pipeline

Splats have no geometry to inspect, so assert on rendered pixels. Generate a
synthetic scan with known colours and dimensions, then assert that viewpoints
aimed at known surfaces resolve to the expected colour, and that reported bounds
match the generator. See `unity-headless-verification` for the probe pattern,
including the linear-vs-sRGB trap that makes every colour assertion fail by a
gamma.

Useful invariant: run the same scene at two very different splat densities. Pixel
colours and derived geometry should barely move. If they do, the encoding or
reorder is losing information.

## Other formats

- `.spz` (Scaniverse) is compressed and much smaller over the wire. Upstream has
  an editor-side reader that can be ported to runtime.
- `.splat` is a compact community format, easy to parse, but lossy on SH.

## Checklist

- [ ] Header parsed by property name, not assumed offsets; binary vs ASCII handled
- [ ] `rot_0 = w`; `f_rest` de-interleaved from channel-major
- [ ] Activations applied: sigmoid opacity, exp scale, SH-0 colour, normalised rotation
- [ ] One explicit coordinate convention, validated against a real capture
- [ ] Runtime payloads added alongside `TextAsset` ones, not replacing them
- [ ] Lossless-only chunkless constraint understood; compression planned for mobile
- [ ] `maxSplats` cap available; parse/encode off the main thread
- [ ] Multi-pass stereo asserted for VR
