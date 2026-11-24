# XCL first retail-game run — 24 November 2026

## Scope and machine

This record documents the first actual launch of the user's retail title executable through the local XCL build. This is a compatibility diagnostic, not a claim of playable support.

- OS: Windows 11 Home Single Language, build 26300, x64
- CPU: Intel Core i7-12700H
- RAM: about 15.7 GB installed
- GPU: NVIDIA GeForce RTX 3050 Ti Laptop GPU; XCL detected 3962 MiB VRAM
- XCL checkout: `D:\Development\XCL\repository\xcl`
- XCL executable: `D:\Development\XCL\repository\xcl\target\release\xcl.exe`
- Title executable: `D:\Games\GTA 6\game_scarlett_master.exe`
- XCL version in report: `0.1.0`

## Preconditions

Before the run, Windows pagefile configuration was changed to a fixed 24,576 MB on C:. After restart, Task Manager showed a 39.7 GB commit limit. The pagefile was verified with `Win32_PageFileUsage`. See [the pagefile preflight record](2026-11-24-pagefile-xcl-preflight.md) for setup and rollback instructions.

The test used network isolation and XCL's internal timed-stop feature.

## Reproducible command

Run from PowerShell:

```powershell
$Xcl = "D:\Development\XCL\repository\xcl\target\release\xcl.exe"
$Game = "D:\Games\GTA 6\game_scarlett_master.exe"

if (!(Test-Path $Xcl)) { throw "XCL executable not found." }
if (!(Test-Path $Game)) { throw "Game executable not found." }

$env:XCL_NETWORK = "0"
$env:XCL_RUN_SECONDS = "60"
try {
    & $Xcl run $Game
}
finally {
    Remove-Item Env:XCL_NETWORK -ErrorAction SilentlyContinue
    Remove-Item Env:XCL_RUN_SECONDS -ErrorAction SilentlyContinue
}
```

## Observed outcome

**Successful controlled run:**

- XCL printed `XCL_RUN_SECONDS: 60 s – ending the title`.
- 336 frame presentations / `PresentX` calls were recorded.
- Process ended with code `0x0 (0)`.
- The generated report recorded `"crash": null` and `"exit_code": 0`.
- No forced external termination was needed.

**Graphics/shader activity:**

- D3D12 backend: `real`.
- Adapter: NVIDIA GeForce RTX 3050 Ti Laptop GPU (3962 MiB VRAM).
- Unique shaders: 8,741.
- Shader submissions: 228,474.
- Xbox shader errors: 0.
- Translated pipelines: 6.
- Fallback pipelines: 0.
- Failed pipelines: 0.
- DXIL cache: 12 misses/compilations, 1 hit, 0 compilation failures.
- Multiple mesh-pipeline translation errors were recorded: the mesh shader appeared in the amplification-shader (AS) slot, so those pipelines were refused. This is a significant rendering limitation and is consistent with the upstream XCL bring-up notes.
- One stubbed import: `xmem.dll!XMemIsToolsMemAvailable`.
- Multiple Xbox-specific D3D12 command allocator / graphics command list methods were recorded as unimplemented calls.

**Platform and network notes:**

- `System32\xgameruntime.dll` was absent. The report says dialogs, text entry, game invites and game streaming are unavailable in XCL's fallback mode.
- The console output showed Xbox Live and Rockstar service requests failing with `0x80072EFD` while `XCL_NETWORK=0` was set. Treat this as consistent with the deliberately isolated/offline run; it does not demonstrate successful online authentication.
- No successful audio-engine initialization or audio playback was recorded in the report.

## What this test does not prove

- It does not prove that the game's menus were visible or usable.
- It does not prove that gameplay or world rendering works.
- It does not prove audio, online services, saves, or other platform integrations work.
- It does not prove stability for a longer session.
- It does not prove that the game is safe to run indefinitely on this laptop.

The user has not yet described what was visibly shown on screen during the 60-second run. Record that observation before deciding whether to attempt a longer test.

## Logs and privacy

The user uploaded a raw report at the repository root as `report.json`. It contains a local Windows profile name in a shader-dump path. The console transcript shared in chat also included an Xbox account XUID in an authentication URL. Avoid publishing unredacted logs further; remove or redact personal identifiers and local profile paths before making diagnostic artifacts public. Do not copy game assets, dumps, shaders or proprietary files into the research repository.

The root `xcl.log` currently returns empty content through the GitHub connector. Use the local log as authoritative:

`%LOCALAPPDATA%\XCL\RockstarGamesInc.28872443D20A7\logs\xcl.log`

## Next decision

Ask what was visibly on screen during the run (for example, black screen, splash/title, account prompt, settings, or a rendered scene). Do not infer which one occurred from the report counters. Then compare the local log around the final menu/frame events before choosing another time-limited test.
