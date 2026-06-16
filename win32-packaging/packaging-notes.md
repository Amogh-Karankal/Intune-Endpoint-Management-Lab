# Win32 App Packaging Notes — 7-Zip

How the 7-Zip Win32 application was packaged and deployed through Intune. This is the standard pattern for deploying desktop software that isn't available (or isn't ideal) through the Microsoft Store.

## Why Win32 (vs Store app)

| | Store app (new) | Win32 |
|---|---|---|
| Packaging | None — pick from catalog | Wrap installer into `.intunewin` |
| Install pipeline | Store/UWP — fast after enrollment | Intune Management Extension (IME) |
| Detection | Automatic | You define it (MSI product code / file / registry) |
| Best for | Apps in the Store catalog | Custom installers, MSIs/EXEs, full control |

VLC was deployed as a Store app (winget-backed) for a quick win. 7-Zip was deployed as a Win32 MSI to demonstrate the full packaging workflow — and because 7-Zip wasn't reliably available in the Store catalog.

## Prerequisites

Done on the **admin workstation (host)**, not the managed device — the device receives the app from Intune.

1. **7-Zip x64 MSI** from https://www.7-zip.org/download.html
   - Download the **.msi** (x64), not the .exe. The MSI enables clean automatic product-code detection; an EXE would force a manual file/registry detection rule.
2. **Microsoft Win32 Content Prep Tool** (`IntuneWinAppUtil.exe`)
   - https://github.com/microsoft/Microsoft-Win32-Content-Prep-Tool

Folder layout used:
```
IntuneWin32\
├── IntuneWinAppUtil.exe
└── source\
    └── 7z2601-x64.msi      # only the installer in the source folder
```

## Step 1 — Wrap the MSI into .intunewin

```
cd /d "C:\path\to\IntuneWin32"
IntuneWinAppUtil.exe
```
Interactive prompts:
- Source folder: `source`
- Setup file: `7z2601-x64.msi`
- Output folder: `.` (current folder)
- Catalog folder (Y/N): `N`

Produces `7z2601-x64.intunewin`.

> Note: the source folder should contain **only** the installer. If both an `.exe` and `.msi` are present, the tool may wrap the wrong one.

## Step 2 — Create the Win32 app in Intune

**Apps → All apps → Add → Windows app (Win32)** → upload `7z2601-x64.intunewin`.

**App information**
- Name: `7-Zip 26.01 (x64)`
- Publisher: `Igor Pavlov`
- App version auto-populated from the MSI (`26.01.00.0`)

**Program**
| Field | Value |
|-------|-------|
| Install command | `msiexec /i "7z2601-x64.msi" /qn` |
| Uninstall command | `msiexec /x "{23170F69-40C1-2702-2601-000001000000}" /qn` (auto-filled from MSI product code) |
| Install behavior | **System** |
| Device restart behavior | Determine behavior based on return codes |

Return codes (standard MSI, pre-filled): `0` Success, `1707` Success, `3010` Soft reboot, `1641` Hard reboot, `1618` Retry.

**Requirements**
- OS architecture: **x64**
- Minimum OS: lowest Windows 11 version (anything at/below the target build)

**Detection rules**
- Rules format: **Manually configure detection rules** → Add
- Rule type: **MSI**
- MSI product code: auto-populated → `{23170F69-40C1-2702-2601-000001000000}`
- Product version check: No

**Assignments**
- **Required → All devices**

## Step 3 — Sync & verify

On the device (must be powered on and online):
- Settings → **Access work or school → Info → Sync**
- Win32 installs run via the **Intune Management Extension (IME)**, which checks periodically and on startup — a Win32 app can sit at "Waiting for install status" until the IME runs. A reboot or restart of the *Microsoft Intune Management Extension* service forces an immediate cycle.

Verify:
- **On device:** 7-Zip appears in Start / Installed apps.
- **Server-side:** Apps → 7-Zip 26.01 → **Device install status** → device shows **Installed**.
- **IME log (definitive):** `C:\ProgramData\Microsoft\IntuneManagementExtension\Logs\IntuneManagementExtension.log` — search for `7z`; exit code `0` = success.

## Detection method summary

| Installer | Detection method | Notes |
|-----------|------------------|-------|
| MSI | **MSI product code** (automatic) | Cleanest — used here |
| EXE | Manual: file exists (`C:\Program Files\7-Zip\7zFM.exe`) or registry key (`HKLM\SOFTWARE\7-Zip\Path64`) | Required when no MSI is available; install via `7zXXXX-x64.exe /S` |
