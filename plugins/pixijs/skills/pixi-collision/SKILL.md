---
name: pixi-collision
description: Collision detection for PixiJS v8 games — AABB, circle, point tests, spatial hashing, swept collision for fast objects, and tile-based collision. Use when implementing hit detection, physics overlap, pickups, or platformer collision on a Pixi canvas.
---

# Collision Detection in PixiJS

Pixi is a renderer, not a physics engine — it ships no collision system. Write your own for simple
cases, or integrate a physics library for real dynamics.

**Keep collision geometry separate from display objects.** Testing against `sprite.getBounds()`
each frame is slow and ties your logic to art dimensions, including transparent padding.

```js
// ✗ Recomputes world bounds, includes transparent padding
if (playerSprite.getBounds().intersects(enemySprite.getBounds())) { … }

// ✓ Explicit collision boxes you control
const player = { x: 100, y: 200, w: 24, h: 48, sprite: playerSprite };
```

## AABB

Axis-aligned bounding boxes — the workhorse. Fast, and correct for anything non-rotated.

```js
function aabb(a, b) {
  return a.x < b.x + b.w &&
         a.x + a.w > b.x &&
         a.y < b.y + b.h &&
         a.y + a.h > b.y;
}
```

With centre-origin boxes:

```js
function aabbCentered(a, b) {
  return Math.abs(a.x - b.x) * 2 < a.w + b.w &&
         Math.abs(a.y - b.y) * 2 < a.h + b.h;
}
```

## Circles

Cheaper than AABB and rotation-invariant — good for projectiles and blast radii. Compare squared
distances to avoid a square root:

```js
function circleHit(a, b) {
  const dx = a.x - b.x;
  const dy = a.y - b.y;
  const r = a.r + b.r;
  return dx * dx + dy * dy < r * r;
}
```

## Point and circle-rect

```js
function pointInRect(px, py, r) {
  return px >= r.x && px <= r.x + r.w && py >= r.y && py <= r.y + r.h;
}

function circleRect(c, r) {
  const nx = Math.max(r.x, Math.min(c.x, r.x + r.w));   // nearest point on rect
  const ny = Math.max(r.y, Math.min(c.y, r.y + r.h));
  const dx = c.x - nx;
  const dy = c.y - ny;
  return dx * dx + dy * dy < c.r * c.r;
}
```

## Resolving overlap

Detecting a hit is half the job; pushing objects apart is the other half. Resolve along the axis
of **least** penetration:

```js
function resolveAABB(moving, solid) {
  const overlapX = Math.min(moving.x + moving.w, solid.x + solid.w) - Math.max(moving.x, solid.x);
  const overlapY = Math.min(moving.y + moving.h, solid.y + solid.h) - Math.max(moving.y, solid.y);

  if (overlapX < overlapY) {
    moving.x += moving.x < solid.x ? -overlapX : overlapX;
    moving.vx = 0;
  } else {
    moving.y += moving.y < solid.y ? -overlapY : overlapY;
    if (moving.y < solid.y) moving.grounded = true;      // landed on top
    moving.vy = 0;
  }
}
```

For platformers, move and resolve **one axis at a time** — horizontal, resolve, then vertical,
resolve. Doing both at once causes catching on tile seams.

```js
body.x += body.vx * dt;
for (const tile of nearbyTiles(body)) if (aabb(body, tile)) resolveX(body, tile);

body.y += body.vy * dt;
for (const tile of nearbyTiles(body)) if (aabb(body, tile)) resolveY(body, tile);
```

## Tunnelling and swept collision

A fast object can pass through a thin wall between frames. Fixed timestep (see `pixi-game-loop`)
reduces this; for bullets, sweep the movement segment instead of testing a point:

```js
function sweptAABB(box, vx, vy, target) {
  const expanded = {
    x: target.x - box.w / 2,
    y: target.y - box.h / 2,
    w: target.w + box.w,
    h: target.h + box.h,
  };
  return raySegmentRect(box.x + box.w / 2, box.y + box.h / 2, vx, vy, expanded);
}
```

Simpler alternative: subdivide the step. If a bullet moves more than half the thinnest wall's
width per frame, split the movement into several smaller steps and test each.

## Spatial hashing

