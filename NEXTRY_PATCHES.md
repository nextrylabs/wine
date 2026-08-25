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
  6. Static assertions pin both layouts the consumer indexes by offset.
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
  - Layout: `C_ASSERT` on `offsetof` for all four struct fields and six vtable
    slots, so a future reshuffle fails the build instead of silently handing the
    consumer a wrong pointer — which is exactly how `6471a42` broke this. Negative
    control: asserting `0x20` instead of `0x18` fails the build, so the assertion
    is not vacuous.
- **Scope, stated precisely — and one claim withdrawn.** This fixes D3D11
  presentation **for a process presenting to its own window**. That is measured.
  It does **not** make a browser-architecture client reach the GPU. An earlier
  revision of this ledger said such a client "was measured to reach hardware";
  **that is withdrawn.** It was read while the macOS session was locked, so no
  window existed and nothing had asked for a swapchain. Re-measured with a window
  present, Chromium falls back to software (`gpu_compositing: disabled_software`,
  window pure black) because DXMT refuses the swapchain outright:
  `src/d3d11/d3d11_swapchain.cpp:1099` returns `E_FAIL` with "cross-process
  swapchain not supported yet" when the `HWND` belongs to another process, which
  is precisely Chromium's browser/GPU-process split. That rejection happens
  *before* any Metal view is requested, so this patch cannot reach it and never
  could. Fixing it is upstream DXMT work, not a winemac.drv change.
  *(Amended 2026-08-25: half of that last sentence fell. The cross-process
  rejection itself is still upstream DXMT work, but it can be sidestepped by
  running the client's GPU thread in the window-owning process
  (`--in-process-gpu`) — and once it is, a second, winemac.drv-side gap appears
  and is closed by NEXTRY-WINE-0002 below. The combination is measured working
  for the Steam client.)*
- **Upstream PR.** Not yet submitted (§4.6 obligation open). Patches 3 and 4 —
  a defined accessor for out-of-module Metal-view access — are the
  contribution-worthy part. Patches 1 and 2 are a fork-local compatibility shim
  for the consumer's current offset assumption and are unlikely to land as-is.

### NEXTRY-WINE-0002 — winemac: serve a child HWND its toplevel's client view

- **Platform:** macOS (x86_64 lane; same DXMT consumer as NEXTRY-WINE-0001).
- **Component:** `dlls/winemac.drv` (`window.c` only; no header or layout change).
- **Symptom.** A multi-process browser client (the Steam client's CEF, Chromium
  126) run with its GPU thread in the window-owning process — `--in-process-gpu`,
  the only configuration DXMT's cross-process guard admits — still dies in the
  same "Failed to create metal view" `abort()` that 0001 removed for ordinary
  titles. The browser process then wedges mid-teardown: Chromium's threads are
  gone, the Cocoa loop keeps pumping, and the client shows no window at all.
- **Rationale.** Chromium's compositor presents to a **child** HWND. winemac
  gives children `win_data` but never a Cocoa window — `macdrv_create_win_data`
  builds an NSWindow only when the parent is the desktop — so 0001's on-demand
  view path finds nothing to attach a view to and returns it NULL, which the
  consumer treats as fatal. A child's pixels land on its toplevel's surface
  anyway under winemac; the view the renderer needs is the root's.
- **Change.** The vtable's `get_win_data` resolves any HWND whose data lacks a
  Cocoa window (children; windowless cases) to `NtUserGetAncestor(hwnd, GA_ROOT)`
  and serves the root's client view, created on demand as in 0001. Lock
  discipline: `get_win_data` returns holding the global `win_data` mutex, so the
  first lookup is released before the root lookup — re-locking would deadlock.
  Plus a one-line diagnostic at this exact decision point, emitted only when
  `BBX_MACDRV_DIAG` is set, because this is where "no view" becomes the
  consumer's `abort()` and the macdrv TRACE channel is too coarse to leave on
  under a full browser client.
- **Tests.** Measured on Apple M4 Max / macOS 26.5.1, Wine 11.0 base, DXMT
  v0.80-148-g856d9f3 as builtin, current Steam client (CEF/Chromium 126),
  unlocked Aqua session held awake (`caffeinate -disu`), window present — the
  admissibility discipline the 0001 withdrawal established.
  - Resolution observed: `bbx-macdrv: metal-view lookup hwnd 0x1011e root
    0x5010a … client_cocoa_view 0x…` — child resolved to a different root, view
    non-NULL. A toplevel resolves to itself (probe: `hwnd 0x2004e root 0x2004e`).
  - Client outcome, against a same-day 4-run control (identical binaries, no
    `--in-process-gpu`): cross-process swapchain rejections 1/run → **0**;
    `gpu_compositing` `disabled_software` → **`enabled`** across 13 consecutive
    CDP samples; `glRenderer` names `ANGLE (Apple, Apple M4 Max … Direct3D11
    vs_5_0 ps_5_0)`; the on-screen 705×440 login window measures
    `distinct_rgb=1116, mean_luma≈58.3, frac_luma_gt8=1.0` where the control is
    a pure black window (`distinct_rgb=1, mean_luma=0.00`) or none at all. No
    separate GPU process exists in the tree, no SwiftShader process appears, and
    no GPU-disable token is present.
  - Regression: the 0001 own-window probe still passes on this build
    (`D3D11CreateDeviceAndSwapChain hr=0`, `FEATURE_LEVEL 11_1`, `Present hr=0`).
