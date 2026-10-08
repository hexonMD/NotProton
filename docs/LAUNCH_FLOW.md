# Observed launch flow

Source: upstream `c3a49486c14cdf6b9716e2e7ac55286aa537fcf1`. This is a source
trace; no live Steam session has been tested in this work session.

## Install and routing

1. The application installs bundle injection and support files.
2. The ordinary native Steam bootstrapper loads `notproton.dylib`.
3. Signature-resolved hooks enable Steam's existing compatibility-tool features.
4. The local tool declares `from_oslist=windows`, `to_oslist=macos`.
5. An unmapped Windows-only title receives the NotProton tool. A title with a macOS
   platform bit retains Steam's normal native selection unless explicitly mapped.
6. Steam's oslist/depot path selects the Windows build for a compatible title.
   Steam owns downloading and installation; NotProton supplies the tool/hooks.

## Pressing Play

```text
Native macOS Steam Play
  -> Steam compatibility tool invocation: /run %verb%
  -> generated run script (source: dylib/feats/compat_run.sh)
  -> parse allowed NAME=VALUE launch overrides
  -> resolve current CrossOver clone and Wine loader/server
  -> determine AppID and STEAM_COMPAT_DATA_PATH
  -> prefix architecture check and steamuser/cloud directory layout
  -> wineboot initialization, registry settings and per-prefix msync marker
  -> stage Steam bridge files and legacycompat; configure DLL paths/overrides
  -> branch by target: Steam helper vs. foreground game
```

AppID precedence is `STEAM_COMPAT_APP_ID`, then `SteamAppId` when empty/zero,
then the basename of `STEAM_COMPAT_DATA_PATH`. Non-numeric values become zero.
`SteamAppId` and `SteamGameId` are filled if absent. Steam's compatibility data
directory supplies the persistent prefix at `$STEAM_COMPAT_DATA_PATH/pfx`.

The script's `getcompatpath` verb returns the compatibility data path before runner
setup. Other verbs follow the shared launch path. Runtime switching and any new
verb handling must retain Steam's expectations, including install-script calls.

## Helpers

Targets outside `STEAM_COMPAT_INSTALL_PATH` run on the raw Wine loader. This
includes Steam's install-script evaluator/redistributable helpers. The script
stages `legacycompat` into the native Steam client location because the macOS
client lacks the Windows evaluator binaries. Helpers do not use the foreground
app bundle path.

## Foreground games

```text
Prepared launch
  -> resolve title and artwork with appinfo/iconmaker helpers
  -> create launchers/<AppID>/<game>.app with game category/Game Mode metadata
  -> link/copy loader and neighboring Wine libraries into generated bundle
  -> generate launcher script with Wine/Steam environment and overlay setup
  -> open -n -W -a <game>.app ... --args <prefix Steam steam.exe> <game arguments>
  -> Wine runs steam.exe (steam-shim)
  -> steam-shim configures Steam paths/registry and starts the Windows game
  -> patched ntdll + PE lsteamclient + matching Unix lsteamclient half
  -> native steamclient.dylib / communication with native Steam
```

Bridge staging uses `$HOME/Library/Application Support/notproton/bridge`. Files in
the prefix's `Program Files (x86)/Steam` include Valve client DLLs, lsteamclient
and the `steam.exe` helper. `WINEDLLOVERRIDES` requests native Steam client DLLs
and builtin lsteamclient. The 32-bit trigger is also installed in `syswow64`.

`ntdll-patch/detour.c`, `detour32.c` and their assembly shims intercept loader
behavior, load lsteamclient and redirect relevant Steam client exports. This is
more than setting a DLL environment variable: an alternative Wine runtime must
implement equivalent bridge behavior with a matching ABI.

`lsteamclient/unixlib.cpp` synchronizes Steam environment and opens
`$STEAM_COMPAT_CLIENT_INSTALL_PATH/steamclient.dylib` using `dlopen`. Its Unix
architecture must match the loader host: x86-64 for Rosetta, ARM64 for FEX.

## Overlay and lifetime

The generated launcher reconstructs `DYLD_INSERT_LIBRARIES` using the NotProton
overlay shim and `STEAM_DYLD_INSERT_LIBRARIES`. FEX's hardened loader can strip
dyld environment, so lsteamclient additionally loads the shim/renderer manually
and calls the shim installation export. Preserve both mechanisms.

steam-shim manages child launching/waiting and Steam process tracking. The shell
script additionally tracks processes using the prefix's Wine server directory,
`lsof` and `ps`, excluding known Wine/helper processes. It waits for the app bundle
and/or game tree to finish, kills/waits for that prefix's wineserver and sweeps
stragglers. TERM/INT/HUP trigger prefix teardown. Prefix files remain for later
launches.

The shell's exit status can reflect helper/bundle/session failure; it must not be
described as an independently verified Windows game exit code. With `set -e`,
some failing helper paths exit before the explicit status message. Structured
diagnostics should capture these failures without changing teardown semantics.

## Current logging gap

Upstream writes `notproton-run.log` in the compatibility data directory and
`notproton-wine.log` under the AppID launcher directory. The dylib logs to the
NotProton application-support directory. The launch script logs arguments and
Steam/SDL environment broadly. Replace broad logging with explicit safe fields
when implementing diagnostics; do not include Steam tokens, auth values, raw
credential-bearing arguments or full environment dumps. Raw runtime output also
needs a deliberate privacy policy.
