# Server Network Migration: Incident Review and Change Record

> **Address sanitization:** All deployment IPv4 addresses in this public edition are fictional, consistently substituted examples. Network roles, protocol ports, validation outcomes, dates, and test counts are preserved. The examples do not reveal the deployed addresses. Public test endpoint `1.1.1.1` and standard loopback notation are not deployment identifiers. Configuration excerpts are sanitized illustrations, not verbatim live configuration dumps. Original evidence and the private substitution mapping are not included.

## Segmented Infrastructure Migration with DNS, NAS, Reverse-Proxy, and VPN Continuity

> **Environment:** Personal, service-bearing homelab. This is a technical project record, not a claim of enterprise employment, enterprise scale, or a formally declared production SEV incident.
>
> **Change window:** November 8–9, 2026, local Central Standard Time, UTC−06:00.
>
> **Record type:** Planned infrastructure migration with migration-related service incidents, root-cause analysis, controlled remediation, and post-change validation.
>
> **Status:** Both servers migrated; critical local service paths verified; external NAS reachability and subsequent phone SSH success reported by the operator. On November 10, the operator also reported restored VPN internet access after correcting the phone tunnel's stale DNS-server address. Independent packet-level, persistence, and other remaining validation items are explicitly recorded below.
>
> **Attribution:** Owner/operator-led implementation with AI-assisted diagnostics, automation development, evidence review, and documentation. The operator supplied local-console access, approvals, privileged terminal actions where required, and external-client testing.
>
> **Publication notice:** This is the address-sanitized public portfolio edition. Credentials, private keys, public WAN addresses, peer identities, device MAC addresses, and configuration-backup payloads are excluded.

---

## 1. Executive Summary

Two Linux-based infrastructure hosts were migrated from a flat ISP-managed IPv4 network into a routed **IEEE 802.1Q Servers VLAN** behind an **OpenWrt security gateway**:

- **NAS-01:** OpenMediaVault storage, SMB, Pi-hole, and co-hosted web applications.
- **VPN-01:** WireGuard remote access and the Unbound recursive DNS service used by Pi-hole.

The migration changed Layer 2 broadcast-domain membership, Layer 3 addressing and default gateways, inter-zone forwarding policy, DNS source-address semantics, and the inbound remote-access NAT path. It was therefore treated as a dependency-sensitive infrastructure change rather than a simple IP renumbering exercise.

The implementation used a **staged cutover**, host-specific DHCP reservations, access-port/PVID changes, narrowly scoped firewall exceptions, configuration backups, strict SSH host-key continuity, daemon-context configuration validation, and independent application-layer checks.

The principal migration-related faults were:

1. Resolver access-control and service-account file-readability constraints during preparation.
2. A shared nginx instance failing because a custom HTTPS virtual host still bound the NAS's retired address.
3. Pi-hole's local-subnet acceptance policy rejecting routed clients despite firewall permission.
4. A private application's reverse-proxy allowlist and canonical public-origin configuration retaining the old network identity.
5. Ethernet clients retaining old DHCP state because VLAN reassignment did not necessarily produce physical link loss.
6. A nominal Wi-Fi recovery connection whose reply path still used the Ethernet interface scheduled for migration.
7. An upstream WireGuard port forward that initially targeted the server's retired ISP-LAN address.

These were resolved without replacing WireGuard peer keys, opening public SSH, removing NAS authentication, deleting backup data, or rebuilding either operating system.

**Outcome:** NAS and VPN/DNS services now operate on the Servers VLAN. Pi-hole's forward/reply logs demonstrate the new filtering-to-recursion path. The WireGuard ingress path was changed to a two-stage DNAT arrangement through OpenWrt, and the operator reported successful remote NAS access.

---

## 2. Scope, Service Criticality, and Impact

### In scope

- Migration of two wired Linux servers into VLAN 30.
- DHCP reservations, IPv4 addressing, gateway selection, and lease renewal.
- TP-Link managed-switch VLAN membership and ingress PVID changes.
- OpenWrt inter-zone policy and WireGuard UDP DNAT.
- Pi-hole-to-Unbound DNS dependencies, listener selection, and resolver ACLs.
- NAS web administration and a co-hosted private HTTPS application.
- Strict SSH identity continuity across address changes.
- Removal of an unintended upstream Wi-Fi bypass after wired validation.
- Configuration backup, guarded automation, rollback planning, and evidence capture.

### Not included as completed work

- An organization-wide zero-trust architecture deployment.
- A high-availability cluster, redundant resolver pair, or automated failover platform.
- A blanket password-authentication shutdown or passwordless-sudo rollout.
- Anonymous SMB access or a redesign of storage authorization.
- Network-wide DNS interception, DHCP DNS redirection, or a demonstrated policy forcing every client through Pi-hole.
- A full TLS/PKI remediation, penetration test, or comprehensive security audit.
- Controlled reboot/power-cycle persistence testing of every device.
- A full authenticated SMB file-read acceptance test.
- A measured remote VPN throughput, MTU, packet-loss, or full-tunnel IPv6 assessment.

### Observed impact and measurement limitations

Service-affecting interruptions occurred during network cutover and service activation. The NAS's shared web server also required recovery from a stale address binding. Changing the private application's runtime origin required an application restart that cleared its in-memory sessions.

No data loss was reported. No deletion or modification of NAS backup-source data formed part of the migration procedure. This is not equivalent to a full filesystem integrity or end-to-end backup-restore certification.

A formal incident severity was not assigned. No continuous availability telemetry or complete outage start/stop series was captured, so this report does **not** claim an MTTR, total downtime, availability percentage, RTO achievement, or RPO achievement. Elapsed administration time includes preparation, operator interaction, and waiting; it must not be presented as continuous outage duration.

---

## 3. Architecture and Addressing

### 3.1 Before and after

