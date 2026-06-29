# Android Enterprise (Personally-Owned Work Profile)

Cross-platform extension of the Intune endpoint-management lab. This document covers the
Android side; the Windows 11 side is documented in the main README.

**Tenant:** `<yourtenantname>outlook.onmicrosoft.com`
**Enrollment model:** Android Enterprise — Personally-Owned Work Profile (BYOD)
**Test device:** Xiaomi Redmi Note 10S, Android 13 / MIUI
**Licensed user:** `alex.helpdesk` (Intune license; Entra Joined Device Local Administrator role)

---

## Why work profile (BYOD)

Personally-owned work profile creates a cryptographically separated container on the device.
Intune manages **only** the work side — corporate apps, policies, and data — while the user's
personal apps, photos, accounts, and location stay private and unmanaged. On unenrollment,
**only the work profile is wiped**; personal data is never touched. This is the most common
real-world model for employee personal phones, which is why it was chosen over fully-managed
(which requires a factory reset).

---

## Architecture

```
Microsoft Intune (Entra tenant)
        │
        │  Managed Google Play binding (one-time tenant config)
        ▼
Google Play (managed) ──► approved apps only
        │
        │  Company Portal enrollment as alex.helpdesk
        ▼
Xiaomi Redmi Note 10S
 ├── PERSONAL profile  (unmanaged — user's apps/data, untouched)
 └── WORK profile      (managed by Intune — briefcase-badged)
        ├── Compliance policy   → measures device posture
        ├── Configuration profile → enforces work-profile restrictions
        └── Managed Google Play app (VLC, Required → auto-installed)
```

---

## Setup steps

### 0. Managed Google Play binding (one-time, tenant-level)
`Intune → Devices → Enrollment → Android → Managed Google Play` → agree → connect with a
**dedicated Google account** (not a personal one — it becomes the tenant's managed Play
account). Organization name set to "Amogh Lab". Until this is bound, all Android enrollment
options are greyed out.

### 1. Enroll the device
On the phone: install **Intune Company Portal** → sign in as `alex.helpdesk` → choose
**work profile** setup. Android provisions the isolated work container and the management
agent inside it automatically (no separate "Intune" app needed for the work-profile model).

### 2. Harden against MIUI background-kill
MIUI/HyperOS aggressively kills background apps, which causes the Intune agent to stop
checking in and compliance to flap. Mitigation: `Settings → Apps → Company Portal & Intune →
Battery = No restrictions` (and Autostart = ON where the build exposes it). On newer HyperOS
builds the Autostart toggle is folded into the No-restrictions battery setting.

---

## The three pillars

### Pillar 1 — Compliance policy
**Name:** `Android WP - Baseline Compliance`
Mirrors the Windows baseline. Per-setting evaluation, all Compliant on the test device:

| Setting | Value |
|---|---|
| Rooted devices | Block |
| Minimum OS version | 11 |
| Require password to unlock | Require |
| Password complexity (Android 12+) | Low |
| Encryption of data storage | Require |
| Block apps from unknown sources | Block |
| Block USB debugging | Block |
| Company Portal app runtime integrity | Require |
| Google Play Services configured | Require |
| Up-to-date security provider | Require |
| Play Integrity device attestation | Check basic integrity |

**Action for noncompliance:** Mark device noncompliant after a **1-day** grace period
(demonstrates understanding of grace periods vs. immediate enforcement).

### Pillar 2 — Configuration profile
**Name:** `Android WP - Work Profile Restrictions`
**Type:** Android Enterprise → Device restrictions (Personally-Owned Work Profile)

| Setting | Value |
|---|---|
| Screen capture | Block |
| Copy/paste between work and personal profiles | Block |

Verified on-device: attempting a screenshot inside a work-badged app returns
*"This app doesn't allow taking screenshots."* Screenshots on the personal side still work —
confirming the restriction scopes to the work container only.

### Pillar 3 — App deployment
App **VLC for Android** (Videolabs) approved in Managed Google Play and assigned
**Required → All Users**. Intune auto-installs it into the work profile (no user action).
Verified by the briefcase-badged VLC icon in the device's Work tab and an "Installed" state in
`Apps → Monitor → App install status`. Mirrors the Windows-side VLC deployment for clean
cross-platform parity.

---

## Troubleshooting notes (real issues hit and resolved)

**Policies not applying after assignment.** A freshly enrolled work-profile device does not
pull newly assigned compliance/configuration policies until it checks in. The Intune schedule
can lag by hours. **Resolution:** force a check-in via `Company Portal → Settings → Sync`.
Confirmed correct assignment first (All Users, Active) to rule out a targeting error before
treating it as a sync-timing issue.

**Sync getting killed mid-flight on MIUI.** A backgrounded sync was being terminated by MIUI
battery optimization before it completed. **Resolution:** set Company Portal + Intune agent to
No-restrictions battery, then run the sync while staying on-screen until it reports complete.

**Config-profile reporting lag.** Device-level configuration reporting in Intune is
significantly slower than compliance reporting (the portal flags this with a "report may be
delayed" banner). "No items found" in the device's Device Configuration view is not, on its
own, proof a profile failed to apply — behavioral testing on the device is the authoritative
check.

**Screen-capture block on Xiaomi (noted, not hit here).** MIUI/HyperOS has a documented history
of inconsistently honoring the Android Enterprise `disableScreenCapture` flag. On this device
the hardware-button screenshot was correctly blocked inside work apps. In environments where
OEM screenshot gestures bypass the flag, the appropriate fix is at the policy layer — specifying
approved device manufacturers in the MDM hardware standard — rather than the Intune config.

---

## Outcome

A single Intune tenant managing **both Windows 11 and Android** endpoints across all three
core capabilities — compliance, configuration, and application deployment — demonstrating
cross-platform endpoint management rather than a Windows-only portfolio.
