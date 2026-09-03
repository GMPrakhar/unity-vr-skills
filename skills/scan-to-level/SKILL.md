---
name: scan-to-level
description: Turn a real-world 3D scan (Gaussian splat cloud, photogrammetry mesh or point cloud) into a playable level by deriving floor height, walkable area, obstacles and spawn points from the geometry, and constraining player movement without colliders. Use when building VR or games around user-captured spaces such as room-scale or whole-house scans.
---

# Turning a scan into a playable level

A scan is scenery, not a level. It has no floor, no navmesh, no colliders and no
notion of where a player may stand. This skill covers deriving that automatically,
so a space the developer has never seen becomes playable.

The core requirement: **the same room must produce the same level regardless of
capture quality**. If a better scan yields a different level, the analysis is
keying on noise.

## Why not just generate colliders

Splat clouds have no surfaces to collide with, and photogrammetry meshes are
usually a single non-manifold shell with holes where the scanner could not see.
Mesh colliders over that produce a player who falls through the floor or sticks
to invisible fragments. A coarse 2.5D occupancy grid derived from point
statistics is far more robust and is cheap to compute.

## Finding the floor and ceiling

Build a histogram of point heights across the whole scan and take the peaks.
Floors and ceilings are the two largest flat, densely-sampled surfaces in almost
any indoor capture, so they dominate.

- Use a bin size around 2–5 cm. Too fine and noise splits the peak; too coarse and
  a low sofa merges into the floor.
- Take the **lowest** major peak as the floor, not the global maximum: ceilings
  are often more densely sampled than furniture-covered floors.
- Reject a "ceiling" that is implausibly close to the floor.
- Multi-storey scans have several floor peaks. Either segment per storey or
  document the single-storey assumption. Do not silently take the first peak.

Report the derived heights. A wrong floor height is the single most common cause
of "the player is buried in the ground" and is instantly obvious in a report.

## Occupancy grid

Project points onto the horizontal plane into cells of roughly 20–30 cm — about a
foot, which is the resolution that matters for standing.

For each cell, classify by what sits above the floor:

- **Floor points near floor height** and **no substantial points in the body
  band** (roughly 0.2–1.8 m above the floor) → walkable.
- **Points in the body band** → obstacle: furniture, walls, clutter.
- **Too few points at all** → unknown, and should be treated as *not* walkable.
  Unobserved space is where the scanner never went, which is exactly where you
  should not send the player.

Use point *counts* with a threshold rather than "any point at all". Splat clouds
and photogrammetry both contain floaters, and a single stray point must not
delete a walkable cell.

## Spawn points

Pick walkable cells spread out by a minimum separation, using a seeded shuffle so
a given scan always produces the same level.

**Require clearance, not merely walkability.** A cell can be walkable while
sitting 10 cm from a wall; spawning there puts the player's head inside the wall,
which in VR is both disorienting and looks like a rendering bug. Require every
cell within a clearance radius to be walkable too:

```csharp
int margin = Mathf.CeilToInt(clearance / cellSize);   // clearance > player radius
// reject the cell unless all cells within `margin` are walkable
```

Make the spawn clearance **larger** than the movement clearance, or the player
spawns in a spot the movement code immediately considers illegal.

Always keep a fallback: a tightly furnished or noisy room can yield zero cells
with full clearance, and returning no spawn points at all is a worse failure than
a slightly tight one.

## Constraining movement without colliders

Test the destination against the grid and commit only if it is legal. To avoid
sticking on corners, retry each axis separately so the player slides along
obstacles:

```csharp
if (IsWalkable(target))            { Commit(target); return; }
if (IsWalkable(new(target.x, p.y, p.z))) { Commit(...); return; }   // slide X
if (IsWalkable(new(p.x, p.y, target.z))) { Commit(...); return; }   // slide Z
// otherwise: blocked, do not move
```

Check a small clearance radius around the destination, not just the point itself.
In VR the camera is a head, and letting it touch a wall reveals the scan as a
smear of noise from the inside — one of the fastest ways to break presence.

## VR-specific concerns

- The rig transform is the **floor-level tracking origin**; the headset drives the
  camera relative to it. Do not also drive the camera yourself when a headset is
  present, or you will fight the tracking and cause sickness.
- Players physically walk. Room-scale movement can carry the headset into a wall
  without any input, so validate the *headset* position, not just locomotion
  input.
- Keep a desktop fallback (keyboard/mouse) so the scene stays testable without
  hardware. Detect a connected HMD rather than relying on a build flag:

```csharp
var devices = new List<InputDevice>();
InputDevices.GetDevicesWithCharacteristics(InputDeviceCharacteristics.HeadMounted, devices);
bool hasHeadset = devices.Count > 0;
```

## Verifying the analysis

Two properties are worth asserting automatically, and both catch real bugs:

**1. Density independence.** Analyse the same room captured at very different
point densities. Floor height, walkable area and spawn count should be
effectively identical. Divergence means the analysis is keying on sample density
rather than geometry.

**2. Containment.** Walk the rig from every spawn point in several directions,
further than the largest room, and require:

- every run is eventually stopped by geometry,
- no step lands outside the walkable area,
- no step escapes the scan bounds,
- every spawn point is itself a legal standing position.

That last assertion is the one that catches spawn clearance bugs, and it will
catch them — spawning inside walls is easy to do and invisible until someone puts
the headset on.

## Checklist

- [ ] Floor/ceiling from height histogram peaks; lowest major peak is the floor
- [ ] Multi-storey either handled or explicitly documented as unsupported
- [ ] Occupancy grid at ~20–30 cm, using point-count thresholds
- [ ] Unobserved cells treated as not walkable
- [ ] Spawn points require clearance, with a fallback when none qualify
- [ ] Spawn clearance greater than movement clearance
- [ ] Movement checked against the grid with per-axis sliding
- [ ] Headset position validated, not just locomotion input
- [ ] Density independence and containment asserted automatically
