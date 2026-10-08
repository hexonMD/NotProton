# CrossOver dependencies and extraction checklist

Source audit at upstream `c3a49486c14cdf6b9716e2e7ac55286aa537fcf1`. No files below
have been refactored. CrossOver remains the existing compatibility baseline.

## Runtime-specific assumptions

| Surface | CrossOver-specific dependency |
| --- | --- |
| `SupportPaths.swift` | Installed bundle payload at `Contents/SharedSupport/CrossOver`; clones under `runners/crossover-<build>/CrossOver`; `runners/current` is a symlink. |
| `CrossOverSource.swift` | Searches `/Applications`, `~/Applications`, and a saved manual bundle; expects `lib/wine`; detects support through loader hashes. |
| `SupportedRunner.swift` | Known bundle versions/flavors, loader hash, clean and patched ntdll hashes; display release date and Rosetta/FEX labels. |
| `CrossOverLicense.swift`, `RunnerSetup.swift` | License inspection and validation before clone/activation. Keep these in the CrossOver implementation. |
| `RunnerInstaller.swift` | APFS clone (`cp -c -R`) with copy fallback, CrossOver-shaped staging, marker scrubbing, clone input validation, current symlink selection. |
| `NtdllPatcher.swift`, `ntdll-patch/` | Exact Wine PE layouts, hook instructions/RVAs, payloads and hashes for pinned CrossOver releases/flavors. Generic Wine cannot inherit those offsets. |
| `RunnerPatcher.swift` | Installs patched ntdll and builtin bridge modules into `lib/wine/<arch>`; detects Unix loader layout; preserves signing rules, including cross-architecture entitlements. |
| `RunnerStore.swift` | Recognizes `crossover-` directory identity; verifies patches and reports clone/active-build state. |
| `PrefixTools.swift` | Duplicates shell loader/server selection, CrossOver environment and per-prefix msync settings for maintenance tools. |
| `PrefixRebuild.swift`, `PrefixPrompt.swift`, prefix UI | Repair and architecture-switching assumptions based on Rosetta/FEX CrossOver; prefix rebuilding must preserve saves. |
| `SystemStatus.swift`, `AppLog.swift`, app views | CrossOver discovery, licenses, runner status and user-visible build terminology. |
| `compat.c`, `compat.h`, `hook_compat.c` | Compatibility tool display name, API names and messages mention CrossOver. Hook semantics should stay intact while their runtime label is eventually generalized. |
| `webpatch.c`, compatibility-service UI | CrossOver options panel sets graphics/sync/CPU-related environment launch options. |
| `compat_run.sh` | Current root/layout/environment, prefix machine check, crossover-to-steamuser alias, runner patch verification, generated Game Mode loader bundle and CX environment forwarding. |
| `lsteamclient/build.sh` | Installs against the cloned runner when `--install` is used; refuses certain installed CrossOver destinations; Unix half must match loader architecture/ntdll ABI. |
| `bridge/setup-wine-tree.sh` | Uses pinned Wine 11.15 sources for bridge builds, described upstream as corresponding to the CrossOver runtime; builds separate x86-64 and ARM64 trees. This builds bridge prerequisites, not a verified standalone runtime. |
| `ntdll-patch/build-ntdll.sh` | Default CrossOver location, flavor recognition and clean/patched hashes; stages patched copies derived from the user's runtime. |
| `Makefile` | `CX_ROOT` for ntdll resolution; bridge build paths, both host architectures, app payload staging. |

## Executables, layouts and environment

The shell sets `CX_ROOT=~/Library/Application Support/notproton/runners/current`
and `CX_HOME=~/Library/Application Support/CrossOver` (cxcompatdb lookup).

It first tests **both** of:

- `lib/wine/aarch64-unix/wine.app/Contents/MacOS/wine`
- `bin/wineserver-arm64`

If either is unavailable it selects `lib/wine/x86_64-unix/wine` plus
`bin/wineserver`, falling back to `bin/wineserver-x86`. Thus the current code
prefers a present ARM64 layout; it does not independently choose Rosetta over an
installed FEX clone. Rosetta preference is upstream's recommendation and a
product requirement, not an already-implemented runtime-selection algorithm.

It exports `WINELOADER`, `WINESERVER`, `WINEDLLPATH` (x86-64 Windows builtin path
plus selected Unix path), prepends runtime `bin` to `PATH` and sets the prefix
from Steam compatibility data. `WINEMSYNC` is read from launch options or a
per-AppID `notproton-msync` marker, defaulting to **0**. Preserve that baseline;
msync should not become a new default before testing.

Launch options accept families `CX_GRAPHICS*`, `D3DM_*`, `DXMT_*`, `DXVK_*`,
`MTL_*`, `NOTPROTON_*`, `ROSETTA_*` and `WINE*`. The generated app receives explicit
CX/Wine/Steam variables and selected graphics/CPU families. The profile layout
aliases `drive_c/users/crossover` to `steamuser` for cloud path compatibility.

The prefix ntdll PE machine check distinguishes `8664` (Rosetta) from `aa64`
(FEX). A mismatched existing prefix is refused and directed to repair; do not
silently overwrite it. The shared bridge contains staged ntdll from the active
build, so runtime-specific bridge/prefix isolation is necessary before experiments.

## Graphics configuration

`webpatch.c` exposes `CX_GRAPHICS_BACKEND` with runtime default, D3DMetal, DXMT,
DXVK and WineD3D choices. Related controls include `D3DM_ENABLE_METALFX`,
`DXMT_ENABLE_NVEXT`, `DXMT_METALFX_SPATIAL_SWAPCHAIN`, `DXMT_CONFIG`,
`ROSETTA_ADVERTISE_AVX`, `WINEMSYNC` and Retina mode. These reach the launch script
as assignments; the runtime interprets them. There is no independent graphics
backend interface or automatic per-game API selection in this fork yet.

## Safe extraction sequence

1. Verify untouched upstream on the Mac and record one real Windows-only title.
2. Add a runtime contract at the actual launch boundary and in Swift preparation.
3. Wrap existing clone/license/hash/patch/environment behavior without altering it.
4. Test both loader layouts, spaces in paths, failed preparation, prefix mismatch,
   helper calls, Steam stop/exit, bridge staging and native-game selection.
5. Add developer-only experimental runtime discovery. A Wine executable alone is
   not launch readiness; bridge ABI and Steam process tracking must work too.
6. Keep experimental prefix/bridge state from modifying the known-good path.

For additional locations during refactoring, reproduce the inventory with `rg`
over `app/Sources`, `dylib`, `bridge`, `lsteamclient`, `steam-shim`, `ntdll-patch`,
`helpers` and `Makefile` for `CrossOver|CX_|Rosetta|FEX|WINEMSYNC|D3DM|DXMT|DXVK`.
