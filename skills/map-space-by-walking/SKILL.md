---
name: map-space-by-walking
description: Let the player map their own room or house from inside a VR app by walking it, turning headset poses into a playable level - instead of requiring an external scanning app and a sideloaded file. Covers the mapper, the walls-vs-furniture problem, agent sizing, and how to grade the result headlessly. Use when an MR/VR experience needs the player's real space and you do not want to depend on Polycam/Scaniverse exports or the Scene API.
---

# Mapping a real space by walking it

If your app needs the player's real room, the tempting design is "have them
scan it in another app and copy the file in". It works, and almost nobody will
do it. The alternative is to map the space inside your app: the player walks
around, and where they walked becomes the level.

The idea rests on one observation that is worth stating plainly because
everything else follows from it:

> **Floor a person has physically stood on is the strongest possible evidence
> that floor is walkable** — stronger than anything inferred from a point cloud,
> a depth map or a plane detector. It cannot produce a false positive.

Your map should therefore claim *nothing else*. Precision is free and should be
100%; spend your effort on the parts that are genuinely uncertain.

## What to record

Sample the headset pose, not the controllers. Each accepted pose stamps a disc
of walked floor into a sparse grid.

```csharp
// Absolute cell keys in a dictionary, baked to a dense array only at the end.
// The player has not told you how big their house is and should not have to.
static long Key(int x, int z) => ((long)x << 32) ^ (uint)z;
```

Four details that each fix a real failure:

**Reject poses outside a plausible head height** (roughly 0.9–2.3 m above the
floor). A headset taken off and carried to another room at hip height will
otherwise paint a walkable trail straight through your walls. This is not
hypothetical — it is the single most likely thing a real user does during
mapping, and it is trivial to defend against.

**Interpolate between samples.** At a walking pace and a realistic sample rate,
consecutive poses are far enough apart to leave holes down the middle of the
trail. Step along the segment at half a cell.

**Stamp a round footprint by true distance, not an integer cell radius.**

```csharp
int r = Mathf.CeilToInt(bodyRadius / cellSize);
for (int dz = -r; dz <= r; ++dz)
for (int dx = -r; dx <= r; ++dx)
{
    float px = (x + 0.5f) * cellSize - head.x;
    float pz = (z + 0.5f) * cellSize - head.z;
    if (px * px + pz * pz > radiusSq) continue;   // not: if (dx*dx+dz*dz > r*r)
    ...
}
```

Rounding the radius to whole cells first makes the footprint a plus sign at
small radii, and a trail stamped with plus signs is notched along both edges —
enough to break it into disconnected pieces once an agent radius is eroded off
it. Switching to true distance took one test from nine connected regions to one.

**A square footprint over-claims corners**, which is exactly where walls are.
Use the disc.

## Agent radius is part of the level, not a global constant

This is the mistake that costs the most time, so it gets its own section.

A walked map only knows about the strip of floor the player's body swept —
about 0.70 m wide for a 0.35 m body radius. A person-sized agent needs 0.60 m.
After a navigation grid erodes clearance for that agent, almost nothing is left,
and the symptoms do not look like a sizing problem:

- the mapped house fragments into many disconnected regions;
- spawn-point selection returns a handful of points instead of the number asked
  for, because few cells have the clearance it wants;
- with too few points, enemies spawn a metre from the player and the round is
  over instantly.

Three unrelated-looking bugs, one cause. The invariant to hold onto:

> An agent no wider than the person who walked the map fits by construction.
> Anything wider might not.

So expose the radius through whatever interface describes a level:

```csharp
public interface ILevelSource
{
    LevelAnalysis analysis { get; }
    IReadOnlyList<Vector3> spawnPoints { get; }

    /// How wide an agent this level's geometry can actually support. A level is
    /// not just a shape - it comes with a statement about how much of it is
    /// trustworthy. Zero means "use the default".
    float agentRadius { get; }
}
```

A photogrammetry scan measures whole rooms and can return 0. A walked map
returns something comfortably under its body radius (0.20 m against 0.35 m
worked well). Pass the same value as the spawn-clearance requirement.

Make the interface the *only* thing gameplay depends on and a walked room and a
scanned house become interchangeable for free.

## The hard part: what the gaps mean

Everything unwalked is impassable. But not all of it should block *sight*, and
footsteps cannot tell you which is which:

- a sofa you walked around, and
- a partition wall with a corridor walked down either side