- **Scope and limits, stated precisely.** All presenting children of one
  toplevel share that toplevel's client view, and child-rect offsets are not
  applied — correct for a compositor child that fills its window (the measured
  case), untested for partial-area children. The `--in-process-gpu` delivery
  mechanism is the *client's* concern and lives outside this fork (Steam's argv
  whitelist does not forward it; BottleBox measured it via a steamwebhelper
  wrapper). Retirement: this patch's need disappears for browser clients if
  upstream DXMT implements cross-process presentation (their issue #151); the
  child-to-root mapping itself remains useful for any out-of-module renderer
  handed a child HWND.

### NEXTRY-WINE-0003 — ntdll: append env-declared arguments to a multi-process app's main process

- **Platform:** cross-platform mechanism; the reason it exists is the macOS DXMT lane.
- **Component:** `dlls/ntdll/unix` (`process.c` only).
- **Symptom / need.** A multi-process client whose GPU-hosting helper needs a
  command-line flag the parent does not forward has no declarative way to add
  it. The concrete case: Steam's `steam.exe` spawns `steamwebhelper.exe` with a
  fixed argv whitelist carrying no `--in-process-gpu` and no `--use-angle` (read
  from the binary), so the CEF compositor cannot be routed onto the same-process
  swapchain path DXMT requires (see NEXTRY-WINE-0002 and the DXMT cross-process
  guard). The only prior lever was to replace `steamwebhelper.exe` with a wrapper
  *inside Steam's install tree* — which a Steam self-update or integrity-repair
  pass overwrites, silently returning the client to software rendering.
- **Why ntdll and not kernelbase — measured, not assumed.** An earlier draft of
  this patch hooked `kernelbase!CreateProcessInternalW`. Instrumentation showed
  that path sees all six of Steam's `--type=`-tagged CEF *sub*-processes but
  **never the `--type=`-less browser process** — Steam launches the browser
  bypassing the Win32 CreateProcess layer (a direct `NtCreateUserProcess`), so a
  kernelbase hook flags exactly the wrong processes and misses the one that
  matters. `NtCreateUserProcess` in the ntdll unix backend is the one funnel
  every launch — Win32 or direct-syscall — passes through: instrumentation
  confirmed it sees the browser (`--type` absent) *and* the children, and that a
  `BBX_`-prefixed launch-environment variable propagates to it across wine-child
  hops.
- **Change.** `bbx_build_appended_cmdline`, called at the single point where the
  child command line is serialized into its startup info (`create_startup_info`),
  reads two launch-environment variables — `BBX_WINE_APPEND_TARGET` (a child exe
  basename) and `BBX_WINE_APPEND_ARGS` (the arguments) — and, when the spawn's
  image basename matches the target **and** its command line carries no
  `--type=` marker (i.e. the *main* process of a Chromium/CEF-style app, not a
  sub-process) **and** the arguments are not already present, appends them. The
  appended command line is swapped in only for the `create_startup_info` call and
  the caller's `UNICODE_STRING` is restored immediately after, so the caller's
  own teardown is untouched and nothing leaks. The env vars are unset by default,
  so the fast path is two `getenv()`s and a return — no behavioural change for
  anyone who has not opted in.
- **Where the policy lives (§4.5 boundary).** The fork carries only the generic
  mechanism "append env-declared args to the main process of a named multi-process
  app". The concrete policy — that `steamwebhelper.exe` gets `--in-process-gpu` —
  is data BottleBox places in the launch environment (`GameProfile.env_overrides`),
  outside the fork. No store name, product logic, or selection heuristic is in
  this patch; the `--type=` skip is generic Chromium-family knowledge, not
  Steam-specific.
- **Tests.** Apple M4 Max / macOS 26.5.1, Wine 11.0 base, DXMT builtin, unlocked
  Aqua session, **the Valve `steamwebhelper.exe` byte-identical and under its own
  name** (`stat` = 7,697,048, asserted before launch).
  - Instrumented funnel census: the browser spawn (`--type` absent) is seen here
    and carries the propagated env var; the six sub-processes carry `--type=` and
    are correctly skipped.
  - Steam launched with `BBX_WINE_APPEND_TARGET=steamwebhelper.exe`
    `BBX_WINE_APPEND_ARGS=--in-process-gpu` in the environment, no wrapper: `ps`
    shows the live browser process carrying `--in-process-gpu` that no argv on
    `steam.exe`'s side supplied, no separate GPU process, and the same A/B
    outcome as the earlier wrapper run — rejections → 0, `gpu_compositing
    enabled`, on-screen login window composited. [Measured 2026-08-25;
    decomposition doc §C.10.]
  - Negative default: with the env vars unset, no flag appears and the client is
    the software-fallback control, so the mechanism is inert unless opted in.
- **Upstream.** Not offered upstream — a niche, fork-local convenience Wine is
  unlikely to want in this form. Documented as an L3 fork addition with no
  expected retirement (it is not fixing a regression upstream introduced). It
  retires only if a first-class per-child launch-argument facility lands
  upstream, or if upstream DXMT implements cross-process presentation and the
  flag stops being needed at all.

## Operational note (not a patch)

DXMT's PE modules must be installed as Wine **builtins**, which is what its
`-builtin` release archive means. Installed as *native* into a prefix's
`system32`, ntdll never binds `winemetal.dll` to its unix half and `DXGI.dll`
fails to import it, taking the client's GPU process down. This is packaging, not
source, and is recorded here only because it is easy to mistake for this bug.
