# NotProton source audit

Audited checkout: `c3a49486c14cdf6b9716e2e7ac55286aa537fcf1` (application version
`1.0.3`). These are source findings, not evidence of a successful macOS build or
game launch. No runtime refactoring has been applied. See
[MACBOOK_HANDOFF.md](MACBOOK_HANDOFF.md) for the baseline gate and next work.

## Components and boundaries

| Component | Observed role |
| --- | --- |
| `app/` | Swift 6.2 macOS application: installs Steam injection, detects supported CrossOver installs, checks licensing, clones/patches/activates runners, manages prefixes and repairs. The package targets macOS 26.0. |
| `dylib/` | C Steam integration: loads into `steam_osx`, resolves private Steam functions using signatures and instruction probes, enables compatibility-tool functionality, patches Web UI gates and routes Windows-only titles. The main dylib is ARM64 plus an x86-64 inert stub in a universal binary. |
| `signatures/macos.arm64/` | Build-specific JSON signatures for `steamclient.dylib` and related hooks. Files present are `1788400362.json`, `1789086785.json`, and `1790904859.json`. |
| `lsteamclient/` | Local adaptations over fetched Valve Proton sources. Windows PE halves plus a Mach-O Unix half translate Steamworks calls to native `steamclient.dylib`. |
| `steam-shim/` | Port of Proton 9 `steam_helper`, built as `steam.exe`. Starts the Windows game, sets Steam registry/client paths, participates in Steam process tracking and waits for game activity. This is not a Windows Steam client installation. |
| `ntdll-patch/` | Detour sources, assembly and patch tooling. Makes Windows `steamclient` imports resolve through `lsteamclient`; patch addresses and hashes are tied to specific runner builds. |
| `overlay-shim/` | Objective-C Metal/macOS overlay compatibility shim, built for ARM64 and x86-64. |
| `bridge/` | Builds the Wine component trees, fetches/stages supporting Valve files and checks payload provenance. |
| `helpers/` | Swift appinfo reader, PE/game icon tooling, and application icon generator. |
| `vendor/` | Tracked cJSON plus externally fetched pinned Dobby. Sparkle is resolved by SwiftPM. |

## Steam integration

`SteamInstaller.swift` installs `notproton.dylib` into Steam's app bundle, backs
up `Info.plist`, sets `LSEnvironment/DYLD_INSERT_LIBRARIES`, ad-hoc signs the
dylib/bootstrapper/bundle and registers the bundle. It stages signatures, overlay
and bridge support. Installation includes checks for foreign injection and an
existing deployment.

The constructor in `dylib/core/loader.c` only activates in `steam_osx`. It exports
`STEAM_EXTRA_COMPAT_TOOLS_PATHS`, installs the spawn hook, waits for Steam images,
loads a signature database and installs resolved hooks. `hook_spawn.c` keeps
injection across Steam relaunch while removing it from other children.

`compat.c` writes the `notproton` tool under Steam's `compatibilitytools.d`, including
`compatibilitytool.vdf`, `toolmanifest.vdf` and the generated `run` script.
`dylib/embed_script.py` embeds `compat_run.sh` into the built dylib. Editing an
installed `run` script is temporary: it is rewritten by the integration.

`hook_compat.c` enables compatibility functionality and registers the local tool.
Its `FindToolForTargetApp` hook respects an existing tool and the explicit no-tool
sentinel, then supplies the NotProton default only when the platform mask contains
Windows and excludes macOS. `np_compat_would_force` is the native-game protection.
The oslist instrument permits choosing the tool's Windows source depot; the local
resolver redirect permits invoking the local tool instead of fetching a Linux
Proton depot. Web UI patches expose the corresponding client controls. Steam
continues reporting macOS: there is no required global Windows platform override.

Do not conflate a supported Steam client version with a signature filename. The
README names client versions `1788652215` and `1790121765`; the filenames above are
different identifiers. The loader selects the highest numeric installed JSON by
default, or `NOTPROTON_SIG`. Matching and completeness must be verified on the
Mac; filename presence alone is not proof that a current Steam build is safe.

## Runner setup

`RunnerSetup.run` checks the CrossOver license and asks `RunnerInstaller` to copy
the runtime. `activate` verifies the clone, stages patched ntdll files, installs
them and lsteamclient builtins, applies loader entitlement handling, then switches
`runners/current`. It attempts to restore prior bridge staging after a failure.

`SupportedRunner.swift` pins two bundle versions, each with Rosetta/FEX variants:
`27.0.0.40921` (release `20260821`) and `27.0.0.41069` (release `20261006`). The
README's older release spelling differs; use the source/hash allowlist for actual
runner identity. `RunnerBuild.id` includes the flavor, while the release date is
display metadata. Detection checks a loader hash, not just a bundle name.

`NtdllPatcher.swift` contains build-specific patch descriptors: machine type,
expected input/output hashes, payload hashes, hook RVAs and stolen bytes,
code-cave/new-section placement. `RunnerPatcher.swift` installs patched ntdll and
builtins into the clone and handles loader signing. These patches must not be
applied to an arbitrary Wine/GPTK build.

## Proposed extraction, not yet implemented

Steam hooks -> existing NotProton compatibility tool -> RuntimeManager ->
CrossOverRuntime / experimental WineRuntime / later GPTKRuntime.

The actual launch boundary is the POSIX shell tool, while installation and repair
are Swift. Choose a contract spanning those two surfaces instead of introducing a
Swift-only abstraction that the Play path bypasses. It should describe runtime
identity/version, loader/server/layout, preparation, prefix policy, environment,
bridge ABI readiness and launch behavior. Keep graphics configuration independent.

Preserve the existing CrossOver root, prefix paths, environment, license checks,
hash allowlist, patch semantics, overlay behavior and activation rollback as the
first adapter. `PrefixTools` duplicates launch environment/layout logic and must
be considered in the extraction. Separate experimental bridge staging and prefix
state before switching runtimes so experiments cannot corrupt the baseline.

The next agent must establish an unchanged Mac build and a real game baseline
before applying this extraction. Implementation, regression evidence and the
experimental game launch remain outstanding.
