---
name: unity-headless-verification
description: Verify Unity rendering work on a headless Linux machine or in CI by rendering to PNG and asserting against ground truth, instead of relying on a human looking at the Game view. Use when Unity must be built, run, or visually tested with no GUI, no display, and no artist in the loop, or when a Unity render regression needs an automated gate.
---

# Headless Unity verification

Rendering work is normally checked by a human looking at the screen. On a build
server, or on a box you are driving over SSH, nobody is looking. This skill
replaces that human with a **render probe**: a headless `-executeMethod` entry
point that renders known viewpoints, asserts the pixels against ground truth, and
exits non-zero when they are wrong.

## When to use this

- Unity must run with no GUI and you still need to know the picture is correct.
- A rendering feature needs a CI gate.
- You are debugging "it renders black" and need to find out *why*, not just *that*.

## The two failures that waste the most time

### 1. Software GL silently replaces your GPU

Under `xvfb-run` alone, Unity picks up **llvmpipe**, Mesa's software rasteriser.
Simple shaders work, so it looks like everything is fine, but anything using
compute shaders with wave/subgroup intrinsics fails. The signature is a compute
kernel that cannot be found at all:

```
IndexOutOfRangeException: Invalid kernelIndex (0) passed, must be non-negative less than 19
```

and every frame comes out black.

**Fix:** force a real GPU backend.

```bash
xvfb-run -a --server-args="-screen 0 1280x1024x24" \
  "$UNITY" -batchmode -force-vulkan -projectPath . \
  -executeMethod MyProbe.Run -logFile probe.log
```

Note `-batchmode` **without** `-nographics`. `-nographics` disables the graphics
device entirely, so nothing renders; batchmode alone still needs a display, which
is what Xvfb provides.

Always have the probe report what it actually got, so a silent fallback is
visible in the log rather than being inferred from a black image:

```csharp
report.AppendLine($"device={SystemInfo.graphicsDeviceName}");
report.AppendLine($"api={SystemInfo.graphicsDeviceType}");
report.AppendLine($"compute={SystemInfo.supportsComputeShaders}");
```

If `device` says `llvmpipe`, stop and fix the backend before trusting any result.

### 2. Unity exhausts the process/thread limit

Unity spawns bee, `bee_backend`, and one `dotnet` server per ILPP worker. Threads
count against `pids.max`, so a container or a systemd-scoped shell with a modest
limit will starve it. The symptom is either

```
bee_backend: error: posix_spawn failed ... Resource temporarily unavailable
```

or a **silent hang with bee sitting at 0% CPU**, which looks like a deadlock and
is not one.

Diagnose:

```bash
cat /proc/self/cgroup
cat /sys/fs/cgroup/<your-scope>/pids.max
cat /sys/fs/cgroup/<your-scope>/pids.current
```

Mitigate by capping Unity's build parallelism and cleaning up orphans between
runs:

```bash
BEE_BUILD_THREADS=2 "$UNITY" -batchmode ...
```

Identify orphans by their working directory rather than by name, so you only kill
your own:

```bash
for p in $(pgrep -f 'bee_backend|netcorerun|dotnet'); do
  [ "$(readlink -f /proc/$p/cwd 2>/dev/null)" = "$PWD" ] && echo "$p"
done
```

Other tools inherit the same limit. A Go binary such as `gh` will crash with
`newosproc ... errno=11` while a Unity build is running; that is the same
exhaustion, not a broken install.

### A wedged UnityLinker does not recover

If the budget is exhausted at the moment `UnityLinker` starts, it spawns but
deadlocks at 0% CPU, and bee waits on it indefinitely. The build sits at
`[BUSY 1518s] UnityLinker` looking like a slow link rather than a hang.

Two things make this hard to diagnose:

- **Freeing pids afterwards does not unstick it.** The linker is already wedged.
  You must kill the build and restart it once there is headroom.
- **Processes in other cgroups are irrelevant, however old they look.** A pile of
  three-week-old `MSBuild.dll` nodes from another login session is a red herring
  if they live in `user.slice`. Compare cgroups before blaming them:

