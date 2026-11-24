# Local Xbox Compatibility Projects — Windows Setup Notes

**Last updated:** 24 November 2026  
**Scope:** Personal Windows development environment notes for XCL and VIctory. This folder is intentionally separate from the Business OS repository and does not define requirements for that product.

## Repository boundaries

- XCL source checkout: `D:\Development\XCL\repository\xcl`
- VIctory has only been reviewed; it has not been cloned or installed.
- Keep each compatibility project, game files, configuration, cache and generated data separate. Do not place any of these in the Business OS repository.
- Only use game files that you are legally entitled to use. Do not commit game dumps, proprietary assets, converted game data or screenshots to these repositories.

## Windows machine used for the XCL setup

| Component | Observed configuration |
|---|---|
| OS | Windows 11 Home Single Language, 64-bit, build 26300 |
| CPU | Intel Core i7-12700H, 14 cores |
| RAM | 15.7 GB |
| Discrete GPU | NVIDIA GeForce RTX 3050 Ti Laptop GPU |
| Integrated GPU | Intel Iris Xe Graphics |
| NVIDIA driver | 32.0.16.2114 |
| VRAM reported by XCL tests | 3962 MiB |
| Free storage reported during setup | C: about 60 GB; D: about 320 GB |

Use D: for project-specific files where possible. Some Windows and Visual Studio components may still use system-managed locations on C:.

## XCL — installed and built

