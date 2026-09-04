# unity-vr-skills

Agent skills for building and, more importantly, **verifying** VR work in Unity
on machines with no headset and no GUI.

These were extracted from a working project that loads a Gaussian splat scan of a
real house at runtime and turns it into a playable VR level, verified end to end
on a headless Linux box. Every technique here is one that was actually needed to
get that working, not a summary of documentation.

## Skills

| Skill | Use it when |
| --- | --- |
| [`unity-headless-verification`](skills/unity-headless-verification/) | Unity must build, run and be visually tested with no GUI and nobody watching the Game view |
| [`unity-xr-simulator-testing`](skills/unity-xr-simulator-testing/) | VR stereo must be verified without a headset — Mock HMD, built-player probes, Windows-under-Wine |
| [`gaussian-splat-runtime`](skills/gaussian-splat-runtime/) | Players supply their own 3D scans and splats must load at runtime, not via the editor importer |
| [`scan-to-level`](skills/scan-to-level/) | A real-world scan needs to become playable: floor, walkable area, spawns, movement without colliders |
| [`scan-navigation-ai`](skills/scan-navigation-ai/) | Agents must path and hunt inside a scanned space without walking through walls |

## Why this exists

Most Unity VR advice assumes a developer sitting at a workstation with a headset
plugged in, looking at the result. That assumption breaks in CI, on a remote
Linux box, and on any team that wants rendering regressions caught automatically.

The recurring theme across these skills is replacing "look at it and see" with
assertions that fail loudly:

- Render known viewpoints and assert the pixels against generated ground truth.
- Assert that both eyes render, agree on colour, **and differ in detail** — the
  last one is what catches stereo silently collapsing to a flat image.
- Assert that a derived level is density-independent and that a player can never
  leave it.

They also document the traps that cost the most time and produce the most
misleading symptoms — software-GL fallback that makes every frame black, process
limits that make Unity hang at 0% CPU, linear-vs-sRGB mismatches that fail every
colour check by a gamma, and XR providers that ship no plugin for your platform
and so fail only at runtime.

## Honest limits

Simulators verify **correctness**, not **performance**. Nothing here tells you
what your frame time will be on a standalone headset, and desktop numbers should
never be presented as if it did. Where a skill cannot answer a question, it says
so.

## Using these

Copy the skill directories into wherever your agent tooling discovers skills
(commonly `~/.config/<tool>/skills/` or a `.github/skills/` directory in a
repository). Each skill is a self-contained `SKILL.md` with YAML frontmatter, plus
optional helper scripts.

## Contributing

Findings that are specific, reproducible and non-obvious are welcome — especially
corrections. If a claim here is wrong or has been fixed upstream, please open an
issue with the version you observed it in.

## License

MIT. See [LICENSE](LICENSE).

The synthetic scan generator vendored under
`skills/gaussian-splat-runtime/scripts/` is original work under the same license.
The techniques for adapting `aras-p/UnityGaussianSplatting` describe changes to
that MIT-licensed project; it is not redistributed here.
