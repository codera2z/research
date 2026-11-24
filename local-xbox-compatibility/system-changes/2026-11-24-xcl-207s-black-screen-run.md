# XCL — 207-second black-screen run on current local main

**Date:** 24 November 2026  
**Build status:** User confirmed the local XCL checkout/executable is updated to current upstream `main` with all already-merged changes. See [current local source/build status](2026-11-24-xcl-current-main-status.md).  
**Host GPU:** NVIDIA GeForce RTX 3050 Ti Laptop GPU, 3962 MiB reported by XCL  
**Test settings:** `XCL_NETWORK=0`, `XCL_RUN_SECONDS=300`, `XCL_COHERENT_MB=512`, `XCL_ACTIVITY=1`. The user manually stopped the run after about 207 seconds while the screen remained black.

## User-observed sequence

Rockstar logo → opening/menu screens → narration/menu flow → terms and conditions accepted → black screen. The black screen persisted, so the user stopped the run before the 300-second timeout.

## Log and exit interpretation

- XCL was still presenting frames during the black-screen interval; the log shows roughly 1,243 presentations over the observed run (about 6 presentations/s overall).
- The end status `0xC000013A` is consistent with Ctrl+C/manual interruption.
- The PowerShell variable `$XclExit` displayed `-1` because the script was interrupted before it assigned `$LASTEXITCODE`. Do not treat `-1` as XCL's actual game exit code.
- The final `report.json` is not a normal completed-run report. Use the log and GPU telemetry as partial diagnostics only.

## Graphics and DirectStorage

- The same 40 mesh-pipeline refusals occurred early in the run (approximately 17.3–21.8 seconds): mesh-shader container appeared in the AS slot.
- All sampled BCPack requests decoded successfully: BC1 6, BC5 100, BC7 20; 126 total, zero decoder failures.
- No new D3D12 `E_OUTOFMEMORY` command-list-close failure was recorded in this run.
- Frame presentations continuing on a black image do not mean the scene is being rendered correctly.

## GPU and memory observations

From the accompanying NVIDIA telemetry and XCL activity lines:

- NVIDIA telemetry spans the same 207-second interval (17:10:55–17:14:21): GPU memory reached approximately 3,909 MiB of 4,096 MiB; temperature peaked at 46.6°C; GPU utilization peaked at 100% and averaged about 23.4%. Temperature alone does not indicate thermal throttling.
- XCL's rolling present samples were 6.5 FPS at 101.715 s (600 total frames), 6.9 FPS at 144.916 s (900 total) and 5.7 FPS at 197.763 s (1,200 total). The log ends with ~1,243 presentations over ~207 s (~6.0/s overall). These values are consistent with the black-screen phase, not a game-world benchmark.
- Process commit reached about 17,490 MB near 206.5 s. At that point the log reported DXGI local/non-local `CurrentUsage` of 7,743 / 1,362 MB, XCL target pages 4,548 MB (4,608 MB pools), arenas 461 MB, and sparse pools 1,024 MB (982 MB mapped). The coherent-window use was about 116 / 512 MB in a nearby activity line.
- DXGI and XCL counters come from different accounting layers. Do not equate the local/non-local CurrentUsage, target-page pools or sparse-pool counters with physically resident dedicated VRAM. The run did not complete a clean shutdown report, so its partial values are diagnostic snapshots, not final high-water marks.

## Interpretation

The latest local build contains the upstream merges, and the old BCPack refusal issue is resolved in these runs. The remaining black screen is consistent with the mesh-pipeline refusals already documented by upstream. The test does not prove hardware incapability or gameplay performance because the game never reached a correctly rendered world scene.

The later adaptive 4095 MB run is a separate session and crashed at `ID3D12CommandList::Close → E_OUTOFMEMORY` after about 30.5 s; see [the separate adaptive-run note](2026-11-24-xcl-adaptive-4095mb-crash.md). Do not combine that crash's report/memory counters with this 207-second session. The latter's `xcl.prev.log` plus the timestamp-matched CSV support the telemetry above; `report.prev.json` does not.