| Component | Previous address/network | Final address/network | Function |
|---|---|---|---|
| NAS-01 | `10.100.10.10/24`, ISP LAN | `10.100.20.10/24`, VLAN 30 | OMV, SMB, Pi-hole, web applications |
| VPN-01 | `10.100.10.20/24`, ISP LAN | `10.100.20.20/24`, VLAN 30 | WireGuard and Unbound |
| VPN-01 recovery Wi-Fi | `10.100.10.30/24` | Radio disabled after wired validation | Temporary recovery/bypass path |
| OpenWrt upstream interface | `10.100.10.2/24` | Retained | DHCP-based upstream connectivity to ISP gateway |
| Servers gateway | `10.100.20.1/24` | Retained | VLAN 30 routing and firewall boundary |
| Trusted gateway | `10.100.30.1/24` | Retained | Administration/client network |
| Administration workstation | `10.100.30.100` | Retained | Source-restricted administrative access |
| WireGuard tunnel interface | `10.200.0.1/24` | Retained | VPN tunnel network |

The final private destination address of the VPN host is not directly reachable as an ordinary on-link device from the ISP LAN. External access requires the downstream router's upstream address as the first forwarding destination.

### 3.2 Routed topology

```text
Internet / external WireGuard client
                  |
                  | UDP 51820
                  v
       ISP gateway / upstream CPE
       LAN: 10.100.10.0/24
                  |
                  | DNAT #1: UDP 51820 -> 10.100.10.2:51820
                  v
           OpenWrt gateway
       Upstream: 10.100.10.2
       Servers:  10.100.20.1/24
       Trusted:  10.100.30.1/24
       IoT:      10.100.40.1/24
                  |
                  | DNAT #2: UDP 51820 -> 10.100.20.20:51820
                  v
         VLAN 30 / Servers subnet
          |                     |
          v                     v
 NAS-01: 10.100.20.10          VPN-01: 10.100.20.20
 OMV / SMB / Pi-hole     WireGuard / Unbound
          |                     ^
          +--- DNS UDP/TCP 5335--+
```

### 3.3 Switching design

The switch/router interconnect retained a native upstream VLAN and tagged downstream VLANs: a **router-on-a-stick-style** arrangement with VLAN subinterfaces and routed security zones.

Final relevant access-port state:

| Switch port | Role | Untagged VLAN / PVID | Important preservation condition |
|---|---|---|---|
| 1 | ISP-LAN connection | VLAN 1 / PVID 1 | Upstream path retained |
| 2 | VPN-01 | VLAN 30 / PVID 30 | Only port changed during VPN cutover |
| 3 | NAS-01 | VLAN 30 / PVID 30 | Not altered during VPN cutover |
| 4 | Retained legacy assignment | VLAN 10 / PVID 10 | Not repurposed |
| 5 | Previously migrated server | VLAN 30 / PVID 30 | Preserved |
| 6 | OpenWrt trunk | Native VLAN 1; tagged VLANs 20, 30, 50 | Trunk/native membership preserved |
| 7 | Existing Trusted access port | VLAN 50 / PVID 50 | Preserved |
| 8 | Existing IoT access port | VLAN 20 / PVID 20 | Preserved |

A port showing link down was not treated as authorization to delete its VLAN or repurpose it.

### 3.4 Relationship to earlier infrastructure work

The two-host cutover used a segmented fabric established during earlier project work: an OpenWrt wired upstream, a secured Trusted wireless network, routed Trusted/IoT/Servers subnets, managed-switch tagging, and a previously migrated server on VLAN 30. That earlier server acted as a useful management/forwarding reference, but it is not counted as a third host migrated within this report's November 8–9 change window.

A pre-existing HTTP 502 on that separate server was tracked independently. It is not represented here as a migration-induced regression or as a repaired application. This preserves a defensible distinction between infrastructure reachability, unrelated backend application health, and the scope of this incident review.

---

## 4. Dependency Model and Migration Sequencing

The principal dependency chain was:

```text
Client DNS request
  -> Pi-hole on NAS-01, UDP/TCP 53
  -> Unbound on VPN-01, UDP/TCP 5335
  -> Recursive resolution
```

The sequence deliberately migrated the filtering/storage host before the recursive resolver/VPN host:

1. Establish verified administration and recovery access.
2. Back up the router, switch, and relevant host configuration.
3. Prepare routed administrative access and DNS source ACLs.
4. Migrate NAS-01 while Unbound remains reachable at its original address.
5. Repair NAS-specific listener, acceptance-policy, and application-origin dependencies.
6. Pre-stage VPN-01's future Unbound listener without removing the working listener.
7. Prepare the downstream WireGuard DNAT rule while preserving the upstream forward.
8. Move VPN-01's access port and renew its Ethernet connection locally.
9. Verify Unbound at the new address from the NAS.
10. Repoint Pi-hole's upstream to the new resolver address.
11. Verify wired operation, then disable the upstream Wi-Fi bypass.
12. Retarget the ISP gateway's existing WireGuard forward to OpenWrt.
13. Conduct external-client reachability checks and document remaining acceptance gates.

This was **make-before-break preparation for DNS listeners**, followed by a controlled host-address cutover. It was not a fully redundant or zero-downtime migration: each host still had a single authoritative wired service path.

---

## 5. Pre-Change Risk Assessment and Change Controls

### Principal risks

- **Management-plane lockout:** VLAN/PVID changes can invalidate the only usable SSH path.
- **Dependency inversion:** Moving a resolver without updating its dependent filtering service can strand DNS.
- **Control-plane/data-plane divergence:** Saved UCI configuration can differ from installed nftables behavior.
- **Source-address translation mismatch:** A backend ACL may see a router SNAT identity instead of the originating client or NAS.
- **Shared-daemon blast radius:** One invalid nginx listener can affect unrelated virtual hosts in the same process.
- **Residual dual-homing:** Upstream Wi-Fi or global IPv6 state can bypass the intended segmentation boundary.
- **Authentication regression:** An attempt to solve reachability by weakening key verification, share authorization, or application authentication would expand risk.
- **Public-ingress misdirection:** An ISP gateway retaining the old server address breaks remote VPN access after migration.
- **Rollback collateral damage:** Restoring a full historical switch baseline could undo previously successful migrations.

### Implemented safeguards

- Explicit operator approval before disruptive changes.
- Local-console availability for the physical cutover.
- Fresh administration-source/route checks before source-restricted operations.
- Native router/switch configuration exports with private file permissions and checksums.
- Verification that router archives contained expected configuration members.
- Readback of switch VLAN rows and the complete PVID map after each write.
- Inspection for LAG membership before manipulating individual ports.
- Host-specific DHCP reservations keyed to the correct Ethernet identity.
- Per-server SSH identities and strict pinned host-key verification.
- Fail-closed automation guards and explicit opt-in application flags.
- Daemon-readable configuration modes, with separate root-private backups.
- Rollback-protected OpenWrt application and confirmation after kernel-policy inspection.
- Application-level tests instead of treating a link state, listener, or successful API response as sufficient acceptance.

