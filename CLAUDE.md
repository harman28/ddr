# Vanshu's Vroom Vroom World — CLAUDE.md

## What this is

A Three.js open-world driving game built for Harman's 2.5-year-old nephew Vanshu. The whole game is a single file: `index.html`. It's deployed to GitHub Pages at `https://harman28.github.io/ddr/` from the `main` branch.

The spirit of the game is pure toddler joy — bright colours, things that bounce and explode, animals to find, a sandwich boat on a lake. It is not meant to be polished or technically impressive. Keep it playful and whimsical.

---

## Tech stack

- Three.js 0.160.0 (CDN, no build step)
- Single HTML file — all JS, CSS, HTML inline
- Procedural chunk-based world: `CHUNK_SIZE=100`, `CHUNK_RADIUS=3`
- Deployed via `git push` to `main` → GitHub Pages auto-publishes

---

## Ground height — the hardest lesson

**`GND_Y = 0.5`** — all objects and cars are placed at this y height. Do not lower it.

Terrain zones use a y-stagger of `0.002` per zone (so zone 0 is at y=0.01, zone 13 is at y=0.038) to prevent z-fighting between overlapping terrain patches. This stagger alone is sufficient.

**Do NOT add `polygonOffset` back to terrain zone materials.** We tried it. `polygonOffsetFactor: -(i+1)` with 14 zones gives factor=-14 on the last zone, which pulls the terrain visually toward the camera by a significant fraction of a world unit — enough to visually exceed GND_Y=0.12 (the old value) and make objects appear to sink into the ground. We fought this bug for a long time. The fix was:
1. Remove all `polygonOffset` from terrain zone `MeshLambertMaterial`
2. Raise `GND_Y` to `0.5`

At game scale, 0.5 units of lift is completely invisible.

---

## World structure

### Terrain zones
Defined in `TERRAIN_ZONES[]` — large coloured plane patches at various world positions. Overlapping is intentional (river cuts through grass, lake sits inside a zone, etc.). Z-fighting is handled purely by y-stagger.

### Chunks
The world spawns/despawns objects in `CHUNK_SIZE=100` chunks around the player. `spawnChunk(cx, cz)` creates objects; `despawnChunk(key)` removes them. `OBJ_TYPES` is the weighted random pool for object types.

### `worldObj()` factory
Every world object goes through `worldObj(group, wx, wz, type, opts)`. It sets `group.position.y = GND_Y` and registers the object in `allWorldObjects[]` for per-frame updates. Always use this — never place a group in the scene manually.

---

## Coordinate system (important — got this wrong multiple times)

Three.js right-handed system, camera looking toward **+Z**:
- **+X appears LEFT on screen**
- **-X appears RIGHT on screen**
- Heading=0 moves the player in the +Z direction

The lake is at world position `(-370, -270)`. From the spawn point (0,0), the lake is to the **bottom-right (↘)**. The arrow on all lake direction signs is `↘ LAKE`, not `↙`.

The first lake sign (at world pos 0, 18) uses `faceHeading = Math.PI` so it faces toward the player at spawn — not away. Every other sign uses `faceHeading = 0.94`.

---

## River and boats

River centreline points:
```
[[-900,-400],[-550,-150],[-220,60],[90,-90],[380,130],[680,350],[950,560]]
```

`snapToRiver(x, z)` finds the nearest river segment and returns `{cx, cz, dx, dz, ...}`. Boats are placed via this function — they drift linearly along their segment (bouncing at the ends of a `driftRange`).

**Boats must never appear in `OBJ_TYPES`.** They were in the general spawn pool once and ended up all over the land. Now they only spawn in `spawnChunk()` when the chunk is near the river (via a dedicated proximity check). The sandwich boat is a one-off permanent object placed at the lake (`lakeX=-370, lakeZ=-270`) that orbits it.

---

## Labels

`makeLabel(text)` creates a `THREE.Sprite` with a semi-transparent dark background. Labels use `depthTest:false` so they're always visible above geometry.

Labels exist on: COW, BIRD, DUCK, RABBIT, CLINIC, FUN PARK, PARK, CONSTRUCTION, SCHOOL, ROCKET, BOAT, SANDWICH BOAT.

**No labels on: trees, bushes, houses.** These are background fill — labelling them made the world cluttered.

---

## Billboard / sign text visibility

Canvas textures for signs and billboards: the text plane must be positioned **in front of** the backing box in local z-space. The backing box is centred at z=0 (its front face is at z = depth/2). The text plane must be at a z value greater than depth/2.

Example: backing box depth=0.3 → front face at z=0.15 → text plane at z=0.2 (or 0.15+ epsilon). If the plane is inside the box, the text is invisible. This bit us more than once.

---

## Car game mechanics

- **All cars** explode on bump (not just coloured ones — that was an earlier version).
- `carsPopped` is a **cumulative** counter (total bumps ever, not unique cars).
- `aiCars.length` grows as new cars spawn over time.
- Win condition: `carsPopped >= aiCars.length` — triggers the success screen.
- We tried Set-based unique-car tracking (each car can only be counted once). The game became too hard because the player had to hunt down every last car. Reverted to cumulative.
- The success screen waits for player input (ArrowUp, Space, Escape, tap) before resetting. No auto-countdown — Vanshu needs time to celebrate.
- `successPending = true` gates all keydown events while the screen is showing.

---

## Mobile

The title screen (`z-index:10`) was blocking all mobile touches and preventing the game from starting. Fix: added a `touchstart` listener directly on `#title-screen` that dismisses it. Mobile controls (`#mobile-controls`) use `z-index:5`.

---

## Coordinates HUD

Small, dim monospace display in the bottom-left (`#coords`). Updated every frame with `Math.round(player.wx)` and `Math.round(player.wz)`. Exists purely as a navigation aid — useful when exploring the world to find things like the lake.

---

## Deployment

```
git add index.html
git commit -m "..."
git push origin main
```

GitHub Pages serves from `main` root. No build step. Changes are live within ~30 seconds.
