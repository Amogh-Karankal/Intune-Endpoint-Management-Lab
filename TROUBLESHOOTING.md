# Troubleshooting Log

The issues hit during this lab and how each was diagnosed and resolved. These were the most valuable part of the project — endpoint management in the real world is mostly diagnosing why something *isn't* applying.

---

## 1. Local-admin enrollment lockout

**Symptom**
On the first VM build, the Settings → **Access work or school** page showed a red banner: *"Sign in as an administrator to change device management settings."* The device was Entra-joined but I couldn't disconnect, re-enroll, or change any management setting.

**Diagnosis**
```
whoami
# azuread\samitadmin

net localgroup administrators        # threw a localization error (1376)
Get-LocalGroupMember -SID S-1-5-32-544
```
The local Administrators group contained only the built-in `Administrator` account (which was **disabled** — `net user administrator` showed `Account active: No`). The Entra user that performed the join was **not** a local admin, so it was effectively a standard user and couldn't change device-management settings.

**Root cause**
The VM was created by bypassing local-account creation at OOBE and going straight to an Entra join. Two things combined:
1. No local admin account was ever created.
2. The tenant's **device-registration setting** ("Registering user is added as local administrator on the device during Microsoft Entra join") was set to **None**, so the joining user did not become a local admin.

**Resolution**
Rebuilt the device correctly and fixed the tenant setting:
1. At OOBE, forced a local account before any Entra activity:
   - `Shift + F10` → `start ms-cxh:localonly` → created a `localadmin` local administrator account.
2. In Entra: set the device-registration "registering user as local admin" scope to the specific management user (least privilege), **and** assigned that user the **Microsoft Entra Joined Device Local Administrator** role.
3. After the Entra join, verified the dual-admin end state:
```
Get-LocalGroupMember -SID S-1-5-32-544
# Administrator, localadmin, AzureAD\AlexHelpdesk
dsregcmd /status
# AzureAdJoined : YES
```

**Lesson**
An Entra join does **not** automatically make the joining user a local admin — that depends on the device-registration setting and/or a role assignment. Always build managed devices with a known-good local admin account as a safety net so you can never lock yourself out of management.

---

## 2. Configuration profile error 65000 (Lock Screen Image)

**Symptom**
A Settings Catalog configuration profile contained two settings: "Turn Off Windows Copilot" and "Lock Screen Image URL." After sync, the device's lock screen *changed* but showed a default image, not the one I specified. The portal's per-setting status showed:
- Turn Off Windows Copilot → **Succeeded**
- Lock Screen Image URL → **Error, code 65000**

**Diagnosis**
The image URL was a valid public raw GitHub link and loaded correctly *inside the VM's browser*, so it wasn't a network or URL problem. The Copilot setting in the same profile succeeded, so the profile itself was fine. Checked **Devices → Configuration → [profile] → Device install/check-in status** and drilled into the per-setting detail: error type 2, code **65000** on the Lock Screen Image setting only.

**Root cause**
**CSP edition gating.** The `Personalization/LockScreenImageUrl` CSP is not reliably enforceable on **Windows 11 Pro** — that setting is gated to Enterprise/Education editions. Windows accepted the policy assignment but the CSP rejected enforcement and fell back to the default lock screen.

**Resolution**
Removed the Lock Screen Image setting from the profile (kept the working Copilot setting). Documented the finding rather than fighting an edition limitation.

**Lesson**
"Applied successfully" does not always mean "enforceable on this edition." Error 65000 on a personalization/policy CSP is a strong signal of edition gating, not a misconfiguration. The per-setting check-in status — not the device's surface behavior — is the source of truth.

---

## 3. Win32 app stuck at "Waiting for install status"

**Symptom**
A Microsoft Store app (VLC) installed within minutes. A Win32 app (7-Zip MSI), assigned at the same time as Required, sat at **"Waiting for install status"** for 15+ minutes with no error shown in the portal. The app's Device install status page showed **0 items**.

**Diagnosis**
"0 items" (device not even listed) plus "Waiting for install status" indicated the assignment had reached the device but no result had been reported — not a failure. Checked on the device:
- `services.msc` → **Microsoft Intune Management Extension** service was **present and Running**.
- Folder `C:\Program Files (x86)\Microsoft Intune Management Extension\` existed.
- Log: `C:\ProgramData\Microsoft\IntuneManagementExtension\Logs\IntuneManagementExtension.log`

**Root cause**
Store apps and Win32 apps use **different install pipelines**:
- **Store apps** install via the Windows Store/UWP pipeline — work as soon as the device is MDM-enrolled.
- **Win32 apps** install via the **Intune Management Extension (IME)** — a separate agent that deploys *after* enrollment and evaluates assigned Win32 apps on a periodic cycle (roughly hourly) and on startup.

On a freshly-enrolled device, the IME simply hadn't run its Win32 evaluation cycle for this app yet. Not a packaging or detection failure.

**Resolution**
Restarted the IME service (and rebooted the VM) to trigger an immediate evaluation. The IME then downloaded the package, ran `msiexec /i "7z2601-x64.msi" /qn`, and 7-Zip installed silently — confirmed by the app appearing in the Start menu and the portal flipping to **Installed**.

**Lesson**
When a Win32 app lags while a Store app installs instantly, suspect IME timing, not the package. Read `IntuneManagementExtension.log` for the real status/exit code, and remember the IME runs on its own schedule — a service restart or reboot forces an immediate cycle.

---

## General principle

Intune is **pull-based**: the cloud assigns; the device checks in, evaluates locally, and reports back. So when something looks wrong, the first questions are always:
1. Is the device powered on and **checked in**? (Offline = stale status, not failure.)
2. Am I looking at the **detail/per-setting status** or a **cached summary view**? (The detail page is the source of truth; the All-devices grid lags.)
3. Which **pipeline** does this use? (MDM CSP vs Win32/IME — they behave and time differently.)