---

## 6. Incident and Remediation Analysis

### 6.1 DNS ACLs changed meaning across the routed/NAT boundary

**Symptom/risk:** The original DNS backend authorization was tied to the NAS's ISP-LAN address. After moving the NAS behind OpenWrt, that was no longer necessarily the source address observed by Unbound.

**Failure domain:** Layer 3 forwarding, source NAT, and application-layer resolver authorization.

**Analysis:** Firewall permission, route selection, SNAT behavior, and Unbound ACL acceptance were separate gates. A router-translated source can represent more than one downstream client, so allowing that source at the resolver cannot by itself establish per-client isolation.

**Remediation:** Transitional `/32` ACLs were prepared for the existing NAS address, the routed NAS address, and the actual upstream router identity. Source/destination/port-specific forwarding policy and direct-resolver guards limited access through the shared translated identity.

**Validation:** DNS queries were exercised from the actual NAS over both UDP and TCP. Router-origin queries were checked separately. Trusted-client direct access to the transitional backend was rejected as expected during preparation. IoT enforcement was inspected in installed policy, not falsely represented as an IoT-client functional test. See E01.

### 6.2 Root validation did not establish daemon readability

**Symptom/risk:** Resolver configuration handling exposed a difference between root's ability to read a file and the Unbound service account's ability to read included fragments.

**Failure domain:** Unix discretionary access control, UID/GID ownership, service-account privilege dropping, and configuration validation.

**Root cause:** Root-only file permissions were unsuitable for configuration consumed by the daemon after privilege reduction. Existing fragment readability also required correction. Additionally, running a daemon-context checker from an inaccessible administrative directory could emit a `cannot chdir` diagnostic despite a zero exit status.

**Remediation:** Relevant configuration fragments were made deliberately service-readable as `root:unbound` with mode `0640`. Backup archives remained private. The corrected helper validated configuration both as root and as the daemon identity, from working directory `/`, and rejected error/fatal diagnostics rather than trusting the exit code alone.

**Operational lesson:** Configuration syntax validity, filesystem traversal, file readability, service activation, and successful DNS answers are distinct acceptance criteria. No broad permission relaxation of private home directories or key material was used.

### 6.3 A stale nginx bind address affected multiple web services

**Symptom:** NAS web administration became unavailable even though the host was reachable and other service processes could be assessed independently.

**Failure domain:** Application socket binding and a shared reverse-proxy/web-server process.

**Root cause:** A custom HTTPS virtual host still attempted to bind `10.100.10.10:8443` after the NAS moved to `10.100.20.10`. The shared nginx service could not start correctly with that stale listener, affecting the co-hosted OMV interface as well.

**Remediation:** The specific custom site was backed up. Exactly one listener address and one `server_name` address reference were updated. Owner/mode preservation and guarded replacement avoided broad edits to unrelated virtual hosts. Native nginx validation passed, and the failed web service was started.

**Validation:** nginx became active; OMV on TCP 8080 and Pi-hole administration returned HTTP 200. These were separate checks, not an inference that one working web application proved the other. See E02.

### 6.4 Pi-hole's local-subnet policy rejected routed clients

**Symptom:** Routed clients could be permitted by the firewall while their DNS requests still failed at the filtering resolver.

**Failure domain:** Resolver application policy, not merely transport reachability.

**Root cause:** Pi-hole's `LOCAL` listening policy did not serve the intended off-subnet Trusted clients in this topology.

**Remediation:** With operator approval, the existing wired interface was confirmed and Pi-hole was changed to `SINGLE` interface-scoped operation. The configured interface remained the NAS Ethernet interface. Upstream configuration, DHCP settings, authentication, and client DNS advertisements were not bundled into this policy change.

**Validation:** Saved/native setting readback, UDP and TCP queries from the routed workstation, and server-loopback DNS were checked. Wildcard sockets in `ss` were not misinterpreted as proof that application policy served every interface. See E03.

### 6.5 Reverse-proxy authorization and application origin were independent barriers

**Symptom:** The private HTTPS application returned HTTP 403 after nginx itself had recovered.

**First barrier:** The nginx client-network allowlist still permitted only the original `10.100.10.0/24` network. It was updated to the intended Trusted subnet, `10.100.30.0/24`, while retaining `deny all` for other sources.

**Second barrier:** The backend still enforced its original canonical public origin, `https://10.100.10.10:8443`. A JSON 403 persisted even after the network allowlist was corrected.

**Discriminating test:** A credential-free GET with the original canonical Host was accepted, while the new Host was rejected. This separated reverse-proxy source authorization from backend Host/origin validation.

**Remediation:** The systemd service's configured public origin was changed to `https://10.100.20.10:8443`. The staged unit passed `systemd-analyze verify`; the systemd manager was reloaded and only that application was restarted after approval.

**Validation:** Runtime process arguments contained the new public origin. The new URL returned HTTP 200 with the expected application identity; the retired Host remained rejected with HTTP 403. See E04–E05.

**Security preservation:** Host validation, Origin checks, CSRF protection, and authentication were not disabled. Browser Origin headers were not rewritten to impersonate the old URL. Existing in-memory application sessions were cleared by the restart, as disclosed.

### 6.6 VLAN reassignment did not guarantee DHCP renewal

**Symptom/risk:** A host could retain the old address and routes after its access-port VLAN changed.

**Failure domain:** Layer 2 membership versus host-side Layer 3 lifecycle.

**Root cause:** The switch change did not necessarily cause Ethernet carrier loss. A DHCP client can retain its current lease and routing state without a connection deactivation/reactivation event.

**Remediation:** The actual network manager and profile were identified before the outage. The operator used a hostname-guarded local-console renewal procedure after switch membership and PVID readback were complete. The NAS and VPN host used their respective discovered management mechanisms rather than an assumed universal command.

**Validation:** Actual addresses and default routes were read from the hosts. VPN-01 acquired `10.100.20.20/24` with default gateway `10.100.20.1` on `eth0`. Console paste errors were distinguished from later successful NetworkManager activation output.

### 6.7 A Wi-Fi destination address was not an independent recovery path

