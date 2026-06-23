# ⚠️ EXPERIMENTAL `.mobileconfig` conversions — read before deploying

> **These files are NOT supported and will most likely NOT work.**
>
> Every declaration converted here is a macOS 27 (or recent) **DDM-only** capability,
> or maps onto a legacy payload that is **semantically lossy** or **deprecated**. They
> are provided only so the file set is complete and so you can experiment. For anything
> real, deploy the original `.json` DDM declaration in `deliverables/` via Settings
> Catalog or Custom DDM JSON (Graph API) — see [`DEPLOY-INTUNE.md`](../../../DEPLOY-INTUNE.md).

## Why each one is experimental

| File | Wrapper payload type | Why it won't reliably apply |
|---|---|---|
| `P0-T02-swu-enforcement.mobileconfig` | `com.apple.configuration.softwareupdate.enforcement.specific` (not a legacy type) | No legacy payload enforces a **specific OS version by a deadline**. Legacy can only *defer*. |
| `P0-T05-app-settings-privacy.mobileconfig` | `com.apple.TCC.configuration-profile-policy` (PPPC) | Legacy PPPC **cannot pre-grant** Camera/Microphone/Bluetooth/Local Network — the `Allow` entries are ignored. This is the exact gap the DDM `app.settings` privacy declaration was created to fill. CodeRequirement strings are placeholders. |
| `P1-T01-binary-allowlist.mobileconfig` | `com.apple.configuration.app.settings` (not a legacy type) | Binary execution allowlisting is DDM-only; no legacy payload exists. |
| `P1-T02-binary-denylist.mobileconfig` | `com.apple.configuration.app.settings` (not a legacy type) | Binary deny control is DDM-only. |
| `P1-T03-managed-apps-passthrough.mobileconfig` | `com.apple.configuration.app.settings` (not a legacy type) | Same as above. |
| `P2-T03-always-on-vpn.mobileconfig` | `com.apple.vpn.managed` (`VPNType: AlwaysOn`) | Legacy Always-On VPN exists but is iOS-centric and its schema does not match the DDM `TunnelConfigurations` shape; mapping is best-effort. |
| `P2-T04-network-relay.mobileconfig` | `com.apple.configuration.network.relay` (not a legacy type) | HTTP/3 network relay is DDM-only. |
| `P3-T02-content-cache.mobileconfig` | `com.apple.configuration.content-cache.settings` (not a legacy type) | No general-purpose legacy payload accepts this full key set. |
| `P3-T05-intelligence-calendar.mobileconfig` | `com.apple.applicationaccess` (restrictions) | Only the 3 system-wide toggles (`allowWritingTools`, `allowGenmoji`, `allowImagePlayground`) map to legacy keys, and **those were deprecated in the 26.4 release** in favor of DDM. Per-app controls (Calendar/Mail/Safari) are dropped entirely. |

## Not converted at all

`P3-T03-trigger-enhanced-log-collection.json` and `P3-T04-cancel-enhanced-log-collection.json`
are **MDM commands**, not configuration profiles. There is no profile/payload representation
of a command, so no `.mobileconfig` is produced for them. Send these via Intune Remote Actions
or the Graph API as documented in [`DEPLOY-INTUNE.md`](../../../DEPLOY-INTUNE.md) (Path 3).

## What happens if you upload these anyway

When you upload one of these as an Intune **Custom Profile**, expect one of:

- **Intune rejects the file** on upload (schema/type validation), or
- Intune accepts it but the device reports the profile as **Error / Failed**, or
- The profile installs but the settings are **silently ignored** (no effect on the Mac).

Any of these is the *expected* outcome — it confirms the capability is DDM-only on macOS 27.
If you're running the test plan, treat "legacy profile rejected/ignored" as the **pass**
condition and use the DDM JSON declaration for the actual configuration.

## If you still want to try one

1. Pick the matching DDM JSON declaration in `deliverables/` as your real artifact.
2. Upload the experimental `.mobileconfig` via **Devices → macOS → Configuration → Custom**
   (Device channel) only on a **lab** device.
3. Capture the result for your test notes:
   ```bash
   profiles show -type configuration
   log show --predicate 'subsystem == "com.apple.ManagedClient"' --last 30m | grep -i -E "reject|invalid|error|ignored"
   ```
4. Then deploy the DDM JSON declaration the supported way and compare behavior.
