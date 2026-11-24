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

## Next steps / safe operating rules

1. **Pause retail-game testing on this local build.** Two attempts failed at command-list close with `E_OUTOFMEMORY`; do not extend the timeout or clear the caches to retry.
2. XCL updates are manual-only: use `Start-ScheduledTask -TaskName "XCL Auto Update"` when choosing to update. Check upstream for a relevant memory/D3D12 fix before rebuilding; a new version number alone is not enough.
3. Preserve the current logs/reports for comparison, but keep public copies redacted. The first-run diagnostic and repeated-crash records are under `local-xbox-compatibility/system-changes/`.
4. The initial run reached a pre-Rockstar-logo screen; neither menu navigation nor gameplay has been verified on this laptop.
5. If resuming XCL test diagnosis, rerun the full test suite sequentially and compare with the earlier parallel result.
6. Before installing VIctory, review its build scripts and dependencies and use a completely separate project directory, such as `D:\\Development\\VIctory`.
7. Do not attempt real-GPU world gameplay with VIctory until the upstream GPU-hang warning and laptop compatibility have been reassessed.
8. Keep game data and other proprietary assets outside source repositories.

## Upstream references

- [XCL repository](https://github.com/ivall/xcl)
- [VIctory repository and README](https://github.com/Marvisak/VIctory)
- [VIctory architecture](https://github.com/Marvisak/VIctory/blob/main/docs/architecture.md)
- [VIctory asset policy](https://github.com/Marvisak/VIctory/blob/main/docs/asset-policy.md)
- [VIctory pinned Rust toolchain](https://github.com/Marvisak/VIctory/blob/main/rust-toolchain.toml)