**Symptom/risk:** SSH to VPN-01's Wi-Fi address succeeded, apparently suggesting a fallback management channel.

**Failure domain:** Kernel route selection, dual-homing, and administrative return-path dependence.

**Root cause:** A route lookup from the Wi-Fi source address to the server-seen SSH peer selected `eth0`. The reply path therefore depended on the interface whose switch port was about to move.

**Remediation:** The connection was not treated as independent recovery. Physical-console availability was confirmed for the cutover. After the new wired management, DNS, and internet paths passed, the old Wi-Fi radio was disabled without deleting its saved profile.

**Validation:** `wlan0` became DOWN with no address. The old upstream default and connected route disappeared. The remaining default route used `10.100.20.1` on Ethernet. Only link-local IPv6 remained on that interface, with no IPv6 default route. SSH, DNS, and wired HTTPS still worked afterward.

**Important distinction:** This verified the observed post-change state; it was not a controlled reboot-persistence certification.

### 6.8 Historical physical-port inventory was inaccurate

**Risk:** Earlier notes had the NAS and VPN server's switch ports reversed.

**Failure domain:** Inventory/configuration-management accuracy and physical-to-logical topology mapping.

**Remediation:** Operator-confirmed cabling and link-capability observations were reconciled before changes. The NAS was established on port 3 and the VPN host on port 2. During each cutover, unrelated VLAN rows and every other port's PVID were compared against the baseline.

**Outcome:** The NAS remained untouched during the VPN host's subsequent move. This prevented the change from being applied to the wrong service-bearing host.

### 6.9 Public WireGuard ingress required two coordinated DNAT stages

**Symptom/risk:** The existing ISP forward pointed UDP 51820 at `10.100.10.20`, which ceased to be the VPN host's wired address after migration.

**Failure domain:** Nested routing/NAT and upstream destination selection.

**Analysis:** The downstream router's actual ingress zone was `wwan`, including its wired upstream interface. It could not be assumed to be the separately named `wan` zone. The WireGuard host's existing `-o eth0 -j MASQUERADE` rule was interface-based, so renumbering that preserved interface did not require rewriting the rule or replacing peers.

**Remediation:** OpenWrt received a narrowly scoped IPv4 UDP DNAT from `10.100.10.2:51820` to `10.100.20.20:51820`. The ISP forward was preserved during preparation and later changed to the DHCP-associated `OpenWrt-wired` device at `10.100.10.2`.

**Validation:** Saved UCI state, live nftables ingress selection, the specific DNAT rule, and the DNAT forwarding acceptance rule were inspected. A screenshot subsequently showed the upstream device/IP, port, and UDP protocol. After being instructed to test from cellular with WireGuard enabled, the operator reported successful NAS web access.

**Limitation:** No independently captured fresh peer-handshake timestamp, packet capture, sustained transfer test, or complete phone DNS/full-tunnel validation was recorded. Remote application success is operator-reported evidence, not a substitute for those measurements.

### 6.10 Control-command completion and service readiness were not equivalent

A Pi-hole restart control command exceeded an initial short client timeout, while the systemd job subsequently completed successfully. The service was inspected rather than blindly restarted again. A later activation used a longer control-command timeout and completed successfully; that operation took approximately 62 seconds according to the tool result.

A classroom restart also exposed an immediate process-inspection/readiness race: the unit was active before an instantaneous command-line assertion yielded the expected application arguments. A subsequent runtime readback and HTTP check confirmed the correct process and configured origin without repeating the restart.

**Operational lesson:** A client timeout does not prove that a service job was cancelled. Conversely, `active` does not establish application readiness. Inspect job/service state, runtime configuration, and the real endpoint before deciding whether to retry or roll back.

### 6.11 Phone SSH refusal was not proven to be an authentication failure

After the remote NAS test, the phone displayed a TCP connection refusal to NAS port 22. NAS SSH was independently verified as running and reachable from the allowed administration workstation. The operator clarified that the failed attempt occurred on Trusted Wi-Fi with WireGuard off.

A source-restricted administrative policy was a plausible explanation. The error occurred before SSH user-key/password authentication, so adding a public key was not established as the correct fix.

The operator subsequently reported that SSH worked and cancelled the proposed phone-specific access change. **No additional phone firewall allowlist, DHCP reservation, or authorized-key modification was made. The exact cause of the transient phone refusal was not confirmed.**

---

## 7. Method of Procedure: Implemented Change Sequence

### Phase A — Baseline, backups, and management assurance

- Verified actual administration source address, SSID, route selection, host identity, and SSH authority.
- Separated key-based login success from operation-specific privilege checks.
- Used operator-executed sudo where unattended privilege was unavailable.
- Exported private router and switch backups and relevant host configuration backups.
- Verified archive integrity/expected router members and recorded SHA-256 digests.
- Reconciled physical switch-port assignments and preserved local-console recovery.

### Phase B — Resolver transition preparation

- Preserved working backend ACLs while adding transitional permitted sources.
- Retained the old DNS path until replacement-path checks passed.
- Corrected daemon-readability and validation-context issues.
- Added a future-address Unbound listener with Linux `IP_FREEBIND` behavior through the installed `ip-freebind` configuration option.
- Restarted Unbound because interface/listener changes were not a SIGHUP-only change.
- Verified old and future UDP/TCP sockets without claiming the future IP was already assigned.

### Phase C — NAS cutover and application recovery

- Moved only NAS access port 3 to VLAN 30 and PVID 30.
- Renewed the NAS interface from the local console.
- Verified the new address, gateway, strict SSH identity, DNS, and relevant service ports.
- Repaired the stale custom nginx binding, routed DNS acceptance, HTTPS allowlist, and backend canonical origin.
- Kept storage authentication and private backup-source data outside the network change scope.

### Phase D — Downstream VPN ingress preparation

- Read the existing upstream forward rather than inferring protocol or destination from a collapsed UI entry.
- Verified the router's actual upstream zone and target reservation.
- Backed up the router immediately before the firewall change.
- Added a single named UDP redirect with reflection disabled.
- Applied with a 240-second rollback window.
- Verified installed kernel behavior and retained all 37 pre-existing UCI firewall sections.
- Confirmed the change at 45 seconds, then later verified that it persisted beyond the original rollback window.

