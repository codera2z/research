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

## Adaptive coherent-memory detail from the first-run comparison

The first successful timed run's report recorded a 512 MB committed UMA window with 119,552 KB allocated. Both later failing reports recorded 4,095 MB total and 4,094,592 KB allocated. The exact local source revision `f4ce27c` already contains an adaptive mechanism:

- `kernelx.rs` sums the title's GPU-visible graphics allocations and calls `uma::note_coherent_need`.
- `uma.rs` saves that sum to `cache/uma_coherent.txt`; next startup may precommit the saved amount plus 512 MB, capped at 4,095 MB (if free commit is at least 17 GiB).
- Graphics allocations that do not fit in the committed window use a write-watched fallback memory path.
- The log's `coherent used/total` values come from XCL's own `Uma::used()` and `Uma::size()` accounting. They are not direct physical GPU VRAM usage.

The first and subsequent runs are therefore not memory-equivalent. See [the dedicated explanation and read-only cache check](2026-11-24-first-vs-later-memory-windows.md). The actual cache hint has not yet been read on the user's machine.

## Interpretation and limits

- `0x8007000e` is the HRESULT commonly named `E_OUTOFMEMORY`; the immediate failing operation in both logs is a D3D12 command-list `Close`.
- `0x80000003` is the reported game breakpoint exception that follows.
- The repeated signature strongly indicates a resource/memory allocation failure during D3D12 work.
- The 24 GB pagefile improved Windows' reported commit limit to 39.7 GB. A larger pagefile cannot enlarge XCL's 4,095 MB coherent window, increase the laptop's 4 GB dedicated VRAM, or correct unsupported resource/memory behavior.
- The DXIL cache is functioning in these two attempts (460 hits, no compilation failures). Do not delete shader caches as a speculative fix.
- The refused mesh pipelines remain a separate known compatibility gap; they prevent correct world/minimap rendering.

## Upstream status check — 24 November 2026

Local base commit: `f4ce27ceb6d1b0576d0c659b713b674778bf1968`.

Upstream `ivall/xcl` main was subsequently observed at `beaa245ca04e212f064efc7cafcdce104bc5c0a2`, 17 commits ahead. Among the merged changes:

- [PR #792](https://github.com/ivall/xcl/pull/792): range-check UMA memory windows and 64-bit address carry/borrow (relevant to memory-window addressing; not a direct capacity increase).
- [PR #795](https://github.com/ivall/xcl/pull/795) and [PR #797](https://github.com/ivall/xcl/pull/797): Wait64 GPU-fence bridge and mid-list command-list splitting (synchronization / submission changes).
- [PR #798](https://github.com/ivall/xcl/pull/798): safer split-wave64 barriers for wave32 GPUs (relevant to the RTX 3050 Ti execution path).

The PR reports show tests on RTX 4090 / Linux + Proton for #792/#795/#797, without a native Windows retail regression run. #798 reports Windows GPU tests on an RTX 5080 and a 60-second retail smoke run on an AMD RX 9070 XT; a retail run on NVIDIA wave32 hardware was not reported. These are relevant improvements to consider in a later rebuild, but none claims to fix this user's exact `E_OUTOFMEMORY` failure. The current README/HANDOFF still reports that the retail title reaches menus but not gameplay because minimap mesh pipelines are refused.

## Recommendation

**Pause further game launches on this local build.** Do not try an unbounded run, extend the timeout, or clear the shader cache. Wait for an upstream change explicitly addressing D3D12 memory/window allocation, command-list close failures, and/or the mesh-pipeline stream interpretation. Then update/build deliberately and repeat only a short controlled test.

## Public diagnostic-file caution

The raw logs and reports were uploaded at the public repository root. They may expose local Windows profile paths, account identifiers, crash context and process details. Avoid further unredacted uploads. Consider removing or replacing public raw diagnostic files with redacted excerpts after retaining a private local copy. Do not publish game assets, shader dumps, memory dumps or proprietary content.
