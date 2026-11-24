# XCL build checkpoint — historical beaa245 record

**Status:** Superseded on 24 November 2026. The user has since confirmed that the local XCL checkout and release executable are updated to current upstream `main`, with the changes already merged into upstream present locally. Do not use this old checkpoint to describe the present local build.

The previously verified checkpoint was:
- Commit: `beaa245ca04e212f064efc7cafcdce104bc5c0a2`
- Subject: merge PR #797, mid-list Wait64 after Write64.
- The GitHub compare showed current upstream main (`47efa2ecea0a12934719ae69b81930cf02a4b1c3`) was 48 commits ahead of this checkpoint, with no divergence from it.

The earlier PowerShell output showing `beaa245` and executable metadata was valid for that historical check, but it was superseded by the user's subsequent update. The generic `xcl 0.1.0` version string does not encode a Git SHA. The exact current local SHA should be captured from the upstream smoke script's `xcl-smoke` output.

Current main includes the already-merged BCPack decoder (#905), allocator fixes (#1009/#1012/#1023), tiled-resource check (#951), format correction (#990) and other merged changes. The latest user reports confirm 126 BCPack requests now decode with zero failures.

For future runs, use the upstream harness from the XCL checkout root:
```powershell
.\scripts\smoke.ps1 -Game 'D:\Games\GTA 6\game_scarlett_master.exe' -Seconds 120 -Xcl '.\target\release\xcl.exe'
```
Do not force `XCL_COHERENT_MB` for the adaptive-window comparison. This file is retained as a historical checkpoint only.
