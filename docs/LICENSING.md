# Initial licensing inventory

This records upstream notices at `c3a49486c14cdf6b9716e2e7ac55286aa537fcf1`.
It is an initial source/dependency inventory. Binary redistribution has not been
audited or approved, and no release binaries were built or published in this
session. Preserve each component's original notices; do not relicense the fork.

| Component | Evidence and action |
| --- | --- |
| Bulk of NotProton | `LICENSE` and `NOTICE`: GPLv3. Retain license and corresponding-source obligations for distribution. |
| lsteamclient | `LICENSE.lsteamclient` and `NOTICE`: Steamworks SDK license. Review the actual agreement and fetched sources before distribution. |
| steam-shim | `LICENSE.steam-shim` and `NOTICE`: Valve Proton 9 steam_helper BSD-3-Clause code; SDK headers remain under their own Steamworks terms. |
| Wine headers/source | Upstream notice identifies LGPL terms. Audit exact source/version and individual bundled components before shipping a self-built runtime. |
| OpenVR headers | Upstream notice identifies BSD-3-Clause. Preserve notices. |
| Dobby | `vendor/README.md`: Apache-2.0; externally fetched at `5dfc8546954ce3b3198132ab13fddb89ee92cdd7`. |
| cJSON | Tracked `vendor/cJSON.c`/`.h`: MIT notices. |
| JsonCpp | `steam-shim/jsoncpp.cpp` and JSON headers: original public-domain/MIT notices. |
| Sparkle | SwiftPM exact version 2.10.0; upstream notice identifies MIT. Audit the embedded framework and its included notices. |
| Valve client/support DLLs | `bridge/fetch-valve.sh`, package manifest and fetcher stage third-party payloads. Review their specific permissions separately; download availability does not establish redistribution rights. |
| CrossOver runtime | Baseline uses a user-installed licensed product and a local runner copy. Do not commit or publish proprietary CrossOver binaries, license files or user data. |
| GPTK / D3DMetal | User-supplied initially. Inspect the license accompanying the exact downloaded release; the product page does not establish redistribution rights. Do not commit restricted Apple payloads. |
| DXMT / DXVK / MoltenVK / alternative Wine distributions | Future dependencies. Review exact revisions, licenses, bundled components and notices before importing code or binaries. No new dependency was imported here. |

The existing payload manifest distinguishes `built`, `patched` and `valve` files.
That provenance model is useful but does not itself grant legal permission to
redistribute any file. Generated detour payloads and patched ntdll need source
provenance as well as the original runtime's applicable terms.

For a source-built non-CrossOver runtime, obtain CodeWeavers' published open-source
Wine sources, audit them and build those sources. Do not extract proprietary
product binaries and present them as an open-source runtime. Upstream's stock
Wine bridge build is distinct from building the whole Wine distribution.

Primary references for continued review:

- [Valve lsteamclient license](https://github.com/ValveSoftware/Proton/blob/proton_11.0/lsteamclient/LICENSE)
- [Valve Proton repository](https://github.com/ValveSoftware/Proton)
- [Sparkle 2.10.0 license](https://github.com/sparkle-project/Sparkle/blob/2.10.0/LICENSE)
- [Apple Game Porting Toolkit](https://developer.apple.com/games/game-porting-toolkit/)
- [CodeWeavers source archive](https://media.codeweavers.com/pub/crossover/source/)
- [DXMT](https://github.com/3Shain/dxmt)

The source archive listing returned HTTP 403 during the initial browser check;
no archive was downloaded or audited. The DXMT repository identifies D3D11/10
Metal translation and has multiple license files; its exact license scope has
not been reviewed here. Further runtime-source, GPTK-term and packaging research
remains open. No claim of redistribution eligibility should be inferred from this
document.
