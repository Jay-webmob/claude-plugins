---
description: Optimize GLB/glTF models, textures, and images — inspect, compress, and verify the results.
---

Optimize this project's assets. Argument: `$ARGUMENTS` (optional — a path, file, or asset type to
scope to).

Read the `three-assets` skill before running anything. Use its exact command forms — the two CLIs
take arguments differently and copy-pasted commands are frequently wrong.

## Steps

1. **Inventory.** Find assets and their real sizes:

   ```bash
   find . -path ./node_modules -prune -o \( -name "*.glb" -o -name "*.gltf" -o -name "*.png" -o -name "*.jpg" -o -name "*.jpeg" \) -print0 2>/dev/null | xargs -0 du -h 2>/dev/null | sort -rh | head -30
   ```

   Report the largest offenders with actual sizes. Optimize what matters, not everything.

2. **Inspect before compressing** — this decides the strategy:

   ```bash
   npx --yes @gltf-transform/cli inspect model.glb
   ```

   Is it geometry-heavy, texture-heavy, or draw-call-heavy? Optimizing geometry on a texture-bound
   model wastes effort.

3. **Check texture dimensions specifically.** VRAM is `width × height × 4 × 1.333` and is
   **independent of file size** — PNG/JPEG decode to raw RGBA before GPU upload. A 4096² texture is
   **85.33 MB** of VRAM no matter how small the PNG is. Flag anything above 2048² for a web target.

4. **Confirm before modifying anything.** Show the user the plan and the expected effect. Never
   overwrite source assets — write to new files or a separate output directory, so the originals
   survive a bad compression.

5. **Run the pipeline** (adjust to what step 2 found):

   ```bash
   gltf-transform prune  raw.glb step1.glb
   gltf-transform dedup  step1.glb step2.glb
   gltf-transform resize step2.glb step3.glb --width 1024 --height 1024
   gltfpack -i step3.glb -o final.glb -cc -tc          # NOTE: -i/-o flags, not positional
   ```

   - **Prefer Meshopt over Draco** — ~7.5 KB gzipped decoder vs ~74 KB, much faster decode, and
     Draco cannot compress morph targets or animation at all.
   - `gltf-transform etc1s`/`uastc` need **KTX-Software installed separately**; `gltfpack -tc` has
     the encoder built in.

   For plain web images, use AVIF/WebP with correct dimensions instead — see `responsive-images`.

6. **Verify — do not trust the byte count.**

   ```bash
   gltf-transform inspect final.glb
   ```

   Compression can silently drop UV sets, vertex colours, or morph targets. Confirm the model still
   renders correctly. If a dev server is available, load it and screenshot via `verify-browser`.

7. **Check the loader wiring.** A compressed GLB fails silently without its decoder:
   - Meshopt → `loader.setMeshoptDecoder(MeshoptDecoder)`
   - Draco → `loader.setDRACOLoader(draco)` with the decoder files copied to a served path
   - KTX2 → `loader.setKTX2Loader(ktx2)` **and** `.detectSupport(renderer)`

   Flag any missing wiring — this is the most common post-optimization breakage.

8. **Report** the before/after for each asset, the VRAM change for textures (not just file size),
   and anything you deliberately left alone and why.

Report real measured numbers. Do not quote generic compression ratios — they vary enormously by
content, and the actual figures are right there in the inspect output.
