# DLSS Ray Reconstruction (RTXBoost)

EscheridiaRT can hand its noisy path-traced image to DLSS Ray Reconstruction (RR) through the
RTXBoost mod. RTXBoost replaces the game's DLSS Super Resolution with DLSS RR, which denoises and
upscales in one step. The pack itself never runs RR; it only offers it.

## What it does

- Every DLSS frame, FinalCombine writes a marker texel at the bottom right corner of
  `outputBufferFinal`, so RTXBoost can see that the pack supports RR and which handshake versions
  it speaks.
- When RTXBoost runs RR, it writes an ack back. CopyToFinal carries that ack into the next frame,
  and the next frame switches to RR mode:
  - the primary ray pass writes guides for every pixel: normal, roughness, diffuse and specular
    albedo of the first surface, its linear depth and the distance the next ray travelled
    (RR uses it to move reflections with the camera);
  - on water and glass the albedo guides describe what the pixel shows: the reflection (Fresnel)
    as specular albedo, and the first solid surface behind the water or glass (found with one
    extra ray, tinted by the glass) as diffuse albedo. Without it RR treats the scene behind the
    surface as light on a flat surface and smears it;
  - FinalCombine hands DLSS RR the current filtered sample rather than the pack's progressive
    accumulation;
  - that colour is pre-exposed with the pack's auto exposure (RR denoises best near display
    brightness) and CopyToFinal takes the exposure out again, so the tonemapper sees the usual values;
  - depth and motion can describe the first surface behind a primary fog scatter without using
    the surface color guide for the fog. Water uses its displaced surface and an approximate
    previous-frame wave position. The fog-scattered pixel retains its default sky guide.
- With no ack (RTXBoost missing, RR off, RR not supported) the next frame renders exactly as
  before. The only extra work is reading the ack, the marker texel and a one-texel ack copy in
  CopyToFinal.

## Handshake versions

| version | where the guides go | DLSS settings | RTXBoost |
|---|---|---|---|
| 3 | RTXBoost's own texture, bound to the empty register u8 | all, DLAA included | 1.5.0 and newer (upcoming) |
| 2 | packed into the free L-shaped area of `outputBufferFinal` around the image | Quality, Balanced, Performance, Ultra Performance | 1.4.2 and newer |
| 1 | packed next to the image | Performance, Ultra Performance | all |

The pack offers version 3 in the marker's alpha (1801) and, where it fits, a packed version in
the other three channels: version 1 at Performance and Ultra Performance (every RTXBoost build
reads it), version 2 at Quality and Balanced. RTXBoost answers with the best version it knows,
and the pack follows the version of the ack. At DLAA the marker texel is an image pixel, so only
its alpha carries the code there.

Version 3 guide texture (`rrGuideAtlas`, RGBA16F, 2 x render width by 2 x render height + 8):

| texel | contents |
|---|---|
| (x, y) | world normal, roughness |
| (render width + x, y) | diffuse albedo, linear view depth |
| (x, render height + y) | specular albedo |
| (render width + x, render height + y) | specular hit distance, specular motion vector, diffuse hit distance (`RR_DIFFUSE_HITDIST`, else 0) |
| (0, 2 x render height) | stamp: 1907, frame number, render size (so RTXBoost knows the pack writes it) |
| (1, 2 x render height) | features: 1913, feature bits (1 = diffuse hit distance), frame number |

The pack only touches u8 after RTXBoost acked version 3. Without RTXBoost the register is
not bound at all, so it must never be read or written in other frames.

With version 3, RTXBoost also gives RR the pack's linear depth, the specular hit distance with the
game's camera matrices, the diffuse hit distance when the feature texel says it is there (it only
trusts the texel while it carries the same frame number as the stamp), and resets RR's history on
camera cuts (/tp, respawn; steady flight never counts as one): it reads `steveSpaceDelta` from the
view constants of the ray tracing passes. That reset only works with version 3 (RTXBoost follows
the view constants only while its guide texture is bound).

## Requirements

- RTXBoost with Ray Reconstruction turned on in its window (F12)
- An NVIDIA RTX GPU and driver that support DLSS RR
- RTXBoost 1.5.0 or newer for every DLSS setting and DLAA. 1.5.0 is not released yet; until then
  RTXBoost uses handshake version 1 or 2 and RR works at the settings listed for those in the table

## Turning it off

Set `#define ENABLE_RAY_RECONSTRUCTION 0` in `project/RTXStub/shaders/Include/settings.HLSL`
and rebuild. That removes all RR code. Turning RR off in RTXBoost also works without a rebuild.

