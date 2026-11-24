# Windows pagefile change for XCL — 24 November 2026

## Purpose

Increase Windows' commit limit before a cautious XCL test of the retail title executable. This change raises the amount of memory Windows can promise to processes; it does **not** add physical RAM or GPU VRAM.

## Machine context

- Windows 11 Home Single Language, build 26300, x64
- Installed RAM: 16 GB (Windows reported about 15.7 GB)
- CPU: Intel Core i7-12700H
- GPU: NVIDIA RTX 3050 Ti Laptop GPU, about 4 GB reported VRAM
- C: had about 60 GB free before this change
- XCL/game test is experimental; no retail game has been launched on this laptop yet

## Before the change

The user reported:

- Task Manager committed memory: 15.1 / 27.2 GB; after closing Firefox, 10.3 / 27.2 GB
- `Get-CimInstance Win32_ComputerSystem | Select-Object AutomaticManagedPagefile` returned `True`
- `Get-CimInstance Win32_PageFileSetting` returned no rows
- `Win32_PageFileUsage` showed `C:\pagefile.sys` allocated at 11,776 MB, current usage 887 MB, peak usage 3,982 MB

The absence of a `Win32_PageFileSetting` row did not mean there was no pagefile; the usage query showed the active system-managed pagefile.

## Change made through Windows UI

1. Opened `sysdm.cpl`.
2. Advanced → Performance / Settings → Advanced → Virtual memory / Change.
3. Disabled **Automatically manage paging file size for all drives**.
4. Selected `C:`.
5. Chose **Custom size**.
6. Set Initial size to `24576` MB and Maximum size to `24576` MB.
7. Applied the change and restarted Windows.

## Verification after restart

The user reported:

- Task Manager committed memory immediately after restart: **6 / 39.7 GB**
- After starting Firefox: **9 / 39.7 GB**
- `Get-CimInstance Win32_PageFileUsage | Format-Table Name, AllocatedBaseSize, CurrentUsage, PeakUsage -AutoSize` returned:

```text
Name            AllocatedBaseSize CurrentUsage PeakUsage
C:\pagefile.sys             24576            0         0
```

This confirms that Windows allocated a 24,576 MB pagefile on C: after restart. The reported commit limit increased from 27.2 GB to 39.7 GB.

The 0 MB current/peak values were the values observed at that time; they do not mean the pagefile is absent.

## Why this does not guarantee the game will run

- A pagefile increases commit capacity, not physical RAM. This laptop still has about 16 GB installed RAM.
- Disk-backed paging can be much slower than RAM and can lead to severe stuttering.
- The XCL developer's notes for this particular retail executable reported about 18–19 GB of process commitment at the menu on the developer's test system. That is a reference observation, not a measured result on this laptop.
- The laptop's RTX 3050 Ti has about 4 GB VRAM, and the developer's test GPU was different. Compatibility and stability are not established.
- Keep the first test short and use XCL's own timed-shutdown feature; do not forcibly kill the process while GPU work may be in flight.

## Rollback

To restore Windows-managed pagefile sizing:

1. Open `sysdm.cpl`.
2. Advanced → Performance / Settings → Advanced → Virtual memory / Change.
3. Re-enable **Automatically manage paging file size for all drives**.
4. Apply and restart Windows.

After rollback, verify with:

```powershell
Get-CimInstance Win32_PageFileUsage |
    Format-Table Name, AllocatedBaseSize, CurrentUsage, PeakUsage -AutoSize
```

## Next XCL test — not yet run

The planned probe uses the local built executable and the user's game executable:

- XCL: `D:\Development\XCL\repository\xcl\target\release\xcl.exe`
- Game EXE: `D:\Games\GTA 6\game_scarlett_master.exe`

Run only with network access disabled for the title and a 60-second in-process timeout:

```powershell
$Xcl = "D:\Development\XCL\repository\xcl\target\release\xcl.exe"
$env:XCL_NETWORK = "0"
$env:XCL_RUN_SECONDS = "60"
try {
    & $Xcl run "D:\Games\GTA 6\game_scarlett_master.exe"
}
finally {
    Remove-Item Env:XCL_NETWORK -ErrorAction SilentlyContinue
    Remove-Item Env:XCL_RUN_SECONDS -ErrorAction SilentlyContinue
}
```

Do not mark the test complete until the user actually runs it and its result/logs are reviewed.
