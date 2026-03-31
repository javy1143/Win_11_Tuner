# StackPoint IT — Windows 11 Tuner
### Version 4.3

A WinForms-based Windows 11 optimization and maintenance tool built for MSP deployment. Applies privacy hardening, performance tuning, debloat, scheduled maintenance, and app uninstallation — all with full backup/restore support and a detailed session log.

---

## Requirements

| Requirement | Detail |
|---|---|
| OS | Windows 10 / 11 (64-bit) |
| PowerShell | 5.1 or later |
| Privileges | Administrator (auto-elevates via RunAs if not already elevated) |
| Assemblies | `System.Windows.Forms`, `System.Drawing` (included with Windows) |

---

## Security Notes

- **Domain-joined and Intune-enrolled machines**: The *Disable VBS / Memory Integrity* tweak is automatically blocked on these endpoints by design. Disabling VBS on managed machines violates typical baseline security policy. The tool detects enrollment via `HKLM:\SOFTWARE\Microsoft\Enrollments` and domain membership via `Win32_ComputerSystem.PartOfDomain`.
- **Backup manifests** are written to `%ProgramData%\StackPointTuner\Backups\` as JSON before any changes are applied. Every registry value, service startup type, scheduled task state, and optional feature state is captured prior to modification.
- **High-risk tweaks** (VBS disable, service optimization) require an explicit confirmation dialog before proceeding.
- **DISM StartComponentCleanup** (`/ResetBase`) triggers a second, separate destructive-action confirmation. This operation is irreversible and permanently removes OS rollback capability.
- **No network calls** are made by the tool itself. All operations are local.

---

## Launch

```powershell
# Standard launch — will auto-elevate if needed
powershell.exe -ExecutionPolicy Bypass -File "Win11_Tuner.ps1"
```

If the script is blocked by a Zone.Identifier ADS (downloaded from the internet):

```powershell
Unblock-File -Path ".\Win11_Tuner.ps1"
```

---

## Tabs Overview

### Privacy
Hardens telemetry, advertising, and activity-tracking settings via Group Policy registry keys.

| Tweak | Risk | Notes |
|---|---|---|
| Disable Activity History | Low | Blocks EnableActivityFeed, PublishUserActivities, UploadUserActivities |
| Disable Game DVR | Low | Disables Xbox recording pipeline |
| Reduce Background Apps | Medium | Disables consumer Store apps; safelists M365, Teams, OneDrive, RingCentral, Adobe |
| Disable Storage Sense | Low | Per-user setting |
| Disable Ads, Spotlight, Tips | Low | Advertising ID, Spotlight lock screen, third-party suggestions, Start tracking |
| Minimize Optional Diagnostics | Medium | Sets telemetry to Required (level 1); disables inking/typing personalization, online speech, SIUF feedback |

### Performance
Optimizes CPU scheduling, services, network stack, and removes background noise.

| Tweak | Risk | Reboot | Notes |
|---|---|---|---|
| Optimize UI Visual Effects | Low | No | Sets VisualFXSetting=2, disables transparency |
| High Performance (AC) / Battery-Aware (DC) | Low | No | AC: 100% CPU. DC: 5–80% CPU. Windows switches automatically — no task required |
| Disable Telemetry Tasks | Low | No | Disables 12 CEIP/telemetry/compatibility scheduled tasks |
| Delete Temporary Files | Low | No | Cleans `C:\Windows\Temp` and `%TEMP%`; pauses/resumes OneDrive |
| Optimize Services | High | No | Adjusts startup types for non-essential OS services; explicitly protects wuauserv, RingCentral, Adobe, OneDrive, Teams |
| Network Optimizations | Low | No | Sets TcpNoDelay on all interfaces; flushes DNS |
| Disable VBS / Memory Integrity | High | Yes | Blocked on domain/Intune machines. Not in Recommended preset |
| Disable SysMain / Superfetch | Low | No | Recommended for SSD-equipped machines; warns if no SSD is detected |
| Limit Delivery Optimization | Low | No | HTTP-only mode (DODownloadMode=0); stops background P2P seeding |
| Reduce Search Indexer Scope | Medium | No | Disables Cortana/cloud search, web results, Outlook indexing; throttles I/O |
| Disable Copilot, AI Tasks, Widgets | Low | Yes | Disables Copilot sidebar, Widgets/News, taskbar Chat, Windows Recall tasks |
| Disable Hibernate + Fast Startup | Low | Yes | Runs `powercfg /h off`; reclaims 8–16 GB on 16 GB RAM machines; forces full cold boot (security benefit) |

### Debloat

| Tweak | Risk | Notes |
|---|---|---|
| Debloat Microsoft Edge | Low | Disables shopping assistant, crypto wallet, first-run, Spotlight, personalization reporting |

### Maintenance
Point-in-time maintenance operations. Most are non-reversible by nature.

| Tweak | Notes |
|---|---|
| DISM: Restore Health | `DISM /Online /Cleanup-Image /RestoreHealth` — takes 10–30+ min |
| SFC: System File Checker | `sfc /scannow` — takes 10–30+ min |
| Clear Teams Cache | Stops classic + new Teams; clears all cache folders including Service Worker |
| Clear Outlook Temp Attachments | Reads `OutlookSecureTempFolder` from registry for all detected Office versions |
| Clear Edge Cache | Stops Edge; clears cache, GPUCache, IndexedDB for all profiles |
| Clear Adobe Acrobat Cache | Supports DC, 2020, 2017, 2015, 11.0 versioned paths |
| Clear RingCentral Cache | Covers both RingCentral Phone and Meetings |
| OneDrive Sync Health Diagnostic | Read-only: reports process state, account, folder path, last sync timestamp |
| DISM: Post-Upgrade Cleanup (**IRREVERSIBLE**) | `/StartComponentCleanup /ResetBase` — frees 2–8 GB; permanently removes rollback capability. Requires dedicated confirmation |
| Intel/Dell Driver Diagnostic | Read-only: reports model, BIOS, Intel GPU driver version, PnP errors |

### Uninstaller
A Control Panel-style app removal interface with deep-clean support.

- Scans three registry hives: `HKLM\SOFTWARE\...\Uninstall`, `HKLM\SOFTWARE\WOW6432Node\...\Uninstall`, `HKCU\SOFTWARE\...\Uninstall`
- Excludes system components, Visual C++ runtimes, .NET runtimes, drivers, KB updates, and OEM tools
- Automatically classifies apps as **Silent** (MSI or explicit QuietUninstallString) or **Interactive** (requires user confirmation in the uninstaller UI)
- Silent apps are batched and run without prompts; interactive apps run first, one at a time
- **Deep Clean** (optional, enabled by default): after a successful uninstall, scans `%APPDATA%`, `%LOCALAPPDATA%`, `%ProgramData%`, `%ProgramFiles%`, `%ProgramFiles(x86)%`, and registry under `HKLM\SOFTWARE` / `HKCU\SOFTWARE` for leftover folders and keys matching the app name or publisher

---

## Backup & Restore

Before any tweak is applied, the tool captures the prior state of every affected registry value, service, scheduled task, optional feature, and file into a **backup context**. At the end of a run, the full context is serialized to:

```
%ProgramData%\StackPointTuner\Backups\backup_YYYYMMDD_HHmmss.json
```

A pointer to the most recent manifest is written to:

```
%ProgramData%\StackPointTuner\last_backup.json
```

Use the **Undo Last Run** button to restore all values captured in the most recent manifest. Individual manifests can also be selected via the file dialog.

---

## Logging

Every operation is written to a timestamped session log:

```
%ProgramData%\StackPointTuner\Logs\run_YYYYMMDD_HHmmss.log
```

The log path is displayed in the footer of the main window. Use **Export Log** to save the output pane contents to a `.txt` file of your choice.

---

## Recommended Preset

The **Recommended** button selects tweaks meeting all of the following criteria:

- Risk level is not `High`
- Category is not `Maintenance`, OR the tweak is a low-risk, non-long-running maintenance operation

This excludes: VBS disable, service optimization, DISM RestoreHealth, SFC scan, DISM cleanup, OneDrive diagnostic, and driver diagnostic.

---

## Data Paths

| Path | Purpose |
|---|---|
| `%ProgramData%\StackPointTuner\Backups\` | JSON backup manifests |
| `%ProgramData%\StackPointTuner\Logs\` | Session logs |
| `%ProgramData%\StackPointTuner\last_backup.json` | Pointer to most recent manifest |

---

## Changelog

### v4.3
- `perf_powerplan`: High Performance plan now sets **AC = 100% / DC = 5–80%** processor state instead of a flat 100% on both AC and DC. Windows automatically applies the correct profile based on power source — no scheduled task or trigger required. Prevents CPU running at full speed on battery when user unplugs.

### v4.2
- Added Uninstaller tab with silent/interactive sequencing, MSI quiet-flag injection, and deep-clean post-uninstall scan
- Applied StackPoint IT color palette throughout (`#18191b` background, `#3f4347` surfaces)

### v4.1
- Added SysMain (Superfetch) disable tweak for SSD machines
- Added Delivery Optimization P2P restriction
- Added Windows Search indexer scope reduction
- Added Copilot / AI tasks / Widgets disable (25H2-specific)
- Added Hibernate + Fast Startup disable (reclaims 8–16 GB)
- Added DISM StartComponentCleanup maintenance tweak
- Added Intel/Dell driver diagnostic (read-only)
- Added domain-join advisory for VBS/HVCI to startup log

### v4.0
- Initial release under Win11_Tuner name (formerly SwiftTune)
- WinForms GUI with tabbed layout, backup/restore engine, apply timer, cancel support
- Privacy, Performance, Debloat, and Maintenance tweak categories

---

## License

Internal use — StackPoint IT. Not for redistribution.
