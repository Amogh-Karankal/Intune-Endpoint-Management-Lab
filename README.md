# Microsoft Intune Endpoint Management Lab

Hands-on Microsoft Intune lab demonstrating modern cloud endpoint management across **two platforms — Windows 11 and Android** — in a single Entra tenant: Entra ID join, MDM/work-profile enrollment, device compliance, configuration profiles, and application deployment (Windows: Store **and** packaged Win32; Android: Managed Google Play). The cloud-side counterpart to my [Windows Server Active Directory Lab](https://github.com/Amogh-Karankal/Windows-AD-Lab): together they show device management from every side — GPO on-prem, MDM in the cloud, across Windows and mobile.

![Microsoft Intune](https://img.shields.io/badge/Microsoft%20Intune-MDM-blue?logo=microsoft)
![Entra ID](https://img.shields.io/badge/Microsoft%20Entra%20ID-Device%20Join-green)
![Windows 11](https://img.shields.io/badge/Windows%2011-Managed-blue?logo=windows11)
![Android Enterprise](https://img.shields.io/badge/Android%20Enterprise-Work%20Profile-green?logo=android)
![Win32](https://img.shields.io/badge/Win32-Content%20Prep%20Tool-orange)
![Status](https://img.shields.io/badge/Status-Complete-brightgreen)

## 🎯 Overview

This project takes endpoints from unmanaged to fully managed through Microsoft Intune, covering the three pillars of endpoint management — **compliance**, **configuration**, and **application deployment** — on **both Windows 11 and Android**, plus the real-world troubleshooting that comes with each. The lab was built end-to-end at zero cost using an Intune Plan 1 trial, free MDM enrollment, and a free Managed Google Play binding.

**End state:**
- A **Windows 11** device that is Microsoft Entra joined, enrolled in Intune MDM, reporting **Compliant**, running a deployed configuration profile, with applications installed via two deployment methods.
- An **Android** device enrolled as a personally-owned **work profile**, reporting **Compliant** against an authored policy, enforcing a work-profile restriction, with an application auto-deployed from Managed Google Play.

Most entry-level endpoint portfolios are Windows-only; this lab demonstrates **cross-platform** management — the same three capabilities on the two platforms real environments actually run.

## 🧱 Environment

| Component        | Windows                                                      | Android                                                              |
| ---------------- | ----------------------------------------------------------- | ------------------------------------------------------------------- |
| Device           | Windows 11 Pro (VirtualBox VM, TPM 2.0 + Secure Boot + EFI) | Xiaomi Redmi Note 10S (M2101K7BI), Android 13 / MIUI                |
| Tenant           | Microsoft Entra ID + Microsoft Intune Plan 1 (trial)        | Same tenant + Managed Google Play binding                           |
| Management user  | `alex.helpdesk` — least-privilege, **not** Global Admin     | `alex.helpdesk` (same licensed user)                                |
| Enrollment       | Entra ID join + MDM enrollment (free — no Entra P1 required) | Android Enterprise — personally-owned work profile (BYOD, no wipe)  |
| Device ownership | Corporate                                                   | Personal                                                            |

## 🔐 Identity & Enrollment

- Created a dedicated management user (`alex.helpdesk`) and assigned the **Microsoft Entra Joined Device Local Administrator** role — least privilege, rather than reaching for Global Admin.
- Built the Windows 11 VM with a proper **local administrator account created at OOBE** (`ms-cxh:localonly`) before the Entra join — so the device can never be locked out of management.
- Entra-joined the device as `alex.helpdesk`, then completed **MDM enrollment**. Verified the dual-admin end state (`localadmin` + `AzureAD\AlexHelpdesk`) with `Get-LocalGroupMember` and confirmed the join with `dsregcmd /status`.
- Bound the tenant to **Managed Google Play** with a dedicated Google account (one-time config that unlocks all Android Enterprise enrollment) and enrolled the Android device via the **Company Portal → work profile** flow, creating an isolated, Intune-managed work container alongside the untouched personal profile.

## ✅ Pillar 1 — Device Compliance Policy

**Windows** — `Win11 - Baseline Compliance`:
- Require **Secure Boot** and **code integrity**
- **Firewall**, **antivirus**, and **antispyware** required (Microsoft Defender)
- Minimum OS version baseline
- Password requirements (length, complexity)

**Android** — `Android WP - Baseline Compliance` (parallel baseline, all settings evaluated **Compliant** per-setting):
- Block **rooted** devices · minimum OS version · **Play Integrity** basic-integrity attestation
- Require **password to unlock** · password complexity (Android 12+) · **encryption**
- Block **apps from unknown sources** · block **USB debugging** · Company Portal runtime integrity
- Require Google Play Services configured · up-to-date security provider
- **Action for noncompliance:** mark noncompliant after a **1-day grace period** (vs. immediate)

Both devices evaluate and report **Compliant** against their authored policies (verified server-side and on-device).

## ⚙️ Pillar 2 — Configuration Profile

**Windows** — **Settings Catalog** profile:
- **Turn Off Windows Copilot** — applied and verified **Succeeded** via per-setting check-in status.

> **Troubleshooting note:** a Lock Screen Image (`LockScreenImageUrl`) setting was also attempted but returned **error 65000** — a CSP edition-gating limitation (that personalization CSP is gated to Windows Enterprise/Education, not Pro). Diagnosed from the per-setting check-in status and removed, keeping the working profile clean. *"Applied successfully" does not always mean "enforceable on this edition."*

**Android** — `Android WP - Work Profile Restrictions` (Device restrictions):
- **Block screen capture** and **block copy/paste between work and personal profiles**.
- Verified on-device: a screenshot inside a work-badged app returns *"This app doesn't allow taking screenshots,"* while personal-side screenshots still work — confirming the restriction scopes to the **work container only**.

## 📦 Pillar 3 — Application Deployment

**Windows** (two methods):

| App                   | Method                                   | Detection        | Result    |
| --------------------- | ---------------------------------------- | ---------------- | --------- |
| **VLC**               | Microsoft Store app (new, winget-backed) | Automatic        | Installed |
| **7-Zip 26.01 (x64)** | **Win32 MSI** (Content Prep Tool)        | MSI product code | Installed |

**Win32 packaging walkthrough:**

1. Downloaded the 7-Zip **x64 MSI** (the MSI, not the EXE — enables clean automatic detection).
2. Wrapped it into an `.intunewin` package with the **Win32 Content Prep Tool** (`IntuneWinAppUtil.exe`).
3. Defined a **silent install** command: `msiexec /i "7z2601-x64.msi" /qn`
4. Defined the **uninstall** command via the MSI product code.
5. Configured **MSI product-code detection** (`{23170F69-40C1-2702-2601-000001000000}`).
6. Assigned as **Required → All devices**; install behavior **System**.

**Android** (Managed Google Play):

| App                   | Method                          | Assignment            | Result    |
| --------------------- | ------------------------------- | --------------------- | --------- |
| **VLC for Android**   | Managed Google Play (approved)  | Required → All Users  | Installed |

Approved VLC in the Intune-embedded Managed Google Play store, set future-permission handling to stay-approved, and assigned it **Required** — Intune **auto-installed** it into the work profile with no user action. Verified by the briefcase-badged icon on the device and an **Installed** state in `Apps → Monitor → App install status`. Mirrors the Windows-side VLC deployment for clean cross-platform parity.

> Full Android walkthrough, architecture, and per-step detail: **[Android-Enterprise.md](./Android-Enterprise.md)**

## 🛠️ Troubleshooting Log (the part that taught the most)

| Issue                                            | Root cause                                                                                                                                                               | Resolution                                                                                                                                                                                                                            |
| ------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Local-admin enrollment lockout** (Windows)     | First VM was OOBE-bypassed with no local admin; the management user was a standard user and built-in Administrator was disabled                                          | Diagnosed with `net localgroup administrators`. Rebuilt with a forced local account at OOBE + assigned the Entra Joined Device Local Administrator role + corrected the device-registration "registering user as local admin" setting |
| **Config profile error 65000** (Windows)         | CSP edition gating — Lock Screen Image not enforceable on Windows 11 Pro                                                                                                 | Identified from per-setting check-in status; removed the setting, kept the working Copilot profile                                                                                                                                    |
| **Win32 app stuck "Waiting for install status"** | Store apps install via the Store pipeline (instant); Win32 apps install via the **Intune Management Extension (IME)**, which deploys after enrollment and checks ~hourly | Confirmed IME running, read `IntuneManagementExtension.log`, triggered with a service restart / reboot                                                                                                                                |
| **Android policies not applying after assignment** | A freshly enrolled work-profile device doesn't pull newly assigned policies until it checks in; the Intune schedule can lag hours                                       | Verified assignment was correct (All Users, Active) to rule out targeting, then forced a check-in via **Company Portal → Settings → Sync** — policies then evaluated immediately                                                      |
| **Android sync killed mid-flight / compliance flap** | MIUI/HyperOS battery optimization aggressively kills background apps, terminating the Intune agent and the sync before it completes                                     | Set Company Portal + Intune agent to **No-restrictions** battery (and Autostart where exposed); ran sync while staying on-screen until it reported complete                                                                           |
| **Android config-profile "No items found"**       | Device-level **configuration reporting** lags far behind compliance reporting (portal flags "report may be delayed")                                                    | Treated on-device behavioral testing as authoritative rather than the lagging report; confirmed the profile applied by triggering the screen-capture block in a work app                                                              |
| **Device showing wrong ownership** (Windows)      | All-devices grid caches; detail page is source of truth                                                                                                                  | Trusted the device Overview page over the cached list                                                                                                                                                                                 |

## 🧠 Key Concepts Demonstrated

- **Pull-based MDM model** — a managed device must check in for policies/apps to apply; offline = stale status, not failure. (The Android sync issue is this concept in practice.)
- **Cross-platform endpoint management** — the same three pillars (compliance, configuration, app deployment) applied to both Windows and Android in one tenant.
- **Work-profile (BYOD) isolation** — Android Enterprise creates a cryptographically separated work container; Intune manages only the work side, personal apps/data stay private, and unenrollment wipes only the work profile.
- **Least privilege** — Entra Joined Device Local Administrator role instead of Global Admin.
- **Store vs Win32 vs Managed Google Play pipelines** — and why each installs differently.
- **Win32 packaging & detection** — Content Prep Tool, silent install, MSI product-code detection vs manual file/registry detection.
- **CSP edition gating** — recognizing error 65000 as a Windows-edition limitation, not a misconfiguration.
- **Compliance vs configuration** — one judges a device against a security bar (and can feed Conditional Access); the other pushes settings.
- **OEM variance in mobile MDM** — vendor skins (MIUI) affect agent reliability and how faithfully Android Enterprise restrictions are honored; a reason enterprises specify approved device manufacturers.

## 🗂️ Repository Structure

```
Intune-Endpoint-Management-Lab/
├── README.md
├── Android-Enterprise.md   # full Android work-profile walkthrough, architecture, troubleshooting
├── TROUBLESHOOTING.md
├── screenshots/            # Windows + Android: Compliant, config Succeeded/screenshot-blocked, apps Installed, work-badged icon
├── policies/               # exported compliance policies + configuration profiles (JSON)
└── win32-packaging/        # Content Prep Tool notes, install/uninstall/detection commands
```

## 🔧 Skills

`Microsoft Intune` · `MDM/MAM` · `Microsoft Entra ID` · `Android Enterprise` · `Managed Google Play` · `Work Profile (BYOD)` · `Device Compliance` · `Configuration Profiles (Settings Catalog)` · `Application Deployment` · `Win32 Packaging (Content Prep Tool)` · `Windows 11 Administration` · `Cross-Platform Endpoint Management` · `Endpoint Troubleshooting`

---

*Part of a hands-on IT & cybersecurity portfolio. See also: [Windows Server Active Directory Lab](https://github.com/Amogh-Karankal/Windows-AD-Lab) and [Secure Azure AI Chatbot](https://github.com/Amogh-Karankal/azure-secure-ai-chatbot).*
