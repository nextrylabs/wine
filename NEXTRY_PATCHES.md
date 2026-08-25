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
(§4.3). A parallel branch `nextry/winemac-macdrv-functions` carries the earlier
form of this work rebased onto `wine-10.0`.

## Patches

### NEXTRY-WINE-0001 — winemac: make the client Metal view reachable from a sibling unix module

- **Platform:** macOS (x86_64 lane; the D3D10/D3D11 → Metal path served by DXMT).
- **Component:** `dlls/winemac.drv` (`macdrv.h`, `window.c`).
- **Symptom.** Every D3D11 title that presents through a window `abort()`s inside
  the translation layer with "Failed to create metal view, it seems like your
  Wine has no exported symbols needed by DXMT" — a process death, not a bad
  frame.
- **Rationale.** DXMT's unix module (`winemetal.so`) reaches winemac.drv through
  `dlsym(RTLD_DEFAULT, "macdrv_functions")` and then reads the client Cocoa view
  at offset `0x18` of `struct macdrv_win_data`. Three independent things break
  that on 11.0, and fixing any two of them still leaves the abort:
  1. winemac.drv is built `-fvisibility=hidden`, so no such symbol is exported.
  2. commit `6471a42` folded `cocoa_view`/`client_cocoa_view` into a single
     `client_view`, so offset `0x18` now holds `rects` rather than a view.
  3. even once exported, the symbol is unreachable: ntdll opens every unix module
     with plain `RTLD_NOW` (`dlls/ntdll/unix/loader.c`), i.e. `RTLD_LOCAL`, and
     macOS excludes `RTLD_LOCAL` images from `dlsym(RTLD_DEFAULT, …)`.
  A fourth appears only once the lookup succeeds: since `6471a42` a client view
  exists solely after a client surface has presented through it, and a renderer
  that bypasses Wine's own D3D never creates one — so the field is NULL exactly
  when it is needed. Wine 9/10 had no such gap because `create_client_cocoa_view`
  built the view up front.
- **Change.**
  1. `struct macdrv_win_data` regains `client_cocoa_view` immediately after
     `client_view`, restoring offset `0x18` ahead of `rects`.
  2. `get_win_data` refreshes that field from `client_view` — but only when one
     exists, so a NULL never erases a view created on demand.
  3. An exported `macdrv_functions` data symbol whose 10-member layout matches
     the consumer's `struct macdrv_functions_t`. Only the members it dereferences
     carry a real function; the rest are NULL. Per-symbol
     `__attribute__((visibility("default")))`; nothing else becomes visible.
  4. A constructor re-opens this image by its own path with
     `RTLD_GLOBAL | RTLD_NOLOAD`, promoting the already-resident copy into the
     global namespace so the export is actually reachable. `RTLD_NOLOAD` keeps it
     a promotion and never a second load.
  5. The vtable's `get_win_data` creates the client Cocoa view on demand for that
     caller alone, so no window pays for a view it never uses.
- **Tests.** Measured on Apple M4 Max / macOS 26.5.1, Wine 11.0, DXMT v0.80
  installed as builtin, in an Aqua session.
  - Symbol: `nm winemac.so` → `D _macdrv_functions`.
  - Visibility: with the module opened `RTLD_LOCAL`,
    `dlsym(RTLD_DEFAULT, "macdrv_functions")` returns NULL before patch 4 and a
    valid pointer after it.
  - Function: a minimal D3D11 client calling `D3D11CreateDeviceAndSwapChain` on a
    real `HWND` — the exact abort path — returns `hr=0x00000000`,
    `FEATURE_LEVEL 11_1`, adapter `Apple M4 Max` (`vendor=0x106b`), and a
    subsequent `IDXGISwapChain::Present` returns `hr=0x00000000`.
  - Control: the same client on the same tree with an unpatched `winemac.so`
    still aborts with the message above, so the pass is attributable to this
    patch and not to the environment around it.
- **Scope, stated precisely.** This fixes windowed D3D11 presentation. It is
  *not* what makes an offscreen compositor reach the GPU: a Chromium/ANGLE client
  rendering offscreen never calls `CreateMetalViewFromHWND`, and was measured to
  reach hardware with and without this patch once its translation layer was
  installed correctly. Both results are real; they are different halves.
- **Upstream PR.** Not yet submitted (§4.6 obligation open). Patches 3 and 4 —
  a defined accessor for out-of-module Metal-view access — are the
  contribution-worthy part. Patches 1 and 2 are a fork-local compatibility shim
  for the consumer's current offset assumption and are unlikely to land as-is.

## Operational note (not a patch)

DXMT's PE modules must be installed as Wine **builtins**, which is what its
`-builtin` release archive means. Installed as *native* into a prefix's
`system32`, ntdll never binds `winemetal.dll` to its unix half and `DXGI.dll`
fails to import it, taking the client's GPU process down. This is packaging, not
source, and is recorded here only because it is easy to mistake for this bug.
