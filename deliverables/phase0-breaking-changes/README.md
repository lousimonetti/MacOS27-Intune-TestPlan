# Phase 0 — Breaking Changes (P0) Deliverables

Production artifacts for the **P0 — Breaking Changes** tests in [`plan.md`](../../plan.md).
These are the highest-priority tests: each validates a macOS 27 change that breaks an
existing Intune configuration on upgrade day.

All files validate with `plutil -lint` (mobileconfig) and `python3 -m json.tool` (JSON).

## Files

| File | Test | Type | Deploy in Intune via | Expected result |
|---|---|---|---|---|
| `P0-T01-legacy-softwareupdate.mobileconfig` | P0-T01 | Legacy `com.apple.SoftwareUpdate` | Custom profile (Devices → macOS → Configuration profiles → Custom) | **Fails / rejected** — payload type deleted in 27.0 |
| `P0-T02-swu-settings.json` | P0-T02 | DDM `softwareupdate.settings` | Settings Catalog → Software Update (or Custom DDM JSON) | Deferrals applied (14d major / 7d minor) |
| `P0-T02-swu-enforcement.json` | P0-T02 | DDM `softwareupdate.enforcement.specific` | Settings Catalog → Software Update → Enforce | Enforcement deadline countdown appears |
| `P0-T03-removed-restriction-keys.mobileconfig` | P0-T03 | `com.apple.applicationaccess` | Custom profile | **Installs**, removed SU keys silently ignored |
| `P0-T04-legacy-pppc.mobileconfig` | P0-T04 | Legacy PPPC `com.apple.TCC.configuration-profile-policy` | Custom profile | Installs (deprecated, not removed) — document behavior |
| `P0-T05-app-settings-privacy.json` | P0-T05 | DDM `app.settings` (Privacy) | Settings Catalog → App Management (or Custom DDM JSON) | Single consolidated consent prompt |
| `P0-T06-lockscreen-restrictions.mobileconfig` | P0-T06 | `com.apple.applicationaccess` (new keys) | Custom profile | Lock-screen captive portal / Wi-Fi config enabled |
| `P0-T07-platformsso-weblogin.mobileconfig` | P0-T07 | `com.apple.extensiblesso` (new PSSO keys) | Custom profile | Web-login password sync gated by allow list |

## Deployment notes

- **Custom mobileconfig (`.mobileconfig`)** — Devices → macOS → Configuration profiles →
  Create → **Custom**. Upload the file, assign to the `macOS-27-AppleSilicon` device group.
- **DDM JSON (`.json`)** — prefer **Settings Catalog** (macOS 27 entries) where the
  declaration is surfaced. Where it is not yet available (e.g. `app.settings`), deploy the
  raw declaration via Graph API `POST /deviceManagement/deviceConfigurations`, or pilot
  through Apple Configurator / Jamf Pro 11.x+. Verify Settings Catalog availability before
  assuming UI support.
- The DDM JSON files carry an empty `ServerToken` — Intune/the MDM server populates this on
  deployment. Leave it empty in the source artifact.

## Placeholders to replace before real deployment

These test artifacts use sample tenant values. Update before using outside a lab:

- `P0-T02-swu-enforcement.json` → `DetailsURL` (`https://yourcompany.com/...`),
  `TargetLocalDateTime`, `TargetOSVersion` to your enforced build.
- `P0-T05` / `P0-T07` → bundle IDs, Team IDs, and allow-list URLs to match your fleet's
  actual managed apps and identity endpoints.

## Pre-deployment checklist

- [ ] Apple Silicon, ADE-enrolled/supervised macOS 27.0 Beta 2+ test device.
- [ ] Existing legacy SoftwareUpdate profiles removed before P0-T02 (avoid conflicts).
- [ ] Existing PPPC profiles removed before P0-T05 (clean consent-prompt observation).
- [ ] Confirmation of whether each DDM type is in the Intune Settings Catalog yet.

See `plan.md` (P0 section) for full per-test steps and pass criteria.
