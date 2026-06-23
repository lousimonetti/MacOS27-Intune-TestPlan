# Phase 2 — New Network DDM Configurations (P2) Deliverables

Production artifacts for the **P2 — New Network DDM Configs** tests in [`plan.md`](../../plan.md).
These validate the six new declarative network types introduced in macOS 27, replacing legacy
VPN profiles, `com.apple.dnsSettings.managed`, `com.apple.relay.managed`, and similar payload types.

**Key advantage over legacy profiles:** Network declarations reference credential asset
declarations by identifier (`ClientCertificateAssetReference`). When a certificate renews via
SCEP or ACME, the VPN and DNS configurations automatically use the new cert — no profile reinstall.

All files validate with `python3 -m json.tool`.

## Files

| File | Test | Declaration type | Deploy in Intune via | Notes |
|---|---|---|---|---|
| `P2-T01-vpn-cert-scep.json` | P2-T01 | `com.apple.asset.credential.scep` | Custom DDM JSON | Deploy **before** the IKEv2 VPN declaration |
| `P2-T01-ikev2-vpn.json` | P2-T01 | `com.apple.configuration.network.vpn.ikev2` | Custom DDM JSON | References the SCEP asset by identifier |
| `P2-T02-dns-settings.json` | P2-T02 | `com.apple.configuration.network.dns-settings` | Settings Catalog → Network → DNS or Custom DDM JSON | DoH (HTTPS) — change `DNSProtocol` to `TLS` for DoT |
| `P2-T03-always-on-vpn.json` | P2-T03 | `com.apple.configuration.network.vpn.always-on` | Custom DDM JSON | Supervised only; reuses the P2-T01 SCEP asset |
| `P2-T04-network-relay.json` | P2-T04 | `com.apple.configuration.network.relay` | Custom DDM JSON | HTTP/3 MASQUE relay for split-tunneled internal domains |

## Deployment order for P2-T01

The IKEv2 VPN config (`P2-T01-ikev2-vpn.json`) references the SCEP asset by its `Identifier`.
Deploy in this order:

1. `P2-T01-vpn-cert-scep.json` — creates the managed credential.
2. `P2-T01-ikev2-vpn.json` — references it via `"ClientCertificateAssetReference": "com.test.macos27.p2-t01.vpn-cert"`.

If deploying to Intune via Custom DDM JSON, submit both declarations to the same device
assignment. The MDM server resolves the asset reference before applying the VPN config.

## Deployment notes

- **P2-T03 (Always-on VPN)** reuses the SCEP asset from P2-T01. Deploy P2-T01 first.
  Always-on VPN requires supervised macOS 27 — user cannot disable it in System Settings.
- **P2-T04 (Relay)** uses `HTTP3RelayURL` — ensure your relay server supports HTTP/3 MASQUE
  (RFC 9298). Set `RawPublicKeys` to the relay server's raw public key bytes if using QUIC
  pinning; leave empty `[]` to use the system trust store.
- **P2-T02 (DNS)** `DNSProtocol` options: `HTTPS` (DoH, port 443) or `TLS` (DoT, port 853).
  `OnDemandRules` here activate the encrypted resolver on both Wi-Fi and Cellular —
  remove or narrow these if you only want encrypted DNS on corporate Wi-Fi.

## Placeholders to replace before real deployment

| File | Placeholder | Replace with |
|---|---|---|
| `P2-T01-vpn-cert-scep.json` | `https://scep.yourcompany.com/scep` | Your SCEP server URL |
| `P2-T01-vpn-cert-scep.json` | `REPLACE_WITH_SCEP_CHALLENGE` | Static SCEP challenge or leave empty for dynamic |
| `P2-T01-ikev2-vpn.json` | `vpn.yourcompany.com` | Your IKEv2 VPN gateway FQDN |
| `P2-T02-dns-settings.json` | `https://dns.yourcompany.com/dns-query` | Your DoH resolver URL |
| `P2-T02-dns-settings.json` | `192.168.1.53`, `192.168.1.54` | Your DNS server IPs |
| `P2-T03-always-on-vpn.json` | `always-on-vpn.yourcompany.com` | Your always-on VPN gateway FQDN |
| `P2-T04-network-relay.json` | `https://relay.yourcompany.com/relay` | Your HTTP/3 relay URL |
| `P2-T04-network-relay.json` | `*.yourcompany.com` domains | Your internal domain names |

## Pre-deployment checklist

- [ ] Apple Silicon, ADE-enrolled/supervised macOS 27.0 Beta 2+ test device.
- [ ] SCEP server accessible from test device (for P2-T01 cert issuance).
- [ ] VPN gateway supports IKEv2 with certificate authentication.
- [ ] Relay server supports HTTP/3 MASQUE (for P2-T04).
- [ ] Verify Settings Catalog availability for each declaration type before choosing Custom DDM JSON path.
- [ ] Remove any conflicting legacy VPN or DNS profiles from the device before testing.

See `plan.md` (P2 section) for full per-test steps and pass criteria.