### Phase E — VPN/recursive-resolver host cutover

- Confirmed local-console readiness and port 2 identity.
- Took fresh prepared-router and switch backups.
- Added port 2 as untagged in VLAN 30, removed it from VLAN 1, and set PVID 30.
- Read back each step and the complete final PVID map.
- Renewed the discovered `netplan-eth0` connection locally.
- Verified SSH, new addressing, the wired default route, and active Unbound/WireGuard services.
- Queried the new Unbound address from NAS-01 over UDP and TCP.

### Phase F — Final DNS path and bypass removal

- Backed up Pi-hole's configuration privately.
- Changed only the upstream array to `10.100.20.20#5335`.
- Performed native configuration validation and service activation.
- Compared parsed configuration to ensure other settings were unchanged.
- Correlated fresh client queries with Pi-hole forward and reply log entries for the new backend.
- Tested wired public HTTPS using the new source address and checked the selected route.
- Disabled VPN-01's upstream Wi-Fi radio only after wired-path success.
- Repeated management, DNS, and wired-egress checks after bypass removal.

### Phase G — Upstream retargeting and operator acceptance

- Retargeted the existing ISP UDP 51820 forward to `OpenWrt-wired`, `10.100.10.2`.
- Preserved the port/protocol rather than opening unrelated services.
- Requested an external cellular/WireGuard NAS-reachability test.
- Recorded the operator's successful result and separately documented the later phone SSH observation.

---

## 8. Final Configuration Excerpts

These are address-sanitized documentation excerpts, not complete replacement configurations or instructions to overwrite a live system.

### 8.1 Supplemental Unbound listener fragment

```conf
server:
    interface: 10.100.20.20@5335
    ip-freebind: yes
```

Existing loopback and transitional listeners/ACLs were preserved in their existing configuration files. The future socket was pre-bound before the DHCP cutover; successful socket creation was not misrepresented as network reachability before address assignment.

### 8.2 Pi-hole DNS settings

```toml
[dns]
interface = "enu1u1"
listeningMode = "SINGLE"
upstreams = ["10.100.20.20#5335"]
```

The interface name is specific to the NAS. It is not a generic Linux Ethernet interface recommendation.

### 8.3 OpenWrt UDP forwarding

```uci
config redirect 'vpn_ingress'
    option name 'Forward-WireGuard-to-VPN01'
    option src 'wwan'
    option src_dip '10.100.10.2'
    option src_dport '51820'
    option dest 'servers'
    option dest_ip '10.100.20.20'
    option dest_port '51820'
    option proto 'udp'
    option family 'ipv4'
    option target 'DNAT'
    option reflection '0'
    option enabled '1'
```

The protocol and exact destination were constrained. TCP 22 was not forwarded from the internet.

### 8.4 Final DNS flow

```text
Trusted client
  -> NAS-01 / Pi-hole: 10.100.20.10:53
  -> VPN-01 / Unbound: 10.100.20.20:5335
  -> DNS response returned through the same filtering path
```

This validated explicit use of Pi-hole. It did not demonstrate that every client/application was configured or compelled to select that resolver.

---

## 9. Security and Identity Controls

- **Least privilege:** Administrative and application paths were expressed as service-specific inter-zone permissions rather than blanket Trusted-to-Servers acceptance.
- **Stateful filtering:** OpenWrt fw4/nftables policy was inspected at the actual packet-processing chains, including ingress selection, DNAT, and forwarding acceptance.
- **Cryptographic host continuity:** Independently verified SSH host keys were retained across address changes using strict checking, dedicated identities, pinned known-hosts files, and `HostKeyAlias` where needed.
- **No authentication downgrade:** SMB/NAS web authentication, application authentication, Host validation, and CSRF controls were retained.
- **VPN identity preservation:** WireGuard server/peer keys were not replaced as a workaround for routing or NAT changes.
- **Configuration secrecy:** Backup archives remained private; credential-bearing configuration and WireGuard dumps were not used as public troubleshooting output.
- **Service-account permissions:** Daemon-readable configuration and root-private backups were treated as different access-control requirements.
- **Recovery-path discipline:** A successful SSH connection was not assumed to prove physical interface independence.
- **IPv6 awareness:** Global addresses and default routes were inspected separately from IPv4; the observed post-cutover host state contained no IPv6 default route.
- **No public SSH:** The remote-access design exposed the existing WireGuard UDP service rather than adding an internet-facing SSH forward.

These controls are least-privilege and zero-trust-aligned practices. They do not constitute a complete zero-trust architecture. Hosts sharing VLAN 30 still share a Layer 2 segment; the report does not claim host-level microsegmentation between those servers.

---

## 10. Automation, Test Strategy, and Operational Safety

Two resolver-preparation test suites were exercised:

| Suite | Cases | What the evidence establishes |
|---|---:|---|
| Earlier ACL/preparation helpers | 55 | Local automated safety/regression cases passed |
| Future-listener helper | 37 | Local automated cases passed; the same suite also passed on VPN-01's Python 3.13.5 |
| Total distinct test cases across these suites | 92 | Not 129: rerunning the 37-case suite on the target does not create additional distinct cases |

The suites were rerun while preparing this report, returning 55 and 37 passing tests. The target-host record independently reports the 37-case suite passing under Python 3.13.5.

The future-listener helper used standard-library Python with:

- Explicit `--apply` opt-in; default execution was inert.
- Root and hostname guards.
- Configuration-directory locking.
- Symlink, hard-link, ownership, and mode checks.
- Exclusive fragment creation and inode/content identity checks.
- Verified private configuration archives and SHA-256 integrity checks.
- Root-context and daemon-context native validation.
- Effective-interface/ACL preservation checks.
- Bounded service commands and readiness deadlines.
- Timeout settlement before retry/rollback decisions.
- Guarded removal of only the helper-owned fragment on failure.
- Recovery validation and explicit manual-intervention errors if rollback could not be established.

The tests used real temporary files/metadata with injected service-command runners and clocks. They did not restart live services or manipulate real networking during unit execution. Live integration was separately checked after the operator applied the helper.

**Reviewed helper SHA-256:**

```text
b2c6ba82af8130ad43b04e60b4fca159e103f45149e4f49949944502bb05340a
```

The staged target copy's digest matched the reviewed source. Test execution/default-mode checks did not restart Unbound; the recorded daemon PID remained unchanged until the explicit operator apply step. See E06.

