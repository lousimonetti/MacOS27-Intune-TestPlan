# Legacy `.mobileconfig` conversions

These are Apple Configuration Profile (`.mobileconfig`) versions of the macOS 27
DDM declaration JSON files in `deliverables/`. They are provided for admins who want
to deploy via Intune's **Custom Profile** path (Path 1 in
[`DEPLOY-INTUNE.md`](../../DEPLOY-INTUNE.md)) instead of the DDM JSON / Graph API path.

## Two buckets

| Folder | What's in it | Will it actually apply? |
|---|---|---|
| `./` (this folder) | **Faithful** conversions onto real, supported legacy payloads | Yes — these use documented Apple payload types and should deploy and take effect on macOS 27 |
| `./experimental/` | **Best-effort** conversions of DDM-only features that have no real legacy payload | **Probably not** — read `experimental/README.md` first |

> The original `.json` DDM declarations remain the source of truth and the recommended
> deployment artifact. These `.mobileconfig` files are a convenience/compatibility layer.

## Faithful conversions in this folder

| File | Legacy payload type(s) | Source declaration |
|---|---|---|
| `P0-T02-swu-settings.mobileconfig` | `com.apple.SoftwareUpdate` + `com.apple.applicationaccess` | `softwareupdate.settings` |
| `P2-T01-vpn-cert-scep.mobileconfig` | `com.apple.security.scep` | `asset.credential.scep` |
| `P2-T01-ikev2-vpn.mobileconfig` | `com.apple.vpn.managed` (IKEv2) | `network.vpn.ikev2` |
| `P2-T02-dns-settings.mobileconfig` | `com.apple.dnsSettings.managed` | `network.dns-settings` |
| `P3-T01-content-filter-plugin.mobileconfig` | `com.apple.webcontent-filter` | `webcontent-filter.plugin` |

### Conversion notes (lossy spots to be aware of)

- **P0-T02 SWU settings** — `Download: AlwaysOn` → `AutomaticDownload`/`AutomaticCheckEnabled`;
  `InstallSecurityUpdate: AlwaysOn` → `CriticalUpdateInstall` + `ConfigDataInstall`;
  `InstallOSUpdates: Allowed` → `AutomaticallyInstallMacOSUpdates = false`. Deferral periods
  map to the Restrictions payload (`enforcedSoftwareUpdate*DeferredInstallDelay`).
  `AllowStandardUserOSUpdates` has **no** legacy key and is omitted.
- **P2-T01 SCEP + IKEv2** — these are **two separate profiles**, mirroring the DDM asset +
  config split. The VPN profile's `PayloadCertificateUUID` references the SCEP payload UUID,
  so **install the SCEP profile first** and keep both installed. The DDM `{{SERIALNUMBER}}`
  subject variable is written as the legacy `%SerialNumber%`; if your Intune SCEP connector
  expects a different variable token, adjust the `Subject` CN.
- **P3-T01 content filter** — the DDM `ProviderComposedIdentifier` is split into
  `FilterDataProviderBundleIdentifier` + `FilterDataProviderDesignatedRequirement`. The
  vendor's system/network extension app must be installed (managed app) first.
- All files use `yourcompany.com` / `REPLACE_WITH_*` placeholders — substitute real values
  before deploying outside a lab.

## How to test these in Intune

### 1. Upload as a Custom profile

1. Sign in to the [Intune admin center](https://intune.microsoft.com).
2. **Devices → macOS → Configuration → Create → New Policy**.
3. Platform **macOS**, Profile type **Templates → Custom**.
4. Name it (e.g. `P2-T02 DNS (legacy mobileconfig)`).
5. **Deployment channel: Device** (all of these use `PayloadScope = System`).
6. Under **Configuration profile file**, upload the `.mobileconfig`.
7. **Assignments** → add your `macOS-27-AppleSilicon` test group → **Create**.

> Deploy `P2-T01-vpn-cert-scep.mobileconfig` **before** `P2-T01-ikev2-vpn.mobileconfig`.

### 2. Confirm Intune accepts and delivers it

- In Intune: open the profile → **Device status** should reach **Succeeded** (not **Error**).
  An **Error** here usually means a malformed payload or a key the OS version rejects.
- **Devices → All devices →** *(device)* **→ Device configuration** → the profile shows **Succeeded**.

### 3. Confirm it landed on the Mac

On the test device (`System Settings → General → Device Management`) the profile should be
listed. Then verify the payload actually took effect:

```bash
# The profile is present and shows the expected payload keys
profiles show -type configuration

# Software Update (P0-T02): effective managed values
sudo defaults read /Library/Managed\ Preferences/com.apple.SoftwareUpdate 2>/dev/null
sudo defaults read /Library/Managed\ Preferences/com.apple.applicationaccess 2>/dev/null

# DNS (P2-T02): encrypted DNS should show as managed
scutil --dns | grep -i -A2 "DNS configuration (for scoped queries)"

# VPN (P2-T01): the configuration should appear in Network settings
scutil --nc list

# Web content filter (P3-T01): the content filter should be registered
systemextensionsctl list | grep -i filter

# Watch profile installation / MDM activity live
log stream --predicate 'subsystem == "com.apple.ManagedClient"' --info
```

### 4. Functional check per profile

| Profile | Pass signal |
|---|---|
| P0-T02 SWU settings | Software Update pane shows managed/deferred behavior; deferral days reflected |
| P2-T01 SCEP | A client certificate is issued and visible in **Keychain Access → System** |
| P2-T01 IKEv2 VPN | VPN config appears in **System Settings → VPN**; connects with the SCEP cert |
| P2-T02 DNS | `scutil --dns` shows the DoH server; DNS queries resolve through it |
| P3-T01 content filter | Vendor extension active; filtered traffic behaves per policy |

### Common issues

| Symptom | Likely cause | Fix |
|---|---|---|
| Profile status **Error** in Intune | OS rejected a key/payload for this macOS version | Check the payload against the device's macOS version; compare to the DDM JSON |
| VPN cert not found | SCEP profile not installed, or installed after VPN | Install `P2-T01-vpn-cert-scep` first; keep both profiles assigned |
| Content filter never activates | Vendor extension app not installed | Push the managed app first |
| Values not in `Managed Preferences` | Profile not delivered to Device channel | Confirm **Deployment channel = Device** and group assignment |