```bash
cat /proc/self/cgroup
cat /proc/<suspect-pid>/cgroup
```

Attribute usage properly, by thread count, within your own cgroup:

```bash
C=/sys/fs/cgroup/<your-scope>
for p in $(cat $C/cgroup.procs); do
  echo "$(awk '/^Threads:/{print $2}' /proc/$p/status 2>/dev/null) $p $(cat /proc/$p/comm 2>/dev/null)"
done | sort -rn | head
```

Idle `UnityShaderComp` workers are usually the largest reclaimable block, at
around 12 threads each; Unity respawns them on demand.

## Writing a probe that actually catches regressions

A probe is only worth having if it fails when the picture is wrong. Weak
assertions are worse than none, because they create false confidence.

### Assert on the centre of the frame, not the whole frame

A whole-frame mean mixes every surface in view, so it drifts toward grey and
stops discriminating. Aim each viewpoint at one known surface and sample the
central ~30%.

### Compare chromaticity, then require the nearest match

Absolute brightness moves with exposure, overlap and tone mapping. Normalise by
total intensity, and require that the **nearest** entry in the ground-truth
palette is the one you aimed at. A distance threshold alone will happily accept a
wrong-but-close colour:

```csharp
static Color Normalize(Color c)
{
    float s = c.r + c.g + c.b;
    return s < 1e-5f ? new Color(0.333f, 0.333f, 0.333f)
                     : new Color(c.r / s, c.g / s, c.b / s);
}

// Fail unless the nearest palette entry is the expected one AND it is close.
if (NearestPaletteKey(centre) != view.expect || ChromaDistance(want, centre) > 0.12f)
    Fail(...);
```

### Watch out for colour space

In linear colour space, `ReadPixels` from a non-sRGB render target gives you
**linear** values while your reference colours are almost certainly authored in
sRGB. Comparing them directly makes everything fail by roughly a 2.2 gamma.
`0.42` sRGB reads back as `0.15`. Convert the references once:

```csharp
static float SrgbToLinear(float c) =>
    c <= 0.04045f ? c / 12.92f : Mathf.Pow((c + 0.055f) / 1.055f, 2.4f);
```

If every channel of every view is off by a consistent power, this is why.

### Keep a coverage guard

The chroma check can pass on a nearly empty frame. Keep a blunt "is there
anything here" assertion alongside it:

```csharp
if (coverage < 0.50f) Fail($"coverage {coverage:F3} below 0.50");
```

### Generate ground truth, do not eyeball it

Build the test scene from a script that also emits the expected palette and
dimensions. Then the probe asserts against numbers that came from the same source
as the geometry, and a drift in either is caught. Keep the generator's palette and
the probe's palette explicitly documented as ground truth for each other.

Avoid golden-image comparison as the primary gate: it breaks on every driver
update and tells you nothing about *what* changed.

### Exit code is the interface

```csharp
EditorApplication.Exit(ok ? 0 : 1);
```

Write a flat `key=value` report next to the PNGs. It diffs well between runs and
is readable in CI logs without downloading artifacts.

## Verifying non-visual behaviour in the same pass

Edit-mode `-executeMethod` cannot run `Start`/`OnEnable`, so components that
configure themselves through lifecycle callbacks will be unconfigured. Give such
components an explicit entry point (`Bind(...)`, `Configure(...)`) that both the
lifecycle callback and the probe can call. Without it, a probe silently tests an
uninitialised object and passes for the wrong reason.

This makes it practical to assert on gameplay too, for example walking a
character controller in several directions from every spawn point and requiring
that every run is stopped by geometry and none leave the walkable area.

## Checklist

- [ ] `-force-vulkan` (or another real backend), `-batchmode`, **not** `-nographics`
- [ ] Probe logs `graphicsDeviceName`; confirm it is not `llvmpipe`
- [ ] Build parallelism capped if `pids.max` is low; orphans cleaned between runs
- [ ] Reference colours converted to the render target's colour space
- [ ] Centre-crop sampling, nearest-match assertion, plus a coverage guard
- [ ] Ground truth generated by script, not eyeballed
- [ ] Non-zero exit on failure; `key=value` report written