---

## 11. Verification Matrix and Evidence Strength

| Acceptance gate | Result | Evidence strength / qualification |
|---|---|---|
| NAS wired address/gateway | Verified | Host state and strict SSH at new address |
| VPN host wired address/gateway | Verified | Console output and independent SSH readback |
| VLAN membership/PVID | Verified | Native switch UI readback; complete map comparison |
| New Unbound listener | Verified | UDP and TCP sockets; service state; NAS-origin queries |
| Pi-hole upstream | Verified | Native configuration readback; only upstream field changed |
| Filtering-to-recursion path | Verified | Client queries correlated with forward/reply log entries to the new backend |
| Routed DNS UDP and TCP | Verified | Successful responses; NODATA correctly interpreted for unique descendant controls |
| Wired public internet | Verified | Source-bound HTTPS with a corresponding Ethernet route |
| Upstream Wi-Fi removal | Verified current state | Radio disabled, interface down, old routes absent; reboot not tested |
| Global IPv6 bypass removal | Verified observed state | Link-local only and no IPv6 default route at inspection |
| OMV web availability | Verified | HTTP 200 from the intended management endpoint |
| Private HTTPS application | Verified response | Correct application page at new URL; retired Host rejected |
| Application TLS trust | Not certified | Existing self-signed certificate unchanged; credential-free diagnostic verification bypass disclosed |
| OpenWrt WireGuard DNAT | Verified configuration/kernel state | Exact redirect, ingress, and DNAT forwarding acceptance inspected |
| ISP WireGuard forward | Screenshot-confirmed | Correct device, reserved upstream IP, UDP protocol, and port |
| External NAS access through VPN | Operator-reported success | Reported after cellular/WireGuard test instruction; no independent handshake capture |
| Phone SSH | Operator-reported success | Earlier refusal not definitively root-caused; no extra phone rule/key change made |
| Authenticated SMB file read | Not completed in this record | Share advertisement/port reachability are not a file-read test |
| Reboot/power-cycle persistence | Not completed | No controlled reboot/power-cycle validation recorded |
| Full VPN DNS/internet/throughput | Not completed | Requires additional external-client acceptance tests |
| Backup restore execution | Not completed | Archive/export verification is not an exercised restore |

### Important diagnostic distinctions

- Link up is not a DHCP lease.
- A DHCP lease is not a working default route.
- A listener is not an application authorization decision.
- A firewall allow rule is not a route.
- Saved configuration is not installed kernel policy.
- `NOERROR` can legitimately contain no A records; NODATA is not automatically a resolver outage.
- A cached or stale answer alone does not establish current upstream forwarding.
- HTTP 200 is not proof of authenticated application functionality or TLS trust.
- A WireGuard interface or active oneshot unit is not proof of a recent peer handshake.
- TCP refusal is not SSH key rejection.
- A configuration backup is not a tested disaster-recovery restore.

---

## 12. Validation Constraints and Tool Boundaries

The standalone `fw4 check` operation was denied by the available management RPC permissions. That result was recorded as an unavailable validation path, not as a clean syntax check.

The denied command was not tunneled through another shell or endpoint. The separately authorized native UCI transaction instead used:

1. A verified pre-change router archive.
2. Exact staged-diff scope inspection.
3. Rollback-protected application.
4. The installed firewall-status view's supported nftables JSON reader.
5. Verification of the exact live DNAT and forwarding rules.
6. Regression checks for management, NAS DNS, and web access.
7. Native confirmation and later saved/kernel-state readback.

This distinction matters for auditability: **the installed policy was verified; an unavailable checker was not falsely reported as passed.**

---

## 13. Rollback and Recovery Strategy

Rollback was designed to limit blast radius rather than revert the entire environment indiscriminately.

### Host access-port rollback

For a failed VPN host cutover, the recovery design was to restore only port 2 to the previous VLAN 1 untagged membership/PVID 1, remove its unintended VLAN 30 access membership, and renew the host's Ethernet connection locally. NAS port 3 and other migrated workloads were to remain untouched.

The NAS cutover similarly had a host-specific port-3 rollback rather than a blanket switch reset.

### DNS rollback

Resolver preparation retained the working listener and ACLs. Helper rollback verified ownership/content before removing only its own fragment, then validated and reactivated the prior service configuration. It did not blindly extract an archive over a changing live tree.

After Pi-hole's upstream moved to the new VPN-host address, any later host-network rollback would also require a coordinated upstream reversal. Reverting only the switch while leaving the filtering resolver pointed at an unreachable address would be an incomplete rollback.

### Router rollback

The new UDP redirect was applied with a 240-second automatic rollback window and confirmed only after relevant readback and regression checks. Later inspection showed the rule still present beyond the original rollback interval.

### Application rollback

The custom nginx site, Pi-hole configuration, and classroom service unit had private pre-change copies. Edits were constrained to expected values and validated before activation.

### Recovery limitations

Local-console recovery was available. No full backup restoration, power-loss recovery, or device-replacement disaster-recovery exercise was performed. Those remain separate tests.

---

## 14. Timeline and Provenance

This is a milestone chronology, not a continuous outage timeline.

| Local date / phase | Milestone |
|---|---|
| November 8 | Baselines and private backups; resolver ACL/source-NAT preparation and service-account permission correction |
| November 8 | NAS access-port cutover to VLAN 30; management and DNS checks; stale nginx binding and application-policy recovery |
| November 8–9, staged follow-up | Routed Pi-hole acceptance and private HTTPS canonical-origin checks completed |
| November 9 | VPN host inventory, local-console confirmation, and existing UDP 51820 forward inspection |
| November 9 | Future Unbound listener helper reviewed, locally tested, target-runtime tested, and operator-applied |
| November 9 | Downstream WireGuard DNAT installed with rollback protection and confirmed |
| November 9 | VPN host port-2 cutover, Ethernet renewal, new resolver verification, and Pi-hole upstream update |
| November 9 | Wi-Fi bypass removed; wired management, DNS, and HTTPS rechecked |
| November 9 | Upstream forward shown targeting OpenWrt; operator reported remote NAS and later phone SSH success |
| November 9 | Incident/change record assembled from artifacts, terminal results, screenshots, and operator reports |

