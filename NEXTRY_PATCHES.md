# NEXTRY_PATCHES

Patch ledger for the NEXTRY public fork of Wine, per the BottleBox third-party
dependency strategy (§4, "Modified upstream source"). This branch carries only
BottleBox's macOS runtime patches on top of an exact upstream base. No private
GameProfile data, telemetry, or proprietary selection logic lives here (§4.5);
that logic stays in the BottleBox integration repo behind adapters.

## UPSTREAM_BASE

- Upstream: `https://gitlab.winehq.org/wine/wine.git`
  (mirror `https://github.com/wine-mirror/wine`)
- Base tag: `wine-11.0`
- Base commit: `db11d0fe6a169c457e23d007e20404643d067aa8`

The branch is exactly the base commit plus the isolated patch commits below
(§4.3). A parallel branch `nextry/winemac-macdrv-functions` carries the same
change series rebased onto `wine-10.0`, where the struct layout already matches
DXMT and only patch 3 is required.

## Patches

### NEXTRY-WINE-0001 — winemac: export `macdrv_functions`, reinstate `client_cocoa_view`

- **Platform:** macOS (x86_64 lane; the D3D10/D3D11 → Metal path served by DXMT).
- **Component:** `dlls/winemac.drv` (`macdrv.h`, `window.c`).
- **Rationale.** DXMT's unix module (`winemetal.so`) reaches into winemac.drv two
  ways: first `dlsym(RTLD_DEFAULT, "macdrv_functions")`, and on failure a set of
  individual `dlsym`s (`get_win_data`, `release_win_data`,
  `macdrv_view_create_metal_view`, `macdrv_view_get_metal_layer`,
  `macdrv_view_release_metal_view`). It then reads the client Cocoa view at
  offset `0x18` of `struct macdrv_win_data`. Upstream 11.0 defeats both paths:
  1. winemac.drv is built `-fvisibility=hidden`, so none of those symbols are
     exported and every `dlsym` returns NULL.
  2. commit `6471a42` folded `cocoa_view`/`client_cocoa_view` into a single
     `client_view`, so offset `0x18` now holds `rects`, not a view.
  DXMT's swapchain constructor (`d3d11_swapchain.cpp`, `CreateMetalViewFromHWND`)
  therefore gets a NULL/garbage view and calls `abort()` — for **every** D3D11
  title on this driver, not just a black window in one client. On the macOS
  Stable pack, where DXMT is the declared D3D11 default, that abort is the
  user-visible failure.
- **Change.**
  1. `struct macdrv_win_data` regains `client_cocoa_view` immediately after
     `client_view`, restoring offset `0x18` ahead of `rects`.
  2. `get_win_data` — the single choke point every reader passes through —
     refreshes `data->client_cocoa_view = data->client_view` before returning,
     so the DXMT-facing field tracks the live view without editing the many
     `client_view` assignment sites.
  3. An exported `macdrv_functions` data symbol whose 10-member layout matches
     DXMT's `struct macdrv_functions_t`. Only the members DXMT dereferences carry
     a real function; the rest are NULL. Exported with per-symbol
     `__attribute__((visibility("default")))` against the module default.
  On the `wine-10.0` branch only change 3 applies; 10.0 already lays the struct
  out the way DXMT expects.
- **Test (symbol level).** `nm winemac.so` → `D _macdrv_functions` — the exact
  symbol kind and name DXMT's `dlsym` looks up. The 10-member vtable order is
  byte-identical between DXMT v0.80 and the currently pinned DXMT commit
  `856d9f3`, so the layout is stable across the DXMT revisions in play.
- **Not claimed (functional level).** This ledger does **not** assert that DXMT
  creates a swapchain or renders a frame with this patch. That verification needs
  a complete macOS RuntimePack: this fork built on a macOS-26-capable loader
  recipe (vanilla WineHQ's loader does not run on macOS 26 — the segment
  reservation faults under Rosetta and the preloader references the removed
  `__dyld_func_lookup`; a buildable base needs the Gcenx macOS loader patches),
  plus a DXMT built from source against that same Wine so the unix ABIs match.
  Under the BottleBox constitution that build is a candidate that must pass the
  Golden Game Farm gate (WO-014) before any RuntimePack promotion; this entry
  is a candidate, not a promotion.
- **Upstream PR.** Not yet submitted (§4.6 obligation open). The exported-accessor
  approach is the contribution-worthy part; the `client_cocoa_view` reinstatement
  is a fork-local compatibility shim for DXMT's current offset assumption and is
  unlikely to be upstreamed as-is.
