---
name: unity-xr-simulator-testing
description: Test Unity VR/XR stereo rendering without a headset, using the Mock HMD XR provider or the OpenXR mock runtime in a built player, including running a Windows player under Wine when the host platform has no XR native plugins. Use when VR work must be verified in CI, on Linux, or on any machine with no VR hardware attached.
---

# Testing Unity XR without a headset

"You need a real device" is usually false. Unity ships simulated XR providers,
and most stereo regressions can be caught without hardware. What *cannot* be
simulated is performance and comfort. Be clear about which of the two you need.

## What a simulator can and cannot tell you

| Question | Simulated? |
| --- | --- |
| Does each eye render at all? | Yes |
| Is stereo actually stereo, or the same image twice? | Yes |
| Does the stereo rendering mode match what the shaders support? | Yes |
| Do eye textures get sane dimensions? | Yes |
| Does XR initialise at all in a built player? | Yes |
| Frame time on the target device | **No** |
| Comfort, judder, reprojection artefacts | **No** |

Desktop-GPU numbers say nothing about a standalone headset. Never present them as
if they do.

## Step 0: check the provider actually has native plugins for your platform

This is the step that decides everything, and it is easy to skip.

```bash
ls Library/PackageCache/com.unity.xr.mock-hmd@*/Runtime/
ls Library/PackageCache/com.unity.xr.openxr@*/Runtime/
```

Look for per-platform directories. As of Unity 6, **Mock HMD ships `android`,
`ios`, `osx` and `windows` — there is no Linux desktop plugin**. Unity's OpenXR
package is the same: `android`, `osx`, `windows`, no Linux desktop.

The consequence: **Unity XR cannot initialise on a Linux desktop player or Linux
editor at all**, regardless of what the package manager says is installed, and
regardless of whether Monado or another OpenXR runtime is present on the host.
Adding the package succeeds; the loader then fails at runtime.

If you find no plugin for your platform, do not keep fighting the editor. Jump to
"Running a Windows player under Wine".

## Configuring the Mock HMD provider headlessly

Referencing the loader asset is not enough — the provider is written into the
build via the metadata store.

```csharp
using UnityEditor.XR.Management;
using UnityEditor.XR.Management.Metadata;
using UnityEngine.XR.Management;
using Unity.XR.MockHMD;

EditorBuildSettings.TryGetConfigObject(
    XRGeneralSettings.k_SettingsKey, out XRGeneralSettingsPerBuildTarget perTarget);
// create + AddConfigObject(XRGeneralSettings.k_SettingsKey, perTarget, true) if null

var settings = perTarget.SettingsForBuildTarget(BuildTargetGroup.Standalone);
settings.InitManagerOnStart = true;

bool added = XRPackageMetadataStore.AssignLoader(
    settings.Manager, nameof(MockHMDLoader), BuildTargetGroup.Standalone);

// Provider-specific settings are a separate config object.
var mock = ScriptableObject.CreateInstance<MockHMDBuildSettings>();
mock.renderMode = MockHMDBuildSettings.RenderMode.MultiPass;
EditorBuildSettings.AddConfigObject(MockHMDBuildSettings.BuildSettingsKey, mock, true);
```

Always log and assert the result — a silently empty loader list produces a player
that runs happily in mono and passes weak tests:

```csharp
if (settings.Manager.activeLoaders.Count == 0)
    Debug.LogError("no XR loader registered; the player will not run in stereo");
```

At runtime, Mock HMD is configurable:

```csharp
MockHMD.SetEyeResolution(1024, 1024);
MockHMD.SetRenderMode(MockHMDBuildSettings.RenderMode.MultiPass);
```

Guard these calls behind a scripting define so non-XR builds still compile.

## Choose the stereo mode your shaders actually support

Single-pass instanced renders both eyes in one draw and requires shaders to be
instancing-aware. Custom renderers frequently are not — a common shortcut is to
read only `XRSettings.eyeTextureWidth`, which silently produces a wrong or
mono-looking image under single-pass.

Assert the mode rather than assuming it:

```csharp
if (XRSettings.stereoRenderingMode != XRSettings.StereoRenderingMode.MultiPass)
    Fail($"stereo mode is {XRSettings.stereoRenderingMode}, expected MultiPass");
```

## Verifying stereo in a built player

Editor `-executeMethod` runs in edit mode, where XR is not active. Real stereo
verification needs a **built player** that runs the probe itself and quits with an
exit code.

Wait for XR with a bounded loop, never an unbounded one:

```csharp
var mgr = XRGeneralSettings.Instance.Manager;
if (mgr.activeLoader == null) { yield return mgr.InitializeLoader(); mgr.StartSubsystems(); }
for (int i = 0; i < 300 && !XRSettings.enabled; ++i) yield return null;
```

