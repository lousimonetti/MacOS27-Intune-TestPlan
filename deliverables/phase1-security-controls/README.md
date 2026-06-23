# Phase 1 — New Security Controls (P1) Deliverables

Production artifacts for the **P1 — New Security Controls** tests in [`plan.md`](../../plan.md).
These validate the new binary allow/deny list capability in `com.apple.configuration.app.settings`
introduced in macOS 27 — the most significant new enterprise security control in this release.

All files validate with `python3 -m json.tool`.

## Files

| File | Test | Declaration type | Deploy in Intune via | Expected result |
|---|---|---|---|---|
| `P1-T01-binary-allowlist.json` | P1-T01 | DDM `app.settings` (AllowedBinaries) | Settings Catalog → App Management or Custom DDM JSON | Only Apple, Microsoft (UBF8T346G9), 1Password (2BUA8C4S2C), and Chrome can run; all others blocked |
| `P1-T02-binary-denylist.json` | P1-T02 | DDM `app.settings` (DeniedBinaries) | Settings Catalog → App Management or Custom DDM JSON | Chrome is blocked; all other apps unaffected |
| `P1-T03-managed-apps-passthrough.json` | P1-T03 | DDM `app.settings` (AlwaysAllowManagedApps) | Settings Catalog → App Management or Custom DDM JSON | MDM-managed apps run despite not being in explicit AllowedBinaries list |

## Key design notes

- **P1-T01** uses `TeamID`-level matching for Microsoft (`UBF8T346G9`) — covers all Microsoft 365
  apps (Word, Excel, Teams, Defender, Company Portal) without listing each one. Both `AppStore`
  and `DeveloperID` signing states are included for Microsoft apps that ship via both channels.
- **P1-T02** uses `TeamID` + `SigningID` to deny Chrome specifically without denying other
  Google apps. Adjust to CDHash matching for exact-build blocking if needed
  (`codesign -dvvv /Applications/Google\ Chrome.app`).
- **P1-T03** is a safety validation: the explicit AllowedBinaries list contains only `*APPLE*`,
  but `AlwaysAllowManagedApps: true` should let MDM-deployed apps (e.g. Defender) through.
  Sideloading an unmanaged binary with the same TeamID should still be blocked.
- **`AlwaysAllowManagedApps`** applies to apps deployed via Intune app management
  (`applicationmanagement`), NOT all apps from a given developer's TeamID.

## Deployment notes

- **DDM JSON (`.json`)** — prefer **Settings Catalog** (Devices → macOS → Configuration profiles
  → Settings Catalog → App Management) if the `app.settings` binary control keys are surfaced
  in Intune's macOS 27 schema. If not yet available, deploy via Graph API:
  `POST /deviceManagement/deviceConfigurations` with the raw declaration body.
- `ServerToken` is intentionally empty — Intune populates it on deployment.
- Binary control requires **supervised** macOS 27 devices (system scope via Endpoint Security).
- Do **not** combine P1-T01 (AllowedBinaries) and P1-T02 (DeniedBinaries) in the same
  deployment — test them separately to isolate behavior. In production, the two keys can
  coexist in a single `app.settings` declaration.

## Placeholders to replace before real deployment

- **P1-T01**: Review the `SigningState` values per app — apps distributed outside the App Store
  use `DeveloperID`; apps from the Mac App Store use `AppStore`. Verify with
  `codesign -dvvv <app_path>`.
- **P1-T02**: Add CDHash entries after running `codesign -dvvv` on the target binary on a
  macOS 27 device. CDHash entries lock to an exact build — update them on each app version
  change or use TeamID+SigningID for broader matching.
- Add or remove `AllowedBinaries` entries to match your actual approved vendor list.

## Pre-deployment checklist

- [ ] Apple Silicon, ADE-enrolled/supervised macOS 27.0 Beta 2+ test device.
- [ ] `AlwaysAllowManagedApps: true` confirmed in P1-T01 before broad rollout — prevents
  managed apps from being blocked if a TeamID is missing from the explicit list.
- [ ] At least one test binary that should be blocked (e.g. an unsigned shell script or
  an unsigned `.app` bundle) staged in `/tmp` for negative testing.
- [ ] Verify Settings Catalog availability for `app.settings` binary control keys before
  choosing Custom DDM JSON path.
- [ ] Endpoint Security framework access confirmed (no conflicting ES client software).

See `plan.md` (P1 section) for full per-test steps and pass criteria.
