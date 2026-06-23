# Claude Project Context — Intune macOS 27 Test Plan

This file exists to give a new Claude Code project full context when building the
"Intune macOS 27 Test Plan" deliverable. Read this before writing any code, profiles,
or documentation.

---

## What This Project Is

The goal is to build production-ready custom mobileconfig XML files and DDM declaration
JSON files for uploading to Microsoft Intune, covering all new and changed device
management features in macOS 27 Golden Gate. The source of truth for Apple's MDM schema
is the `apple/device-management` repo (this repo). The test plan lives in `plan.md`.

---

## Source Repo Structure

```
device-management/
├── CHANGES.md                          ← Delta for macOS 27.0 — read this first
├── mdm/
│   ├── profiles/                       ← Legacy mobileconfig payload schemas (YAML)
│   ├── commands/                       ← MDM command schemas
│   ├── checkin/                        ← MDM check-in message schemas
│   └── errors/                        ← MDM error schemas
├── declarative/
│   ├── declarations/
│   │   ├── configurations/             ← DDM configuration declaration schemas
│   │   ├── assets/                     ← DDM asset (credential) schemas
│   │   ├── activations/                ← DDM activation schemas
│   │   └── management/                 ← DDM management declaration schemas
│   ├── status/                         ← DDM status item schemas
│   └── protocol/                       ← DDM protocol schemas
├── other/
│   └── skipkeys.yaml                   ← Setup Assistant skip key definitions
├── docs/
│   └── schema.md                       ← How to read the YAML schema files
└── plan.md                             ← The test plan (this project's output)
```

### How to read YAML schema files

Each file has a `payload` section (OS support, enrollment requirements, scopes) and
a `payloadkeys` section (all available keys with types, defaults, and OS introductions).
`introduced: n/a` means the key does NOT apply to that OS. `introduced: '27.0'` means
the key is new in this release.

---

## Priority 1: Breaking Changes (Fleet Risk)

These changes break existing Intune configurations on upgrade day. Address these first.

### 1A. Software Update — Everything Removed

The entire legacy software update management stack was removed in macOS 27:

**Removed profile type:**
- `com.apple.SoftwareUpdate` — payload type deleted entirely

**Removed MDM commands:**
- `ScheduleOSUpdate`, `AvailableOSUpdates`, `OSUpdateStatus`, `ScheduleOSUpdateScan`
- `mdm/commands/system.update.available.yaml` — deleted
- `mdm/commands/system.update.scan.yaml` — deleted
- `mdm/commands/system.update.schedule.yaml` — deleted
- `mdm/commands/system.update.status.yaml` — deleted

**Removed restriction keys from `com.apple.applicationaccess`:**
- `enforcedSoftwareUpdateDelay`
- `enforcedSoftwareUpdateMajorOSDeferredInstallDelay`
- `enforcedSoftwareUpdateMinorOSDeferredInstallDelay`
- `enforcedSoftwareUpdateNonOSDeferredInstallDelay`
- `forceDelayedAppSoftwareUpdates`
- `forceDelayedMajorSoftwareUpdates`
- `forceDelayedSoftwareUpdates`
- `allowRapidSecurityResponseInstallation`
- `allowRapidSecurityResponseRemoval`

**Removed DeviceInformation queries:**
- `OSUpdateSettings`, `SoftwareUpdateDeviceID`, `SoftwareUpdateSettings`

**Replacement path (DDM — required):**
- `com.apple.configuration.softwareupdate.settings` → deferrals, automatic actions, beta programs
  - File: `declarative/declarations/configurations/softwareupdate.settings.yaml`
  - macOS support: introduced 15.0, supervised
- `com.apple.configuration.softwareupdate.enforcement.specific` → deadline enforcement
  - File: `declarative/declarations/configurations/softwareupdate.enforcement.specific.yaml`
  - macOS support: introduced 14.0, supervised

**Intune mapping:** Devices → macOS → Settings Catalog → Software Update section.

### 1B. PPPC Deprecated

`com.apple.TCC.configuration-profile-policy` (PPPC) keys are deprecated in macOS 27.
The profile type is NOT removed yet — it still installs — but behavior may change in
future releases. Begin migration now.