Testing every object against every other is O(n²) — 1,000 objects is ~500,000 checks per frame.
Bucket objects into a grid and only test within neighbouring cells:

```js
class SpatialHash {
  constructor(cellSize = 64) {
    this.cellSize = cellSize;
    this.cells = new Map();
  }

  key(x, y) {
    return `${Math.floor(x / this.cellSize)},${Math.floor(y / this.cellSize)}`;
  }

  clear() { this.cells.clear(); }

  insert(obj) {
    const x0 = Math.floor(obj.x / this.cellSize);
    const y0 = Math.floor(obj.y / this.cellSize);
    const x1 = Math.floor((obj.x + obj.w) / this.cellSize);
    const y1 = Math.floor((obj.y + obj.h) / this.cellSize);

    for (let x = x0; x <= x1; x++) {
      for (let y = y0; y <= y1; y++) {
        const k = `${x},${y}`;
        if (!this.cells.has(k)) this.cells.set(k, []);
        this.cells.get(k).push(obj);
      }
    }
  }

  nearby(obj) {
    const found = new Set();
    const x0 = Math.floor(obj.x / this.cellSize);
    const y0 = Math.floor(obj.y / this.cellSize);
    const x1 = Math.floor((obj.x + obj.w) / this.cellSize);
    const y1 = Math.floor((obj.y + obj.h) / this.cellSize);

    for (let x = x0; x <= x1; x++) {
      for (let y = y0; y <= y1; y++) {
        for (const other of this.cells.get(`${x},${y}`) ?? []) {
          if (other !== obj) found.add(other);
        }
      }
    }
    return found;
  }
}
```

Rebuild each frame for moving objects — clearing and reinserting a `Map` is far cheaper than the
O(n²) it replaces. Set `cellSize` to roughly the average object size; too small wastes memory,
too large degrades toward brute force.

Static geometry (tilemaps, walls) should be hashed **once**, not per frame.

## Tile collision

For grid worlds, skip collision objects entirely and index the tile array:

```js
function tileAt(px, py) {
  const tx = Math.floor(px / TILE);
  const ty = Math.floor(py / TILE);
  if (tx < 0 || ty < 0 || ty >= map.length || tx >= map[0].length) return 1;  // out of bounds is solid
  return map[ty][tx];
}

function boxHitsSolid(box) {
  const x0 = Math.floor(box.x / TILE);
  const x1 = Math.floor((box.x + box.w - 1) / TILE);
  const y0 = Math.floor(box.y / TILE);
  const y1 = Math.floor((box.y + box.h - 1) / TILE);

  for (let y = y0; y <= y1; y++)
    for (let x = x0; x <= x1; x++)
      if (map[y]?.[x]) return true;

  return false;
}
```

This is O(tiles covered), independent of world size — always prefer it for tile-based games.

## Layers and filtering

Avoid testing pairs that can never interact:

```js
const LAYER = { PLAYER: 1, ENEMY: 2, PLAYER_BULLET: 4, ENEMY_BULLET: 8, WALL: 16 };

const MASK = {
  [LAYER.PLAYER]:        LAYER.ENEMY | LAYER.ENEMY_BULLET | LAYER.WALL,
  [LAYER.PLAYER_BULLET]: LAYER.ENEMY | LAYER.WALL,
};

function canCollide(a, b) {
  return (MASK[a.layer] & b.layer) !== 0;
}
```

## When to use a physics engine

Hand-rolled collision is right for AABB platformers, top-down games, and bullet hell. Reach for a
library when you need stacking, joints, slopes, friction, or realistic bounce:

- **matter-js** — accessible 2D rigid body physics, easy to pair with Pixi
- **planck.js** — Box2D port, more accurate, steeper learning curve
- **rapier2d** — Rust/WASM, fastest, deterministic

The integration pattern is the same: physics owns position, Pixi mirrors it each frame.

```js
app.ticker.add(() => {
  Matter.Engine.update(engine, 1000 / 60);

  for (const { body, sprite } of pairs) {
    sprite.position.set(body.position.x, body.position.y);
    sprite.rotation = body.angle;
  }
});
```

Never write to `sprite.x` directly when a physics body owns it — set the body's position instead,
or the two desynchronise.