Some router archives are named with a November 10 date while the operator's local working date was November 9. This report uses local CST, UTC−06:00. The differing filename dates are retained as observed; the router's complete clock/timezone configuration was not audited for this report. They are not treated as evidence of an additional outage or an extra day of downtime.

---

## 15. Corrective and Preventive Actions

### Completed corrective actions

- Migrated both target hosts into the intended server broadcast domain and subnet.
- Corrected stale web listener, allowlist, and public-origin references.
- Established the new Pi-hole/Unbound path without dropping resolver ACL safeguards.
- Added and verified the downstream VPN UDP redirect.
- Retargeted the upstream UDP forward, as shown by the operator's screenshot.
- Removed the VPN host's upstream Wi-Fi route/bypass after wired validation.
- Preserved SSH host identity, application security checks, NAS authentication, and VPN keys.
- Documented source-sensitive verification and the limits of each check.

### Follow-up register

| Priority | Follow-up | Completion evidence required |
|---|---|---|
| High | Instrument external VPN acceptance | Fresh cellular peer handshake/counters plus NAS, DNS, and intended internet access checks |
| Medium | Validate authenticated SMB access | Successful read of an authorized existing file using the legitimate account; no backup-source writes |
| Medium | Perform controlled persistence testing | Approved reboot/power-cycle plan; correct VLANs, reservations, routes, Wi-Fi state, and service recovery afterward |
| Medium | Review private application TLS identity | Intended hostname/address SAN coverage and explicit trust validation; authenticated tests without disabling certificate verification |
| Medium | Improve configuration/dependency inventory | Authoritative port map, service bind addresses, canonical origins, resolver dependencies, and external forwards maintained together |
| Medium | Add service-level observability | DNS UDP/TCP synthetic probes, completed forward/reply checks, systemd failure alerts, and WireGuard handshake-age monitoring |
| Low | Retire transitional resolver state | Coordinated removal/review of obsolete listener, ACL, and router exceptions only after final paths and policy dependencies are proven |
| Low | Review administrative key lifecycle | Per-device public keys, secure client storage, recovery access, and separately approved authentication-hardening policy |
| Low | Formalize configuration management | Version-controlled reviewed configuration/automation with secret separation; do not claim a complete GitOps/IaC deployment from this project alone |

No follow-up above is represented as already implemented merely because it appears in this report.

---

## 16. Lessons Learned

1. **Renumbering is dependency migration.** Listener addresses, canonical origins, subnet allowlists, backend ACLs, routes, and upstream forwarding all encode assumptions about network identity.
2. **Design the management return path, not just the destination.** A Wi-Fi IP can answer through Ethernet and fail at the exact moment it is expected to provide recovery.
3. **Model both control and data planes.** UCI intent, kernel nftables state, active sockets, and user-visible application behavior can disagree.
4. **Configuration validation must match execution context.** Root-only checks miss daemon-account file-access failures; working-directory errors can be operationally significant even with a zero exit code.
5. **Treat layered authorization as separate gates.** Router policy, nginx source ACLs, application Host/Origin validation, and user authentication are not interchangeable controls.
6. **Prepare before starting the outage.** Resolve privileges, test helpers, confirm physical ports, and prepare renewal commands before an access-port move.
7. **Observe before retrying.** A timeout or instantaneous startup assertion is not proof that a state-changing action failed.
8. **Use positive and negative controls.** Successful new-origin access and rejection of the old Host are stronger evidence than a single successful page load.
9. **Preserve the last working path until the replacement is demonstrated.** Remove bypasses and transitional allowances as controlled follow-up, not speculative cleanup.
10. **Classify evidence honestly.** Operator-reported remote access is useful, but it is not a packet capture, an authenticated storage read, or an availability SLO measurement.
11. **Avoid security regression as a troubleshooting shortcut.** No public SSH, anonymous shares, disabled CSRF, or blind host-key acceptance was needed.
12. **Keep the retrospective blameless.** Inventory drift, tightly coupled configuration, and inadequate observability explain the difficulty more usefully than attributing fault to the operator.

---

## 17. Portfolio and Interview-Ready Experience

### Project summary

> Executed a staged migration of two service-bearing Linux infrastructure hosts from a flat ISP-managed network into an 802.1Q server VLAN behind OpenWrt. Integrated managed-switch access-port/PVID changes, DHCP reservations, least-privilege inter-zone policy, Pi-hole/Unbound DNS continuity, strict SSH host-key continuity, and two-stage WireGuard DNAT. Diagnosed and remediated stale application bindings, daemon-permission failures, routed DNS acceptance issues, canonical-origin mismatches, and a non-independent Wi-Fi recovery path. Used AI-assisted operational automation with 92 distinct automated helper test cases and evidence-based post-change validation.

### Defensible resume/project bullets

- Executed a dependency-aware, two-host Linux server migration into an IEEE 802.1Q Servers VLAN while preserving existing Trusted/IoT topology and unrelated access-port configurations.
- Implemented and validated service-specific OpenWrt fw4/nftables forwarding and an IPv4 UDP DNAT path for a WireGuard endpoint behind an upstream ISP gateway.
- Re-established Pi-hole-to-Unbound DNS continuity through source-ACL preparation, interface-scoped query acceptance, future-address listener pre-staging, and UDP/TCP forward/reply correlation.
- Diagnosed shared-nginx failure propagation and stale backend canonical-origin enforcement; restored web availability without weakening Host validation, CSRF controls, or authentication.
- Maintained SSH cryptographic host continuity across IP renumbering with dedicated client identities, pinned host keys, and strict verification.
- Developed and exercised AI-assisted Python operational safeguards covering backups, locking, file permissions, configuration validation, bounded readiness, timeout handling, and guarded rollback.
- Produced an evidence-based incident review, validation matrix, host-specific rollback procedure, and corrective/preventive-action register with explicit testing limitations.

Use these as personal-project experience. Do not relabel the work as enterprise employment, claim sole unaided authorship, or add unsupported availability/performance improvements.

---

## 18. Terminology and Competency Map

