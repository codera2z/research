# XCL local checkout — current upstream main

**Date:** 24 November 2026  
**User-confirmed status:** local XCL source checkout and release executable are updated to current upstream `main`, with all changes already merged into upstream present locally.

## Current upstream reference

At the time of this record, upstream `main` points to:

- Commit: `47efa2ecea0a12934719ae69b81930cf02a4b1c3`
- Link: https://github.com/ivall/xcl/commit/47efa2ecea0a12934719ae69b81930cf02a4b1c3
- The GitHub compare API showed this revision 48 commits ahead of the old `beaa245ca04e212f064efc7cafcdce104bc5c0a2` checkpoint, with no divergence from that base.

**Important correction:** the previous note treated `beaa245` as the current local build and described the checkout as eight commits behind. That was stale. The old SHA is a historical checkpoint, not the current local revision. Do not repeat the “8 commits behind” statement.

The local HEAD was not directly read through the user's PowerShell in this conversation. The user explicitly confirmed the checkout is up to date, and the latest run's successful BCPack decoding is consistent with the merged #905 implementation. For an independently recorded exact local SHA, use the `commit:` line from the next official XCL smoke output.

## Relevant upstream merges present in current main

Among the current-main changes relevant to this machine and the recent title tests:

- #905 — BCPack decoding for BC1, BC3, BC4, BC5, BC6H and BC7.
- #1009 — retain reusable command allocators after a failed Reset.
- #1012 — released command lists no longer retain allocator recordings.
- #1023 — update allocator recordings only after successful Reset.
- #951 — query tiled-resource support before creating reserved depth textures.
- #990 — correct GFX10 image format 134.
- #1016 — reject removal of non-empty virtual directories.

The latest uploaded 120-second report records all 126 sampled BCPack requests as successfully unpacked (BC1 6, BC5 100, BC7 20; zero decoder failures), corroborating that the old “BCPack refused” assessment is obsolete for the current executable.

## What is not implied by being up to date

Open pull requests are not included merely because the local checkout is current. The mesh-pipeline translation, streamed page-mapping, depth-UAV and NVIDIA PSO-performance work must be evaluated by their actual merge state. In particular, the 40 mesh-pipeline refusals still present in the latest logs show that the gameplay-rendering blocker has not been eliminated just by updating to current main.

## Standard next run

Use the upstream regression harness without patches:

```powershell
.\scripts\smoke.ps1 -Game 'D:\Games\GTA 6\game_scarlett_master.exe' -Seconds 120 -Xcl '.\target\release\xcl.exe'
```

The script sets `XCL_NETWORK=0`, `XCL_RUN_SECONDS=120` and `XCL_ACTIVITY=1`, then prints the exact local source commit and run summary. Do not set `XCL_COHERENT_MB` for this run; leave XCL's adaptive coherent-window selection intact.

This status update does not change the XCL source, executable, title files, settings or caches.
