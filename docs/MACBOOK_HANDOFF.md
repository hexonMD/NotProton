# MacBook Codex takeover: Mac Steam Play / NotProton fork

## Assignment and product contract

Continue this project on the user's Apple Silicon MacBook. The user requested a
working fork and implementation, then explicitly moved the work from Windows to
the Mac so the unchanged upstream build and real Steam behavior can be verified.
Do not stop at a plan. Perform the baseline, then make and test logical changes.

The product is **Proton for the native macOS Steam client**:

1. Install our app once.
2. Open normal native macOS Steam.
3. Windows-only games become installable; Steam downloads Windows depots.
4. Press Play; the compatibility layer selects and launches the right runtime.

Users should not create bottles, choose prefixes, install Windows Steam, copy
DLLs, enter launch arguments, choose translators/graphics backends, run another
Steam client or use Terminal. Developer commands below are for engineering only.
The app eventually serves as installer, runtime manager, updater, diagnostics and
profile manager; Steam remains the library UI. Preserve native macOS games.

## Repository and verified session state

- Fork created successfully: <https://github.com/hexonMD/NotProton>.
- Upstream: <https://github.com/NotProtonNot/NotProton>.
- Branch created: `dev/runtime-abstraction`.
- Upstream starting commit: `c3a49486c14cdf6b9716e2e7ac55286aa537fcf1`.
- Upstream app version in source: `1.0.3`.
- Windows checkout:
  `C:\Users\Mike Doyle\Documents\Codex\2026-10-07\project-mac-steam-play-notproton-fork`.
- Local remotes: `origin` is hexonMD's fork; `upstream` is NotProtonNot's repository.
- The handoff branch contains source-audit documentation. **Production code and
  launch behavior have not been changed.** No RuntimeManager or WineRuntime has
  been implemented. No developer tool scripts have been added yet.
- No binaries, game data, proprietary runtimes, SDK downloads, credentials or
  tokens were added. Existing upstream licenses and notices are preserved.

Checks actually performed on unchanged upstream:

| Check | Actual result |
| --- | --- |
| `python dylib/tests/sigcheck.py` | Passed: three databases, 17 signatures each, all referenced. This is structural validation, not validation against a running Steam client. |
| `sh -n dylib/feats/compat_run.sh` using Git Bash | Passed shell syntax. ShellCheck was not run. |
| `make dobby all` using Git Bash | Failed immediately: `make: command not found`. No compilation occurred. |
| macOS/C/Swift build and platform tests | Not run. Windows lacks macOS SDK/frameworks, Swift/Xcode/signing and the required Mac environment. Dobby was not fetched. |
| CrossOver baseline / experimental Steam launch | Not run. No Mac/Steam/game test was available in the Windows session. |

Do not describe the baseline as known good on this machine, the abstraction as
finished or this branch as free of tested runtime regressions. The Mac baseline
is the next gate. There was no outstanding background build when the handoff was
requested. Earlier Git/GitHub sandbox access needed approved escalation; that is
Windows-session permission history, not a Mac build problem.

## Get the project onto the Mac

Use a fresh local checkout of the fork's handoff branch if it is not already on
the Mac. These are developer setup commands:

```sh
git clone --branch dev/runtime-abstraction https://github.com/hexonMD/NotProton.git NotProton
cd NotProton
git remote add upstream https://github.com/NotProtonNot/NotProton.git
git fetch upstream
git status --short --branch
git remote -v
git log --oneline -5
```

For an existing checkout, inspect local changes/remotes first and reuse it; do
not reset user work. Check for `AGENTS.md` and current session instructions.
Read this handoff and the four documents beside it. The docs-only branch still
contains unchanged upstream executable sources, so it can supply the baseline.
Verify that with `git diff c3a49486c14cdf6b9716e2e7ac55286aa537fcf1 -- app dylib
lsteamclient steam-shim ntdll-patch overlay-shim bridge helpers signatures vendor
Makefile`. Do not claim a baseline if those sources changed unexpectedly.

## First actions: unchanged build and real game baseline

1. Inspect the laptop's macOS version, chip/model and architecture, selected
   Xcode/SDK/Swift versions, Rosetta, native Steam install/build and available
   supported CrossOver Preview. Do not assume the Mac meets the app's deployment
   target or that a current Steam build matches the signatures.