Upstream: [ivall/xcl](https://github.com/ivall/xcl)

Local checkout:
`D:\Development\XCL\repository\xcl`

The checkout was on branch `main`, at commit `f4ce27ceb6d1b0576d0c659b713b674778bf1968` when built.

### Toolchain and locations

| Component | Version / location | Result |
|---|---|---|
| Visual Studio Build Tools | `D:\DevTools\VisualStudio\BuildTools` | Installed |
| MSVC x64/x86 toolset | 14.52.37025 | Compiler and linker found |
| MSVC environment script | `VC\Auxiliary\Build\vcvars64.bat` under Build Tools | Found |
| Windows SDK | 10.0.26100.0 | Found |
| Rust stable MSVC | rustc 1.100.0; cargo 1.100.0 | Installed |
| Cargo home | `D:\Development\XCL\tools\Rust\.cargo` | Configured |
| Rustup home | `D:\Development\XCL\tools\Rust\.rustup` | Configured |
| DirectX Shader Compiler | 1.9.2609.5 | Installed |
| DXC directory | `D:\Development\XCL\tools\DXC\1.9.2609\bin\x64` | Contains `dxc.exe`, `dxcompiler.dll`, `dxil.dll` |

User-level environment settings configured for this setup:

- `CARGO_HOME=D:\Development\XCL\tools\Rust\.cargo`
- `RUSTUP_HOME=D:\Development\XCL\tools\Rust\.rustup`
- `XCL_DXC_DIR=D:\Development\XCL\tools\DXC\1.9.2609\bin\x64`
- Cargo's `.cargo\bin` and the DXC x64 directory were added to the user PATH.
- `TEMP` and `TMP` were redirected to `D:\Development\XCL\temp` only in the PowerShell sessions used for installation/building. No persistent user-level TEMP/TMP change was confirmed.

### Build result

Build command, after setting up the MSVC environment with `vcvars64.bat`:

```powershell
cargo build --release --workspace --jobs 4
```

Result: **successful build** in approximately 3 minutes 18 seconds.

Executable:
`D:\Development\XCL\repository\xcl\target\release\xcl.exe`

Observed file size: 6,832,640 bytes.

- `xcl.exe --version` returned `xcl 0.1.0`.
- `xcl.exe --help` displayed the expected CLI commands: `run`, `inspect`, `disasm`, `shaders`, and `rpf`.
- One Rust deprecation warning involving `AtomicU32::fetch_update` appeared; it did not stop the build.

**A first retail-game test was run on 24 November 2026.** The 60-second XCL probe exited cleanly with code `0`; this is evidence that the title process ran through a timed session, not evidence that the game is playable. See [the recorded first-run diagnostics](system-changes/2026-11-24-first-retail-run.md).


## XCL — latest upstream update verified locally (24 November 2026)

The user updated the local XCL checkout and release executable. Read-only PowerShell checks verified:

- Source commit: `beaa245ca04e212f064efc7cafcdce104bc5c0a2` (2026-11-24).
- Subject: merge PR #797, splitting title command lists around a mid-list Wait64 after Write64.
- Branch/status: `main...origin/main`, with no working-tree changes shown.
- Executable: `D:\Development\XCL\repository\xcl\target\release\xcl.exe`.
- Executable size: 6,873,600 bytes; Windows LastWriteTime: `24-Nov-26 11:41:00`.
- `xcl --version`: `xcl 0.1.0`.

The source checkout is verified at the observed upstream `main` commit and is clean. The new executable size and timestamp are consistent with a refreshed build compared with the earlier 6,832,640-byte executable. However, the generic version string does not encode a Git SHA, so the executable's exact source provenance is not independently embedded in the binary.

### Diagnostic options confirmed in the updated source

- `XCL_COHERENT_MB=512` explicitly overrides the adaptive coherent-window hint for one process. The updated `crates/xcl/src/gpu/uma.rs` parses and clamps this value. It does not require deleting or changing `cache/uma_coherent.txt`.
- `XCL_ACTIVITY=1` enables periodic activity diagnostics. Upstream notes describe process commit, local/non-local video-memory use, target pages, arenas and sparse pools.
- `XCL_NETWORK=0` remains required for this title because upstream documents that its crash reporter otherwise attempts to send a minidump to an external server.
- `XCL_RUN_SECONDS=n` provides XCL-managed title shutdown after draining GPU work and queues; do not forcibly kill the process while GPU work may be outstanding.

Upstream references at the verified commit:
- [UMA/coherent-window implementation](https://github.com/ivall/xcl/blob/beaa245ca04e212f064efc7cafcdce104bc5c0a2/crates/xcl/src/gpu/uma.rs)
- [Host activity and timed-shutdown implementation](https://github.com/ivall/xcl/blob/beaa245ca04e212f064efc7cafcdce104bc5c0a2/crates/xcl/src/host.rs)
- [Upstream merge commit #797](https://github.com/ivall/xcl/commit/beaa245ca04e212f064efc7cafcdce104bc5c0a2)

### Next controlled diagnostic (not yet run)

The next proposed comparison is a 60-second run with `XCL_COHERENT_MB=512`, `XCL_ACTIVITY=1`, and `XCL_NETWORK=0`. This tests the smaller coherent window seen in the first successful timed run while retaining the saved adaptive cache. It is a diagnostic comparison, not a claim that 512 MB is the correct long-term setting or that the new build fixes the D3D12 allocation failure.

Do not delete the saved cache, change the pagefile, or infer physical VRAM exhaustion from coherent-window counters alone. Record the screen reached, exit code, activity lines and new report/log paths after the controlled test.

## XCL — test results and limitations

Full test command:

```powershell
cargo test --workspace -- --test-threads=4
```

Observed result: **657 passed, 3 failed, 21 ignored**.

The first failure was:
`hle::d3d12x::title_textures::tests::placed_title_textures_share_vram_and_preserve_pixels`

It reported a cubemap extra-pages reuse assertion mismatch. Two later failures reported mutex `PoisonError`, apparently after the first panic. The first failing test was rerun on its own with `--test-threads=1` and passed (1 passed, 0 failed). Parallel/shared GPU-state interference is a possibility, but the root cause has not been established. The entire suite has **not** been rerun sequentially.

The test output detected the D3D12 backend on the NVIDIA RTX 3050 Ti Laptop GPU (3962 MiB VRAM). Two GPU tests were skipped because the hardware reported `no wave64`. This confirms that many backend tests ran on this laptop; it does not mean every GPU feature or title is supported.

### Gaming Services

The Windows file `System32\xgameruntime.dll` was not present during the checks. XCL used its own XTaskQueue/XAsync fallback. The test output noted that dialogs, text entry, game invites, game streaming and other unsupported APIs are unavailable in that fallback mode.

Gaming Services was not installed or changed during this setup.

## XCL — updater configuration (manual only)

**Verified 24 November 2026.** The Windows Task Scheduler task is named `XCL Auto Update` and is registered in the root Task Scheduler Library.

The task's configured action is:

- Program: `powershell.exe`
- Arguments: `-NoProfile -ExecutionPolicy Bypass -File D:\\Development\\XCL\\tools\\Update-XCL.ps1`

The user removed the recurring one-minute trigger in Task Scheduler. After that change, this read-only inspection returned no trigger entries:

```powershell
$t = Get-ScheduledTask -TaskName "XCL Auto Update"
$t.Triggers | Format-List *
```

Therefore the task is now **manual-only**: it has no configured scheduled trigger. The task and its action were left in place; the script was not deleted.

### Run the updater manually

From PowerShell:

```powershell
Start-ScheduledTask -TaskName "XCL Auto Update"
```

Alternatively, open Task Scheduler → Task Scheduler Library → right-click **XCL Auto Update** → **Run**.

### Verification still outstanding

- The task's action path is recorded, but the script contents have not been re-audited in this session.
- A manual updater run and its resulting update/build outcome have not yet been verified.
- The user removed the one-minute trigger; the task currently has no scheduled triggers. Do not describe XCL as automatically updating.
- To inspect the task's last result after a manual run:

```powershell
Get-ScheduledTaskInfo -TaskName "XCL Auto Update" |
    Format-List LastRunTime, LastTaskResult, NextRunTime, NumberOfMissedRuns
```


## XCL — first retail-game test (24 November 2026)

The user ran the locally built XCL executable against `D:\Games\GTA 6\game_scarlett_master.exe`, with `XCL_NETWORK=0` and `XCL_RUN_SECONDS=60`.

Verified from the generated report and console output:

- Exit code: `0`; report field `crash` was `null`.
- XCL's timed stop ended the title after 60 seconds; 336 frame presentations were recorded.
- Real D3D12 backend: NVIDIA GeForce RTX 3050 Ti Laptop GPU (3962 MiB VRAM).
- Shader report: 8,741 unique shaders; 228,474 submissions; zero Xbox shader errors.
- Six translated pipelines were recorded; the run summary listed no fallback or failed pipelines.
- Multiple mesh pipelines were refused because the mesh shader appeared in the amplification-shader slot. Rendering remains incomplete.
- One stubbed import was reported: `xmem.dll!XMemIsToolsMemAvailable`.
- `System32\xgameruntime.dll` was absent; dialogs, text entry, invites and game streaming remain unavailable in XCL's fallback mode.
- Xbox Live / Rockstar service requests failed with `0x80072EFD` while network isolation was enabled. No successful online authentication was shown.

The user later clarified that the first 60-second run displayed a screen immediately before the Rockstar logo, after shader/pipeline loading. The logo itself, menus and gameplay were not confirmed.

See [the sanitized first-run record](system-changes/2026-11-24-first-retail-run.md). The raw `report.json` uploaded by the user at the repository root includes a local Windows profile path; avoid copying raw identifiers or local paths into further public notes. The user subsequently uploaded `xcl.log`, `xcl.prev.log`, `report.json` and `report.prev.json` to the repository root. Both logs and reports now document two repeated out-of-memory failures; see [the repeated-crash record](system-changes/2026-11-24-repeated-crashes.md). These raw diagnostic files are public and may contain local profile paths or account identifiers; avoid further unredacted uploads.

## XCL — repeated startup/rendering failures (24 November 2026)

After the initial 60-second run that displayed a screen just before the Rockstar logo, the user performed two further attempts. Both terminated with the same D3D12 command-list close failure and an unhandled game breakpoint. The detailed evidence is in [the repeated-crash record](system-changes/2026-11-24-repeated-crashes.md).

Latest logs show XCL did perform substantial graphics work before the failure: about 228 translated pipelines, 460 translated-DXIL cache hits, and draw/dispatch activity. The coherent window reached 3,998 of 4,095 MB and the sparse allocation pool reached about 7,445 MB. `ID3D12CommandList::Close` returned `0x8007000e` (`E_OUTOFMEMORY`), followed by a game `0x80000003` breakpoint. The same pattern occurred after approximately 31.5 seconds and 34.9 seconds in the two attempts.

**Do not repeat the test on this build for now.** The 24 GB pagefile increased Windows' commit limit but does not expand XCL's 4 GiB coherent window or fix its D3D12 resource-management limitations. The available reports do not establish whether the allocation failure is caused by a specific XCL memory-window limit, GPU/driver allocation, or another D3D12 resource exhaustion condition. Do not label it conclusively as physical VRAM exhaustion.

The upstream repository was checked again on 24 November 2026. Its current README still describes the newer retail title as reaching menus but not gameplay due to refused mesh pipelines. The local build is 8 commits behind current upstream; the diff is primarily a DXIL cache-key/build-symbol change and documentation, not a demonstrated fix for this game's mesh or command-list failure. Updating now is not expected to make this title playable.

## XCL — first run versus later memory-window behaviour (24 November 2026)

The first 60-second run and the two later crashes used different coherent-memory window sizes: first run 512 MB (119,552 KB allocated); later runs 4,095 MB (4,094,592 KB allocated, about 3,998.6 MB). XCL's exact local source commit contains adaptive coherent-window sizing: it saves the title's cumulative GPU-visible graphics allocation request in `cache/uma_coherent.txt`, then on the next launch may commit `saved request + 512 MB` up to 4,095 MB when free system commit is at least 17 GiB. The first run can therefore complete on the default 512 MB path while later warm runs take a different, much larger committed-memory path. The exact local cache value still needs a read-only check. See [the detailed first-run / later-run comparison](system-changes/2026-11-24-first-vs-later-memory-windows.md).

This does **not** establish that dedicated VRAM is or is not the root cause. The `uma_mb` / `uma_used_kb` counters are XCL's committed coherent UMA-window accounting, not a direct NVIDIA dedicated-VRAM meter. The later runs were near the committed coherent-window limit when D3D12 command-list `Close` returned `E_OUTOFMEMORY`.

The user subsequently read XCL's `cache/uma_coherent.txt` value: **4,582.2 MB** cumulative requested graphics allocations. The next-run hint adds 512 MB and clamps the coherent window to 4,095 MB. XCL intentionally holds about 96 MB back for internal/readback allocations, explaining the later `~3,998.6 / 4,095 MB` counter. The saved value is cumulative allocation requests, not peak simultaneous VRAM. See [the confirmed first-vs-later memory comparison](system-changes/2026-11-24-first-vs-later-memory-windows.md).

## VIctory — source review only

Upstream: [Marvisak/VIctory](https://github.com/Marvisak/VIctory)

At the time checked (24 November 2026):

- Latest commit inspected: `d2a4bae1f49533ef5cb512fd1b1ba1513f2e1989`.
- No GitHub releases were listed.
- The project's README status dated 23 November 2026 says **“Not playable yet.”**
- The README reports working menus, an intro video with sound, start/profile/save screens and loading into the world. It also reports that the first mission scene remains mostly dark because rendering is incomplete; world frame times were reported around 0.5–1.8 seconds per frame.
- The README's reported test machine was Windows 11 with an AMD RX 6700 XT.

### Important GPU warning

The VIctory README warns that entering the world on a real GPU can currently hang the GPU and cause Windows to reset the display driver. It suggests `VICTORY_D3D12_WARP=1` for diagnostics with a software adapter. WARP is expected to be very slow and is not a gameplay fix.

The project's stated requirements include Windows 10/11 x64, an AVX2 CPU, Direct3D 12, Shader Model 6.6 and mesh shaders for world rendering. It also requires several GiB of available Windows committed memory at startup.

The laptop used for XCL has an RTX 3050 Ti with about 4 GB of reported VRAM. VIctory's compatibility with this laptop has **not** been established; the reported test GPU was different. Do not assume the real-GPU world path is safe on this system.

### Build and data notes from the README

- VIctory pins Rust `1.98.1` in `rust-toolchain.toml` for the Windows MSVC target. Rustup may install that toolchain alongside the existing Rust installation when building VIctory.
- The README's workspace build command is `cargo build --release -p victory`.
- The README describes `cargo xtask install-hooks`, copying `victory.example.toml` to a local `victory.toml`, and configuring game paths outside the source repository.
- Its run form is `target\release\victory.exe run <game directory>\<title executable> --execute`.
- Keep game directories, configuration, cache and data outside the VIctory source checkout.
- The asset policy forbids committing game files, dumps, converted assets, screenshots or other proprietary game material.

VIctory is GPL-3.0-or-later; XCL is GPL-2.0-only. Keep their source trees separate and do not copy code between them without reviewing the licensing implications.

**Current status:** VIctory was reviewed only. It has not been cloned, built, configured or run on this laptop.

## Windows pagefile preflight

Before the first retail-game test, Windows automatic pagefile management was disabled and a fixed 24,576 MB pagefile was configured on C: through the Virtual Memory UI. After restart, the user verified `C:\pagefile.sys` at 24,576 MB and Task Manager reported a 39.7 GB commit limit (6 GB committed immediately after restart; 9 GB after starting Firefox). This increases commit capacity but does not add physical RAM. See [the exact change and rollback instructions](system-changes/2026-11-24-pagefile-xcl-preflight.md).


## XCL — 120-second retail-game run with 512 MB coherent window (24 November 2026)

The user completed a 120-second run on the verified upstream build `beaa245`, using `XCL_COHERENT_MB=512`, `XCL_ACTIVITY=1`, and `XCL_NETWORK=0`. The log confirms that `XCL_RUN_SECONDS=120` ended the process intentionally. The report records `crash: null` and exit code `0`. The user observed the Rockstar logo and a narration screen before the timer ended.

Key results: 757 PresentX calls; 235 translated pipelines; no fallback/failed pipeline counters; no DXIL compilation failures; 40 mesh-pipeline translation refusals (MS container in AS slot); 90 Opus/XAPU packets decoded with zero decoder errors; 126 DirectStorage BCPack requests refused (BC1 6, BC5 100, BC7 20).

The coherent UMA window remained at 512 MB, with `uma_used_kb=119552` (about 116.75 MB). It was not close to full. The activity trace's process-private commit rose to a maximum of about 16,074 MB and then stayed around 16,029–16,074 MB through the final minute. DXGI reported local video-memory `CurrentUsage` peaking around 6,654 MB and non-local around 1,359 MB. XCL's activity log does not record the DXGI budget, CPU load, total system physical-memory availability, GPU temperature, CPU temperature, or clocks, so it cannot by itself establish thermal headroom or exact physical residency.

FPS: XCL logged 600 frames presented and 7.8 FPS over the last 300 at about 97.7 seconds; the total run recorded 757 presentations over 120 seconds (about 6.3 average across startup and loading). The user's observed on-screen rate was approximately 9.5 FPS. Keep these as different measurement windows rather than treating them as contradictory.

Interpretation: forcing 512 MB successfully avoided the previous 4,095 MB coherent-window saturation, and the title survived the full timed interval. This is the strongest local progress so far, but not proof of gameplay. The remaining mesh pipeline refusals, DirectStorage BC pack failures, 32-lane wave split on the RTX 3050 Ti, and memory pressure likely contribute to the low frame rate / incomplete rendering.

For the local video-memory numbers, XCL calls `IDXGIAdapter3::QueryVideoMemoryInfo` and logs `CurrentUsage` for local and non-local segments. Microsoft defines this as the application's current video-memory usage; the same structure's `Budget` field gives the OS-provided budget, and exceeding that budget can cause stuttering/performance penalties. Because the log does not record `Budget`, it is not possible to quantify the over-budget amount from this run.

Next diagnostic: a bounded 180-second repeat with the same 512 MB override and network isolation, while collecting live Windows physical-memory/commit, CPU utilization, NVIDIA GPU temperature/utilization/memory/power, and pagefile usage. Do not increase the coherent window, delete caches, change the pagefile, or run without a timeout based on this result alone. The 16 GB process-private commit is not the same as its resident working set, and the log has no temperatures.

See [the detailed 120-second diagnostic record](system-changes/2026-11-24-xcl-120s-512mb-run.md). Raw root-level `report.json` and `xcl.log` contain local Windows paths / profile identifiers; do not duplicate unredacted diagnostics in further public notes.

## Next steps / safe operating rules

1. **Latest local retail test: 120 seconds, normal XCL-managed exit, and the coherent window stayed at 512 MB / about 116.75 MB used.** The user saw the Rockstar logo and narration screen. See [the 120-second run record](system-changes/2026-11-24-xcl-120s-512mb-run.md).
2. The next reasonable test is a bounded 180-second repeat with `XCL_COHERENT_MB=512`, `XCL_ACTIVITY=1`, and `XCL_NETWORK=0`, while separately monitoring Windows physical memory/commit, pagefile, CPU load and NVIDIA GPU temperature/load/memory. Do not run indefinitely or increase the coherent window until the budget and thermal data are checked.
3. The local source is verified at upstream `beaa245`; the working tree was clean. XCL updates remain manual-only: use `Start-ScheduledTask -TaskName "XCL Auto Update"` when choosing to update. The previous one-minute trigger was removed.
4. The current run still reports mesh pipelines refused (MS shader container in AS slot) and 126 DirectStorage BCPack failures; gameplay/rendering is not yet verified.
5. Preserve logs for comparison, but keep public copies redacted. The raw root-level report/log include local Windows profile paths; do not duplicate these unredacted.
6. Before installing VIctory, review its build scripts and dependencies and use a completely separate project directory, such as `D:\\Development\\VIctory`. Do not attempt real-GPU world gameplay with VIctory until its upstream GPU-hang warning and laptop compatibility have been reassessed.
7. Keep game data and proprietary assets outside source repositories.

## Upstream references

- [XCL repository](https://github.com/ivall/xcl)
- [VIctory repository and README](https://github.com/Marvisak/VIctory)
- [VIctory architecture](https://github.com/Marvisak/VIctory/blob/main/docs/architecture.md)
- [VIctory asset policy](https://github.com/Marvisak/VIctory/blob/main/docs/asset-policy.md)
- [VIctory pinned Rust toolchain](https://github.com/Marvisak/VIctory/blob/main/rust-toolchain.toml)
