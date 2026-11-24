# XCL updated to current upstream main — verification record

**Date:** 24 November 2026
**Scope:** Local Windows XCL installation only. No game files, raw diagnostics, account identifiers or credentials are included.

## 1. Local source checkout

Repository: `D:\Development\XCL\repository\xcl`
Branch: `main`

PowerShell checks:
```powershell
$Repo = "D:\Development\XCL\repository\xcl"
$Xcl = Join-Path $Repo "target\release\xcl.exe"
git -C $Repo log -1 --format="%H%n%cs%n%s"
git -C $Repo status --short --branch
Get-Item $Xcl | Format-List FullName, Length, LastWriteTime
& $Xcl --version
```

Observed results:
- Commit: `beaa245ca04e212f064efc7cafcdce104bc5c0a2`.
- Date: `2026-11-24`.
- Subject: `Merge pull request #797 from gsucristo/d3d12x/wait64-midlist-split`.
- Working-tree status: `## main...origin/main`; no modified/untracked files were shown.
- Executable path: `D:\Development\XCL\repository\xcl\target\release\xcl.exe`.
- File length: 6,873,600 bytes.
- Windows LastWriteTime: `24-Nov-26 11:41:00`.
- Version output: `xcl 0.1.0`.

The earlier executable size recorded at the initial build was 6,832,640 bytes. The current size and timestamp are consistent with the executable having been rebuilt during the update. The generic `--version` string does not encode the Git SHA, so it cannot independently prove binary-to-source provenance.

## 2. Upstream changes at this revision

The verified `main` commit is the merge of PR #797, which splits title command lists around a mid-list Wait64 after Write64. Other recent merged work in the newer upstream history includes:
- PR #792: UMA memory-window addressing / out-of-range access corrections.
- PR #795: GPU fence synchronization improvements.
- PR #797: command-list splitting around mid-list waits.
- PR #798: barrier handling for split-wave64 execution on wave32 GPUs.

These changes are not a verified fix for this laptop's previous `E_OUTOFMEMORY` failure or for the game's mesh-pipeline limitation.

## 3. Memory diagnostics available in the updated source

At commit `beaa245`, `crates/xcl/src/gpu/uma.rs` supports `XCL_COHERENT_MB=512` as an explicit per-process coherent-window override. It takes precedence over the saved adaptive hint for that run; it does not require deleting or editing `cache/uma_coherent.txt`.

`XCL_ACTIVITY=1` enables periodic activity diagnostics, including process commit, local/non-local video memory, target pages, arenas and sparse pools.

Also retain:
- `XCL_NETWORK=0`: required for the retail test because upstream notes the crash reporter may otherwise attempt to send a minidump externally.
- `XCL_RUN_SECONDS=60`: use XCL-managed shutdown after GPU work is drained. Do not externally kill a process with GPU work in flight.

Source links:
- [UMA implementation at beaa245](https://github.com/ivall/xcl/blob/beaa245ca04e212f064efc7cafcdce104bc5c0a2/crates/xcl/src/gpu/uma.rs)
- [Host activity and timed shutdown at beaa245](https://github.com/ivall/xcl/blob/beaa245ca04e212f064efc7cafcdce104bc5c0a2/crates/xcl/src/host.rs)
- [Upstream merge commit](https://github.com/ivall/xcl/commit/beaa245ca04e212f064efc7cafcdce104bc5c0a2)

## 4. Proposed next diagnostic — not yet executed

Run the game for 60 seconds with the smaller coherent window forced, while leaving the saved cache untouched:

```powershell
$Xcl = "D:\Development\XCL\repository\xcl\target\release\xcl.exe"
$Game = "D:\Games\GTA 6\game_scarlett_master.exe"

if (!(Test-Path $Xcl)) { throw "XCL executable not found." }
if (!(Test-Path $Game)) { throw "Game executable not found." }

$names = "XCL_NETWORK", "XCL_RUN_SECONDS", "XCL_COHERENT_MB", "XCL_ACTIVITY"
$previous = @{}
foreach ($name in $names) {
    $previous[$name] = [Environment]::GetEnvironmentVariable($name, "Process")
}

try {
    $env:XCL_NETWORK = "0"
    $env:XCL_RUN_SECONDS = "60"
    $env:XCL_COHERENT_MB = "512"
    $env:XCL_ACTIVITY = "1"
    & $Xcl run $Game
}
finally {
    foreach ($name in $names) {
        [Environment]::SetEnvironmentVariable($name, $previous[$name], "Process")
    }
}
```

This is a diagnostic comparison, not an established fix and not a claim that 512 MB is a suitable permanent configuration. It keeps the adaptive cache file intact and restores previous process-level values afterwards.

After the run, record the last visible screen, exit code, activity diagnostics and new report/log paths. Do not delete the shader cache, delete `uma_coherent.txt`, or make further pagefile changes before this comparison. Do not infer physical VRAM exhaustion from coherent-window accounting alone.