Capture each eye properly — this is the API that distinguishes a real stereo test
from a camera moved by hand:

```csharp
yield return new WaitForEndOfFrame();
var left  = ScreenCapture.CaptureScreenshotAsTexture(ScreenCapture.StereoScreenCaptureMode.LeftEye);
yield return new WaitForEndOfFrame();
var right = ScreenCapture.CaptureScreenshotAsTexture(ScreenCapture.StereoScreenCaptureMode.RightEye);
```

### The assertions that matter

Three properties together catch essentially every stereo breakage:

1. **Both eyes contain the scene** — coverage above a floor. Catches one-eye-black.
2. **The eyes agree on colour** — chromaticity distance below ~0.02. They look at
   the same surfaces, so a large difference means they are not.
3. **The eyes differ in detail** — a meaningful fraction of pixels changed, with
   the mean absolute difference bounded above. This is the important one: if the
   images are *identical*, the provider is handing the same view to both eyes and
   the scene will look flat in a headset, while every naive test still passes.

```csharp
if (parallaxFraction < 0.02f) Fail("no stereo parallax; eye position is being ignored");
if (meanAbsDiff > 0.25f)      Fail("eyes are not looking at the same place");
```

Pass the scan/scene and report paths as command-line arguments and read them with
`Environment.GetCommandLineArgs()`, so one player binary covers many cases.

## Running a Windows player under Wine

When the host has no XR plugins but Windows does, build for Windows and run it
under Wine. Requirements:

```bash
wine --version
ls /usr/share/vulkan/icd.d/                                  # host GPU ICD
ls /usr/lib/x86_64-linux-gnu/wine/x86_64-windows/winevulkan.dll
```

Build with **Vulkan** rather than D3D11, so rendering goes through `winevulkan`
straight to the host GPU instead of needing a D3D translation layer:

```csharp
PlayerSettings.SetUseDefaultGraphicsAPIs(BuildTarget.StandaloneWindows64, false);
PlayerSettings.SetGraphicsAPIs(BuildTarget.StandaloneWindows64,
    new[] { GraphicsDeviceType.Vulkan });
PlayerSettings.runInBackground = true;   // otherwise an unfocused window stalls
```

Run it headlessly, remembering that Wine needs Windows-style paths (the Linux
root is mounted at `Z:`):

```bash
xvfb-run -a --server-args="-screen 0 1280x1024x24" \
  wine /path/to/Player.exe \
    -scan 'Z:\path\to\input' \
    -report 'Z:\path\to\report.txt' \
    -logFile 'Z:\path\to\player.log'
```

Have the player write its report to a path you pass in, then read the exit code.

### This works — verified numbers

A Unity 6 URP project rendering a 115k-splat Gaussian cloud, built for Windows
x64 and run under Wine 9.0 on a headless Linux box with an NVIDIA GTX 1650:

```
device=NVIDIA GeForce GTX 1650   api=Vulkan (through winevulkan)
xr.active_loader=MockHMDLoader   xr.enabled=True
xr.device=Mock HMD Display       xr.stereo_mode=MultiPass
xr.eye_width=1512                xr.eye_height=1680
cam.stereo_enabled=True
left.coverage=0.690              right.coverage=0.679
stereo.eye_chroma_delta=0.0069   stereo.parallax_pixel_fraction=0.2531
RESULT=PASS
```

The captured eyes show the expected circular lens viewport, and their difference
map shows structured, depth-dependent disparity rather than noise. No headset,
no Android SDK, no device.

## A word on Monado and other host OpenXR runtimes

Installing Monado on Linux does not help Unity. Unity talks to OpenXR through
`UnityOpenXR`, and there is no Linux desktop build of it, so there is nothing to
connect to the runtime. Monado is useful for engines with native Linux OpenXR
support; Unity is not currently one of them.

## Building for a real device from headless Linux

**First: check whether you actually cannot build, rather than assuming.** Unity's
release metadata lists the Android (and iOS) target for the *Linux* editor as a
macOS `.pkg`, which looks like Linux is unsupported. It is not. A `.pkg` is an
xar archive, and Unity Hub only extracts its payload into
`Editor/Data/PlaybackEngines/AndroidPlayer`. Every sub-component is a real
linux-x64 build — the NDK toolchain is literally `prebuilt/linux-x86_64`.

So a headless Linux machine with no Hub can still install the module and build a
Quest APK. Drive it from Unity's own metadata:

```
https://services.api.unity.com/unity/editor/release/v1/releases?version=<VERSION>
```

