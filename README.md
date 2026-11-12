# Server VLAN Migration & Incident Review

An evidence-based homelab project covering a staged Linux server migration, DNS continuity, least-privilege routed segmentation, and WireGuard remote-access recovery.

**Public portfolio edition: every deployment IP address below and in the accompanying files is a fictional example.** Protocols, dates, test counts, observed outcomes, and the architecture's technical relationships are retained. No original-address mapping is published.

## Project documentation

- [Full incident/change report](server-migration-incident-report.md) — architecture, root-cause analysis, Method of Procedure, rollback strategy, validation matrix, corrective/preventive actions, and a terminology glossary.
- [Sanitized validation summary](sanitized-validation-summary.json) — evidence classes, test counts, observed outcomes, and explicit limitations. This is a derived summary, not fabricated raw logs.

## Scope

Two service-bearing Linux hosts moved from a flat upstream network into a routed IEEE 802.1Q Servers VLAN behind OpenWrt:

- **NAS-01:** OpenMediaVault, SMB, Pi-hole, and co-hosted web applications.
- **VPN-01:** WireGuard and Unbound recursive DNS.

The work combined managed-switch access-port/PVID changes, DHCP reservations, source-restricted administration, resolver dependency migration, nginx recovery, canonical-origin correction, and two-stage inbound UDP DNAT.

## Technologies and competencies

- Linux administration, NetworkManager, systemd, and UID/GID permissions
- IEEE 802.1Q VLANs, access ports, PVIDs, and trunking
- OpenWrt UCI, fw4/nftables, stateful filtering, SNAT, and DNAT
- WireGuard, Pi-hole, Unbound, and DNS UDP/TCP diagnostics
- nginx virtual hosts, Host/Origin checks, and CSRF preservation
- Strict SSH host-key continuity and scoped administrative access
- AI-assisted Python operational automation, regression tests, guarded rollback, and evidence-driven incident review

## Sanitized network illustration

| Role | Fictional example subnet |
|---|---|
| Upstream | `10.100.10.0/24` |
| Servers | `10.100.20.0/24` |
| Trusted administration | `10.100.30.0/24` |
| IoT | `10.100.40.0/24` |
| WireGuard tunnel | `10.200.0.0/24` |

```text
External WireGuard client
  -> ISP gateway, UDP 51820
  -> OpenWrt upstream example 10.100.10.2:51820
  -> VPN-01 example 10.100.20.20:51820

DNS client
  -> Pi-hole example 10.100.20.10:53
  -> Unbound example 10.100.20.20:5335
```

These fictional private networks support distinct roles without overlapping the illustrated subnets. They are not the deployed addressing plan, and these excerpts should not be applied to live infrastructure.

## Selected results

- Both target hosts moved to the intended server VLAN; unrelated access-port configuration was preserved.
- DNS controls were exercised over UDP and TCP and correlated with Pi-hole forward/reply records for the new backend.
- Web-service regressions were resolved without disabling authentication, Host validation, or CSRF controls.
- Strict SSH identity checks were retained across address changes.
- The old upstream Wi-Fi bypass was disabled after wired-path validation.
- The helper suites contain **92 distinct automated test cases**: 55 in the earlier preparation suite and 37 in the future-listener suite. The 37-case suite also passed on the target's Python 3.13.5; it is not counted twice as additional distinct cases.
- The operator reported restored remote NAS access and, after correcting stale DNS in the phone's WireGuard profile, working VPN internet access.

## Post-migration DNS incident

The phone's tunnel profile retained its old DNS-server address. Local resources remained reachable by IP while internet name resolution failed. Server-side checks showed DNS responses, enabled IPv4 forwarding, and working DNS-independent public HTTPS. Updating the client DNS setting restored internet access according to the operator. No additional server restart, firewall change, or WireGuard key replacement was needed for that fix.

## Evidence boundaries

Direct observations, screenshots/operator reports, and unfinished acceptance tests are separately classified. This project does not claim measured MTTR, an availability SLA, enterprise scale, high availability, or a complete zero-trust deployment.

Controlled reboot persistence, full authenticated SMB acceptance, complete TLS-trust validation, independent external peer capture, and comprehensive VPN IPv6/throughput assessment remain separately documented follow-up work.

## Publication safety

Only this README, the sanitized incident report, and the sanitized validation summary are included. Raw archives, deployment-address mappings, passwords/hashes, Wi-Fi PSKs, SSH/WireGuard private keys, peer identities, browser sessions, public WAN addresses, personal screenshots, and DNS browsing histories remain excluded.

## Attribution

This is owner/operator-led **personal-project experience**, with AI-assisted diagnostics, automation development, evidence review, and documentation. It is not presented as unsupported enterprise employment or sole unaided authorship.
