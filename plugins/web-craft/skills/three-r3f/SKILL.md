---
name: three-r3f
description: React Three Fiber and three.js for the web — Canvas setup, the useFrame rule, drei helpers, loading models, lighting, and disposal. Use when building 3D web scenes with React Three Fiber, three.js, or @react-three/drei.
---

# React Three Fiber

R3F is a React renderer for three.js. JSX elements map to three.js constructors — `<mesh>` becomes
`new THREE.Mesh()` — so anything in the three namespace is available without a wrapper.

## Versions

| Package | Version | Peers |
|---|---|---|
| `@react-three/fiber` | **9.7.0** | `react >=19 <19.3`, `three >=0.156` |
| `@react-three/drei` | **10.7.8** | `@react-three/fiber ^9.0.0`, `react ^19`, `three >=0.159` |
| `three` | **0.186.0** | — |

R3F's major tracks React's major: **fiber 8 → React 18, fiber 9 → React 19.** The current
generation is **fiber 9 + drei 10 + React 19**.

**three has no semver-major** — every `0.x.0` may contain breaking changes. Pin it, and read the
migration notes when bumping. This is why libraries declare `three >= …` rather than a caret range.

```bash
npm i three @react-three/fiber @react-three/drei
```

## Canvas

```jsx
import { Canvas } from '@react-three/fiber';

export function Scene() {
  return (
    <Canvas
      camera={{ position: [0, 0, 5], fov: 50 }}
      dpr={[1, 2]}                              // cap DPR — 4x pixels on retina otherwise
      gl={{ antialias: true, alpha: false }}
      shadows
    >
      <ambientLight intensity={0.5} />
      <directionalLight position={[5, 5, 5]} intensity={1} castShadow />
      <mesh>
        <boxGeometry args={[1, 1, 1]} />
        <meshStandardMaterial color="hotpink" />
      </mesh>
    </Canvas>
  );
}
```

- **`args`** maps to constructor arguments: `<boxGeometry args={[1, 1, 1]} />` is
  `new THREE.BoxGeometry(1, 1, 1)`. Changing `args` **recreates** the object.
- **Dash props** set nested properties: `position-x={2}`, `material-color="red"`,
  `rotation-y={Math.PI}`.
- **`dpr={[1, 2]}`** is the single biggest mobile win — full retina means 4× the pixels to shade.

`<Canvas>` creates its own WebGL context. See the context-limit warning below.

## The useFrame rule

```jsx
import { useFrame } from '@react-three/fiber';
import { useRef } from 'react';

function Spinner() {
  const ref = useRef();

  useFrame((state, delta) => {
    ref.current.rotation.y += delta * 0.5;    // delta is SECONDS
  });

  return <mesh ref={ref}>…</mesh>;
}
```

The official docs are explicit: **"You should never setState in there!"**

```jsx
// ✗ 60 React re-renders per second — destroys the frame rate
useFrame(() => setRotation((r) => r + 0.01));

// ✓ Mutate the ref directly; React never re-renders
useFrame((state, delta) => { ref.current.rotation.y += delta * 0.5; });
```

Always scale by `delta` (seconds since last frame) so motion is frame-rate independent. Keep the
callback slim — it runs 60+ times a second.

Ordering: `useFrame(cb, priority)` runs callbacks in **ascending priority**. Passing any non-zero
priority takes over the render loop, and you must then render manually
(`state.gl.render(state.scene, state.camera)`).

## drei

The helper library — reach for it before writing three.js by hand.

| Area | Helpers |
|---|---|
| Loading | `useGLTF`, `useTexture`, `useFBX`, `useVideoTexture`, `useFont` |
| Controls | `OrbitControls`, `CameraControls`, `PresentationControls`, `ScrollControls`, `KeyboardControls` |
| Staging | `Environment`, `Stage`, `ContactShadows`, `AccumulativeShadows`, `Lightformer`, `Sky` |
| Performance | `Instances`/`Instance`, `Merged`, `Detailed` (LOD), `AdaptiveDpr`, `PerformanceMonitor`, `BakeShadows` |
| Abstractions | `Text`, `Html`, `Billboard`, `Float`, `MeshTransmissionMaterial`, `shaderMaterial` |