Walk `downloads[platform=LINUX].modules[id=android]` and its nested
`subModules`, honouring each node's `destination` and `extractedPathRename`.
Extraction is `7z` (xar) → `Payload` → `gzip` → `cpio`; note that some 7z
versions already decompress the gzip layer and leave `Payload~`, so handle both.
This also yields `adb`, at `.../AndroidPlayer/SDK/platform-tools/adb`.

Two traps in the renames: the destination is often the *parent* of the source
(extract into `NDK/`, then collapse `NDK/android-ndk-r27c` up into `NDK`), so
clearing the destination first deletes what you are about to move — stage
through a temp directory. And verify afterwards that the binaries are executable
and actually run (`adb version`, `java -version`).

Still split configuration from building (`ConfigureQuest` / `BuildQuestApk`), so
configuration can be applied and reviewed on a machine that genuinely lacks the
module. Four traps:

- **Unity only materialises XR settings for *installed* build targets.**
  `OpenXRSettings.GetSettingsForBuildTargetGroup(BuildTargetGroup.Android)`
  returns `null`, and `FeatureHelpers.GetFeatureWithIdForBuildTarget` finds
  nothing, when Android Build Support is missing. Detect this with
  `BuildPipeline.IsBuildTargetSupported` and **warn loudly**, rather than letting
  the run look successful while silently configuring nothing.
- **Switch the active build target before configuring**, for the same reason:
  ```csharp
  if (EditorUserBuildSettings.activeBuildTarget != BuildTarget.Android)
      EditorUserBuildSettings.SwitchActiveBuildTarget(NamedBuildTarget.Android, BuildTarget.Android);
  ```
- **`PlayerSettings.colorSpace` is global, not per-platform.** Setting it for a
  headset target changes desktop rendering too — which will move the colours your
  render probe reads back. Expect to fix the probe in the same change.
- **OpenXR defaults to `SinglePassInstanced`** (`OpenXRSettings.RenderMode`).
  Force `MultiPass` if your shaders lack an instancing-aware path, and re-check
  it after any settings regeneration.

### Unity misreports build failures as compiler errors

`Aborting batchmode due to failure: Scripts have compiler errors` is Unity's
catch-all. It is frequently not a code error at all. Grep the log for the real
cause before touching any code:

- `Out of memory` — Unity leaves Roslyn `VBCSCompiler` servers running between
  builds, and they can hold several GB each. Kill the idle ones.
- `pthread_create failed (EAGAIN)` / `posix_spawn failed` / `Resource
  temporarily unavailable` — process-limit exhaustion, not memory. Gradle
  defaults to a resident daemon and one worker per core; on a constrained
  cgroup that alone breaks the build. Force it serial via
  `Assets/Plugins/Android/gradleTemplate.properties`:
  ```properties
  org.gradle.parallel=false
  org.gradle.daemon=false
  org.gradle.workers.max=2
  ```
  and lower `BEE_BUILD_THREADS`. Compare `/proc/<pid>/cgroup` against
  `/proc/self/cgroup` before blaming a process for your pid budget — processes
  in another cgroup never counted against it.
- `The name 'Permission' does not exist in the current context` — the built-in
  **Android JNI** module is disabled. Add `com.unity.modules.androidjni` to
  `Packages/manifest.json`.

Have your build script detect these and print the remedy; otherwise you will
debug imaginary compiler errors.

### Verify the artifact, not just the exit code

A "Succeeded" result is weak evidence. Inspect the APK:

```bash
aapt2 dump badging app.apk      # package, minSdk, native-code, uses-feature
aapt2 dump xmltree app.apk --file AndroidManifest.xml
apksigner verify --verbose app.apk
unzip -l app.apk | grep '\.so$'
```

For a Quest build expect `arm64-v8a`, `libopenxr_loader.so`,
`com.oculus.intent.category.VR`, `android.hardware.vr.headtracking`, and
`com.oculus.supportedDevices` containing your target (Quest 3 is `eureka`).
Also confirm your compute kernels actually shipped, e.g.
`strings assets/bin/Data/sharedassets0.assets.split* | grep <KernelName>`.

Be honest about what was never run. A structurally verified APK that has never
been on a headset is *built*, not *proven*: frame rate and runtime behaviour are
still unknown.

## Checklist

- [ ] Confirmed the provider ships native plugins for the target platform
- [ ] Loader registered via `XRPackageMetadataStore.AssignLoader`, and asserted non-empty
- [ ] Stereo mode explicitly set **and** asserted at runtime
- [ ] Verification runs in a **built player**, not edit mode
- [ ] Eyes captured with `StereoScreenCaptureMode`, not a hand-moved camera
- [ ] Asserted: both eyes lit, eyes agree on colour, eyes differ in detail
- [ ] Performance claims labelled as desktop-simulated, never as device numbers
