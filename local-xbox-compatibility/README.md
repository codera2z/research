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


## XCL — current local source/build status (24 November 2026)

**Correction to the earlier build record:** the user has confirmed that the local XCL checkout and release executable are now updated to the current upstream `main`, with the changes already merged into upstream included. Do not describe the local build as `beaa245`, eight commits behind, or missing the BCPack/allocator merges. `beaa245` is a historical checkpoint only.

At the time of this update, upstream `main` resolves to [`47efa2ecea0a12934719ae69b81930cf02a4b1c3`](https://github.com/ivall/xcl/commit/47efa2ecea0a12934719ae69b81930cf02a4b1c3). A GitHub comparison showed this main revision is 48 commits ahead of the old `beaa245` checkpoint and has no divergence from it. The user confirms the local checkout has caught up with current main. The exact local HEAD SHA was not extracted from the uploaded logs; the user has confirmed the checkout is aligned with current upstream `main`. Do not infer the local SHA from `xcl --version`, which only reports `0.1.0`.

The current-main merged work includes, among other changes:
- PR #905: BCPack BC1/BC3/BC4/BC5/BC6H/BC7 decoding.
- PRs #1009, #1012 and #1023: D3D12 allocator reset/release/recording fixes.
- PR #951: query tiled-resource support before reserving affected depth textures.
- PR #990: correct GFX10 image format 134.
- PR #1016: refuse removal of non-empty virtual directories.

The current reports confirm that the earlier BCPack refusal is no longer present in the 120-second 512 MB run: 126 sampled chunks decoded with zero failures. The 40 mesh-pipeline refusals remain. The relevant proposed mesh work is still open in [#304](https://github.com/ivall/xcl/pull/304) and [#541](https://github.com/ivall/xcl/pull/541); neither is part of current `main`.

### Current-main run comparison

| Run | Coherent window | Outcome | Performance / diagnostic |
|---|---:|---|---|
| 120 s, current main, forced 512 MB | 512 MB | Timed exit 0; no gameplay | 669 presents / 120.103 s (~5.58/s); rolling sample 6.3/s |
| 207 s, current main, forced 512 MB | 512 MB | Manual Ctrl+C on a black screen after terms/menu flow | ~1243 presents (~6.0/s overall); rolling samples 5.7–6.9/s |
| Latest adaptive run, current main | 4095 MB | D3D12 `Close` returned `E_OUTOFMEMORY` at 30.514 s; process exited `0x80000003` at 31.859 s | 481 presents; 60 presents/2 s at 24 s, 51/2 s at 26 s, then zero in the next two activity windows |

The detailed memory counters, telemetry attribution and upstream comparison are kept in the individual run notes linked below. The 30 FPS interval in the adaptive run was brief and ended before the crash; it is not a stable 30 FPS benchmark. **Do not repeat the adaptive 4095 MB launch just to reproduce the failure.**

The binary's `xcl 0.1.0` version string does not encode a Git SHA. The official 
`scripts/smoke.ps1` harness normally prints the source commit and summary, but the latest uploaded adaptive-run log/report ended in the crash before a completed 120-second result was available. The exact local HEAD SHA is therefore still not established by these uploads.

No source patches, PR cherry-picks, game-file changes, executable modifications or pagefile changes were made for these tests.

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

See [the sanitized first-run record](system-changes/2026-11-24-first-retail-run.md). The raw root uploads contain local Windows profile paths and stack/thread details. They are separate sessions: `xcl.log` + `report.json` are the current-main adaptive 4095 MB crash; `xcl.prev.log` is the 207-second 512 MB run; `report.prev.json` is stale/incomplete for that 207-second run. The older pair of pre-current-main OOM crashes is recorded separately in [the historical crash note](system-changes/2026-11-24-repeated-crashes.md). Do not copy raw identifiers or paths into public summaries.

## XCL — repeated startup/rendering failures (24 November 2026)

The two repeated crashes in [the historical failure note](system-changes/2026-11-24-repeated-crashes.md) occurred on the earlier `f4ce27c` build, before the user updated the local checkout to current upstream `main`. They are useful as historical evidence but are not the current build result. The current-main adaptive 4095 MB crash is documented separately in [the latest run note](system-changes/2026-11-24-xcl-adaptive-4095mb-crash.md).

## XCL — first run versus later memory-window behaviour (24 November 2026)

The first launch used a 512 MB coherent window; the saved hint later caused adaptive launches to select 4,095 MB. The observed `uma_coherent.txt` value was 4,582.2 MB of cumulative allocation requests (not peak live or physical VRAM), and XCL's deliberate ~96 MB reserve explains the ~3,998.6 MB use counter. The same near-cap 4,095 MB path is present in the latest current-main crash. Full allocator mechanics and the earlier run comparison are kept in [the dedicated memory-window note](system-changes/2026-11-24-first-vs-later-memory-windows.md); the new crash's readings are in [its own note](system-changes/2026-11-24-xcl-adaptive-4095mb-crash.md). Do not equate XCL's UMA counters with NVIDIA dedicated VRAM.

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


## Detailed current-main run records

The concise comparison table above is the index; detailed counters are kept once in the linked notes rather than repeated in this README.

- [120-second current-main run, forced 512 MB](system-changes/2026-11-24-xcl-120s-512mb-run.md): timed exit 0, ~5.58 presents/s across the session, no gameplay, BCPack success, 40 mesh-pipeline refusals.
- [207-second current-main run, forced 512 MB](system-changes/2026-11-24-xcl-207s-black-screen-run.md): manually stopped on a black screen; ~6.0 presents/s overall; the timestamp-matched CSV supplies the GPU telemetry.
- [Current-main adaptive 4095 MB crash](system-changes/2026-11-24-xcl-adaptive-4095mb-crash.md): command-list-close `E_OUTOFMEMORY` at 30.514 s; process exit `0x80000003` at 31.859 s.
- [Earlier out-of-memory pair](system-changes/2026-11-24-repeated-crashes.md): historical pre-current-main `f4ce27c` only.
- [Adaptive coherent-window mechanics](system-changes/2026-11-24-first-vs-later-memory-windows.md): cache hint, 17 GiB free-commit guard and 96 MiB reserve.

## Next steps / safe operating rules

1. **Pause further adaptive 4095 MB launches** on this machine until relevant upstream memory/resource changes are merged or the cause is better isolated; the latest attempt crashed at ~32 s.
2. The 512 MB override lasted longer but delivered only ~5–7 presents/s and later went black; do not use those figures as gameplay-FPS measurements.
3. Keep `XCL_NETWORK=0` for any future controlled test. Do not add local patches, cherry-picks, shader-cache deletions, game-file edits, executable modifications, cache edits or pagefile changes as speculative fixes.
4. Keep current-main merged fixes separate from open upstream work. In particular, mesh pass-through and AS-slot fixes #304/#541 are still open, and are not included in the user-confirmed current-main checkout.
5. Raw root-level diagnostic uploads may contain local paths/profile identifiers; use sanitized summaries in long-term research notes.

## Upstream references

- [XCL repository](https://github.com/ivall/xcl)
- [XCL #284 — coherent-window size and performance reports](https://github.com/ivall/xcl/issues/284)
- [XCL #345 — E_OUTOFMEMORY on a 6 GB NVIDIA GPU](https://github.com/ivall/xcl/issues/345)
- [XCL PR #304 — DXC mesh pass-through vertex-chunk budget](https://github.com/ivall/xcl/pull/304)
- [XCL PR #541 — console MS carried in the AS subobject](https://github.com/ivall/xcl/pull/541)
- [VIctory repository and README](https://github.com/Marvisak/VIctory)
- [VIctory architecture](https://github.com/Marvisak/VIctory/blob/main/docs/architecture.md)
- [VIctory asset policy](https://github.com/Marvisak/VIctory/blob/main/docs/asset-policy.md)
- [VIctory pinned Rust toolchain](https://github.com/Marvisak/VIctory/blob/main/rust-toolchain.toml)
