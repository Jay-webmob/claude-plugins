---
name: three-assets
description: Optimizing GLB/glTF models and textures — gltf-transform and gltfpack commands, Draco vs Meshopt, KTX2/Basis compression, and VRAM budgets. Use when a 3D model is too large, a WebGL scene runs out of memory, or you need to compress GLB files and textures for the web.
---

# 3D Asset Optimization

Two separate problems with different solutions:

- **Download size** — geometry compression (Draco/Meshopt) and texture format.
- **VRAM** — texture *dimensions* and GPU format. Compressing a PNG does **nothing** for VRAM.

Most people optimize the first and are surprised when mobile still crashes.

## Inspect first

```bash
npm i -g @gltf-transform/cli
gltf-transform inspect model.glb
```

This reports whether the file is geometry-heavy, texture-heavy, or draw-call-heavy — which decides
the strategy. Optimizing geometry on a texture-bound model wastes effort.

Current versions: `@gltf-transform/cli` **4.5.0**, `gltfpack`/`meshoptimizer` **1.2.0**.

## The two CLIs — mind the argument style

```bash
# gltf-transform: POSITIONAL arguments
gltf-transform <command> <input> <output>

# gltfpack: -i / -o FLAGS
gltfpack -i input.glb -o output.glb -cc -tc
```

```bash
# ✗ A very common broken copy-paste
gltfpack input.glb output.glb

# ✓
gltfpack -i input.glb -o output.glb -cc -tc
```

## gltf-transform

```bash
# One-shot pipeline (Meshopt + WebP textures + cleanup)
gltf-transform optimize model.glb out.glb --compress meshopt --texture-compress webp

# Individual steps
gltf-transform dedup   model.glb out.glb    # merge duplicate accessors/materials
gltf-transform prune   model.glb out.glb    # drop unused nodes, materials, textures
gltf-transform draco   model.glb out.glb    # Draco geometry compression
gltf-transform meshopt model.glb out.glb    # Meshopt compression
gltf-transform webp    model.glb out.glb    # textures → WebP
gltf-transform resize  model.glb out.glb --width 1024 --height 1024
gltf-transform etc1s   model.glb out.glb    # KTX2 ETC1S (small)
gltf-transform uastc   model.glb out.glb    # KTX2 UASTC (high quality)
```

