# Microsoft Intune Endpoint Management Lab

Hands-on Microsoft Intune lab demonstrating modern cloud endpoint management of a Windows 11 device — Entra ID join, MDM enrollment, device compliance, configuration profiles, and application deployment (Store **and** packaged Win32). The cloud-side counterpart to my [Windows Server Active Directory Lab](https://github.com/Amogh-Karankal/Windows-AD-Lab): together they show device management from both sides — GPO on-prem and MDM in the cloud.

![Microsoft Intune](https://img.shields.io/badge/Microsoft%20Intune-MDM-blue?logo=microsoft)
![Entra ID](https://img.shields.io/badge/Microsoft%20Entra%20ID-Device%20Join-green)
![Windows 11](https://img.shields.io/badge/Windows%2011-Managed-blue?logo=windows11)
![Win32](https://img.shields.io/badge/Win32-Content%20Prep%20Tool-orange)
![Status](https://img.shields.io/badge/Status-Complete-brightgreen)

## 🎯 Overview

This project takes a Windows 11 device from unmanaged to fully managed through Microsoft Intune, covering the three pillars of endpoint management — **compliance**, **configuration**, and **application deployment** — plus the real-world troubleshooting that comes with each. The lab was built end-to-end at zero cost using an Intune Plan 1 trial and free MDM enrollment.

**End state:** Windows 11 device that is Microsoft Entra joined, enrolled in Intune MDM, reporting **Compliant**, running a deployed configuration profile, and with applications installed via two deployment methods.

## 🧱 Environment

| Component | Detail |
|-----------|--------|
| Device | Windows 11 Pro (VirtualBox VM, TPM 2.0 + Secure Boot + EFI) |
| Tenant | Microsoft Entra ID + Microsoft Intune Plan 1 (trial) |
| Management user | `alex.helpdesk` — least-privilege, **not** Global Admin |
| Enrollment | Entra ID join + MDM enrollment (free — no Entra P1 required) |
| Device ownership | Corporate |

## 🔐 Identity & Enrollment

- Created a dedicated management user (`alex.helpdesk`) and assigned the **Microsoft Entra Joined Device Local Administrator** role — least privilege, rather than reaching for Global Admin.
- Built the Windows 11 VM with a proper **local administrator account created at OOBE** (`ms-cxh:localonly`) before the Entra join — so the device can never be locked out of management.
- Entra-joined the device as `alex.helpdesk`, then completed **MDM enrollment**. Verified the dual-admin end state (`localadmin` + `AzureAD\AlexHelpdesk`) with `Get-LocalGroupMember` and confirmed the join with `dsregcmd /status`.

## ✅ Pillar 1 — Device Compliance Policy

Authored a Windows compliance policy (`Win11 - Baseline Compliance`):

- Require **Secure Boot** and **code integrity**
- **Firewall**, **antivirus**, and **antispyware** required (Microsoft Defender)
- Minimum OS version baseline
- Password requirements (length, complexity)

Device evaluates and reports **Compliant** against the authored policy (verified server-side and on the device).

## ⚙️ Pillar 2 — Configuration Profile (Settings Catalog)

Created a **Settings Catalog** configuration profile:

- **Turn Off Windows Copilot** — applied and verified **Succeeded** via per-setting check-in status.

> **Troubleshooting note:** a Lock Screen Image (`LockScreenImageUrl`) setting was also attempted but returned **error 65000** — a CSP edition-gating limitation (that personalization CSP is gated to Windows Enterprise/Education, not Pro). Diagnosed from the per-setting check-in status and removed, keeping the working profile clean. *"Applied successfully" does not always mean "enforceable on this edition."*

## 📦 Pillar 3 — Application Deployment (two methods)

| App | Method | Detection | Result |
|-----|--------|-----------|--------|
| **VLC** | Microsoft Store app (new, winget-backed) | Automatic | Installed |
| **7-Zip 26.01 (x64)** | **Win32 MSI** (Content Prep Tool) | MSI product code | Installed |

**Win32 packaging walkthrough:**
1. Downloaded the 7-Zip **x64 MSI** (the MSI, not the EXE — enables clean automatic detection).
2. Wrapped it into an `.intunewin` package with the **Win32 Content Prep Tool** (`IntuneWinAppUtil.exe`).
3. Defined a **silent install** command: `msiexec /i "7z2601-x64.msi" /qn`
4. Defined the **uninstall** command via the MSI product code.
5. Configured **MSI product-code detection** (`{23170F69-40C1-2702-2601-000001000000}`).
6. Assigned as **Required → All devices**; install behavior **System**.

## 🛠️ Troubleshooting Log (the part that taught the most)

| Issue | Root cause | Resolution |
|-------|-----------|------------|
| **Local-admin enrollment lockout** | First VM was OOBE-bypassed with no local admin; the management user was a standard user and built-in Administrator was disabled | Diagnosed with `net localgroup administrators`. Rebuilt with a forced local account at OOBE + assigned the Entra Joined Device Local Administrator role + corrected the device-registration "registering user as local admin" setting |
| **Config profile error 65000** | CSP edition gating — Lock Screen Image not enforceable on Windows 11 Pro | Identified from per-setting check-in status; removed the setting, kept the working Copilot profile |
| **Win32 app stuck "Waiting for install status"** | Store apps install via the Store pipeline (instant); Win32 apps install via the **Intune Management Extension (IME)**, which deploys after enrollment and checks ~hourly | Confirmed IME running, read `IntuneManagementExtension.log`, triggered with a service restart / reboot |
| **Device showing wrong ownership** | All-devices grid caches; detail page is source of truth | Trusted the device Overview page over the cached list |

## 🧠 Key Concepts Demonstrated

- **Pull-based MDM model** — a managed device must check in for policies/apps to apply; offline = stale status, not failure.
- **Least privilege** — Entra Joined Device Local Administrator role instead of Global Admin.
- **Store vs Win32 deployment pipelines** — and why one installs instantly while the other waits on the IME.
- **Win32 packaging & detection** — Content Prep Tool, silent install, MSI product-code detection vs manual file/registry detection.
- **CSP edition gating** — recognizing error 65000 as a Windows-edition limitation, not a misconfiguration.
- **Compliance vs configuration** — one judges a device against a security bar (and can feed Conditional Access); the other pushes settings.

## 🗂️ Repository Structure

```
Intune-Endpoint-Management-Lab/
├── README.md
├── screenshots/          # compliance Compliant, config Succeeded, VLC + 7-Zip Installed, device Overview
├── policies/             # exported compliance policy + configuration profile (JSON)
└── win32-packaging/      # Content Prep Tool notes, install/uninstall/detection commands
```
## 🔧 Skills

`Microsoft Intune` · `MDM/MAM` · `Microsoft Entra ID` · `Device Compliance` · `Configuration Profiles (Settings Catalog)` · `Application Deployment` · `Win32 Packaging (Content Prep Tool)` · `Windows 11 Administration` · `Endpoint Troubleshooting`

---

*Part of a hands-on IT & cybersecurity portfolio. See also: [Windows Server Active Directory Lab](https://github.com/Amogh-Karankal/Windows-AD-Lab) and [Secure Azure AI Chatbot](https://github.com/Amogh-Karankal).*