**Replacement path:**
- `com.apple.configuration.app.settings` (DDM, new in 27.0)
  - File: `declarative/declarations/configurations/app.settings.yaml`
  - `Privacy.PermissionDefaults` sub-key replaces PPPC services dict
  - Key difference: presents a single consolidated consent prompt (user can Accept All or Not Now)
  - App identifier format in macOS: `"Bundle-ID (Team-ID)"` or `"Bundle-ID {Designated-Requirement}"`
  - Permissions supported: Accessibility, Bluetooth, Camera, Dictation, LocalNetwork, Location, Microphone

**Critical distinction:** PPPC silently granted permissions without user consent (pre-grant).
The new `app.settings` Privacy model presents a consent prompt — user must accept. The
organization provides `OrganizationJustification` text that appears in the prompt.

### 1C. Backup Behavior Change

macOS 27 devices no longer restore MDM enrollment, supervision status, or management
configuration from Time Machine or iCloud backups. This is an intentional security change.

**Impact on IT operations:**
- Wiped-and-restored devices will NOT have MDM profiles — must re-enroll via ADE.
- ADE ensures re-enrollment happens automatically during Setup Assistant.
- Helpdesk runbooks must be updated — "restore from backup" does not restore MDM state.
- The `do_not_use_profile_from_backup` key in enrollment profiles no longer has effect.

### 1D. TLS / ATS Requirements

macOS 27 enforces App Transport Security (ATS) requirements on all device management
connections:
- TLS 1.2 minimum (TLS 1.3 recommended)
- Forward secrecy cipher suites only
- ATS-compliant certificates

Intune's infrastructure is expected to already meet these requirements, but validate
before fleet upgrade via `nscurl --ats-diagnostics` from a macOS 27 device.

---

## Priority 2: New Security Controls (Opportunity)

### 2A. Binary Control (app.settings — AllowedBinaries / DeniedBinaries)

This is the most significant new security capability in macOS 27 for enterprise.
Uses the Endpoint Security framework. macOS-only, supervised, system scope.

**Key concepts:**
- `AllowedBinaries`: Only listed binaries may run (allowlist mode)
- `DeniedBinaries`: Listed binaries are blocked (denylist mode)
- `AlwaysAllowManagedApps: true`: MDM-deployed apps automatically bypass allowlist —
  critical safety net to prevent breaking managed app deployments
- Matching hierarchy: CDHash (exact build) > TeamID+SigningID (app) > TeamID (vendor) > PathPrefix

**Binary identifier fields (from `codesign -dvvv <path>`):**
- `CDHash`: Hex string, most specific
- `TeamID`: 10-character alphanumeric string
- `SigningID`: Reverse-DNS bundle identifier
- `PathPrefix`: File system path string
- `SigningState`: All | TestFlight | DeveloperID | Enterprise | AppStore | Apple
- Apple binaries with empty TeamID: use `"*APPLE*"` as TeamID value

**AllowedBinaries rules:** Either CDHash or TeamID must be present.
**DeniedBinaries rules:** Either CDHash, TeamID, or SigningID must be present.

File: `declarative/declarations/configurations/app.settings.yaml`

**Compliance mapping:** Satisfies NIST SP 800-53 CM-7 (Least Functionality) and
ISO 27001:2022 A.8.19 without third-party tools.

### 2B. Privacy Permission Defaults (app.settings — Privacy)

Replaces PPPC. User-scope on macOS, system-scope on iOS/tvOS.
Only applies to AppKit-based apps on macOS (not Catalyst apps).

**Permission types available on macOS:**
- `Accessibility` (macOS only)
- `Bluetooth`
- `Camera`
- `Dictation`
- `LocalNetwork`
- `Location` (WhileUsing = Always on macOS, no distinction)
- `Microphone`

Note: `LocationAccuracy` is NOT available on macOS (iOS/visionOS only).

---

## Priority 3: New DDM Capabilities

### 3A. Network Configurations (all new in 27.0)

Six new DDM declaration types replace legacy network profiles:

| Declaration Type | Replaces | File |
|---|---|---|
| `com.apple.configuration.network.vpn.ikev2` | Legacy VPN profile | `declarative/declarations/configurations/network.vpn.ikev2.yaml` |
| `com.apple.configuration.network.vpn.ipsec` | Legacy VPN profile | `declarative/declarations/configurations/network.vpn.ipsec.yaml` |
| `com.apple.configuration.network.vpn.always-on` | Legacy Always-on VPN | `declarative/declarations/configurations/network.vpn.always-on.yaml` |
| `com.apple.configuration.network.vpn.vpn-plugin` | Legacy VPN plugin profile | `declarative/declarations/configurations/network.vpn.vpn-plugin.yaml` |
| `com.apple.configuration.network.dns-settings` | `com.apple.dnsSettings.managed` | `declarative/declarations/configurations/network.dns-settings.yaml` |
| `com.apple.configuration.network.dns-proxy` | `com.apple.dnsProxy.managed` | `declarative/declarations/configurations/network.dns-proxy.yaml` |
| `com.apple.configuration.network.relay` | `com.apple.relay.managed` | `declarative/declarations/configurations/network.relay.yaml` |

**Key advantage of DDM network configs:** Use credential asset declarations
(`com.apple.asset.credential.scep`, `com.apple.asset.credential.acme`,
`com.apple.asset.credential.identity`) for certificate lifecycle management.
When the certificate renews, the network configuration automatically uses the
new cert without a profile reinstall.

### 3B. Content Caching DDM (macOS 27, supervised only)

`com.apple.configuration.content-cache.settings` replaces `com.apple.AssetCache.managed`.
File: `declarative/declarations/configurations/content-cache.settings.yaml`

New capabilities vs. legacy:
- `DeclarativeStatusInterval` for real-time status reporting via DDM
- `ManagementStatusTarget` + `ManagementSecurityConfig` for custom monitoring webhooks
- Paired with four new DDM status items: `content-cache.info`, `content-cache.status`,
  `content-cache.parents`, `content-cache.peers`

### 3C. Web Content Filter DDM

`com.apple.configuration.webcontent-filter.plugin` (new in 27.0)
File: `declarative/declarations/configurations/webcontent-filter.plugin.yaml`

Supports: Browsers (WebKit), Sockets (network-level), Packets (macOS), URLs (PIR-based).
Uses `CredentialsAssetReference` for authentication — shared credential declarations
work across VPN, DNS, SSO, and content filter configurations.

### 3D. Intelligence Settings Updates (27.0)

New key in `com.apple.configuration.intelligence.settings`:
`Apps.Calendar.AllowNaturalLanguageEditing` (introduced macOS 27.0, iOS 27.0, visionOS 27.0)

Controls whether Apple Intelligence can parse natural language input in Calendar and Reminders.

### 3E. Enhanced Log Collection Commands (new in 27.0)

Two new MDM commands for supervised devices:
- `TriggerEnhancedLogCollection` — requires AppleCare Enterprise token
  - File: `mdm/commands/trigger.enhanced.log.collection.yaml`
- `CancelEnhancedLogCollection`
  - File: `mdm/commands/cancel.enhanced.log.collection.yaml`

Three new DDM status items:
- `enhanced-logging.status`
- `enhanced-logging.timestamp`
- `enhanced-logging.applecare-token`

### 3F. PlatformSSO New Keys

Two new keys in `com.apple.extensiblesso` under `PlatformSSO`:
- `AllowWebLoginPasswordSync` (boolean) — enables password sync from web logins
- `WebLoginURLAllowList` (array of strings) — restricts which URLs trigger sync

File: `mdm/profiles/com.apple.extensiblesso.yaml`

### 3G. Package Declaration UninstallBehavior

New key in `declarative/declarations/configurations/package.yaml`:
`UninstallBehavior: Remove` — removing the declaration actually uninstalls the package.
Previously, removing a package declaration left the package installed.

### 3H. ProfileAssetReference (Legacy Profile Migration)

New key in `declarative/declarations/configurations/legacy.yaml` and
`declarative/declarations/configurations/legacy.interactive.yaml`:
`ProfileAssetReference` — allows legacy configuration profiles to be delivered as
declarative assets with flexible hosting, authentication, and built-in integrity verification.

---

