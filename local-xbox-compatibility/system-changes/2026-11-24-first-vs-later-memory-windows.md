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

The user's actual cache value was read-only inspected on 24 November 2026 and is **4,582.2 MB**. This confirms the cumulative allocation-request hint and the 4,095 MB cap for this machine; it is not a peak simultaneous usage measure.

## User's actual cached value — read on 24 November 2026

The user ran the read-only PowerShell check and obtained:

```text
CacheFile             C:\Users\Shardul\AppData\Local\XCL\RockstarGamesInc.28872443D20A7\cache\uma_coherent.txt
SavedTitleRequestMB   4582.2
ExpectedWindowMB      4095
```

This confirms that the local XCL cache records a cumulative GPU-visible graphics-allocation request of about **4,582.2 MB**. Important: `kernelx.rs` does `ASKED.fetch_add(size, ...)` and does not decrement this counter on frees. Thus 4,582.2 MB is a cumulative allocation-request sum across the run, **not proof that 4,582.2 MB was simultaneously live or physically resident in VRAM**.

At the next launch, `coherent_hint()` adds 512 MB of runtime headroom: 4,582.2 + 512 = 5,094.2 MB, which exceeds the D3D12/XCL maximum of 4,095 MB, so it clamps to 4,095 MB. The free-commit guard is 17 GiB; the post-pagefile system had a 39.7 GB commit limit and roughly 30.7 GB free after Firefox was started (based on the user's 9 / 39.7 GB reading), so the threshold was likely met.

## Why the reports show 3,998.6 MB used from a 4,095 MB window

In the exact local source, `kernelx.rs` uses:

```rust
const KEEP: usize = 96 << 20;
let fits = u.size().saturating_sub(u.used()) >= size + KEEP;
```

This deliberately leaves about 96 MiB in the coherent window for XCL's own readback / internal allocations (per-draw arguments have a separate pool). The difference between 4,095 MB and the reported 3,998.6 MB used is about 96.4 MB, consistent with that reserve policy.

So the later reports show the title/XCL coherent allocations reaching the designed usage ceiling, **not necessarily an accidental attempt to allocate beyond the entire window**. The reserve is intentional. This does not explain the D3D12 `Close` returning `E_OUTOFMEMORY`; that failure may still involve driver resource allocation, local/non-local GPU budget, the fallback memory path, or another interaction. The reports do not prove that dedicated VRAM is full.

## Revised interpretation of the three runs

- First run: 512 MB coherent window; 119,552 KB (~116.75 MB) actually allocated from that XCL window; no crash within 60 seconds; the user saw a pre-Rockstar-logo startup screen.
- Later runs: 4,095 MB coherent window; 4,094,592 KB (~3,998.6 MB) used, leaving approximately 96 MB held back by XCL's reserve policy; both fail at D3D12 command-list `Close` with `0x8007000e`.
- The coherent-window size change is directly explained by the saved cumulative request. This makes the three runs different memory-path experiments.
- The first run demonstrates that the game can get to an early visible startup screen on this RTX 3050 Ti with a 512 MB coherent window and fallback allocations. It does **not** prove that later menu/rendering workloads will fit, but it also means we should not conclude that the GPU cannot run the game at all.
- `uma_used_kb` and `sparse_used` are XCL UMA allocator counters, not NVIDIA dedicated VRAM telemetry. Dedicated VRAM usage/budget would need separate GPU telemetry or XCL activity logs.
- Do not interpret the cache's 4,582.2 MB as peak simultaneous GPU memory demand. It is cumulative requested allocation size.

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

The first clean 60-second run is meaningful evidence that the title can start on this laptop under the smaller initial 512 MB coherent window. The user's cache hint confirms a cumulative 4,582.2 MB graphics-allocation request, which makes XCL clamp the next coherent window to 4,095 MB. The later logs show about 3,998.6 MB allocated and approximately 96 MB deliberately held in reserve. This is not proof that dedicated VRAM is exhausted, nor does it explain the subsequent `E_OUTOFMEMORY` by itself. The exact cache value and 96 MB reserve are documented above.
