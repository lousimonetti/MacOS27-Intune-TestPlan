# macOS 27 Golden Gate — Intune Test Plan

**Branch:** `seed_OS_27_0`  
**Repo:** [Apple Device Management Client Schema — seed_OS_27_0](https://github.com/apple/device-management/tree/seed_OS_27_0)  
**Target MDM:** Microsoft Intune  
**Status:** Beta — macOS 27.0 (Golden Gate Beta 2)  
**Goal:** Validate all macOS 27 device management changes against an Intune-managed fleet before general availability.

**Artifacts:** `deliverables/` — deployable mobileconfig and DDM JSON files, one directory per phase.

---

## Environment Requirements

| Item | Requirement |
|---|---|
| Hardware | Apple Silicon Mac only (M-series) — Intel Macs are not supported in macOS 27 |
| Enrollment | Supervised via ADE (Apple Business Manager) — most new features require supervised |
| Intune | Tenant with DDM (Declarative Device Management) support; Settings Catalog schema updated for macOS 27 |
| Test devices | Minimum 2: one ADE-enrolled/supervised, one User Enrollment for user-scope tests |
| OS | macOS 27.0 Beta 2+ installed via Apple Seed or ABM beta program |
| AppleCare | AppleCare Enterprise agreement required for Enhanced Log Collection tests (P3-T03/T04) |

## Profile Deployment Methods in Intune

| Type | How to deploy in Intune | Used for |
|---|---|---|
| Legacy mobileconfig (XML) | Devices → macOS → Configuration profiles → Create → Custom | Traditional payload types (restrictions, PPPC legacy, SSO, etc.) |
| DDM Declaration (JSON) | Devices → macOS → Configuration profiles → Settings Catalog (macOS 27 entries) | New DDM-only declarations: app.settings, content-cache.settings, network VPN/DNS, intelligence.settings |
| MDM Commands | Intune remote actions or Graph API | TriggerEnhancedLogCollection, CancelEnhancedLogCollection |

> **Note:** For DDM declarations not yet in Intune's Settings Catalog, use the Graph API endpoint `deviceManagement/deviceConfigurations` with a custom declarative JSON body, or pilot via Apple Configurator 3 / another MDM that has day-one DDM schema support (Jamf Pro 11.x+).

---

## Test Matrix Summary

| Priority | Area | Tests | Artifacts | Risk if Skipped |
|---|---|---|---|---|
| P0 | Breaking Changes — Software Update | 3 | **Ready** (`phase0-breaking-changes/`) | Fleet loses update enforcement on day 1 of upgrade |
| P0 | Breaking Changes — PPPC / Restrictions | 4 | **Ready** (`phase0-breaking-changes/`) | Privacy controls silently stop working |
| P1 | New Security: Binary Control | 3 | **Ready** (`phase1-security-controls/`) | Missed hardening opportunity; compliance gap |
| P1 | New Security: Privacy Defaults (PPPC Replacement) | 2 | See P0-T05 | User permission prompts behave unexpectedly |
| P2 | New Network DDM Configs | 4 | **Ready** (`phase2-network-ddm/`) | VPN/DNS not deployable via new declarative path |
| P3 | New MDM Features | 4 | **Ready** (`phase3-new-features/`) | Enhanced logging, PlatformSSO gaps |
| P3 | Content Caching DDM | 2 | **Ready** (`phase3-new-features/`) | Legacy AssetCache profile stops working |
| P4 | Status & Monitoring | 3 | Planned | Intune device health reporting incomplete |
| P5 | Setup Assistant / Enrollment | 3 | Planned | Enrollment UX broken by new setup panes |

---

## P0 — Breaking Changes: Software Update Management

### Background
All legacy software update MDM commands and restrictions are **removed** in macOS 27. The `com.apple.SoftwareUpdate` profile payload type is deleted. Organizations **must** migrate to DDM before upgrading their fleet.

**Removed (will break silently or error):**
- MDM commands: `ScheduleOSUpdate`, `AvailableOSUpdates`, `OSUpdateStatus`, `ScheduleOSUpdateScan`
- Profile payload: `com.apple.SoftwareUpdate` (entire payload type deleted)
- Restriction keys from `com.apple.applicationaccess`:
  - `enforcedSoftwareUpdateDelay`
  - `enforcedSoftwareUpdateMajorOSDeferredInstallDelay`
  - `enforcedSoftwareUpdateMinorOSDeferredInstallDelay`
  - `enforcedSoftwareUpdateNonOSDeferredInstallDelay`
  - `forceDelayedAppSoftwareUpdates`
  - `forceDelayedMajorSoftwareUpdates`
  - `forceDelayedSoftwareUpdates`
  - `allowRapidSecurityResponseInstallation`
  - `allowRapidSecurityResponseRemoval`
- DeviceInformation query keys: `OSUpdateSettings`, `SoftwareUpdateDeviceID`, `SoftwareUpdateSettings`
- Settings command: `SoftwareUpdateSettings`

---

### P0-T01: Legacy SoftwareUpdate Profile Removed

> **Artifact:** `deliverables/phase0-breaking-changes/P0-T01-legacy-softwareupdate.mobileconfig`

**What:** Verify that a deployed `com.apple.SoftwareUpdate` profile is rejected or ignored on macOS 27.

**Profile to test (upload to Intune as Custom profile — expect rejection):**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>PayloadContent</key>
    <array>
        <dict>
            <key>PayloadType</key>
            <string>com.apple.SoftwareUpdate</string>
            <key>PayloadVersion</key>
            <integer>1</integer>
            <key>PayloadIdentifier</key>
            <string>com.test.macos27.p0-t01.softwareupdate</string>
            <key>PayloadUUID</key>
            <string>A1B2C3D4-0001-0001-0001-000000000001</string>
            <key>PayloadDisplayName</key>
            <string>P0-T01 Legacy SoftwareUpdate Test</string>
            <key>AutomaticCheckEnabled</key>
            <true/>
            <key>AutomaticDownload</key>
            <true/>
            <key>AutomaticallyInstallMacOSUpdates</key>
            <false/>
            <key>CriticalUpdateInstall</key>
            <true/>
        </dict>
    </array>
    <key>PayloadDisplayName</key>
    <string>P0-T01 Legacy SoftwareUpdate Test</string>
    <key>PayloadIdentifier</key>
    <string>com.test.macos27.p0-t01</string>
    <key>PayloadType</key>
    <string>Configuration</string>
    <key>PayloadUUID</key>
    <string>A1B2C3D4-0001-0001-0001-000000000000</string>
    <key>PayloadVersion</key>
    <integer>1</integer>
</dict>
</plist>
```

**Steps:**
1. Upload profile to Intune as a Custom configuration profile for macOS.
2. Assign to a macOS 27 test device group.
3. Sync the device.
4. Check device profile installation status in Intune.
5. On device: System Settings → General → Device Management → verify profile status.

**Pass Criteria:** Profile installation fails or reports an error. The `com.apple.SoftwareUpdate` payload type is no longer accepted. No unintended update behavior occurs.

---

### P0-T02: DDM Software Update Enforcement (Replacement Path)

> **Artifacts:** `deliverables/phase0-breaking-changes/P0-T02-swu-settings.json` and `P0-T02-swu-enforcement.json` — deploy both.

**What:** Verify that `com.apple.configuration.softwareupdate.enforcement.specific` and `com.apple.configuration.softwareupdate.settings` function as the required replacement for legacy update management.

**DDM Declarations to deploy via Intune Settings Catalog (macOS 27 → Software Update section):**

*Declaration 1 — softwareupdate.settings (JSON for manual DDM deployment):*

```json
{
  "Type": "com.apple.configuration.softwareupdate.settings",
  "Identifier": "com.test.macos27.p0-t02.swu-settings",
  "ServerToken": "",
  "Payload": {
    "Notifications": true,
    "Deferrals": {
      "MajorPeriodInDays": 14,
      "MinorPeriodInDays": 7,
      "SystemPeriodInDays": 3
    },
    "AutomaticActions": {
      "Download": "AlwaysOn",
      "InstallOSUpdates": "Allowed",
      "InstallSecurityUpdate": "AlwaysOn"
    },
    "AllowStandardUserOSUpdates": false
  }
}
```

*Declaration 2 — softwareupdate.enforcement.specific:*

```json
{
  "Type": "com.apple.configuration.softwareupdate.enforcement.specific",
  "Identifier": "com.test.macos27.p0-t02.swu-enforce",
  "ServerToken": "",
  "Payload": {
    "TargetOSVersion": "27.0",
    "TargetLocalDateTime": "2026-11-01T09:00:00",
    "DetailsURL": "https://yourcompany.com/macos27-update-info"
  }
}
```

**Steps:**
1. Remove any existing legacy SoftwareUpdate profiles from device.
2. Deploy both DDM declarations via Intune Settings Catalog.
3. On device: System Settings → General → Software Update — verify update deferral countdown shows 14 days for major, 7 days for minor.
4. Verify standard user cannot install major OS updates (standard user account test).
5. Verify enforcement countdown appears and deadline notification fires at T-1 hour.
6. Check `softwareupdate.install-state` status item in Intune device report.

**Pass Criteria:**
- Update deferrals visible and enforced in System Settings.
- Standard user cannot trigger major OS update.
- Enforcement deadline notification appears correctly.
- `softwareupdate.install-state` DDM status item reports in Intune.

---

### P0-T03: Removed Restriction Keys — Software Update Deferrals

> **Artifact:** `deliverables/phase0-breaking-changes/P0-T03-removed-restriction-keys.mobileconfig`

**What:** Verify that removed restriction keys (`enforcedSoftwareUpdateDelay`, `forceDelayedSoftwareUpdates`, etc.) no longer function or cause errors on macOS 27. This is a regression check.

**Profile (test that removed keys are silently ignored or rejected):**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>PayloadContent</key>
    <array>
        <dict>
            <key>PayloadType</key>
            <string>com.apple.applicationaccess</string>
            <key>PayloadVersion</key>
            <integer>1</integer>
            <key>PayloadIdentifier</key>
            <string>com.test.macos27.p0-t03.restrictions</string>
            <key>PayloadUUID</key>
            <string>A1B2C3D4-0003-0003-0003-000000000003</string>
            <key>PayloadDisplayName</key>
            <string>P0-T03 Removed SU Restriction Keys</string>
            <key>enforcedSoftwareUpdateDelay</key>
            <integer>30</integer>
            <key>enforcedSoftwareUpdateMajorOSDeferredInstallDelay</key>
            <integer>30</integer>
            <key>enforcedSoftwareUpdateMinorOSDeferredInstallDelay</key>
            <integer>14</integer>
            <key>forceDelayedSoftwareUpdates</key>
            <true/>
            <key>forceDelayedMajorSoftwareUpdates</key>
            <true/>
            <key>allowRapidSecurityResponseInstallation</key>
            <false/>
            <key>allowRapidSecurityResponseRemoval</key>
            <false/>
        </dict>
    </array>
    <key>PayloadDisplayName</key>
    <string>P0-T03 Removed SU Restriction Keys Test</string>
    <key>PayloadIdentifier</key>
    <string>com.test.macos27.p0-t03</string>
    <key>PayloadType</key>
    <string>Configuration</string>
    <key>PayloadUUID</key>
    <string>A1B2C3D4-0003-0003-0003-000000000000</string>
    <key>PayloadVersion</key>
    <integer>1</integer>
</dict>
</plist>
```

**Steps:**
1. Deploy profile to macOS 27 device.
2. Verify profile installs (payload type itself still exists — keys removed but type not).
3. On device: confirm no software update deferrals are applied (updates show immediately).
4. Verify no errors in MDM logs from unrecognized keys.

**Pass Criteria:** Profile installs, removed keys are silently ignored, no update deferrals applied (macOS 27 ignores deleted keys). Intune shows profile as installed successfully.

---

## P0 — Breaking Changes: PPPC and Restrictions

### P0-T04: PPPC Profile Deprecation Validation

> **Artifact:** `deliverables/phase0-breaking-changes/P0-T04-legacy-pppc.mobileconfig`

**What:** Verify that `com.apple.TCC.configuration-profile-policy` (PPPC) still installs but confirm behavior is deprecated. Test that the new `com.apple.configuration.app.settings` Privacy section is the correct replacement.

**Legacy PPPC profile (deploy to verify deprecation warning):**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>PayloadContent</key>
    <array>
        <dict>
            <key>PayloadType</key>
            <string>com.apple.TCC.configuration-profile-policy</string>
            <key>PayloadVersion</key>
            <integer>1</integer>
            <key>PayloadIdentifier</key>
            <string>com.test.macos27.p0-t04.pppc-legacy</string>
            <key>PayloadUUID</key>
            <string>A1B2C3D4-0004-0004-0004-000000000004</string>
            <key>PayloadDisplayName</key>
            <string>P0-T04 Legacy PPPC Test</string>
            <key>Services</key>
            <dict>
                <key>Microphone</key>
                <array>
                    <dict>
                        <key>Allowed</key>
                        <true/>
                        <key>CodeRequirement</key>
                        <string>identifier "com.microsoft.teams" and anchor apple generic</string>
                        <key>Identifier</key>
                        <string>com.microsoft.teams</string>
                        <key>IdentifierType</key>
                        <string>bundleID</string>
                    </dict>
                </array>
                <key>Camera</key>
                <array>
                    <dict>
                        <key>Allowed</key>
                        <true/>
                        <key>CodeRequirement</key>
                        <string>identifier "com.microsoft.teams" and anchor apple generic</string>
                        <key>Identifier</key>
                        <string>com.microsoft.teams</string>
                        <key>IdentifierType</key>
                        <string>bundleID</string>
                    </dict>
                </array>
            </dict>
        </dict>
    </array>
    <key>PayloadDisplayName</key>
    <string>P0-T04 Legacy PPPC</string>
    <key>PayloadIdentifier</key>
    <string>com.test.macos27.p0-t04</string>
    <key>PayloadType</key>
    <string>Configuration</string>
    <key>PayloadUUID</key>
    <string>A1B2C3D4-0004-0004-0004-000000000000</string>
    <key>PayloadVersion</key>
    <integer>1</integer>
</dict>
</plist>
```

**Steps:**
1. Deploy legacy PPPC profile to macOS 27 device.
2. Verify installation status in Intune.
3. Launch Microsoft Teams on the device and observe camera/microphone permission behavior.
4. Check System Settings → Privacy & Security for override status.
5. Review system logs for TCC deprecation warnings: `log show --predicate 'process == "tccd"' --last 5m`

**Pass Criteria:** Legacy profile installs (still accepted in 27.0 as deprecated, not removed). Document observed behavior for migration planning. Record whether privacy controls still apply — this informs fleet migration timeline.

---

### P0-T05: New PPPC Replacement — app.settings Privacy Defaults

> **Artifact:** `deliverables/phase0-breaking-changes/P0-T05-app-settings-privacy.json`

**What:** Verify `com.apple.configuration.app.settings` Privacy.PermissionDefaults functions as the PPPC replacement and presents the single consolidated consent prompt.

**DDM Declaration (JSON for Intune Settings Catalog or custom deployment):**

```json
{
  "Type": "com.apple.configuration.app.settings",
  "Identifier": "com.test.macos27.p0-t05.app-privacy",
  "ServerToken": "",
  "Payload": {
    "Privacy": {
      "PermissionDefaults": {
        "com.microsoft.teams (2BUA8C4S2C)": {
          "OrganizationJustification": "Microsoft Teams requires camera and microphone access for video calls. Your organization uses Teams for all video conferencing.",
          "Camera": "Allow",
          "Microphone": "Allow",
          "Bluetooth": "Allow"
        },
        "com.microsoft.onenote (UBF8T346G9)": {
          "OrganizationJustification": "OneNote requires microphone access for audio note recording.",
          "Microphone": "Allow"
        },
        "com.google.Chrome (EQHXZ8M8AV)": {
          "OrganizationJustification": "Google Chrome requires camera and microphone access for web-based video conferencing tools.",
          "Camera": "Allow",
          "Microphone": "Allow",
          "LocalNetwork": "Allow"
        }
      }
    }
  }
}
```

**Steps:**
1. Remove any existing PPPC profiles.
2. Deploy app.settings DDM declaration.
3. Launch Microsoft Teams for the first time on a fresh user account.
4. Observe the consent prompt — verify it shows a single consolidated dialog listing all permissions (Camera, Microphone, Bluetooth) with the `OrganizationJustification` text.
5. Click "Allow" — verify all three permissions are granted simultaneously.
6. Relaunch with a second user account → click "Not Now" — verify individual permission prompts appear on first use.
7. Check System Settings → Privacy & Security → Camera/Microphone to confirm allow state.

**Pass Criteria:** Single consolidated consent prompt appears with custom organization text. Accepting grants all listed permissions atomically. Declining reverts to standard per-subsystem prompting.

---

### P0-T06: New Lock Screen Restriction Keys

> **Artifact:** `deliverables/phase0-breaking-changes/P0-T06-lockscreen-restrictions.mobileconfig`

**What:** Validate the two new keys in `com.apple.applicationaccess`: `ForceCaptivePortalConnectionFromLockScreen` and `ForceWifiConfigurationOnLockScreen`.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>PayloadContent</key>
    <array>
        <dict>
            <key>PayloadType</key>
            <string>com.apple.applicationaccess</string>
            <key>PayloadVersion</key>
            <integer>1</integer>
            <key>PayloadIdentifier</key>
            <string>com.test.macos27.p0-t06.lockscreen-restrictions</string>
            <key>PayloadUUID</key>
            <string>A1B2C3D4-0006-0006-0006-000000000006</string>
            <key>PayloadDisplayName</key>
            <string>P0-T06 Lock Screen Restrictions</string>
            <key>ForceCaptivePortalConnectionFromLockScreen</key>
            <true/>
            <key>ForceWifiConfigurationOnLockScreen</key>
            <true/>
        </dict>
    </array>
    <key>PayloadDisplayName</key>
    <string>P0-T06 Lock Screen Restrictions Test</string>
    <key>PayloadIdentifier</key>
    <string>com.test.macos27.p0-t06</string>
    <key>PayloadType</key>
    <string>Configuration</string>
    <key>PayloadUUID</key>
    <string>A1B2C3D4-0006-0006-0006-000000000000</string>
    <key>PayloadVersion</key>
    <integer>1</integer>
</dict>
</plist>
```

**Steps:**
1. Deploy profile to supervised macOS 27 device.
2. Lock the screen.
3. Connect to a network with a captive portal — verify the captive portal connection dialog appears on the lock screen.
4. Navigate to Wi-Fi settings from the lock screen — verify Wi-Fi configuration is accessible before authentication.

**Pass Criteria:** Both lock screen features activate as configured. Profile installs cleanly in Intune.

---

### P0-T07: PlatformSSO New Keys — AllowWebLoginPasswordSync and WebLoginURLAllowList

> **Artifact:** `deliverables/phase0-breaking-changes/P0-T07-platformsso-weblogin.mobileconfig`

**What:** Validate two new PlatformSSO keys in `com.apple.extensiblesso` that control web login password sync behavior. Requires a PlatformSSO-capable extension (e.g., Microsoft Enterprise SSO plugin).

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>PayloadContent</key>
    <array>
        <dict>
            <key>PayloadType</key>
            <string>com.apple.extensiblesso</string>
            <key>PayloadVersion</key>
            <integer>1</integer>
            <key>PayloadIdentifier</key>
            <string>com.test.macos27.p0-t07.psso</string>
            <key>PayloadUUID</key>
            <string>A1B2C3D4-0007-0007-0007-000000000007</string>
            <key>PayloadDisplayName</key>
            <string>P0-T07 PlatformSSO Web Login Test</string>
            <key>ExtensionIdentifier</key>
            <string>com.microsoft.CompanyPortalMac.ssoextension</string>
            <key>TeamIdentifier</key>
            <string>UBF8T346G9</string>
            <key>Type</key>
            <string>Redirect</string>
            <key>URLs</key>
            <array>
                <string>https://login.microsoftonline.com</string>
                <string>https://login.microsoft.com</string>
            </array>
            <key>PlatformSSO</key>
            <dict>
                <key>AuthenticationMethod</key>
                <string>Password</string>
                <key>AllowWebLoginPasswordSync</key>
                <true/>
                <key>WebLoginURLAllowList</key>
                <array>
                    <string>https://login.microsoftonline.com</string>
                    <string>https://myapps.microsoft.com</string>
                    <string>https://portal.azure.com</string>
                </array>
            </dict>
        </dict>
    </array>
    <key>PayloadDisplayName</key>
    <string>P0-T07 PlatformSSO Web Login Keys</string>
    <key>PayloadIdentifier</key>
    <string>com.test.macos27.p0-t07</string>
    <key>PayloadType</key>
    <string>Configuration</string>
    <key>PayloadUUID</key>
    <string>A1B2C3D4-0007-0007-0007-000000000000</string>
    <key>PayloadVersion</key>
    <integer>1</integer>
</dict>
</plist>
```

**Steps:**
1. Ensure Microsoft Enterprise SSO extension is deployed.
2. Deploy profile with `AllowWebLoginPasswordSync: true` and defined `WebLoginURLAllowList`.
3. Log in via Safari to a URL in the allow list — verify SSO extension syncs the password/token.
4. Attempt login to a URL NOT in the allow list — verify SSO does not sync the password.
5. Set `AllowWebLoginPasswordSync: false` and repeat — verify no password sync occurs for any web login.

**Pass Criteria:** Password sync only occurs for URLs in the allow list when `AllowWebLoginPasswordSync` is true. Non-listed URLs do not trigger sync. Disabling the key prevents all web login syncing.

---

## P1 — New Security Controls: Binary Control

### Background
macOS 27 introduces native binary allow/deny lists via the Endpoint Security framework through the `com.apple.configuration.app.settings` DDM declaration. This replaces the need for third-party tools for application allowlisting and maps to NIST SP 800-53 and ISO 27001:2022 software-integrity requirements.

**Key identifiers (run `codesign -dvvv <path_to_binary>` on device to extract values):**
- `CDHash`: Cryptographic hash of code directory — most specific, locks to exact build
- `TeamID`: All software signed by a developer team — broadest
- `SigningID`: Specific application within a team's suite
- `PathPrefix`: File system path constraint
- `SigningState`: One of All, TestFlight, DeveloperID, Enterprise, AppStore, Apple

**Special value:** Use `"*APPLE*"` as TeamID for Apple-signed binaries with empty team identifiers.

---

### P1-T01: Binary AllowList — Enforce Allowed Binaries

**What:** Deploy `AllowedBinaries` configuration to enforce an allowlist of approved applications. The `AlwaysAllowManagedApps: true` key ensures MDM-deployed apps are automatically included.

**DDM Declaration (JSON):**

```json
{
  "Type": "com.apple.configuration.app.settings",
  "Identifier": "com.test.macos27.p1-t01.binary-allowlist",
  "ServerToken": "",
  "Payload": {
    "Allowed": {
      "AlwaysAllowManagedApps": true,
      "AllowedBinaries": [
        {
          "BinaryIdentifier": {
            "TeamID": "*APPLE*",
            "SigningState": "Apple",
            "SigningID": "com.apple.*"
          }
        },
        {
          "BinaryIdentifier": {
            "TeamID": "UBF8T346G9",
            "SigningState": "AppStore"
          }
        },
        {
          "BinaryIdentifier": {
            "TeamID": "2BUA8C4S2C",
            "SigningState": "DeveloperID"
          }
        },
        {
          "BinaryIdentifier": {
            "TeamID": "EQHXZ8M8AV",
            "SigningState": "DeveloperID",
            "SigningID": "com.google.Chrome"
          }
        }
      ]
    }
  }
}
```

**Steps:**
1. Deploy declaration — `AlwaysAllowManagedApps: true` ensures already-managed apps run.
2. Attempt to launch an Apple-signed system app (Terminal) — should run.
3. Attempt to launch a Microsoft 365 app (Teams, Word) — should run.
4. Attempt to run an unsigned/downloaded binary from `/tmp` — should be blocked.
5. Attempt to run a DeveloperID-signed binary not in TeamID list — should be blocked.
6. Run `codesign -dvvv /path/to/blocked/binary` to confirm why it was blocked.

**Pass Criteria:** Apple and Microsoft (UBF8T346G9) binaries run. Unlisted unsigned binaries are blocked by the system. System critical processes are never blocked.

---

### P1-T02: Binary DenyList — Block Specific Binaries

**What:** Deploy `DeniedBinaries` to block specific applications by CDHash (exact build) or SigningID.

**DDM Declaration (JSON):**

```json
{
  "Type": "com.apple.configuration.app.settings",
  "Identifier": "com.test.macos27.p1-t02.binary-denylist",
  "ServerToken": "",
  "Payload": {
    "Allowed": {
      "DeniedBinaries": [
        {
          "BinaryIdentifier": {
            "TeamID": "EQHXZ8M8AV",
            "SigningID": "com.google.Chrome",
            "SigningState": "DeveloperID"
          }
        },
        {
          "BinaryIdentifier": {
            "CDHash": "REPLACE_WITH_ACTUAL_CDHASH_FROM_CODESIGN_OUTPUT",
            "TeamID": "REPLACE_WITH_TEAMID"
          }
        }
      ]
    }
  }
}
```

**Steps:**
1. Run `codesign -dvvv /Applications/Google\ Chrome.app` and capture CDHash and TeamID.
2. Update declaration with real CDHash values.
3. Deploy declaration.
4. Attempt to launch Chrome — should be blocked.
5. Attempt to launch Safari — should run (not in deny list).
6. Verify Endpoint Security framework blocks the process at launch.

**Pass Criteria:** Chrome launch is blocked. Other browsers/apps not in deny list are unaffected. Block is logged in Unified Log (`log show --predicate 'subsystem == "com.apple.endpointsecurity"' --last 5m`).

---

### P1-T03: Binary Control — Combined AllowList + AlwaysAllowManagedApps

**What:** Validate the `AlwaysAllowManagedApps` key as a safety net — confirm MDM-deployed apps are automatically included in the effective allow list without being explicitly listed.

**DDM Declaration (JSON):**

```json
{
  "Type": "com.apple.configuration.app.settings",
  "Identifier": "com.test.macos27.p1-t03.managed-apps-passthrough",
  "ServerToken": "",
  "Payload": {
    "Allowed": {
      "AlwaysAllowManagedApps": true,
      "AllowedBinaries": [
        {
          "BinaryIdentifier": {
            "TeamID": "*APPLE*",
            "SigningState": "Apple"
          }
        }
      ]
    }
  }
}
```

**Steps:**
1. Pre-deploy Microsoft Defender for Endpoint via Intune as a managed app.
2. Deploy this declaration (only Apple-signed in explicit list).
3. Launch Microsoft Defender — should run despite TeamID `UBF8T346G9` not being listed, because it's a managed app.
4. Sideload an unmanaged binary with the same TeamID — should be blocked (not an MDM-managed app).

**Pass Criteria:** MDM-managed apps bypass the AllowedBinaries restriction. Unmanaged apps from the same vendor are blocked. This validates the `AlwaysAllowManagedApps` scope — it applies only to apps explicitly deployed through MDM, not all apps from that developer.

---

## P2 — New Network DDM Configurations

### Background
macOS 27 introduces six new declarative network configuration types. These use credential asset declarations for lifecycle management (ACME, SCEP, identity), simplifying certificate renewal vs. embedding in profiles.

---

### P2-T01: DDM IKEv2 VPN Configuration

**What:** Deploy an IKEv2 VPN using the new `com.apple.configuration.network.vpn.ikev2` declaration, with a certificate credential asset reference.

**DDM Declarations (JSON — deploy as a set):**

*Asset Declaration — Certificate for VPN authentication:*

```json
{
  "Type": "com.apple.asset.credential.scep",
  "Identifier": "com.test.macos27.p2-t01.vpn-cert",
  "ServerToken": "",
  "Payload": {
    "Name": "VPN Client Certificate",
    "URL": "https://scep.yourcompany.com/scep",
    "Subject": [
      [["CN", "{{SERIALNUMBER}}"]]
    ],
    "Challenge": "REPLACE_WITH_SCEP_CHALLENGE",
    "KeyType": "RSA",
    "KeySize": 2048,
    "KeyUsage": 5
  }
}
```

*VPN Configuration Declaration:*

```json
{
  "Type": "com.apple.configuration.network.vpn.ikev2",
  "Identifier": "com.test.macos27.p2-t01.ikev2-vpn",
  "ServerToken": "",
  "Payload": {
    "Name": "Corporate IKEv2 VPN (DDM)",
    "Server": "vpn.yourcompany.com",
    "AuthenticationMethod": "Certificate",
    "ClientCertificateAssetReference": "com.test.macos27.p2-t01.vpn-cert",
    "IKESecurityAssociationParameters": {
      "EncryptionAlgorithm": "AES-256-GCM",
      "IntegrityAlgorithm": "SHA2-384",
      "DiffieHellmanGroup": 20
    }
  }
}
```

**Steps:**
1. Deploy both declarations (asset first, then VPN configuration).
2. Check System Settings → VPN — verify "Corporate IKEv2 VPN (DDM)" appears.
3. Connect to VPN — verify certificate auth succeeds.
4. Verify credential lifecycle: revoke cert → verify VPN prompts for reenrollment rather than hard-failing.

**Pass Criteria:** VPN connection established via DDM declaration. Certificate managed via SCEP asset reference. Profile-level VPN is NOT required. Credential renewal works without profile reinstall.

---

### P2-T02: DDM DNS Settings (Encrypted DNS)

**What:** Deploy encrypted DNS (DoH/DoT) via `com.apple.configuration.network.dns-settings` DDM declaration.

**DDM Declaration (JSON):**

```json
{
  "Type": "com.apple.configuration.network.dns-settings",
  "Identifier": "com.test.macos27.p2-t02.dns-settings",
  "ServerToken": "",
  "Payload": {
    "DNSProtocol": "HTTPS",
    "ServerURL": "https://dns.yourcompany.com/dns-query",
    "ServerAddresses": [
      "192.168.1.53",
      "192.168.1.54"
    ],
    "OnDemandRules": [
      {
        "Action": "Connect",
        "InterfaceTypeMatch": "WiFi"
      },
      {
        "Action": "Connect",
        "InterfaceTypeMatch": "Cellular"
      }
    ]
  }
}
```

**Steps:**
1. Deploy DNS settings declaration.
2. On device: run `scutil --dns` — verify the corporate DoH server appears as primary resolver.
3. Capture DNS traffic with `tcpdump` — verify queries are sent over HTTPS (port 443) not UDP/53.
4. Disconnect from Wi-Fi and reconnect — verify DNS settings persist.

**Pass Criteria:** DoH resolver is applied system-wide. DNS queries go to corporate resolver. Configuration persists across network changes. No legacy `com.apple.dnsSettings.managed` profile required.

---

### P2-T03: DDM Always-On VPN

**What:** Validate `com.apple.configuration.network.vpn.always-on` for supervised devices — network traffic is routed through VPN before user login.

**DDM Declaration (JSON):**

```json
{
  "Type": "com.apple.configuration.network.vpn.always-on",
  "Identifier": "com.test.macos27.p2-t03.always-on-vpn",
  "ServerToken": "",
  "Payload": {
    "TunnelConfigurations": [
      {
        "Identifier": "com.test.macos27.p2-t03.ikev2-always-on",
        "ServiceType": "IKEv2",
        "Server": "always-on-vpn.yourcompany.com"
      }
    ],
    "AllowedCaptiveNetworkPlugins": [
      {
        "BundleID": "com.apple.captiveagent"
      }
    ]
  }
}
```

**Steps:**
1. Requires supervised macOS 27 device.
2. Deploy always-on VPN declaration.
3. Reboot device — verify VPN connects before user login screen appears.
4. Verify network traffic captured before authentication is tunneled.
5. Attempt to bypass by disabling VPN in System Settings — verify the option is grayed out.
6. Test captive portal exception — verify captive portal still works despite always-on VPN.

**Pass Criteria:** VPN connects at boot before user login. Cannot be disabled by end user. Captive portal exception allows Wi-Fi network authentication.

---

### P2-T04: DDM Network Relay

**What:** Validate `com.apple.configuration.network.relay` for routing traffic through a relay server (HTTP/3 MASQUE or other relay protocols).

**DDM Declaration (JSON):**

```json
{
  "Type": "com.apple.configuration.network.relay",
  "Identifier": "com.test.macos27.p2-t04.network-relay",
  "ServerToken": "",
  "Payload": {
    "Relays": [
      {
        "HTTP3RelayURL": "https://relay.yourcompany.com/relay",
        "RawPublicKeys": []
      }
    ],
    "MatchDomains": [
      "internal.yourcompany.com",
      "corp.yourcompany.com"
    ],
    "ExcludedDomains": [
      "public.yourcompany.com"
    ]
  }
}
```

**Steps:**
1. Deploy relay declaration.
2. Access `internal.yourcompany.com` — verify traffic routes through relay.
3. Access `public.yourcompany.com` — verify traffic does NOT route through relay (excluded domain).
4. Check `netstat` or `Charles Proxy` to verify relay routing.

**Pass Criteria:** Matched domain traffic routes through corporate relay. Excluded domains bypass relay. No legacy proxy profile required.

---

## P3 — New MDM Features

### P3-T01: Web Content Filter via DDM (webcontent-filter.plugin)

**What:** Deploy a web content filter using the new `com.apple.configuration.webcontent-filter.plugin` DDM declaration — replaces the legacy `com.apple.webcontent-filter` profile. Requires a compatible Network Extension content filter app.

**DDM Declaration (JSON):**

```json
{
  "Type": "com.apple.configuration.webcontent-filter.plugin",
  "Identifier": "com.test.macos27.p3-t01.content-filter",
  "ServerToken": "",
  "Payload": {
    "VisibleName": "Corporate Web Filter (DDM)",
    "PluginBundleID": "com.yourvendor.contentfilter.extension",
    "ServerAddress": "filter.yourcompany.com",
    "Organization": "Your Company",
    "VendorConfig": {
      "LicenseKey": "REPLACE_WITH_LICENSE_KEY",
      "PolicyGroup": "corporate-default"
    },
    "Filter": {
      "Grade": "inspector",
      "Sockets": {
        "Enabled": true,
        "ProviderComposedIdentifier": "com.yourvendor.contentfilter.dataprovider {anchor apple generic and identifier \"com.yourvendor.contentfilter.dataprovider\"}"
      }
    }
  }
}
```

**Steps:**
1. Ensure the content filter Network Extension app is installed (via Intune managed app).
2. Deploy the DDM declaration.
3. Browse to a blocked URL category — verify block page appears.
4. Browse to an allowed URL — verify normal access.
5. Check System Settings → Network — verify "Corporate Web Filter (DDM)" listed as active filter.
6. Verify filter shows Intune-labeled management (not a separate profile in profile list).

**Pass Criteria:** Content filter is active via DDM declaration. Blocked categories are enforced. Filter is tied to DDM lifecycle, not a separate mobileconfig profile.

---

### P3-T02: Content Caching via DDM (replaces com.apple.AssetCache.managed)

**What:** Deploy content caching on a supervised Mac using `com.apple.configuration.content-cache.settings` — the replacement for the deprecated `com.apple.AssetCache.managed` profile.

**DDM Declaration (JSON):**

```json
{
  "Type": "com.apple.configuration.content-cache.settings",
  "Identifier": "com.test.macos27.p3-t02.content-cache",
  "ServerToken": "",
  "Payload": {
    "AutoActivation": true,
    "AllowPersonalCaching": false,
    "AllowSharedCaching": true,
    "CacheLimit": 107374182400,
    "LocalSubnetsOnly": false,
    "ListenRanges": [
      {
        "type": "IPv4",
        "first": "10.0.0.1",
        "last": "10.0.255.254"
      }
    ],
    "ListenRangesOnly": true,
    "KeepAwake": true,
    "DisplayAlerts": false,
    "DeclarativeStatusInterval": 300,
    "ManagementStatusTarget": "https://monitoring.yourcompany.com/content-cache",
    "ManagementSecurityConfig": "no-cert",
    "ManagementReportingInterval": 300
  }
}
```

**Steps:**
1. Deploy on a supervised Mac on a corporate network (Mac Mini or MacBook Pro with adequate storage).
2. Check System Settings → General → Sharing → Content Caching — verify it's enabled and activated.
3. From a client Mac on the same subnet, download a large macOS app from the App Store — verify cache registers the download.
4. Download the same app from another client — verify it serves from cache (significantly faster).
5. Check DDM status items in Intune: `content-cache.info`, `content-cache.status`.
6. Verify management stats are reported to `ManagementStatusTarget`.

**Pass Criteria:** Content caching activates automatically. Shared caching serves subnet clients. Status items report in Intune. No `com.apple.AssetCache.managed` profile required.

---

### P3-T03: Enhanced Log Collection — Trigger

**What:** Test the new `TriggerEnhancedLogCollection` MDM command for remote diagnostic log collection. **Requires an AppleCare Enterprise agreement and AppleCare token.**

**Intune Steps (Graph API or Intune Remote Actions):**

```
POST https://graph.microsoft.com/beta/deviceManagement/managedDevices/{deviceId}/triggerEnhancedLogCollection
Body: {
  "appleCareSupportToken": "REPLACE_WITH_APPLECARE_TOKEN"
}
```

**Steps:**
1. Verify AppleCare Enterprise token is available in the Intune tenant.
2. Trigger log collection from Intune device → Remote actions → Trigger Enhanced Log Collection.
3. Check DDM status item `enhanced-logging.status` — verify it reports "collecting".
4. Monitor `enhanced-logging.timestamp` — note collection start time.
5. Wait for collection to complete — verify status changes to "complete".
6. Check `enhanced-logging.applecare-token` status item for the session reference.
7. Retrieve logs from AppleCare portal.

**Pass Criteria:** Log collection initiates via MDM command. Status items track progress. User receives a notification (non-interactive session). Logs are retrievable via AppleCare portal.

---

### P3-T04: Enhanced Log Collection — Cancel

**What:** Test `CancelEnhancedLogCollection` command to abort an in-progress log collection session.

**Steps:**
1. Trigger a log collection session (see P3-T03).
2. While `enhanced-logging.status` is "collecting", issue the cancel command via Intune remote action.
3. Verify `enhanced-logging.status` changes to "cancelled" or "idle".
4. Verify no partial logs are submitted to AppleCare.

**Pass Criteria:** In-progress collection is cleanly cancelled. Status item updates correctly. Repeat trigger after cancel functions normally.

---

## P3 — Intelligence Settings

### P3-T05: Intelligence Settings — Calendar Natural Language Editing (New in 27.0)

**What:** Validate the new `Apps.Calendar.AllowNaturalLanguageEditing` key added in macOS 27.0 (was not present in 26.x).

**DDM Declaration (JSON):**

```json
{
  "Type": "com.apple.configuration.intelligence.settings",
  "Identifier": "com.test.macos27.p3-t05.intelligence-calendar",
  "ServerToken": "",
  "Payload": {
    "AllowWritingTools": true,
    "AllowGenmoji": false,
    "AllowImagePlayground": false,
    "Apps": {
      "Calendar": {
        "AllowNaturalLanguageEditing": false
      },
      "Mail": {
        "AllowSmartReplies": true,
        "AllowSummary": true
      },
      "Safari": {
        "AllowSummary": false
      }
    }
  }
}
```

**Steps:**
1. Deploy declaration with `AllowNaturalLanguageEditing: false` for Calendar.
2. Open Calendar app → create a new event.
3. Verify natural language input (e.g., "lunch with Sarah next Tuesday at noon") does NOT auto-parse into a calendar event via Intelligence.
4. Update declaration to `AllowNaturalLanguageEditing: true`.
5. Sync and re-test — verify natural language event creation now works.

**Pass Criteria:** Natural language calendar editing is controlled by the MDM policy. Toggling the key has immediate effect after declaration update and sync.

---

## P4 — Status and Monitoring

### P4-T01: New MDM Status Items — Enrollment Type and MDM Properties

**What:** Verify new DDM status items report correctly in Intune device inventory.

**Status items to validate (via Intune device reports or Graph API):**

| Status Item | Expected Value |
|---|---|
| `mdm.enrollment-type` | `Supervised` (for ADE-enrolled device) |
| `mdm.is-awaiting-configuration` | `false` (post-enrollment) |
| `mdm.is-return-to-service` | `false` (normal device) |
| `mdm.is-shared-ipad` | `false` (standard Mac) |
| `mdm.push-magic` | (non-empty UUID string) |
| `mdm.push-token` | (non-empty base64 string) |

**Management Status Subscription Declaration (JSON):**

```json
{
  "Type": "com.apple.configuration.management.status-subscriptions",
  "Identifier": "com.test.macos27.p4-t01.status-subscriptions",
  "ServerToken": "",
  "Payload": {
    "StatusItems": [
      {"Name": "mdm.enrollment-type"},
      {"Name": "mdm.is-awaiting-configuration"},
      {"Name": "mdm.is-return-to-service"},
      {"Name": "mdm.push-magic"},
      {"Name": "mdm.push-token"},
      {"Name": "device.system.health"},
      {"Name": "security.lockdown-mode"}
    ]
  }
}
```

**Steps:**
1. Deploy status subscription declaration.
2. Check Intune device details page → Properties → look for new status fields.
3. Alternatively, query via Graph API: `GET /deviceManagement/managedDevices/{id}?$select=*`
4. Verify enrollment type shows "Supervised" for ADE device.
5. Enable Lockdown Mode on a test device (System Settings → Privacy & Security → Lockdown Mode).
6. Verify `security.lockdown-mode` status item reports `true` in Intune.

**Pass Criteria:** All new status items report expected values. `security.lockdown-mode` changes dynamically when Lockdown Mode is toggled. `mdm.enrollment-type` correctly identifies enrollment channel.

---

### P4-T02: Device System Health Status

**What:** Validate the new `device.system.health` status item for hardware component integrity reporting.

**Steps:**
1. Ensure `device.system.health` is included in the management status subscriptions declaration (see P4-T01).
2. Check device report in Intune for health data.
3. Query via Graph API for the raw status item value.
4. Compare reported health components against physical device (storage health, memory status).

**Pass Criteria:** `device.system.health` reports hardware component status. Data is accessible in Intune device inventory. Abnormal component health triggers appropriate alerting in Intune compliance policies.

---

### P4-T03: Backup Behavior Change — No MDM Enrollment from Backup

**What:** Verify the critical behavioral change: restoring a macOS 27 device from a Time Machine or iCloud backup does NOT restore MDM enrollment, supervision, or configuration profiles. Devices must re-enroll via ADE.

**Steps:**
1. Enroll a macOS 27 device in Intune via ADE.
2. Deploy several profiles and apps.
3. Create a Time Machine backup.
4. Erase the Mac (Apple menu → System Settings → General → Transfer or Reset → Erase All Content and Settings).
5. Restore from the Time Machine backup.
6. Check System Settings → General → Device Management — verify NO MDM profiles are present.
7. Go through Setup Assistant — verify ADE enrollment re-triggers automatically (device is supervised via ABM).
8. Confirm device re-enrolls cleanly and receives current profiles.

**Pass Criteria:** Backup restore does NOT restore MDM enrollment or profiles. ADE re-enrollment occurs automatically. This is the expected new behavior — confirm it works reliably and document for helpdesk runbook.

---

## P5 — Setup Assistant and Enrollment

### P5-T01: New Skip Keys — LiquidGlass and AccessibilityAppearance

**What:** Validate two new Setup Assistant skip keys added in macOS 27: `LiquidGlass` (skips the new Liquid Glass UI design introduction pane) and `AccessibilityAppearance` (skips the Accessibility Appearance selection pane shown during new user creation).

**Profile (deploy via Intune or embed in ADE enrollment profile):**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>PayloadContent</key>
    <array>
        <dict>
            <key>PayloadType</key>
            <string>com.apple.SetupAssistant.managed</string>
            <key>PayloadVersion</key>
            <integer>1</integer>
            <key>PayloadIdentifier</key>
            <string>com.test.macos27.p5-t01.setupassistant</string>
            <key>PayloadUUID</key>
            <string>A1B2C3D4-5001-5001-5001-000000000001</string>
            <key>PayloadDisplayName</key>
            <string>P5-T01 Setup Assistant Skip Keys</string>
            <key>SkipSetupItems</key>
            <array>
                <string>LiquidGlass</string>
                <string>AccessibilityAppearance</string>
                <string>Privacy</string>
                <string>SiriSetup</string>
                <string>iMessageAndFaceTime</string>
                <string>ScreenTime</string>
                <string>TOS</string>
                <string>iCloudStorage</string>
            </array>
        </dict>
    </array>
    <key>PayloadDisplayName</key>
    <string>P5-T01 Setup Assistant — macOS 27 Skip Keys</string>
    <key>PayloadIdentifier</key>
    <string>com.test.macos27.p5-t01</string>
    <key>PayloadType</key>
    <string>Configuration</string>
    <key>PayloadUUID</key>
    <string>A1B2C3D4-5001-5001-5001-000000000000</string>
    <key>PayloadVersion</key>
    <integer>1</integer>
</dict>
</plist>
```

**Steps:**
1. Add `LiquidGlass` and `AccessibilityAppearance` to the ADE enrollment profile skip keys in ABM (Apple Business Manager → Enrollment Profiles).
2. Alternatively, deploy the SetupAssistant.managed profile before a new user logs in.
3. Factory reset / erase a test Mac enrolled in ABM.
4. Go through Setup Assistant — verify the Liquid Glass introduction pane does NOT appear.
5. Log in as a new user — verify the Accessibility Appearance pane does NOT appear.
6. Verify all other specified skip items are also skipped.

**Pass Criteria:** `LiquidGlass` pane is skipped during initial Setup Assistant. `AccessibilityAppearance` pane is skipped during new user login Setup Assistant flow. No unexpected panes appear.

---

### P5-T02: Return to Service — ShouldRetryEnrollment

**What:** Validate the new `ShouldRetryEnrollment` key in the Return to Service erase command. When enrollment fails, the device retries with increasing delays (up to 5 minutes) rather than staying in a failed state.

**MDM Command (send via Graph API to trigger erase with Return to Service):**

```json
{
  "EraseDevice": {
    "ReturnToService": {
      "Enabled": true,
      "ShouldRetryEnrollment": true,
      "WifiSSID": "CorpNetwork",
      "WifiPassword": "REPLACE_WITH_WIFI_PASSWORD"
    }
  }
}
```

**Steps:**
1. Temporarily take the Intune MDM enrollment endpoint offline (test environment only — coordinate with IT).
2. Send erase + Return to Service command with `ShouldRetryEnrollment: true`.
3. Device erases and attempts enrollment — fails (enrollment endpoint offline).
4. Verify device retries enrollment with increasing delays (1 min, 2 min, 4 min... up to 5 min).
5. Bring MDM enrollment endpoint back online.
6. Verify device successfully enrolls on next retry.
7. Repeat test with `ShouldRetryEnrollment: false` — verify device stops after first failure.

**Pass Criteria:** Device automatically retries enrollment when `ShouldRetryEnrollment: true`. Retry delays increase progressively. Device recovers and fully enrolls once endpoint is available. With `false`, device stays in failed state.

---

### P5-T03: TLS Requirements — ATS Compliance for MDM Infrastructure

**What:** Verify that Intune MDM services meet the new App Transport Security (ATS) requirements mandated in macOS 27 for device management connections. This is an infrastructure validation test.

**Requirements to validate:**
- TLS 1.2 minimum (TLS 1.3 preferred)
- Forward secrecy cipher suites (ECDHE, no static RSA key exchange)
- ATS-compliant certificate (SHA-256 or better, RSA 2048+ or EC 256+)
- Valid certificate chain (no self-signed, no expired intermediates)

**Steps:**
1. Run SSL Labs test against Intune MDM endpoints (use Intune documentation for endpoint FQDNs).
2. On a macOS 27 device, test enrollment — verify it completes successfully.
3. Run `nscurl --ats-diagnostics https://manage.microsoft.com` to check ATS compliance from the device.
4. Check MDM communication logs via `log show --predicate 'subsystem == "com.apple.ManagedClient"' --last 1h` for TLS errors.

**Pass Criteria:** All Intune MDM endpoints pass ATS diagnostics. No TLS handshake failures in MDM communication logs. Enrollment and profile delivery complete without TLS errors. (Microsoft Intune's infrastructure should already be ATS-compliant — this is a verification test.)

---

## Appendix A: Intune DDM Declaration Deployment Reference

| Declaration Type | Intune Path | Notes |
|---|---|---|
| `com.apple.configuration.softwareupdate.settings` | Devices → macOS → Settings Catalog → Software Update | Available in Intune for macOS 15+ |
| `com.apple.configuration.softwareupdate.enforcement.specific` | Devices → macOS → Settings Catalog → Software Update → Enforce | Replaces legacy update deferral restriction keys |
| `com.apple.configuration.intelligence.settings` | Devices → macOS → Settings Catalog → Apple Intelligence | macOS 26.4+ scope |
| `com.apple.configuration.app.settings` | Devices → macOS → Settings Catalog → App Management | Binary control (macOS 27), Privacy defaults (macOS 27) |
| `com.apple.configuration.content-cache.settings` | Custom DDM JSON or Settings Catalog (when available) | macOS 27 supervised |
| `com.apple.configuration.network.vpn.*` | Custom DDM JSON or Settings Catalog | macOS 27, replaces legacy VPN profiles |
| `com.apple.configuration.webcontent-filter.plugin` | Custom DDM JSON | macOS 27 |

## Appendix B: Key File References in This Repo

| Area | File Path |
|---|---|
| App settings (binary control, privacy) | `declarative/declarations/configurations/app.settings.yaml` |
| Intelligence settings | `declarative/declarations/configurations/intelligence.settings.yaml` |
| External Intelligence | `declarative/declarations/configurations/external-intelligence.settings.yaml` |
| Content Cache DDM | `declarative/declarations/configurations/content-cache.settings.yaml` |
| Software Update enforcement | `declarative/declarations/configurations/softwareupdate.enforcement.specific.yaml` |
| Software Update settings | `declarative/declarations/configurations/softwareupdate.settings.yaml` |
| Web Content Filter | `declarative/declarations/configurations/webcontent-filter.plugin.yaml` |
| IKEv2 VPN | `declarative/declarations/configurations/network.vpn.ikev2.yaml` |
| Always-on VPN | `declarative/declarations/configurations/network.vpn.always-on.yaml` |
| DNS Settings | `declarative/declarations/configurations/network.dns-settings.yaml` |
| Network Relay | `declarative/declarations/configurations/network.relay.yaml` |
| Restrictions (applicationaccess) | `mdm/profiles/com.apple.applicationaccess.yaml` |
| Extensible SSO | `mdm/profiles/com.apple.extensiblesso.yaml` |
| Setup Assistant skip keys | `other/skipkeys.yaml` |
| Enhanced log commands | `mdm/commands/trigger.enhanced.log.collection.yaml`, `cancel.enhanced.log.collection.yaml` |
| All changes summary | `CHANGES.md` |

---

## Sources Referenced

- [Apple Device Management Client Schema repo — seed_OS_27_0](https://github.com/apple/device-management/tree/seed_OS_27_0) — `CHANGES.md`, `declarative/`, `mdm/`
- [Apple WWDC26 Device Management Updates](https://support.apple.com/guide/deployment/device-management-updates-depd638aa061/1/web/1.0)
- [Stabilise.io — macOS 27 Binary Control / PPPC Replacement analysis](https://stabilise.io/blog/macos-27-mdm-binary-control-pppc-replacement-mac-admins)
- [macOS 27 Release Notes](https://developer.apple.com/documentation/macos-release-notes/macos-27-release-notes)
- [Mac Admins News — WWDC26 Golden Gate coverage](https://www.macadmins.news/410-golden-gate/)