Other switches in `settings.HLSL`:

- `RR_TRANSPARENT_GUIDES`: the water / glass albedo guides described above (one extra ray per
  water or glass pixel, only in RR mode). 0 = the surface's own colour, as before.
- `RR_FOG_DIRECT_SAMPLING` reduces primary air-fog direct-light variance in RR mode by sampling a
  scattering point independently of the camera path. It can require an extra shadow ray per pixel;
  set it to 0 to restore the previous sampling. Indirect fog and underwater scattering are unchanged.
- `RR_FOG_BACKGROUND_MOTION` supplies background surface depth and motion for primary fog-scattered
  pixels in RR mode without writing the background material guide. Set it to 0 to use the previous
  far-depth and sky-motion behavior.
- `RR_PSR` (primary surface replacement): on polished metal, still water and glass seen at a
  grazing angle (roughness below 0.1) the guides describe the first rough surface seen in the
  reflection or under the water, with the mirror's colour folded into the albedo guides, and depth
  and motion describe its mirror image. The mirror's other lobes are added to the albedo guides where
  their albedo is known (a water surface's own reflection and colour, a metal's leftover diffuse
  colour): the path still samples them, and without them RR took that light for noise. RR then keeps
  the mirrored image sharp instead of smearing it as light on a flat surface. Up to three extra rays
  per such pixel, only in RR mode, about 3-4 % FPS (RTX 3050). Looking through glass keeps
  `RR_TRANSPARENT_GUIDES` (with the scene behind a pane in depth and motion, RR dragged the pane's
  own reflection along and the glass smeared while moving). On by default since 2026-11-23: in
  repeated tests it is as noisy as without it, still and after camera moves. 0 = the mirror's own
  guides.
- `RR_SPEC_CHAIN_HITDIST`: the specular hit distance of glossy surfaces follows mirror-like bounces
  up to the first rough surface instead of ending after the first bounce. 0 = first bounce only.
- `RR_SPEC_ALBEDO_MULTISCATTER`: the specular albedo guide includes the multiple-scattering energy
  `brdf.hlsl` adds to rough specular (Kulla-Conty). 0 = single scattering only.
- `RR_DIFFUSE_HITDIST`: the guides also carry the diffuse hit distance, the length of the diffuse
  bounce off the surface whose diffuse albedo the guides describe (the primary surface, the surface
  seen in a mirror with `RR_PSR`, or the solid surface behind water and glass). It is meant to keep
  contact shadows and corners, where the diffuse light comes from close by, out of RR's wide blur.
  The renderer picks one lobe per path, so where the path did not bounce diffusely off that surface
  this frame, one probe ray with the same cosine distribution measures the distance (only the hit,
  no shading; skipped where the diffuse albedo is zero). RTXBoost versions older than 1.5.0 ignore
  it. Off by default: it costs 3-7 % FPS (RTX 3050) and the RR model of the 2026-11 NVIDIA driver
  showed no change at all, not even with the distance forced to 0 or to "far" for every pixel.
  0 = specular hit distance only.

## Files

- `project/RTXStub/shaders/Include/RayReconstruction.hlsl`: everything RR-specific (handshake,
  guide layouts, ack carry). The protocol is described at the top of that file.
- `project/RTXStub/shaders/Include/Generated/Signature.hlsl`: `rrGuideAtlas` at u8
- `project/RTXStub/shaders/Include/settings.HLSL`: the switches above
- `project/RTXStub/shaders/PrimaryCheckerboardRayGenInline.hlsl`: `rrBeginPixel` before and
  `rrFinishDiffuseHitDistance` (the probe ray) and `rrEndPixel` (writes the guides) after
  `RenderRay`; `rrGuidePoint` moves depth and motion vector to the mirror image on replaced pixels
- `project/RTXStub/shaders/Include/Renderer.hlsl`: primary surface guide, primary surface
  replacement and water / glass guide in `RenderVanilla`, bounce distances and lobes and the surface
  behind fog in `RenderRay`
- `project/RTXStub/shaders/FinalCombine.hlsl`: marker, keeps packed guides, pre-exposes the RR input
- `project/RTXStub/shaders/CopyToFinal.hlsl`: carries the ack, removes the pre-exposure

IncidentLight slot 279 carries the ack and slot 278 the frame's RR state and exposure from
FinalCombine to CopyToFinal. Slot 277 stores the previous frame's water time and wave origin for
water motion vectors, independently of RR.
