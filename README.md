# wintty-bin

Unofficial nightly Windows builds of [deblasis/wintty](https://github.com/deblasis/wintty), published as prereleases of this repository.

This repository is not a fork. It contains no source code, no patches, and nothing to keep in sync with upstream — only the workflow that builds them. Every night it reads the tip of upstream's `windows` branch, builds that commit, and attaches the result to a release tagged with it. Nothing is ever pushed anywhere, so a run cannot leave a branch in a state nobody built, and there is no merge to resolve.

## Downloading

Take the newest release from the [releases page](releases). Each release carries the platform zips that built successfully, with a `.sha256` next to each:

| Asset | Architecture |
| --- | --- |
| `wintty-nightly-win-x64.zip` | Windows on x86-64 |
| `wintty-nightly-win-arm64.zip` | Windows on Arm64 |

Unzip anywhere and run `Wintty.exe`. The build is portable and needs no installer. Verify the download with `Get-FileHash .\wintty-nightly-win-x64.zip -Algorithm SHA256` against the `.sha256` file.

> **These are not Wintty releases.** They are unsigned, unreviewed builds of whatever happened to be at the tip of the upstream branch that night, so any given nightly can be broken. The official builds live at [deblasis/wintty](https://github.com/deblasis/wintty). This project is not affiliated with upstream.

## What the builds are

- **ReleaseFast**, with the DirectX 12 renderer and its HLSL shaders compiled in.
- **NativeAOT** .NET, so startup does not depend on a JIT warm-up.
- **Portable**, including the bundled `conpty.dll` (Kitty graphics and Sixel), `dxcompiler.dll`, and the `bin\wintty.exe` console shim.

One thing worth knowing before you file a bug against a nightly: the CPU level is pinned deliberately, and it is lower than your machine's.

- **x64 builds target `x86-64-v3`** — AVX, AVX2, BMI1/2, F16C, FMA, LZCNT, MOVBE, and no AVX-512. That covers every x86-64 CPU since Haswell (2013) and Excavator (2015).
- **arm64 builds target zig's `baseline`** (`-Dcpu=baseline`) — ARMv8.0-A plus NEON, the lowest level zig has, and the level upstream's own packaging paths use. ARMv8.1-A would add LSE atomics and hardware CRC-32 and is worth having, but raising it is a separate experiment: it is not what used to break the arm64 build, which was the toolchain bug above. The CPU level was never at fault.

Without that pin, zig compiles for whatever CPU the CI machine happens to have — which for the x64 runner means AVX-512, and an `0xC000001D STATUS_ILLEGAL_INSTRUCTION` crash on launch for most consumer Intel parts since 12th-gen Core.

So the floor for these builds is a 2013-or-newer x86-64 CPU, and on Arm it is ARMv8.0-A — the lowest arm64 level there is, so the Arm builds impose no floor of their own. What rules Arm hardware out is Windows rather than the build: Windows 11 24H2 requires ARMv8.1 to boot at all, so ARMv8.0 parts (Snapdragon 835-class) only ever ran Windows 10 on Arm or Windows 11 23H2 and older. On x86-64 the floor rules out pre-2013 hardware. On either, build from source.

## How it works

`.github/workflows/nightly.yml` runs daily at 00:00 UTC, and can be started by hand from the Actions tab. Four jobs:

1. **plan** — resolves the upstream commit, reads the required zig version out of upstream's `build.zig.zon`, and decides per platform whether that commit still needs a build.
2. **build-x64** — clones upstream at the exact commit the plan job resolved, builds libghostty with zig and the WinUI shell with .NET, publishes the NativeAOT binary, zips it, and uploads it as a build artifact. Runs on `windows-latest`.
3. **build-arm64** — the same, cross-compiled on the same x64 runner with `-Dtarget=aarch64-windows` rather than built on a native ARM64 runner. Native ARM64 compilation is impossible with the pinned toolchain: zig 0.16.0's `zig.exe` for `aarch64-windows` is itself miscompiled by the LLVM TLS bug [ziglang/zig#31865](https://codeberg.org/ziglang/zig/issues/31865), so it crashes on any compilation on an ARM64 host. The fix is on zig master but not in a release before 0.18.0.
4. **release** — waits for both, then publishes one prerelease containing whichever zips arrived.

One platform succeeding is enough. A broken arm64 leg does not cost the x64 build its release, and the platform that failed is rebuilt on the next run and added to the same release rather than a second one. When a release is missing a platform, its notes say so and name the build job's outcome.

Two manual dispatch inputs are available: `force_build` rebuilds even if the commit already has a nightly, and `skip_release` builds without publishing.

## Credits

Wintty is [deblasis/wintty](https://github.com/deblasis/wintty), itself built on [Ghostty](https://github.com/ghostty-org/ghostty). All credit for the software belongs to those projects; this repository only compiles it on a schedule.
