# XCL — current-main adaptive 4095 MB run crashes at D3D12 command-list close

**Date:** 24 November 2026  
**Build:** User-confirmed local checkout and release executable are synced to upstream `main`; current upstream reference is [`47efa2e`](https://github.com/ivall/xcl/commit/47efa2ecea0a12934719ae69b81930cf02a4b1c3). The exact local HEAD SHA is not present in the uploaded report.  
**Host:** Windows 11 build 26300; Intel Core i7-12700H; 15.7 GB RAM; NVIDIA RTX 3050 Ti Laptop GPU (XCL reports 3962 MiB).  
**Target:** the user's unmodified local retail-game executable, run with `XCL_NETWORK=0`.

## Result

**The game crashed before the intended 120-second window.** This is the new adaptive-window run represented by root-level `xcl.log` and `report.json`; it is not the separate 207-second manual-stop test documented in [the 512 MB run note](2026-11-24-xcl-207s-black-screen-run.md).

- XCL opened a **4095 MB coherent window** (three sparse VRAM windows).
- At **30.514 s**, `ID3D12CommandList::Close` returned `0x8007000e` (`E_OUTOFMEMORY`).
- At the same timestamp, the game raised unhandled `0x80000003` with `RAX=0x8007000e`.
- XCL logged process exit status `0x80000003` at **31.859 s**.
- `report.json` agrees: `crash` describes the breakpoint with `RAX=0x8007000e`; `exit_code=2147483651` (`0x80000003`).

This reproduces the earlier command-list-close allocation failure **on the user's current-main build**, after the previous two examples on an older build. It is not a clean 120-second smoke completion.

## Rendering, shader and present activity before the crash

- 481 `PresentX` calls / frame boundaries were reported.
- 744 translated pipelines; DXIL cache had **1490 hits, 0 misses, 0 compilations and 0 compilation failures**.
- There were still **40 mesh-pipeline translation refusals**: a console mesh-shader container was found in the AS subobject and the pipeline was refused. This is separate from the E_OUTOFMEMORY failure.
- The report counted 8742 unique shaders, 228478 submissions and 20,308,180 decoded RDNA2 instructions, with zero Xbox shader errors.
- Activity at 24 s was 60 presents in two seconds (about 30 presents/s). At 26 s it was 51 in two seconds (about 25.5/s). The 28 s and 30 s activity windows recorded zero presents, draws or dispatches before the command-list-close failure.
- No DXIL compilation work was occurring in the report; the high cache-hit count indicates shader cache reuse, not a shader-compiler failure.
- No DirectStorage request failures were counted in the activity windows leading to the crash. The new report's BCPack-specific summary fields are empty, so do not use it to make a new BCPack success/failure claim.
- The report has no frame-dump sample. Present counters alone do not establish that the displayed pixels were correct.

The brief 25–30 presents/s intervals are **not a stable benchmark**: presentation stopped in the final activity intervals and the process crashed. This run does not show that the game sustains 30 FPS.

## Memory readings at the failure boundary

The last complete two-second memory line before the error (about 30.027 s) reports:

| Counter | Reported value |
|---|---:|
| Process commit | 14,668 MB |
| DXGI local CurrentUsage | 6,080 MB |
| DXGI non-local CurrentUsage | 4,857 MB |
| XCL target pages | 3,505 MB (3,520 MB pools) |
| XCL arenas | 401 MB in use / 465 MB made |
| XCL sparse pools | 896 MB (838 MB mapped) |
| XCL coherent window | 4,095 MB |
| XCL coherent use | 4,094,592 KB (~3,998.6 MiB) |

These are counters from different allocators/accounting layers. DXGI local/non-local `CurrentUsage`, target pages, pools and sparse mapping are **not interchangeable with dedicated-VRAM residency or a direct WDDM budget reading**. The data show that the coherent window was near its designed usage ceiling and that reported graphics-memory accounting was substantial; they do **not** prove the single cause of the `E_OUTOFMEMORY` result or establish physical VRAM exhaustion.

The earlier read-only `uma_coherent.txt` value was 4,582.2 MB of cumulative allocation requests, which leads to the adaptive 4,095 MB cap after the 512 MB headroom is added. That cumulative hint is not peak live memory. See [the detailed adaptive-window explanation](2026-11-24-first-vs-later-memory-windows.md).

## DXC version discrepancy in the uploaded diagnostics

The report's environment summary lists the custom DXC 1.9.2609.5 directory, but the actual D3D12 startup log and the report's `dxc` object identify the Windows SDK DXC **1.9.2602.17** at `Windows Kits\\10\\bin\\10.0.26100.0\\x64`. The report's runtime-loaded path/version should be treated as the observed compiler for this run. Since DXIL cache misses and compilations were both zero, the log does not show a DXC compile failure or link this version discrepancy to the crash. Resolve the environment/path discrepancy only during a later read-only preflight; do not make a speculative toolchain change for this incident.

## Why the other uploaded files are not the same run

- Root `xcl.prev.log` is the earlier **207-second, 512 MB override** run. It ended with `0xc000013a`, consistent with the user's manual Ctrl+C; it is the source for the long-run FPS and NVIDIA telemetry in [the 207-second note](2026-11-24-xcl-207s-black-screen-run.md).
- Root `xcl-gpu-20261124-171054.csv` spans 17:10:55–17:14:21 and accompanies that 207-second 512 MB run, not the 31.9-second adaptive crash.
- Root `report.prev.json` contains zero presentations, zero shaders, no compiler path and no crash. It does not describe the observed 207-second session and should not be used to derive its results.
- Root `xcl.log` and `report.json` are the adaptive 4095 MB crash pair described above.

Raw uploads contain local Windows profile paths and crash/thread details. Keep public research notes sanitized; avoid duplicating the raw files or quoting their personal path data.

## Performance context from other XCL users

These are upstream reports from different machines and should be treated as comparative data, not predictions for this RTX 3050 Ti:

- [Issue #284](https://github.com/ivall/xcl/issues/284) records Windows/NVIDIA cases where a learned 4095 MB coherent window materially improves title presentation rate over 512 MB. One RTX 3080 Ti 10 GB example reports 13.4 FPS with 512 MB versus about 29.8–30.0 FPS with 4095 MB. A separate RTX 3060 Laptop 6 GB comparison reports 10–14 presents per two seconds with 512 MB versus 60 with 4095 MB, with slightly lower local-video-memory accounting in the full-window runs.
- The same issue also contains counterexamples: a Linux/RADV RX 9060 XT test became black/very slow with 4095 MB while 512 MB reached menus/settings. This shows the effect is platform, driver and memory-path dependent; it is not a universal rule that a 4095 MB window is faster.
- [Issue #345](https://github.com/ivall/xcl/issues/345) reports another `E_OUTOFMEMORY` / game-breakpoint failure for the same title family on a 6 GB GTX 1660 SUPER, after roughly 140–217 seconds, with coherent use near 3998/4095 MB and sparse accounting around 7445 MB. The reporter observed 64 MB heap refusals and explicitly raised video-memory pressure as a concern. That is a separate machine and a later failure point; it supports keeping GPU-resource pressure under investigation, but does not prove the root cause on this 4 GB laptop.

The local 512 MB runs are separately documented: the 120-second run averaged about 5.58 presents/s over the whole timed session; the 207-second run averaged about 6.0 presents/s overall and remained black after the first-run menu/terms flow. Those are not valid gameplay-FPS benchmarks. Their detailed counters are not repeated here.

## Relevant upstream work not yet in current main

The current upstream `main` reference above does **not** include these open branches:

- [PR #304](https://github.com/ivall/xcl/pull/304) addresses DXC's 32-vertex-chunk accounting in the generated mesh pass-through, a known source of mesh-shader translation failures.
- [PR #541](https://github.com/ivall/xcl/pull/541), stacked on #304, handles a console mesh-shader container carried in the AS subobject, correct mesh stand-ins and PC-compatible target formats. The PR description reports clean 90/120-second regression runs at 60 presents per two seconds on an RTX 3060 Ti 8 GB, but it remains open and is not part of this user's current-main build.
- The current logs' 40 AS-slot mesh-pipeline refusals match the kind of case described by #541. These PRs are plausible relevant work for the rendering blocker, **not a demonstrated fix for the separate command-list `E_OUTOFMEMORY` crash**.

Other streamed page-mapping and depth-UAV work is tracked separately in the project. Do not treat any open PR as merged or as part of the user's local build until upstream main actually contains it.

## Conclusion / next action

The current-main adaptive-window path is **not stable on this machine in the tested state**: it crashes around 32 seconds while the coherent window is at its 4,095 MB cap. The current-main 512 MB override survives longer but stays at roughly 5–7 presents/s and eventually displays black; it is not a successful gameplay path either.

**Do not repeat the adaptive 4,095 MB launch just to reproduce this failure.** No game files, XCL source, executable, cache, pagefile or configuration need to be changed for this record. Revisit the title after relevant upstream runtime/memory work is merged and a controlled test can be planned; keep any future comparison on the unmodified upstream harness and record the actual window chosen.