2. Read `Makefile`, `app/Package.swift`, `vendor/README.md`, the bridge/component
   scripts and `.github/workflows/app.yml`. Document missing prerequisites before
   modifying code. Make required dependencies available under the user's current
   authorization and host permission rules.
3. Build upstream unchanged, record all commands/results and resolve ordinary
   setup problems before changing architecture. If blocked, record the exact
   diagnostic and prerequisite, and work on independent tasks.
4. Use the existing NotProton setup to activate a licensed supported **Rosetta**
   CrossOver clone, configure Steam and launch a small owned Windows-only game
   through ordinary native Steam. Choose from the user's actual library; no title
   or AppID was selected in the Windows session.
5. Also launch an existing native macOS game. Confirm normal native selection.
6. Record the baseline below. Then implement the runtime abstraction.

Source-established build dependencies:

- The Swift package requires Swift tools 6.2 and targets macOS **26.0**.
- C dylib/helpers use `clang`, Apple SDK/frameworks and a 15.0 minimum deployment
  setting. That does not lower the whole application's 26.0 target.
- Full app icon generation uses `xcrun actool` and the `.icon` pipeline. Upstream
  CI names Xcode 27 and the runner label `xcode-27`; do not assume this label is
  usable on a public fork or that that exact Xcode is installed on the laptop.
- GNU/BSD developer `make`, Git, CMake, Python 3, bison, flex and MinGW-w64 cross
  compilers are needed. Script tooling also uses curl, rsync and standard macOS
  utilities. Node and ShellCheck support relevant checks.
- Dobby is not vendored: pin it to
  `5dfc8546954ce3b3198132ab13fddb89ee92cdd7` as `vendor/README.md` specifies.
- SwiftPM resolves Sparkle exactly at 2.10.0.
- `make bridge` builds x86-64 and ARM64 host component trees against pinned stock
  Wine 11.15 commit `2df1ee28039cf84776eb1421ed90bd154cebb65f`. It fetches Valve
  component/header sources via the existing scripts. Keep their license terms.
- Baseline gameplay requires supported user-installed CrossOver Preview, its
  licensing, Rosetta and a native Steam client compatible with the hooks.

Typical build sequence, to verify against the actual laptop/toolchain:

```sh
# Inspect first; install only missing dependencies.
xcodebuild -version
xcrun --sdk macosx --show-sdk-version
swift --version
sw_vers
uname -m
sysctl -n hw.model
sysctl -n hw.optional.arm64
arch -x86_64 /usr/bin/true

# If Homebrew is installed, these are the relevant build packages.
brew install cmake mingw-w64 bison flex

git clone https://github.com/jmpews/Dobby.git vendor/dobby
git -C vendor/dobby checkout 5dfc8546954ce3b3198132ab13fddb89ee92cdd7
make dobby
make all
make overlay-shim iconmaker appinfo
make bridge
make app-payload
make app

make sigcheck compatcheck compatsvc-check callscheck
make webpatch-fixtures peicon-fixtures panel-behavior spawn-env
make app-tests
```

Reuse an existing correct Dobby checkout instead of cloning over it. ShellCheck
0.11.0 is upstream's lint pin. `make runcheck scriptcheck` can report skips when
ShellCheck is missing; skips are not passed checks. Some other targets also skip
missing inputs. Confirm what actually ran. Full build output is
`out/NotProton.app`; use the app's setup flow for the first install, rather than
inventing a new Steam injection procedure. Avoid publishing a release during
bring-up. The upstream updater/release configuration also needs fork-specific
review before distribution.

Baseline record must include timestamp, Steam client/build identification, macOS,
Mac model, chip generation, CrossOver version/flavor, NotProton SHA, translator,
AppID, graphics backend, prefix and launch result. Record Steamworks, Steam
running-state recognition, overlay, controller/Steam Input and audio separately
as passed/failed/not tested. Also verify Windows depot installation, clean exit,
prefix persistence and relaunch. Do not fill untested fields with success.

## Source findings to preserve

Read [ARCHITECTURE.md](ARCHITECTURE.md), [LAUNCH_FLOW.md](LAUNCH_FLOW.md),
[CROSSOVER_DEPENDENCIES.md](CROSSOVER_DEPENDENCIES.md) and
[LICENSING.md](LICENSING.md). They contain source observations and open questions,
not runtime test evidence.