`<Environment />` with an HDRI is the fastest visual-quality win available — image-based lighting
does more for realism than any number of point lights:

```jsx
import { Environment, OrbitControls } from '@react-three/drei';

<Environment preset="city" />
<OrbitControls enableDamping makeDefault />
```

## Loading models

```jsx
import { useGLTF } from '@react-three/drei';
import { Suspense } from 'react';

function Model() {
  const { scene } = useGLTF('/model.glb');
  return <primitive object={scene} />;
}

export function App() {
  return (
    <Canvas>
      <Suspense fallback={null}>
        <Model />
      </Suspense>
    </Canvas>
  );
}
```

`useGLTF` suspends, so a `<Suspense>` boundary is required. Preload to start fetching before render:

```jsx
useGLTF.preload('/model.glb');
```

For compressed models, point at the decoders — see `three-assets` for producing them:

```jsx
useGLTF('/model.glb', '/draco/');   // Draco decoder path
```

`gltfjsx` generates typed components from a GLB, which is far more maintainable than
`<primitive>` for anything you need to manipulate:

```bash
npx gltfjsx model.glb --types --transform
```

## Instancing

Thousands of copies of one mesh in one draw call:

```jsx
import { Instances, Instance } from '@react-three/drei';

<Instances limit={1000}>
  <boxGeometry />
  <meshStandardMaterial />
  {positions.map((p, i) => <Instance key={i} position={p} />)}
</Instances>
```

Without instancing, 1,000 meshes is 1,000 draw calls. With it, one.

## Measuring

```js
renderer.info.render.calls        // draw calls — the number that matters most
renderer.info.render.triangles
renderer.info.memory.textures     // rising over time = a leak
renderer.info.memory.geometries
```

Adaptive quality under load:

```jsx
import { PerformanceMonitor, AdaptiveDpr } from '@react-three/drei';

<PerformanceMonitor onDecline={() => setQuality('low')}>
  <AdaptiveDpr pixelated />
</PerformanceMonitor>
```

Rough guidance, to be measured rather than trusted: aim for **under ~100–200 draw calls** on
mobile, and keep triangle counts modest. These are starting budgets, not thresholds — a scene's
real limit depends on shader cost, overdraw, and texture bandwidth.

## Disposal

R3F disposes objects it created when a component unmounts. It does **not** dispose things you
created imperatively or assets in the loader cache.

```jsx
useEffect(() => {
  const geometry = new THREE.BoxGeometry();
  const material = new THREE.MeshStandardMaterial();
  return () => {
    geometry.dispose();
    material.dispose();
  };
}, []);

useGLTF.clear('/model.glb');   // drop a cached model
```

Textures and geometries hold **GPU memory that a JS heap snapshot will not show**. Watch
`renderer.info.memory` instead.

## WebGL context limits

**Chromium allows 8 concurrent WebGL contexts on Android and 16 elsewhere**, per renderer process.
Past the cap the least-recently-used context is destroyed — the source of "context lost" errors.

Each `<Canvas>` is one context. A gallery page with 20 independent canvases will break. Use **one**
canvas with multiple viewports instead:

```jsx
import { View } from '@react-three/drei';

<Canvas eventSource={root}>
  <View track={ref1}> … </View>
  <View track={ref2}> … </View>
</Canvas>
```

## Pixi or three?

- **PixiJS** — 2D: sprites, particles, filters, data viz, 2D games. Batches tens of thousands of
  sprites efficiently.
- **three.js** — 3D: cameras, lights, materials, glTF, post-processing.

They cannot share a WebGL context, but can coexist as separate canvases on one page — mind the
context budget above. For 2D work, Pixi is lighter and faster; don't reach for three.js to draw
rectangles.

## Related skills

| Need | Skill |
|---|---|
| Model and texture optimization | `three-assets` |
| Scroll-driven 3D | `three-scroll` |
| 2D canvas rendering | `pixi-v8-core` |
| Making 3D accessible | `a11y-canvas` |
