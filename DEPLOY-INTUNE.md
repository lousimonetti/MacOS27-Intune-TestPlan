# How to Deploy These Artifacts in Microsoft Intune

This guide covers the three deployment paths used across all test phases.
Each artifact's README identifies which path to use.

---

## Quick Reference

| Artifact type | File extension | Intune path |
|---|---|---|
| Legacy mobileconfig | `.mobileconfig` | Custom Profile |
| DDM Declaration (profile-like) | `.json` | Settings Catalog **or** Custom DDM JSON |
| MDM Command reference | `.json` (with `_comment` key) | Graph API / Remote Actions |

---

## Path 1 — Custom Profile (mobileconfig)

Use for: `P0-T01`, `P0-T03`, `P0-T04`, `P0-T06`, `P0-T07`, `P5-T01`

1. Sign in to [Intune admin center](https://intune.microsoft.com).
2. Go to **Devices → macOS → Configuration**.
3. Click **Create → New Policy**.
4. Platform: **macOS** · Profile type: **Templates → Custom**.
5. Click **Create**.
6. Give the profile a name (e.g. `P0-T01 Legacy SoftwareUpdate Test`).
7. **Custom configuration profile name** — enter a display name.
8. **Deployment channel** — select **Device channel** (most profiles here are device-scoped).
9. Under **Configuration profile file**, click **Upload** and select the `.mobileconfig` file.
10. Click **Next** through Scope tags.
11. Under **Assignments**, add your `macOS-27-AppleSilicon` test device group.
12. Review and click **Create**.

**To verify deployment:** Device → view the device → **Device configuration** tab → find the profile → status should show **Succeeded**.

On the device: **System Settings → General → Device Management** → tap the profile to inspect keys.

---

## Path 2a — Settings Catalog (preferred for DDM declarations)

Use for: `P0-T02`, `P0-T05`, `P1-T01`–`P1-T03`, `P2-T02`, `P3-T05` — wherever the declaration type is surfaced in Intune's macOS 27 Settings Catalog.

1. Go to **Devices → macOS → Configuration**.
2. Click **Create → New Policy**.
3. Platform: **macOS** · Profile type: **Settings catalog**.
4. Click **Create** and give it a name.
5. Click **+ Add settings**.
6. Use the search bar to find the relevant category:
   - Software Update → maps to `softwareupdate.settings` / `softwareupdate.enforcement.specific`
   - App Management → maps to `app.settings` (binary control, privacy defaults)
   - Apple Intelligence → maps to `intelligence.settings`
   - Network → DNS → maps to `network.dns-settings`
7. Toggle the settings on and enter values matching the artifact JSON's `Payload` keys.
8. Assign to the `macOS-27-AppleSilicon` test group and create.

> **Tip:** If a setting is not listed in the Settings Catalog, fall back to Path 2b (Custom DDM JSON).

---

## Path 2b — Custom DDM JSON (when Settings Catalog doesn't have it yet)

Use for: `P1-T01`–`P1-T03`, `P2-T01`, `P2-T03`, `P2-T04`, `P3-T01`, `P3-T02`

### Option A — Graph API (recommended for automation)

```http
POST https://graph.microsoft.com/beta/deviceManagement/deviceConfigurations
Authorization: Bearer {token}
Content-Type: application/json

{
  "@odata.type": "#microsoft.graph.macOSCustomConfiguration",
  "displayName": "P2-T01 IKEv2 VPN DDM",
  "deploymentChannel": "deviceChannel",
  "payloadFileName": "P2-T01-ikev2-vpn.json",
  "payload": "<BASE64_ENCODED_JSON_FILE_CONTENTS>"
}
```

To base64-encode a file on macOS:

```bash
base64 -i deliverables/phase2-network-ddm/P2-T01-ikev2-vpn.json | pbcopy
```

Then paste the clipboard value as the `payload` string in the request body.

After creating the configuration, assign it to the test device group:

```http
POST https://graph.microsoft.com/beta/deviceManagement/deviceConfigurations/{configId}/assign
Content-Type: application/json

{
  "assignments": [
    {
      "target": {
        "@odata.type": "#microsoft.graph.groupAssignmentTarget",
        "groupId": "YOUR_AAD_GROUP_OBJECT_ID"
      }
    }
  ]
}
```

### Option B — Apple Configurator 3 / Jamf (pilot path)

If you have Jamf Pro 11.x+ or Apple Configurator 3 available for pilot testing:

1. In Apple Configurator 3: **File → New Profile → Declarations**.
2. Paste the JSON declaration body directly.
3. Install on a test device via USB or ABM assignment.

This is the fastest path when Intune hasn't yet surfaced a new DDM declaration type in the UI
and you need to validate behavior before committing to the Graph API path.

---

## Path 3 — MDM Commands via Remote Actions / Graph API

Use for: `P3-T03` (TriggerEnhancedLogCollection), `P3-T04` (CancelEnhancedLogCollection)

### Intune UI (Remote Actions)

1. Go to **Devices → All devices** → select your macOS 27 test device.
2. Click **...** (ellipsis) in the top action bar.
3. Look for **Collect diagnostics** or **Enhanced Log Collection** (if surfaced in the macOS 27 release of Intune).
4. Enter your AppleCare Enterprise token when prompted.

### Graph API

Use the reference body in `P3-T03-trigger-enhanced-log-collection.json`:

```http
POST https://graph.microsoft.com/beta/deviceManagement/managedDevices/{deviceId}/triggerEnhancedLogCollection
Authorization: Bearer {token}
Content-Type: application/json

{
  "appleCareSupportToken": "YOUR_APPLECARE_ENTERPRISE_TOKEN"
}
```

Get `{deviceId}` from: `GET /deviceManagement/managedDevices?$filter=deviceName eq 'your-mac-name'`

Required Graph permission: `DeviceManagementManagedDevices.PrivilegedOperations.All`

---

## Checking DDM Declaration Status on the Device

After deployment, confirm the declaration is active:

```bash
# List all active DDM declarations
profiles -e -o stdout-xml

# Check a specific declaration by identifier
profiles -e -o stdout-xml | grep -A 20 "com.test.macos27"

# Monitor MDM communication
log show --predicate 'subsystem == "com.apple.ManagedClient"' --last 1h

# Force an MDM sync
profiles -N
```

---

## Deployment Order for Declarations with Dependencies

Some declarations reference others by `Identifier`. Deploy in this order:

| Deploy first | Then deploy |
|---|---|
| `P2-T01-vpn-cert-scep.json` (credential asset) | `P2-T01-ikev2-vpn.json` (references the asset) |
| `P2-T01-vpn-cert-scep.json` (credential asset) | `P2-T03-always-on-vpn.json` (references the asset) |
| Vendor content filter app (managed app install) | `P3-T01-content-filter-plugin.json` |

---

## Common Issues

| Symptom | Likely cause | Fix |
|---|---|---|
| Profile status "Not applicable" in Intune | Device not in assigned group or wrong OS filter | Check group membership and macOS version filter |
| Profile status "Error" for mobileconfig | Payload type removed in macOS 27 (P0-T01 expected behavior) | This is the pass condition for P0-T01 |
| DDM declaration not appearing in `profiles -e` | Settings Catalog not yet updated for macOS 27 | Use Custom DDM JSON via Graph API |
| VPN cert not enrolling (P2-T01) | SCEP server unreachable or wrong challenge | Check SCEP URL and network connectivity |
| `TriggerEnhancedLogCollection` returns 403 | Missing `PrivilegedOperations.All` Graph permission | Add permission to app registration and re-consent |
| Binary blocked unexpectedly (P1-T01) | App TeamID or SigningState doesn't match declared rule | Run `codesign -dvvv <app>` and compare to declaration |
