# XCL repeated out-of-memory failures — 24 November 2026

## Summary

After one 60-second run that ended cleanly and displayed a screen immediately before the Rockstar logo, the user ran the title twice more. Both attempts terminated in approximately 31.5 and 34.9 seconds. Both show the same failure pattern:

1. XCL successfully loaded/reused translated graphics pipelines and submitted substantial draw/dispatch work.
2. The coherent memory window reached approximately 3,998 / 4,095 MB.
3. The sparse allocation pool reached approximately 7,444–7,445 MB.
4. `ID3D12CommandList::Close` returned `0x8007000e` (`E_OUTOFMEMORY`).
5. The title then raised unhandled exception `0x80000003` (breakpoint), with `RAX=0x8007000e`.

This is a repeatable allocation/resource failure on this configuration. The logs do **not** by themselves prove whether the limiting resource is a specific XCL coherent-window limit, GPU/driver allocation, or another D3D12 resource exhaustion condition. Do not describe it conclusively as physical VRAM exhaustion.

## Machine / software context

- Windows 11 Home Single Language, build 26300, x64
- Intel Core i7-12700H
- About 15.7 GB installed RAM
- NVIDIA GeForce RTX 3050 Ti Laptop GPU; XCL reports 3962 MiB VRAM
- XCL local checkout: `D:\Development\XCL\repository\xcl`
- Local built executable: `D:\Development\XCL\repository\xcl\target\release\xcl.exe`
- XCL build commit recorded at build time: `f4ce27ceb6d1b0576d0c659b713b674778bf1968`
- Title executable: `D:\Games\GTA 6\game_scarlett_master.exe`
- The title was run with network isolation enabled (`XCL_NETWORK=0`) and a configured 300-second timeout on the later attempts; the process crashed before the timeout could expire.

## Run evidence

Two log files and their report counterparts were uploaded by the user to the repository root:

- `xcl.prev.log` / `report.prev.json` — earlier of the two failed attempts
- `xcl.log` / `report.json` — later failed attempt

The XCL timestamps in the logs show:
- Earlier run: `ID3D12CommandList::Close → 0x8007000e` at approximately **34.926 s**; unhandled breakpoint at 34.927 s.
- Later run: `ID3D12CommandList::Close → 0x8007000e` at approximately **31.467 s**; unhandled breakpoint at 31.470 s.

Both reports contain:
- `crash`: unhandled exception `0x80000003` in `game_scarlett_master.exe+0x72ac`, with `RAX=0x8007000e`
- `exit_code`: decimal `2147483651` (`0x80000003`)
- `d3d12.backend`: `real`
- Adapter: NVIDIA GeForce RTX 3050 Ti Laptop GPU (3962 MiB VRAM)
- `d3d12.uma_mb`: 4095
- `d3d12.uma_used_kb`: 4,094,592 KB (about 3,998.6 MiB)
- `d3d12.translated_pipelines`: 228
- `d3d12.dxil_cache`: 460 hits, 0 misses, 0 compilations, 0 compilation failures
- 40 mesh-pipeline translation errors, where the mesh shader container appears in the amplification-shader slot and the pipeline is refused

The logs additionally show draw/dispatch counters reaching:
- Later run: 1,135 draws and 128 dispatches before the failure.
- Earlier run: 1,459 draws and 256 dispatches before the failure.

The XCL log reports the coherent window as approximately `3998/4095 MB`, with sparse allocation about `7444–7445 MB`. It also logs a buffer view outside UMA receiving an empty shader V# before the command-list close failure.

## Comparison with the first successful timed run

The earlier first run (recorded in [the first-run record](2026-11-24-first-retail-run.md)) ran for 60 seconds and ended through `XCL_RUN_SECONDS` with exit code 0. The user saw a screen immediately before the Rockstar logo after shader/pipeline loading.

In that first report:
- 336 frame presentations
- 6 translated pipelines
- DXIL cache: 1 hit / 12 misses / 12 compilations
- coherent use reported as about 116.7 MB

The two later logs show substantially more D3D12 work and reuse 460 cached translations, but now fill the coherent window to approximately 3.9 GiB and fail at command-list close. This difference suggests the later runs advanced much further into the graphics workload; it is not evidence that the first run was more compatible or that deleting the cache would help.

## Interpretation and limits

- `0x8007000e` is the HRESULT commonly named `E_OUTOFMEMORY`; the immediate failing operation in both logs is a D3D12 command-list `Close`.
- `0x80000003` is the reported game breakpoint exception that follows.
- The repeated signature strongly indicates a resource/memory allocation failure during D3D12 work.
- The 24 GB pagefile improved Windows' reported commit limit to 39.7 GB. A larger pagefile cannot enlarge XCL's 4,095 MB coherent window, increase the laptop's 4 GB dedicated VRAM, or correct unsupported resource/memory behavior.
- The DXIL cache is functioning in these two attempts (460 hits, no compilation failures). Do not delete shader caches as a speculative fix.
- The refused mesh pipelines remain a separate known compatibility gap; they prevent correct world/minimap rendering.

## Upstream status check — 24 November 2026

Local base commit: `f4ce27ceb6d1b0576d0c659b713b674778bf1968`.

Upstream `ivall/xcl` main was observed at `e50836bdf6f5c026f281d545ed86fb2f07ba6e6a`, 8 commits ahead. Comparing these commits showed changes in `Cargo.toml`, `README.md`, `SECURITY.md`, `crates/xcl/build.rs`, `crates/xcl/src/gpu/recomp/dxc.rs`, and `crates/xcl/src/gpu/recomp/pso.rs`. The shader change preserves translated DXIL across unrelated rebuilds; no fix for this title's mesh-pipeline issue or this observed command-list-close failure was established by that comparison.

The current upstream README still states that the newer retail title reaches menus but not gameplay because minimap mesh pipelines are refused. There is no basis to expect simply pulling the currently observed upstream commits to resolve this specific laptop failure.

## Recommendation

**Pause further game launches on this local build.** Do not try an unbounded run, extend the timeout, or clear the shader cache. Wait for an upstream change explicitly addressing D3D12 memory/window allocation, command-list close failures, and/or the mesh-pipeline stream interpretation. Then update/build deliberately and repeat only a short controlled test.

## Public diagnostic-file caution

The raw logs and reports were uploaded at the public repository root. They may expose local Windows profile paths, account identifiers, crash context and process details. Avoid further unredacted uploads. Consider removing or replacing public raw diagnostic files with redacted excerpts after retaining a private local copy. Do not publish game assets, shader dumps, memory dumps or proprietary content.