## Priority 4: Status and Monitoring

### New DDM Status Items in 27.0

| Status Item | Value Type | Notes |
|---|---|---|
| `mdm.enrollment-type` | string | Supervised, Device, User |
| `mdm.is-awaiting-configuration` | boolean | Awaiting ConfigurationWebURL |
| `mdm.is-return-to-service` | boolean | Device in RTS flow |
| `mdm.is-shared-ipad` | boolean | Shared iPad indicator |
| `mdm.push-magic` | string | APNs push magic UUID |
| `mdm.push-token` | string | APNs push token (base64) |
| `device.system.health` | dictionary | Hardware component health |
| `security.lockdown-mode` | boolean | Lockdown Mode active (supervised only) |
| `content-cache.info` | dictionary | Cache stats and identifiers |
| `content-cache.status` | dictionary | Cache operational status |
| `content-cache.parents` | array | Parent cache list |
| `content-cache.peers` | array | Peer cache list |
| `enhanced-logging.status` | string | Log collection state |
| `enhanced-logging.timestamp` | string | Collection start time |
| `enhanced-logging.applecare-token` | string | Session token |

Files: `declarative/status/` directory.

---

## Priority 5: Setup Assistant / Enrollment

### New Skip Keys (27.0)

`LiquidGlass` — skips the macOS 27 Liquid Glass UI design introduction pane.
`AccessibilityAppearance` — skips Accessibility Appearance pane during new user login.

File: `other/skipkeys.yaml`
Configure via ABM enrollment profile or `com.apple.SetupAssistant.managed` mobileconfig.

### Return to Service Improvements

`ShouldRetryEnrollment` (new key in 27.0):
- `mdm/checkin/returntoservice.yaml` — for check-in
- `mdm/commands/device.erase.yaml/ReturnToService/ShouldRetryEnrollment` — for erase command

When `true`, failed enrollment retries with exponential backoff (up to 5 minute intervals).

---

## Intune-Specific Context

### Deployment Methods

**Legacy mobileconfig (XML) → Intune Custom Profile:**
- Devices → macOS → Configuration profiles → Create → Custom
- Upload .mobileconfig file
- Assign to device group
- Use for: com.apple.applicationaccess, com.apple.extensiblesso, com.apple.TCC.*, com.apple.SetupAssistant.managed, etc.

**DDM Declarations → Intune Settings Catalog:**
- Devices → macOS → Configuration profiles → Create → Settings Catalog
- Select macOS 27 settings
- Use for: softwareupdate.settings, softwareupdate.enforcement.specific, intelligence.settings, etc.

**DDM Declarations → Custom DDM (when not in Settings Catalog):**
- Use Graph API: `POST /deviceManagement/deviceConfigurations` with DDM JSON body
- Or use Apple Configurator 3 / Jamf for DDM declarations not yet surfaced in Intune UI

### Apple Silicon Requirement

macOS 27 only runs on Apple Silicon Macs (M-series). Any fleet with Intel Macs
will NOT be running macOS 27. Segment your device groups in Intune accordingly:
- `macOS-27-AppleSilicon` group: ADE-enrolled M-series Macs
- Keep Intel Mac policies on macOS 26.x path (they sunset from Apple support)

### Intune DDM Support Status

Microsoft provides day-zero Intune support for macOS 27, with expanded Settings Catalog
entries for the new DDM declarations. However, some very new declarations (e.g.,
`app.settings` binary control, `content-cache.settings`, network VPN DDM types) may
require Custom DDM JSON deployment until Microsoft's Settings Catalog is updated.
Check the Intune Settings Catalog for macOS 27 entries before assuming UI availability.

---

## Key Command Reference

```bash
# Extract binary identifiers for AllowedBinaries/DeniedBinaries
codesign -dvvv /Applications/Microsoft\ Teams.app

# Check ATS compliance from macOS 27 device
nscurl --ats-diagnostics https://manage.microsoft.com

# Verify software update DDM declarations applied
defaults read /Library/Preferences/com.apple.SoftwareUpdate

# Monitor MDM communication
log show --predicate 'subsystem == "com.apple.ManagedClient"' --last 1h

# Monitor Endpoint Security (binary control)
log show --predicate 'subsystem == "com.apple.endpointsecurity"' --last 5m

# Check active configuration profiles
profiles -P

# Check active DDM declarations
profiles -e -o stdout-xml

# Check DNS configuration
scutil --dns

# Software update status via DDM
softwareupdate -l
```

