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

- NVIDIA GPU memory usage reached approximately 3,909 MiB of 4,096 MiB.
- GPU temperature peaked around 46.6°C. This does not indicate thermal throttling by itself.
- DXGI local `CurrentUsage` and XCL's target-page/pool counters rose during content loading. Those are accounting counters and must not be equated with physically resident dedicated VRAM.
- Process commit rose during a DirectStorage/content-loading burst. The run did not complete a clean shutdown report, so do not treat its partial peak counters as definitive end-of-run measurements.

## Interpretation

The latest local build contains the upstream merges, and the old BCPack refusal issue is resolved in these runs. The remaining black screen is consistent with the mesh-pipeline refusals already documented by upstream. The test does not prove hardware incapability or gameplay performance because the game never reached a correctly rendered world scene.

Next comparison: use the upstream `scripts/smoke.ps1` harness at 120 seconds without `XCL_COHERENT_MB`, so the exact local source commit and normal adaptive memory-window choice are logged.