| Term | Demonstrated meaning in this project |
|---|---|
| IEEE 802.1Q | Tagged VLAN transport between the managed switch and router |
| Broadcast-domain segmentation | Separation of upstream, Servers, Trusted, and IoT Layer 2 membership |
| Access port / PVID | Untagged endpoint membership and ingress VLAN classification |
| Router-on-a-stick | Routed VLAN subinterfaces sharing a physical trunk |
| Management plane | SSH and administrative web paths used to control infrastructure |
| Control plane vs data plane | Saved UCI intent versus actual nftables packet processing |
| Inter-zone policy | Service-specific permissions between routed security zones |
| Least privilege | Narrow sources/destinations/ports instead of blanket administrative access |
| DNAT | Inbound destination translation at the ISP gateway and OpenWrt |
| SNAT / masquerading | Source translation affecting backend identity and VPN egress |
| Conntrack | Stateful handling and acceptance of translated flows/replies |
| Dependency graph | Client → Pi-hole → Unbound service chain |
| Canonical origin | Exact scheme/authority identity enforced by the private application |
| Host/Origin validation | Application request-integrity checks preserved during renumbering |
| CSRF | Browser write-request protection retained during origin remediation |
| DAC / UID / GID | Unix file ownership and permissions controlling daemon readability |
| Privilege dropping | Daemon execution context differing from root validation context |
| IP_FREEBIND | Specific future-address socket preparation before local IP assignment |
| Idempotency | Exact existing helper state validated as no-op, not blindly reapplied |
| Compare-and-check guards | Expected file/configuration identity checked before mutation |
| Bounded readiness | Finite waiting for the real service/listener state |
| Blast-radius control | One host/port/service changed while unrelated workloads remained untouched |
| MOP | Method of Procedure for staged execution and acceptance |
| Backout plan | Host-specific, dependency-aware rollback rather than a blanket reset |
| RCA / PIR | Root-cause analysis and post-implementation/incident review |
| CAPA | Corrective and preventive actions separated from completed remediation |
| Positive/negative controls | Expected successful access and expected rejection tested independently |
| Evidence provenance | Direct checks, screenshots, and operator reports explicitly distinguished |
| Configuration drift | Historical assumptions no longer matching actual ports, addresses, or policy |
| Service restoration vs acceptance | Running endpoints versus complete authenticated end-user workflows |

---

## 19. Evidence Index and Publication Hygiene

The source artifacts are retained privately in the migration working directories. The evidence labels below are document-local references, not externally issued incident/ticket identifiers. A sanitized summary accompanies this report; the raw artifacts are not included in the publication bundle.

| ID | Private artifact / source | Supports |
|---|---|---|
| E01 | `live-verification-v2.json` | Transitional resolver ACL preparation, service permissions, permitted and denied DNS paths |
| E02 | `nginx-binding-repair.json` | Exact stale-listener correction, native validation, service recovery |
| E03 | `finishing-access-verification.json` and captured DNS checks | Routed Pi-hole policy and service readback |
| E04 | `https-trusted-access.json` | Trusted-subnet HTTPS allowlist with deny-all retained |
| E05 | `classroom-origin-repair.json` | Unit validation, runtime origin, new-Host 200 and retired-Host 403 |
| E06 | `target-python313-verification.json` | Target-runtime 37-case suite, transfer hashes, inert default behavior |
| E07 | `pihole-final-upstream.json` | New upstream, native validation, service activation, other settings preserved |
| E08 | `final-dns-path-verification.json` | Client queries correlated with new-backend forward/reply records |
| E09 | `forward-confirmation.json` | Exact UDP redirect, live ingress/forward rules, confirmation |
| E10 | `switch-step3-port2-pvid30.json` and preceding membership snapshots | Port-2-only PVID/membership cutover |
| E11 | Router/switch backup manifests | Verified export presence, sizes/digests, prepared-forward inclusion |
| E12 | Operator screenshots and captured SSH/route/HTTPS results | Host renewal, Wi-Fi cleanup, upstream forwarding UI, user-reported access |

### Never publish with this report

- Router/switch/NAS configuration archives.
- `.env`, vault exports, browser session data, cookies, or API tokens.
- SSH private keys, WireGuard private/preshared keys, client exports, or QR codes.
- Password hashes, Wi-Fi PSKs, recovery codes, or unredacted personal screenshots.
- Full DNS browsing logs or complete raw session transcripts.
- Public WAN addresses or unnecessary client/device identities.

The companion sanitized summary is a derived description of actual observations, **not fabricated raw diagnostic output**. Original logs and configuration backups remain private.

---

## 20. Final Disposition

The target hosts were successfully renumbered and moved into the intended routed Servers VLAN. DNS dependencies were re-established on the final addresses, web-service regressions were remediated, the upstream Wi-Fi bypass was removed, and the two-stage WireGuard forwarding path was configured. Remote NAS access and later phone SSH success were reported by the operator.

The change is documented as an implemented migration with verified core service paths and explicitly bounded acceptance evidence. Outstanding storage-authentication, persistence, TLS-trust, and independently instrumented external VPN validation are follow-up work—not silently assumed successes.

### 20.1 Post-migration addendum — November 10, 2026: stale VPN client DNS

**Reported symptom:** WireGuard could reach home-network resources, but internet access appeared broken.

**Direct server-side checks:** Unbound answered control queries over UDP and TCP; Pi-hole answered through the new backend; VPN-01's IPv4 forwarding settings were enabled; and a source-bound, DNS-independent HTTPS request from VPN-01 to `https://1.1.1.1/cdn-cgi/trace` returned HTTP 200. These were host-side checks, not a claim that the phone's entire forwarded path had already been instrumented.

**Root cause established by the operator:** The phone's WireGuard profile retained the retired DNS-server address after the infrastructure migration. Literal-IP access to local resources could succeed while DNS-dependent internet access failed.

**Corrective action and outcome:** The operator updated the tunnel profile to the new DNS-server address and reported that internet access then worked. The intended client DNS destination is Pi-hole at `10.100.20.10`; Pi-hole forwards to Unbound at `10.100.20.20:5335`. No additional server restart, firewall change, or WireGuard key replacement was required for this fix.

**Evidence classification:** Client configuration diagnosis and successful recovery are operator-reported. No new peer packet capture, throughput measurement, or comprehensive IPv6 assessment is implied.

**Preventive action:** Add client-side DNS, Endpoint, and AllowedIPs review to the migration closeout checklist. Preserve the distinction between local-IP reachability, public-IP egress, and name resolution when validating VPN acceptance. Do not request full client configuration exports or QR codes to inspect these nonsecret fields.
