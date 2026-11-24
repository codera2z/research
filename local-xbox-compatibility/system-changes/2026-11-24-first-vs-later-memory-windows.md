# XCL first run vs later crashes: adaptive coherent-memory window — 24 November 2026

## Main finding

The initial 60-second run and the two later failing runs did **not** use the same XCL coherent-memory configuration.

The report fields from the three runs show:

| Measurement | First timed run | Later failed run A | Later failed run B |
|---|---:|---:|---:|
| XCL report exit | 0 | 0x80000003 | 0x80000003 |
| PresentX calls | 336 | 490 | 470 |
| Committed UMA/coherent size | 512 MB | 4,095 MB | 4,095 MB |
| UMA allocated bytes | 119,552 KB (~116.75 MB) | 4,094,592 KB (~3,998.6 MB) | 4,094,592 KB (~3,998.6 MB) |
| Translated pipelines | 6 | 228 | 228 |
| DXIL cache hits / misses | 1 / 12 | 460 / 0 | 460 / 0 |

The failed-run logs show the coherent allocation was close to full: about 3,998.6 MB of 4,095 MB. They also show approximately 7,444–7,445 MB reserved/allocated in sparse windows. That sparse figure is an XCL sparse-window allocation metric, **not a direct reading of physical dedicated VRAM occupancy**.

## Why the coherent window changed between launches

This behaviour is present in the exact local source commit used for the build:

`f4ce27ceb6d1b0576d0c659b713b674778bf1968`

Relevant source files at that pinned revision:

- [`crates/xcl/src/gpu/uma.rs`](https://github.com/ivall/xcl/blob/f4ce27ceb6d1b0576d0c659b713b674778bf1968/crates/xcl/src/gpu/uma.rs)
- [`crates/xcl/src/hle/kernelx.rs`](https://github.com/ivall/xcl/blob/f4ce27ceb6d1b0576d0c659b713b674778bf1968/crates/xcl/src/hle/kernelx.rs)
- [`crates/xcl/src/hle/d3d12x/bind.rs`](https://github.com/ivall/xcl/blob/f4ce27ceb6d1b0576d0c659b713b674778bf1968/crates/xcl/src/hle/d3d12x/bind.rs)

The mechanism is:

1. XCL starts with a 512 MB committed coherent window by default (`RUNTIME_COMMITTED_CAP`).
2. As the title requests GPU-visible graphics allocations, `kernelx.rs` cumulatively counts their requested sizes and calls `uma::note_coherent_need(asked)`.
3. XCL saves that cumulative requested size in `cache/uma_coherent.txt` beneath the title's XCL data directory.
4. On the next launch, `coherent_hint()` reads that saved value. If Windows reports at least 17 GiB of free commit, XCL creates a coherent window sized approximately `saved title request + 512 MB`, capped at 4,095 MB. If free commit is below 17 GiB, it falls back to the 512 MB runtime window.
5. A graphics allocation that does not fit in the coherent window falls back to write-watched plain memory; this is a different memory path from putting the title's graphics allocations in the committed coherent window.

Therefore, the first run's 512 MB window can be followed by later launches with a nearly 4 GB coherent window. The 4,095 MB size implies that the saved request is at least roughly 3,583 MB before the 512 MB headroom is added (subject to the allocation-granularity rounding and maximum cap).

The exact value in this user's `uma_coherent.txt` file has **not** yet been read. The report figures strongly fit this mechanism, but the cache file should be inspected read-only before treating the explanation as confirmed for this particular machine.

## What the counters mean

In this code revision, XCL's draw/dispatch progress log formats:

- `coherent_used = u.used()`
- `coherent_total = u.size()`
- `sparse_used = u.sparse_used()`

For the runtime layout, `u.size()` is the committed coherent window's size; `u.used()` is the bytes allocated from the committed UMA window. These are XCL memory-window accounting values, not NVIDIA's dedicated/local VRAM meter.

WDDM does charge video-memory allocations against process commit, and XCL's own comments explain that coherent heaps are a trade-off involving system commit and GPU-visible memory. However, the report counters alone cannot tell whether the final `E_OUTOFMEMORY` originated from a coherent-window allocation, the D3D12 driver, local/non-local GPU memory budget, or an interaction between them.

## Interpretation of the three runs

- The first run proves that this executable could start on the RTX 3050 Ti and complete 60 seconds with a 512 MB coherent window using about 116.75 MB of it. It does **not** prove that the full later graphics workload fits in memory.
- The two later runs are not directly comparable to the first: they used a 4,095 MB coherent window and reached about 97.6% of that allocation before the command-list close failure.
- The later logs actually show substantial graphics activity before the crash: 228 translated pipelines, 460 DXIL cache hits, and draw/dispatch activity. Although the visible startup screen did not appear, the logs do not describe a failure before all pipeline work.
- The crash line is `ID3D12CommandList::Close → 0x8007000e` (`E_OUTOFMEMORY`), followed by the game's `0x80000003` breakpoint. This is consistent with a memory/resource allocation problem, but not enough to conclude that the 4 GB dedicated VRAM alone is exhausted.
- The first run makes it inappropriate to say the laptop GPU cannot start the title at all. The later runs make it inappropriate to say VRAM / GPU memory is definitely not a problem.

## Read-only check for the user's cache hint

Expected path (the title directory shown in the XCL report):

`%LOCALAPPDATA%\XCL\RockstarGamesInc.28872443D20A7\cache\uma_coherent.txt`

PowerShell command to inspect its saved byte count and infer the next coherent-window size:

```powershell
$cache = Join-Path $env:LOCALAPPDATA "XCL\RockstarGamesInc.28872443D20A7\cache\uma_coherent.txt"

if (Test-Path -LiteralPath $cache) {
    $bytes = [int64](Get-Content -LiteralPath $cache -Raw).Trim()
    [pscustomobject]@{
        CacheFile = $cache
        SavedTitleRequestMB = [math]::Round($bytes / 1MB, 1)
        ExpectedWindowMB = [math]::Min(4095, [math]::Ceiling($bytes / 1MB) + 512)
    }
} else {
    "No saved coherent-window hint found at: $cache"
}
```

This command only reads the hint. Do not edit, delete or rename it until the value is reviewed.

## Relevant newer upstream work

The local build is at `f4ce27c`; upstream `main` was observed at `beaa245ca04e212f064efc7cafcdce104bc5c0a2`, 17 commits ahead.

Relevant merged changes include:

- [PR #792 — range-check UMA memory windows and 64-bit address carry/borrow](https://github.com/ivall/xcl/pull/792). Relevant to shader memory-window addressing and out-of-range aliases, but not a direct increase to the coherent-window capacity. Its reported tests used Linux + Proton on an RTX 4090; native Windows retail-game regression was not reported.
- [PR #795 — bridge Wait64 operations to GPU fences with cycle detection](https://github.com/ivall/xcl/pull/795), and [PR #797 — split command lists around mid-list Wait64 after Write64](https://github.com/ivall/xcl/pull/797). These change synchronization and command-list submission. Their reported tests used Linux + Proton on an RTX 4090; native Windows retail-game regression was not reported.
- [PR #798 — safer split-wave64 barriers for wave32 GPUs](https://github.com/ivall/xcl/pull/798). This is relevant to the RTX 3050 Ti's wave32 execution path. It had reported Windows GPU tests on an RTX 5080 and a separate 60-second retail smoke run on an AMD RX 9070 XT, but no retail smoke test on an NVIDIA wave32 GPU was reported.

These changes make a current rebuild potentially worthwhile after reviewing the coherent-memory hint, but none of the PR descriptions claims to fix this user's exact `E_OUTOFMEMORY` failure or the unresolved retail minimap mesh-pipeline issue. Do not launch again on the old build just to repeat the same failure.

## Bottom line

The first clean 60-second run is meaningful evidence that the title can start on this laptop under the smaller initial memory path. It does **not** rule out a later GPU/VRAM or WDDM resource-budget problem. The 512 MB → 4,095 MB window change explains why the later attempts reached a very different memory state. Read the adaptive cache hint before choosing whether to test a fallback-memory mode or rebuild current upstream.