**`etc1s` and `uastc` are not self-contained** — they require
[KTX-Software](https://github.com/KhronosGroup/KTX-Software) installed separately, or the command
fails. `gltfpack -tc` has the encoder built in.

A sensible order: `prune` → `dedup` → `resize` → texture compression → geometry compression.

## gltfpack

```bash
npx gltfpack -i in.glb -o out.glb -cc -tc
```

| Flag | Effect |
|---|---|
| `-c` | Meshopt compression |
| `-cc` | Higher compression ratio |
| `-cz` | Higher `KHR_meshopt` compression level |
| `-cf` | Compressed with a fallback for non-supporting viewers |
| `-tc` | KTX2/Basis texture compression (encoder included) |
| `-si <n>` | Simplify meshes to a target ratio |
| `-noq` | Disable quantization |

## Draco vs Meshopt

| | Draco | Meshopt |
|---|---|---|
| Geometry ratio | **Best** | Slightly larger |
| Decode speed | Slow | **Fast** |
| Decoder size (gzipped) | **~74 KB** | **~7.5 KB** |
| Compresses geometry | ✓ | ✓ |
| Compresses morph targets | **✗** | ✓ |
| Compresses animation | **✗** | ✓ |

Decoder sizes measured from a real `three@0.186.0` install: `meshopt_decoder.module.js` is 29,256
raw / 7,714 gzipped; Draco's glTF decoder is `draco_decoder.wasm` 192,420 / 63,240 plus
`draco_wasm_wrapper.js` 58,456 / 11,524 — about 74 KB gzipped combined.

**Default to Meshopt.** The decoder is ~10× smaller, decode is far faster (which matters on mobile
CPUs), and it handles animation and morph targets. Draco's edge in raw ratio rarely repays a 74 KB
decoder plus a slow decode.

**Draco cannot compress morph targets or animation tracks at all** — that alone disqualifies it for
most animated web assets.

Wire up the decoders in three.js:

```js
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';
import { MeshoptDecoder } from 'three/addons/libs/meshopt_decoder.module.js';

const loader = new GLTFLoader();
loader.setMeshoptDecoder(MeshoptDecoder);
```

```js
import { DRACOLoader } from 'three/addons/loaders/DRACOLoader.js';

const draco = new DRACOLoader();
draco.setDecoderPath('/draco/');    // copy from three/examples/jsm/libs/draco/
loader.setDRACOLoader(draco);
```

Forgetting the decoder gives a silent load failure or an error about a missing extension.

## VRAM — the part people miss

**PNG and JPEG are container-only compression.** The CPU fully decodes them to raw RGBA before GPU
upload, so **VRAM cost is identical to an uncompressed bitmap** and depends only on pixel
dimensions. A 2 MB JPEG and a 40 MB PNG of the same dimensions occupy the same VRAM.

```
VRAM = width × height × 4 bytes × 1.333      (1.333 = 4/3, the full mipmap chain)
```

| Texture | Without mips | **With mips** |
|---|---|---|
| 1024 × 1024 RGBA8 | 4.0 MB | **5.33 MB** |
| 2048 × 2048 RGBA8 | 16.0 MB | **21.33 MB** |
| 4096 × 4096 RGBA8 | 64.0 MB | **85.33 MB** |

Four 4K textures on a model is ~341 MB of VRAM — enough to crash a mid-range phone on its own.

**KTX2/Basis stays block-compressed all the way into VRAM**; the GPU samples compressed blocks
natively. That's a roughly 4–8× VRAM reduction, not just a smaller download.

```bash
gltf-transform uastc model.glb out.glb --level 4   # high quality (needs KTX-Software)
gltf-transform etc1s model.glb out.glb             # smaller, lower quality
gltfpack -i model.glb -o out.glb -tc               # encoder built in
```

```js
import { KTX2Loader } from 'three/addons/loaders/KTX2Loader.js';

const ktx2 = new KTX2Loader()
  .setTranscoderPath('/basis/')
  .detectSupport(renderer);       // required — picks the right GPU format
loader.setKTX2Loader(ktx2);
```

`detectSupport(renderer)` is mandatory; without it transcoding picks the wrong target format.

**Resizing is the biggest single VRAM win.** Halving dimensions quarters memory. Most web models do
not need 4K textures:

```bash
gltf-transform resize model.glb out.glb --width 1024 --height 1024
```

## Budgets

Starting points to measure against, not hard limits — real capacity depends on device, scene
complexity, and overdraw:

| | Mobile | Desktop |
|---|---|---|
| Draw calls | ~100–200 | ~500–1000 |
| Triangles | ~100–300k | ~1–3M |
| Texture VRAM | ~100–200 MB | ~500 MB+ |
| Total download | ~5–10 MB | ~20 MB |

Measure with `renderer.info` (see `three-r3f`). A JS heap snapshot does **not** show GPU memory —
`renderer.info.memory.textures` climbing over time is your leak signal.

## A working pipeline

```bash
gltf-transform inspect raw.glb                                    # 1. what kind of heavy?
gltf-transform prune raw.glb step1.glb                            # 2. drop unused
gltf-transform dedup step1.glb step2.glb                          # 3. merge duplicates
gltf-transform resize step2.glb step3.glb --width 1024 --height 1024
gltfpack -i step3.glb -o final.glb -cc -tc                        # 4. meshopt + KTX2
gltf-transform inspect final.glb                                  # 5. verify
```

Always inspect the output. Compression can silently drop UV sets or vertex colours; check the
model still renders correctly rather than trusting the byte count.

Expect large reductions on unoptimized assets — often several-fold — but figures vary enormously
with content. A texture-heavy scan behaves nothing like a low-poly game asset, so measure yours
rather than quoting a ratio.

## Related skills

| Need | Skill |
|---|---|
| R3F/three.js scene setup and disposal | `three-r3f` |
| Scroll-driven 3D | `three-scroll` |
| Page-level performance | `web-vitals` |
| 2D texture/atlas strategy in Pixi | `pixi-performance` |
