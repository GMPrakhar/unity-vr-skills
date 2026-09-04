---
name: scan-navigation-ai
description: Make agents navigate and hunt inside a scanned real-world space - clearance-eroded grids, connected regions, A*, line of sight separated from movement, and headless tests that prove agents cannot walk through walls. Use when adding NPCs, enemies or any pathfinding to a level derived from a user-supplied 3D scan.
---

# Navigating a scanned space

A walkable grid is not a navigation mesh, and getting from one to the other is
where agents start standing inside walls. This skill covers the grid work, the
AI, and — most of the value here — the tests, because every bug listed below
looked correct while reading the code and only showed up under assertion.

Assumes you already have a 2.5D occupancy grid derived from the scan (see
`scan-to-level`).

## Walkable is not navigable

The single most important distinction. *Walkable* means the floor is there.
*Navigable* means a body of a given radius fits.

Compute **clearance** — the Chebyshev distance from each cell to the nearest
blocked cell — with a multi-source BFS seeded from every blocked cell. Then:

```
navigable(c)  ==  walkable(c) && clearance(c) >= ceil(agentRadius / cellSize)
```

Treat the grid border as blocked, or agents path off the edge of the capture.

Conflating the two means agents clip walls at every corner, and you will chase
the symptom in the steering code for a long time before finding it here.

## Connected regions before pathfinding

Flood fill the navigable cells into components once, at build time. This gives
you three things cheaply:

- **A\* early-out.** Different components can never be connected, so reject in
  O(1) instead of exploring the entire reachable set before failing. Failed
  searches are the expensive ones.
- **Spawn validity.** Never spawn an agent in a region it cannot leave. A hunter
  sealed in another room is not a challenge, it is a decoration.
- **A diagnostic.** If a real scan yields far more regions than the room has
  rooms, the capture has holes or the agent radius is too large. Report the
  count; it is the fastest signal that a scan is unusable.

## Line of sight is not movement

Use a **separate** height threshold for sight. A cell blocks movement if the
floor is occupied; it blocks sight only if its geometry reaches above roughly
1.2 m. An 0.85 m sofa should stop a body and not a gaze; a 2.6 m wall stops
both. One combined "blocked" grid gives you enemies blinded by furniture, which
reads as broken AI.

Record a second per-cell count during analysis — geometry above `floorY + 1.2 m`
— rather than trying to recover this later.

## A\* details that matter indoors

- 8-connected with an octile heuristic.
- **Disallow corner cutting**: a diagonal step is legal only if both orthogonal
  neighbours are clear. Otherwise agents slice through wall corners.
- Add a small **clearance penalty** to the cost so routes drift to the middle of
  a corridor instead of scraping the walls. Without it every path hugs geometry
  and every steering error becomes a collision.
- String-pull the result, but validate each shortcut (below).

## The wall-clipping bugs

Three separate bugs, all producing the same symptom, all easy to write:

**1. Waypoint tolerance.** "Close enough — advance to the next waypoint" quietly
teleports the agent onto a segment that was never checked. Follow the polyline
exactly and only advance on arrival within an epsilon.

**2. The unvalidated first segment.** `path[0]` is the centre of the cell the
agent occupies, not where the agent is standing. Skipping straight to `path[1]`
is a shortcut across untested ground. Only take it if that specific segment is
clear; otherwise walk to your own cell centre first.

**3. Point-sampled traversal.** Checking a segment by sampling it every *n*
metres misses a cell the line only clips at a corner, however fine the spacing.
Use exact voxel traversal — **Amanatides & Woo DDA** — for both corridor checks
and line of sight. This is the one that survives all the obvious fixes.

Assert on this directly: simulate the agent and **count steps that end on a
non-navigable cell**. It should be zero. Anything above zero is a real defect,
not a rounding artefact.

## The player is not on the grid

In VR the player stands wherever their body is: pressed against a wall, or
inside the half-metre the agent radius carves around a table. Their cell is
frequently not navigable, and code that assumes otherwise fails in ways that
look like unrelated bugs:

- **Region comparisons return -1**, so spawn filters match nothing and the round
  never starts. Snap the player to the nearest navigable cell first, then fall
  back to the largest region.
- **Pathfinding to the player fails outright**, so agents give up and wander.
  Path to the nearest standable cell instead.
- **The player becomes invincible.** If the closest reachable cell is 1.07 m away
  and the catch radius is 0.70 m, an agent can stand next to them in plain sight
  forever. Decide the rule deliberately: an agent that has reached the nearest
  ground it can stand on, and can see the player, should count as a catch.

Snap once, in one place, and document it. This assumption leaks into every
system that consumes a position.

## Test with a negative control

The essential technique. Generate **two scans that differ in exactly one
property** — with and without doorways — from the same seed at the same density,
and verify the sealed file is byte-identical to the pre-doorway original so the
doorways really are the only variable.

Then assert both directions:

| | With doorways | Sealed |
| --- | --- | --- |
| Connected regions | 1 | 3 |
| Route between rooms | exists, crosses the wall only at the door | none |
| Agent reaches target | yes | never |

A pathfinder that ignored geometry would pass the first column. Only the second
column catches it. A test suite without a negative control mostly proves your
code runs.

Extend the same idea to behaviour: a stationary player should be caught, and a
fleeing one should survive measurably longer — *and assert the fleeing player
actually covered distance*, or a movement bug turns the comparison into two
identical runs that agree and pass.

## Make the AI testable, then test it headlessly

Write the decision-making as a plain class with **no engine object dependencies**
and a **seeded RNG**. No transforms, no coroutines, no delta time from the
engine. Then a full chase runs deterministically in a batch-mode process with no
GPU, in seconds, and its behaviour can be asserted rather than watched.

For the engine-side controller, keep `Update()` as a one-line forwarder:

```csharp
void Update() { Tick(Time.deltaTime); }
public void Tick(float dt) { /* everything */ }
```

Batch mode has no meaningful frame time, so without this the game loop cannot be
driven at all — and a round loop that has only ever compiled is not a round loop
that works. Stepping `Tick(1f/72f)` also makes results reproducible.

## Render the grid

Write the navigable grid to a PNG: navigable, blocked, and outside-the-scan in
three colours, with the computed path and the agent's actual trail drawn over
it. It reads as a floor plan and makes an entire class of bug obvious at a
glance — a doorway eroded shut, a room that fell out of the main region, a trail
that cuts a corner.

Map cell Z to image Y without flipping if your texture origin is bottom-left, or
north ends up at the bottom and every map is confusing to read.

## Grid resolution is a real constraint

At 0.25 m cells a 1 m doorway is only about three free cells wide, because the
partial cells at each edge pick up wall geometry. Any furniture within roughly
half a metre of a door erodes it shut and splits the house in two.

Either use finer cells at doorways, detect and protect narrow passages, or state
the limitation plainly. Do not discover it from a bug report — it will happen on
real scans, and it presents as "the AI won't chase me", which is a long way from
the cause.
