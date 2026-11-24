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

**No retail game was launched.** The successful build does not establish compatibility or playability for any game.

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

## XCL — automatic updater status

An updater script and Task Scheduler commands were drafted during the setup discussion, but there was **no confirmation** that `D:\Development\XCL\tools\Update-XCL.ps1` was created or that a scheduled task was registered. Treat automatic updating as **not configured / not verified**.

Preferred polling interval: **every 30 minutes**.

The intended updater design was:

- Fetch updates from the official XCL `main` branch.
- Skip updates if the working tree has local changes.
- Do not rebuild while `xcl.exe` is running.
- Fast-forward only; do not overwrite local edits.
- Build after updating and preserve the previous working executable for recovery if the new build fails.
- Log update/build outcomes inside the XCL project folder.

Offline behaviour: without internet, Git cannot fetch new commits. A properly written updater should log the failure and exit without changing the local checkout or executable. It will retry at the next scheduled run after connectivity returns; it does not continue downloading while offline. The machine must be awake for Task Scheduler to run. A 30-minute interval means a connection restored just after a poll may not be checked until the next poll.

Before enabling or relying on an updater, verify the script exists and inspect the scheduled task:

```powershell
Test-Path "D:\Development\XCL\tools\Update-XCL.ps1"
schtasks.exe /Query /TN "XCL Auto Update" /V /FO LIST
```

Do not assume the task exists unless those checks confirm it.

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

## Next steps / safe operating rules

1. Keep the established XCL installation intact; no uninstall or rollback was performed.
2. Do not assume XCL automatic updates are enabled. The intended interval remains 30 minutes, but script/task creation is unverified.
3. If resuming XCL test diagnosis, rerun the full test suite sequentially and compare with the earlier parallel result.
4. Before installing VIctory, review its build scripts and dependencies and use a completely separate project directory, such as `D:\Development\VIctory`.
5. Do not attempt real-GPU world gameplay with VIctory until the upstream GPU-hang warning and laptop compatibility have been reassessed.
6. Keep game data and other proprietary assets outside source repositories.

## Upstream references

- [XCL repository](https://github.com/ivall/xcl)
- [VIctory repository and README](https://github.com/Marvisak/VIctory)
- [VIctory architecture](https://github.com/Marvisak/VIctory/blob/main/docs/architecture.md)
- [VIctory asset policy](https://github.com/Marvisak/VIctory/blob/main/docs/asset-policy.md)
- [VIctory pinned Rust toolchain](https://github.com/Marvisak/VIctory/blob/main/rust-toolchain.toml)
