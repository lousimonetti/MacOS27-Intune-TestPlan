# macOS 27 Golden Gate — Intune Test Plan

Production-ready configuration profiles and DDM declarations for validating all new and
changed macOS 27 device management features in a Microsoft Intune-managed fleet, before
general availability.

**OS:** macOS 27.0 Golden Gate (Beta)  
**MDM:** Microsoft Intune  
**Hardware requirement:** Apple Silicon Macs only (M-series) — macOS 27 does not support Intel

---

## What this repo contains

This repo is the canonical artifact store for the macOS 27 Intune test plan. Every test
case in `plan.md` has a corresponding deployable file in `deliverables/`.

```
MacOS27-Intune-TestPlan/
├── plan.md                              ← Full test plan: steps, pass criteria, all phases
├── claude.md                            ← Context file for this project (AI-assisted authoring)
└── deliverables/
    └── phase0-breaking-changes/         ← P0 artifacts (COMPLETE — push to Intune now)
        ├── README.md                    ← Deployment guide for P0 files
        ├── P0-T01-legacy-softwareupdate.mobileconfig
        ├── P0-T02-swu-settings.json
        ├── P0-T02-swu-enforcement.json
        ├── P0-T03-removed-restriction-keys.mobileconfig
        ├── P0-T04-legacy-pppc.mobileconfig
        ├── P0-T05-app-settings-privacy.json
        ├── P0-T06-lockscreen-restrictions.mobileconfig
        └── P0-T07-platformsso-weblogin.mobileconfig
```

---

## How to use this repo

### Step 1 — Read the test plan

Start with [`plan.md`](plan.md). It defines every test case across all priority levels,
including the full steps, pass criteria, and expected Intune behavior.

### Step 2 — Deploy Phase 0 first (breaking changes)

The `deliverables/phase0-breaking-changes/` files are ready to deploy. See
[`deliverables/phase0-breaking-changes/README.md`](deliverables/phase0-breaking-changes/README.md)
for the file-to-test mapping, Intune deployment paths, and placeholders you must
update before deploying outside a lab.

**Deploy order for P0:**
1. `P0-T01` — try the legacy SoftwareUpdate profile (expect rejection)
2. `P0-T02` — deploy both DDM replacement declarations for software update
3. `P0-T03` — deploy removed restriction keys (expect silent ignore)
4. `P0-T04` — deploy legacy PPPC profile (observe deprecation behavior)
5. `P0-T05` — deploy `app.settings` privacy declaration (PPPC replacement)
6. `P0-T06` — deploy new lock screen restriction keys
7. `P0-T07` — deploy PlatformSSO web login keys

### Step 3 — Work through remaining phases

Subsequent phases (P1–P5) are documented in `plan.md`. Artifacts will be added to
`deliverables/` as each phase is built out.

| Phase | Area | Status |
|---|---|---|
| P0 | Breaking Changes (Software Update, PPPC, Restrictions, PlatformSSO) | **Artifacts ready** |
| P1 | New Security Controls (Binary Control, Privacy Defaults) | Planned |
| P2 | New Network DDM Configs (VPN, DNS, Relay) | Planned |
| P3 | New MDM Features (Content Cache, Web Filter, Intelligence, Logging) | Planned |
| P4 | Status & Monitoring | Planned |
| P5 | Setup Assistant / Enrollment | Planned |

---

## Deployment methods at a glance

| File type | Intune path |
|---|---|
| `.mobileconfig` | Devices → macOS → Configuration profiles → Create → **Custom** |
| `.json` (DDM) | Settings Catalog (macOS 27 entries) — or Custom DDM JSON via Graph API if not yet in Catalog |

For DDM JSON not yet surfaced in Intune's Settings Catalog, deploy via:
```
POST https://graph.microsoft.com/beta/deviceManagement/deviceConfigurations
```
with the JSON declaration as the body.

---

## Prerequisites

- Apple Silicon Mac(s) enrolled in Apple Business Manager (ABM), supervised via ADE
- macOS 27.0 Beta 2+
- Intune tenant with DDM support and macOS 27 Settings Catalog entries
- Minimum two test devices: one supervised (ADE), one User Enrollment
- AppleCare Enterprise agreement for P3-T03/T04 (Enhanced Log Collection)

---

## Source of truth

All profiles and declarations are derived from the Apple Device Management Client Schema
(`apple/device-management` repo, branch `seed_OS_27_0`). Key references:

- `CHANGES.md` in that repo — macOS 27.0 delta (all additions and removals)
- `declarative/declarations/configurations/` — DDM declaration schemas
- `mdm/profiles/` — Legacy mobileconfig payload schemas
- `other/skipkeys.yaml` — Setup Assistant skip key definitions
