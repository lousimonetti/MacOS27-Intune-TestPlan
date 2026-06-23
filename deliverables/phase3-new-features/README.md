# Phase 3 — New MDM Features (P3) Deliverables

Production artifacts for the **P3 — New MDM Features** tests in [`plan.md`](../../plan.md).
Covers the web content filter DDM declaration, content caching DDM replacement, enhanced log
collection commands, and the new Apple Intelligence Calendar key.

All JSON files validate with `python3 -m json.tool`.

## Files

| File | Test | Type | Deploy in Intune via | Notes |
|---|---|---|---|---|
| `P3-T01-content-filter-plugin.json` | P3-T01 | DDM `com.apple.configuration.webcontent-filter.plugin` | Custom DDM JSON | Requires vendor Network Extension app pre-installed |
| `P3-T02-content-cache.json` | P3-T02 | DDM `com.apple.configuration.content-cache.settings` | Custom DDM JSON or Settings Catalog | Supervised only; replaces `com.apple.AssetCache.managed` |
| `P3-T03-trigger-enhanced-log-collection.json` | P3-T03 | MDM command reference (Graph API) | Intune Remote Actions / Graph API | **Not a deployable profile** — command body reference |
| `P3-T04-cancel-enhanced-log-collection.json` | P3-T04 | MDM command reference (Graph API) | Intune Remote Actions / Graph API | **Not a deployable profile** — command body reference |
| `P3-T05-intelligence-calendar.json` | P3-T05 | DDM `com.apple.configuration.intelligence.settings` | Settings Catalog → Apple Intelligence | Tests new `Apps.Calendar.AllowNaturalLanguageEditing` key (27.0+) |

## P3-T01: Web Content Filter

The `webcontent-filter.plugin` declaration replaces the legacy `com.apple.webcontent-filter`
profile. It requires a compatible vendor Network Extension content filter app to be installed
on the device first (deploy via Intune as a managed app).

**Critical placeholders:**
- `PluginBundleID` — bundle ID of your vendor's content filter Network Extension
- `ProviderComposedIdentifier` — designated requirement for the Network Extension data provider
- `ServerAddress`, `VendorConfig` — vendor-specific

These values are vendor-specific (Cisco Umbrella, Symantec WSS, Zscaler, etc.). Consult your
vendor's macOS 27 DDM integration guide for the exact values.

## P3-T02: Content Cache

`CacheLimit` is set to 100 GB (`107374182400` bytes). Adjust to match your hardware.
`DeclarativeStatusInterval: 300` causes the device to report content-cache DDM status items
every 5 minutes — pairs with the status subscription in P4-T01.

`ManagementStatusTarget` must be an HTTPS endpoint that accepts POST requests with
cache statistics JSON. Leave this key out entirely if you don't have a monitoring webhook.

## P3-T03 / P3-T04: Enhanced Log Collection (command references)

These are **not deployable profiles**. They are reference files for Graph API calls.

**To trigger from Intune UI:** Device → Remote Actions → Collect Diagnostics (if surfaced for
Enhanced Log Collection). For programmatic use:

```
POST https://graph.microsoft.com/beta/deviceManagement/managedDevices/{deviceId}/triggerEnhancedLogCollection
Authorization: Bearer {token}
Content-Type: application/json

{
  "appleCareSupportToken": "YOUR_APPLECARE_ENTERPRISE_TOKEN"
}
```

**Requires:** AppleCare Enterprise agreement. Token is obtained from the AppleCare Enterprise portal.
The device must be supervised and the `DeviceManagementManagedDevices.PrivilegedOperations.All`
Graph API permission must be granted to the calling app registration.

## P3-T05: Intelligence Settings — Calendar Natural Language Editing

`AllowNaturalLanguageEditing` is new in macOS 27.0 / iOS 27.0. The declaration shown disables
it (`false`) to validate the control, then enables it (`true`) to confirm the feature works when
permitted. The other Apple Intelligence keys (`AllowWritingTools`, `AllowGenmoji`, etc.) are
included to show a realistic production payload — adjust per your organization's AI policy.

## Deployment notes

- **P3-T01** and **P3-T02** require supervised macOS 27 devices.
- **P3-T05** can be deployed to unsupervised macOS devices but requires DDM enrollment.
- All `.json` files carry an empty `ServerToken` — the MDM server populates this on deployment.
- For P3-T02, deploy on a Mac with adequate storage (recommend 500 GB+ SSD for meaningful caching).

## Placeholders to replace before real deployment

| File | Placeholder | Replace with |
|---|---|---|
| `P3-T01-content-filter-plugin.json` | `com.yourvendor.contentfilter.extension` | Vendor plugin bundle ID |
| `P3-T01-content-filter-plugin.json` | `filter.yourcompany.com` | Your filter server address |
| `P3-T01-content-filter-plugin.json` | `REPLACE_WITH_LICENSE_KEY` | Vendor license key |
| `P3-T01-content-filter-plugin.json` | `com.yourvendor.contentfilter.dataprovider {…}` | Vendor designated requirement |
| `P3-T02-content-cache.json` | `107374182400` | Cache size in bytes for your hardware |
| `P3-T02-content-cache.json` | `10.0.0.1` / `10.0.255.254` | Your subnet's IP range |
| `P3-T02-content-cache.json` | `https://monitoring.yourcompany.com/content-cache` | Your monitoring webhook URL (or remove key) |
| `P3-T03-trigger-enhanced-log-collection.json` | `REPLACE_WITH_APPLECARE_ENTERPRISE_TOKEN` | AppleCare Enterprise session token |

## Pre-deployment checklist

- [ ] Supervised macOS 27.0 Beta 2+ test device for P3-T01 and P3-T02.
- [ ] Vendor content filter Network Extension app installed before deploying P3-T01.
- [ ] AppleCare Enterprise agreement and token available before P3-T03/T04.
- [ ] `ManagementStatusTarget` webhook server running (or key removed) before P3-T02.
- [ ] Verify Settings Catalog availability for `intelligence.settings` new Calendar key.

See `plan.md` (P3 section) for full per-test steps and pass criteria.