- `SteamInstaller` backs up/modifies Steam's bundle plist for dyld injection and
  signs/registers it. `loader.c` acts only inside `steam_osx`; the spawn hook
  carries injection across Steam relaunch and strips it from other children.
- `compat.c` embeds `compat_run.sh`, writes `compatibilitytools.d/notproton`, and
  registers a Windows-to-macOS compatibility tool. Keep the tool identity's
  `proton` substring: upstream relies on it for Steam Cloud path mapping.
- `hook_compat.c` enables existing Steam compatibility features, applies local
  tool resolution and the Windows source-depot gate. `np_compat_would_force`
  requires a Windows platform bit and no macOS bit. Preserve explicit mappings,
  the no-tool sentinel and native-game behavior. Do not globally force Windows.
- README client versions and signature filenames differ. Installed JSON files
  are `1788400362`, `1789086785`, `1790904859`; loader defaults to the highest
  numeric database or `NOTPROTON_SIG`. Check actual resolutions on the laptop.
- Supported runtime identities are `27.0.0.40921` / release `20260821` and
  `27.0.0.41069` / release `20261006`, each with Rosetta/FEX flavor. Exact hashes
  and patch sites live in `SupportedRunner.swift` and `NtdllPatcher.swift`.
- AppID comes through Steam environment; prefix is the Steam compatibility data
  directory's `pfx`. `getcompatpath` returns before initialization. Other calls
  initialize/maintain the prefix and stage the bridge. msync defaults to **0**.
- Foreground games use a generated macOS game-category app bundle, `open -n -W`,
  Wine and the `steam.exe` shim. Steam helpers/redistributables use the raw Wine
  loader. Keep both launch paths and cleanup behavior.
- Patched ntdll routes Windows Steam client functionality through lsteamclient;
  the matching Unix half opens native `steamclient.dylib`. DLL overrides alone do
  not replace ntdll integration. Matching loader/bridge/server ABI is essential.
- Overlay uses both dyld insertion and manual loading for hardened FEX. The
  ARM64 Unix half is signed; upstream keeps the x86-64 builtin unsigned.
- `RunnerSetup` validates licensing, verifies/clones/stages/patches, then points
  `runners/current` at the clone, with restoration of prior staging on failure.
- Launch environment/layout logic is duplicated in Swift `PrefixTools` and the
  shell tool. A runtime abstraction must cover the actual Play path and repair
  tools; a Swift protocol alone that Steam never calls is insufficient.

## First implementation milestone after the baseline

Build a runtime contract appropriate to the existing C/Swift/shell structure:
identity/version, availability/readiness, preparation, loader/server/layout,
prefix creation/update, environment construction, bridge requirements and launch.
Do not rewrite the working Steam hooks.

Wrap the existing implementation as CrossOverRuntime. Preserve paths, license
and hash checks, patch addresses, prefix/cloud behavior, msync default, graphics
environment, overlay, Game Mode, helper execution and teardown. Retain its fallback
as experiments develop. Rosetta is the initial production translator; FEX stays
experimental. Existing code selects a present ARM64 layout first, so introducing
a Rosetta-first policy is a behavior change that requires explicit testing.

Add a developer-only experimental WineRuntime skeleton and detection. Keep
runtime installation/preparation state separate from graphics backend selection.
Do not mark arbitrary `wine --version` detection as Steam readiness. A usable
experimental runtime needs matching builtin bridge modules and a valid ntdll
integration strategy. Isolate experimental bridge/prefix state from the baseline;
do not apply CrossOver's hard-coded patches to foreign Wine builds or destructively
switch an existing prefix's architecture.

Improve structured game-launch logging under `~/Library/Logs/notproton/` (or a
consistent chosen project name). Record timestamp, AppID, actual Steam build,
runtime/version, translator, requested/effective graphics state where observable,
prefix, safe overrides, stages and exit status. Do not infer an effective graphics
backend merely from an unset variable, nor a Windows game exit code from `open`.
Cover early errors and termination. No credentials/tokens/full environment dumps.
Existing raw argument/Steam-environment logging must be reviewed too.

Add reproducible engineering commands:

```text
./tools/doctor
./tools/build
./tools/install-dev
./tools/uninstall-dev
./tools/test-runtime
./tools/launch-appid <appid>
./tools/collect-diagnostics <appid>
```

