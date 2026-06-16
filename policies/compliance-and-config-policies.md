# Policies — Compliance & Configuration

The settings configured in this lab. (Intune also lets you export these as JSON via Graph / the portal export — add the exported `.json` files here if you capture them.)

---

## Compliance Policy — `Win11 - Baseline Compliance`

Platform: **Windows 10 and later** · Profile type: Windows 10/11 compliance policy
Assigned to: All devices · Result: **Compliant**

| Category | Setting | Value |
|----------|---------|-------|
| Device Health | Require Secure Boot to be enabled | Require |
| Device Health | Require code integrity | Require |
| Device Properties | Minimum OS version | Baseline below target build |
| System Security | Firewall | Require |
| System Security | Antivirus | Require |
| System Security | Antispyware | Require |
| System Security | Microsoft Defender real-time protection | Require |
| System Security | Require password to unlock | Require |
| System Security | Simple passwords | Block |
| System Security | Minimum password length | 6 |

**Actions for noncompliance:** Mark device noncompliant — immediately (default).

> Settings were chosen to pass cleanly on the target device while still representing meaningful security controls. BitLocker was intentionally left out of the first pass (the VM has a vTPM but BitLocker wasn't enabled) — it's a natural next addition as a compliance + remediation demo.

---

## Configuration Profile — `Win11 - Copilot Config` (Settings Catalog)

Platform: **Windows 10 and later** · Profile type: Settings catalog
Assigned to: All devices

| Setting | Category | Value | Result |
|---------|----------|-------|--------|
| Turn Off Windows Copilot | Windows AI | Enabled | **Succeeded** |

> A Lock Screen Image (`LockScreenImageUrl`) setting was also tested but returned **error 65000** — CSP edition gating (not enforceable on Windows 11 Pro). It was removed; see `TROUBLESHOOTING.md`.

---

## How to export the real JSON (optional)

To add the actual policy definitions to this folder:
- **Portal:** open the policy → `...` (more) → some policy types offer **Export**.
- **Graph / PowerShell:** use the `Microsoft.Graph.Intune` module or Graph Explorer against
  `deviceManagement/deviceCompliancePolicies` and `deviceManagement/configurationPolicies`,
  and save the JSON here as e.g. `Win11-Baseline-Compliance.json`.