produce identical evidence. There is no clever fix; there is only choosing which
way to be wrong. The workable heuristic is a flood fill from the grid border:

- unwalked space **connected to the outside** is the edge of the world — blocks
  movement *and* sight;
- unwalked pockets **enclosed by walked floor** are things walked around —
  block movement only.

Bias errors toward *over*-blocking sight. Treating furniture as opaque costs the
player a sightline. Treating a wall as transparent lets enemies see through the
side of the house, which reads as broken.

### One idea that sounds right and is not

The obvious refinement: morphologically close the walked set first, so a room
the player circled rather than crossed fills in and becomes visible-across.

Measure it before believing it. In one build, sweeping the closing radius:

| Closing reach | Real walls still blocking sight | Furniture correctly seen over |
| --- | --- | --- |
| 0.25 m | 99.5% | 49.1% |
| 0.75 m | 94.0% | 49.1% |
| 1.75 m | 89.1% | 49.1% |

A straight loss. The furniture number never moves, because a trail already
encloses whatever it encloses; all the closing does is swallow thin interior
walls. Keep the sweep in your test suite so the decision stays re-checkable
instead of becoming folklore.

## Grading it without a headset

You can test this properly on a build machine, and you should, because the
alternative is discovering geometry bugs while wearing a headset.

Drive a simulated player around a **synthetic space you have ground truth for**,
convert the route to poses at a realistic walking speed and sample rate, feed
them to the mapper, and grade the output against the truth.

Assertions worth having:

| Check | Why |
| --- | --- |
| Precision of claimed floor | Must be ~100%. A false positive is floor invented through a wall. |
| Recall against the swept band only | Catches holes from dropped frames. Do **not** score against the whole space — that measures how thorough your test route was, not the mapper. |
| Connected regions | You cannot walk through a wall, so a map built by walking must not contain one. |
| Walls that still block sight | The unsafe direction of the heuristic above. |
| A real round played on the map | Enemies spawn far away, path, and arrive. |

Two negative controls that catch real bugs:

- **Headset carried at hip height** across the space adds *zero* floor.
- **Mapping part of the route** yields a proportionally smaller level with no
  cell claimed far from any pose. If a partial walk produces a whole house, the
  map is guessing.

### Two ways these tests go quietly vacuous

**Measuring against the wrong denominator.** An early version scored "furniture
correctly seen over" at 8%, which looked catastrophic. The denominator included
every cell of empty space *outside* the building, since those are also
"not walkable and not sight-blocking". Furniture has to be something actually
there — require obstacle evidence at the cell. The number was really 49%.

**Reusing the camera-tour route.** If you already have a route builder for
recording video, it optimises for a short, pretty path. Someone mapping their
house walks into corners and back out. Using the tour route made mapping look
far worse than it was, because the walk was never trying to cover anything.
Build coverage routes separately — farthest-point sampling over navigable cells,
pathfound between stops.

## Feedback in the headset

The floor lighting up behind the player is the entire UI. No progress bar to
read, no menu to aim at: the floor you covered glows, the floor you have not
does not, and it is obvious what to do next.

Draw it as **one mesh rebuilt a few times a second**, not one object per cell. A
mapped house is thousands of cells, and thousands of draw calls will cost more
frame time than your actual content. Throttle rebuilds and skip them when the
cell count has not changed.

## Wiring and lifecycle traps

- **`Start()` order between components is undefined.** If one component loads a
  saved map and another decides what to prompt the player with, the prompt can
  win the race and talk the user into re-mapping a house that was already
  mapped. Give the loader an idempotent `EnsureLoaded()` and call it from both.
- **Never attach gameplay components that move to the object holding your
  scene geometry.** A director that uses its own transform as the player
  position, attached to the object parenting a point cloud, drags the entire
  world around as the player moves. This is invisible at small displacements and
  obvious at large ones.
- Persist the map (JSON of parallel `x[]/z[]/visits[]` arrays is plenty) and
  reload it on launch. Mapping is a thing the player does once.

## Honest limitations

State these rather than hiding them:

- The map is a corridor network of where someone walked, not a floor plan. It
  will not include the far side of a bed.
- Roughly half of furniture ends up treated as the edge of the map, because
  furniture passed on one side is never enclosed.
- Platform scene understanding (Meta's Scene API and equivalents) already knows
  where the walls are. Walk-mapping is testable on a build machine and Scene API
  is not, which is a good reason to build it first — but the two combine well:
  the platform for walls, footsteps for floor.