doctor reports macOS/chip/architecture, Rosetta, Steam installation/build/hook
compatibility, runtimes/versions/readiness, graphics availability and problems.
These commands do not exist yet; implement them instead of referencing them as
completed tooling. Keep installation/uninstallation aligned with upstream's
backup/repair logic and preserve user data.

Commit logical changes separately, for example runtime contract, CrossOver
extraction, Wine detection, then diagnostics/tooling. Verify the old path after
each launch-affecting change. Never label a mocked launch as a real Steam test.

Milestone acceptance: normal native Steam; Windows-only title installable; Windows
depot downloaded; compatibility prefix used; game launched without Windows
Steam; lsteamclient talks to native Steam; running state recognized; clean exit;
prefix persists; **the same AppID works with both the CrossOver and experimental
runners**. A skeleton/detection does not satisfy the milestone. Proceed into a
real alternative-runtime launch after the extraction, as the user requested.

## Alternative runtime research

Investigate independently:

1. CodeWeavers' published CrossOver Wine sources and macOS Wine CX build tooling.
   Build open-source components; do not copy proprietary CrossOver binaries.
2. User-installed/imported Apple GPTK and D3DMetal. Review the exact acquired
   version's terms before redistribution; do not assume permission from the
   product webpage or an older license.
3. Current macOS Wine projects solving Rosetta WoW64, msync, Metal integration,
   GPTK layouts and packaging. Inspect licenses before reuse.

Research started but **is not complete**. DXMT's primary repository
<https://github.com/3Shain/dxmt> confirms D3D11/10-to-Metal work and has multiple
license files; those scopes have not been audited. Apple's primary page is
<https://developer.apple.com/games/game-porting-toolkit/>. The CodeWeavers source
listing <https://media.codeweavers.com/pub/crossover/source/> returned HTTP 403
to the browser check. An attempted `Gcenx/winecx` URL returned 404; do not reuse
that guessed URL as a verified source. Discover the actual maintained build
repositories, pin commits and document ABI/build choices. Upstream already has
bridge assembly machinery; inspect/reuse it before inventing another.

Do not change project licensing. Retain `LICENSE`, `LICENSE.lsteamclient`,
`LICENSE.steam-shim`, `NOTICE`; audit Valve payload and SDK permissions before
binary distribution. Keep uncertain proprietary components user-supplied.

## Follow-on architecture and acceptance scope

- Independent graphics interface: D3DMetal, DXMT, DXVK/MoltenVK, WineD3D. Evaluate
  D3D12-to-D3DMetal where eligible, D3D11 DXMT vs D3DMetal per game, and older API
  fallbacks. No new D3D12 implementation and no unsupported winner/default claims.
- Local JSON profiles: AppID/status, runtime/version, translator, graphics, sync,
  Windows version, DLL/environment overrides, launch options, known issues,
  minimum OS/chip/GPU and Steam build compatibility. Design for future community
  downloads; do not claim profiles verified without measurements.
- Automatic per-AppID selection with known profiles first, then defensible
  detection/defaults and outcome recording. Rosetta first; FEX behind a flag.
- Installer preflight: Steam, Apple Silicon, Rosetta, runtime, graphics and Steam
  integration. Offer supported Rosetta installation; guide legal user acquisition
  of restricted GPTK components. Normal UX hides technical runtime options.
- Test categories: small Win32/DX11 title, DX11, D3D12, Unity and Unreal. Select
  actual available/owned games. Avoid anti-cheat-heavy titles for initial bring-up.
- Steam tests: ownership/Steamworks, achievements, cloud, friends, controller and
  Steam Input, overlay, DLC, Workshop, launch options, multiple executables,
  redistributables, applicable shader cache, Steam/game updates, prefix persistence,
  uninstall/reinstall, multiple users and offline mode. A game window alone is
  insufficient. Track not-tested items explicitly.
- Performance after alternative launch: startup/prefix time, memory, frame pacing,
  shader compilation, CPU/Wine/graphics translation and Metal utilization. Compare
  the same game/configuration against CrossOver/GPTK and available native/Windows
  references. Keep native ARM64 support code where practical and Rosetta for
  Windows code requiring it; do not remove Rosetta for branding.

The immediate next useful result is an unchanged upstream build plus one real
Windows-only game baseline on this MacBook, followed by the runtime extraction.