---

## Critical Knowledge Gaps / Things to Verify with Beta

1. **Intune Settings Catalog availability** for `app.settings` binary control — may require
   Graph API deployment initially.
2. **PPPC behavior** — the deprecation notice says "deprecated," not "removed." Confirm
   whether PPPC profiles still apply fully or are selectively ignored in 27.0.
3. **app.settings Privacy** — "Only AppKit-based apps on macOS support this feature" —
   test with Microsoft 365 apps (Mix of AppKit and Catalyst — Teams, Word, etc.).
4. **AlwaysAllowManagedApps scope** — confirm "managed apps" means apps deployed via
   `applicationmanagement` (MDM app install), not apps from the same Team ID.
5. **Intune enrollment type** — verify `mdm.enrollment-type` correctly reflects ADE
   supervised enrollment vs. user enrollment in Intune's reporting.

---

## Deliverables Progress

Track what has been built and pushed. Artifacts live in `deliverables/` subdirectories.
Each artifact is a standalone file ready to upload directly to Intune (mobileconfig or DDM JSON).

| Phase | Directory | Status | Files |
|---|---|---|---|
| P0 — Breaking Changes | `deliverables/phase0-breaking-changes/` | **Complete** | T01–T07 (8 files) |
| P1 — Binary Control / Privacy | `deliverables/phase1-security-controls/` | Not started | — |
| P2 — Network DDM | `deliverables/phase2-network-ddm/` | Not started | — |
| P3 — New MDM Features | `deliverables/phase3-new-features/` | Not started | — |
| P4 — Status & Monitoring | `deliverables/phase4-status-monitoring/` | Not started | — |
| P5 — Setup Assistant / Enrollment | `deliverables/phase5-enrollment/` | Not started | — |

### Phase 0 artifacts (built 2026-06-23)

All files validated (`plutil -lint` for mobileconfig, `python3 -m json.tool` for JSON).

| File | Test | Payload type | Purpose |
|---|---|---|---|
| `P0-T01-legacy-softwareupdate.mobileconfig` | P0-T01 | `com.apple.SoftwareUpdate` | Verify payload type is rejected on macOS 27 |
| `P0-T02-swu-settings.json` | P0-T02 | DDM `softwareupdate.settings` | DDM deferral/auto-action replacement |
| `P0-T02-swu-enforcement.json` | P0-T02 | DDM `softwareupdate.enforcement.specific` | Deadline enforcement declaration |
| `P0-T03-removed-restriction-keys.mobileconfig` | P0-T03 | `com.apple.applicationaccess` | Verify removed SU keys are silently ignored |
| `P0-T04-legacy-pppc.mobileconfig` | P0-T04 | `com.apple.TCC.configuration-profile-policy` | Observe PPPC deprecation behavior |
| `P0-T05-app-settings-privacy.json` | P0-T05 | DDM `app.settings` (Privacy) | Validate consolidated consent prompt |
| `P0-T06-lockscreen-restrictions.mobileconfig` | P0-T06 | `com.apple.applicationaccess` | New lock-screen restriction keys |
| `P0-T07-platformsso-weblogin.mobileconfig` | P0-T07 | `com.apple.extensiblesso` | New PlatformSSO web-login sync keys |

---

## Sources Referenced

- Apple Device Management Client Schema repo — `CHANGES.md`, `declarative/`, `mdm/`
- Apple WWDC26 Device Management Updates: https://support.apple.com/guide/deployment/device-management-updates-depd638aa061/1/web/1.0
- Stabilise.io macOS 27 Binary Control analysis: https://stabilise.io/blog/macos-27-mdm-binary-control-pppc-replacement-mac-admins
- macOS 27 release notes: https://developer.apple.com/documentation/macos-release-notes/macos-27-release-notes
- Mac Admins News WWDC26 coverage: https://www.macadmins.news/410-golden-gate/
