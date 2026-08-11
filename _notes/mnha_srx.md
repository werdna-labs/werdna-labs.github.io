---
title: "vSRX Multinode High Availability on ESXi 8 — Lab Build Guide"
description: "Two-node vSRX 3.0 MNHA firewall-on-a-stick on a single standalone ESXi 8 host. Base L4 firewalling, NAT, DHCP, and SRG failover — no subscription services."
updated: 2026-08-11
---

# vSRX Multinode High Availability on ESXi 8
## Working Draft - 

A two-node **Multinode High Availability (MNHA)** vSRX 3.0 build, deployed firewall-on-a-stick on a
single free/standalone ESXi 8 host. Junos 25.4R1.12.

This guide covers **only the capabilities available without security subscription services**:

- Stateful zone-based L4 firewalling
- Source NAT
- Screens (IDS options)
- DHCP relay and on-box DHCP server
- Inter-VLAN routing on 802.1Q subinterfaces
- OSPF and eBGP for the inter-node control plane
- MNHA: ICL with IPsec link encryption, SRG1, floating VIPs, signal-route activeness

**Explicitly out of scope** (subscription-licensed, deliberately omitted): IDP/IPS, application
identification, UTM (AV / web filtering / anti-spam), SecIntel, ATP Cloud / AAMW, and Security
Director Cloud management. Nothing in this document requires a security services subscription,
a cloud tenant, or an outbound management dial-home. If you are interested in these features, trial licensing is available. In the spirit of not embedding dead links, reach out to your HPE Networking sales team, or google (do people even do that anymore?) "SRX Eval".  

> **Licensing note:** vSRX itself still requires a Junos software license for throughput and
> feature entitlement; evaluation licenses are adequate for lab use. MNHA entitlement varies by
> platform and license tier — confirm against current Juniper licensing documentation for your
> deployment. This guide makes no claim that MNHA is license-free.

---

## Table of contents

1. [Design overview](#1-design-overview)
2. [Topology](#2-topology)
3. [ESXi host networking](#3-esxi-host-networking)
4. [Building the vSRX VMs](#4-building-the-vsrx-vms)
5. [Base Junos onboarding](#5-base-junos-onboarding)
6. [Interfaces](#6-interfaces)
7. [Security zones](#7-security-zones)
8. [ICL transport: IPsec, OSPF, eBGP](#8-icl-transport-ipsec-ospf-ebgp)
9. [SRG1: HA chassis, VIPs, activeness](#9-srg1-ha-chassis-vips-activeness)
10. [NAT (and why the WAN needs no VIP)](#10-nat-and-why-the-wan-needs-no-vip)
11. [Security policies and screens](#11-security-policies-and-screens)
12. [Policy map](#12-policy-map)
13. [DHCP](#13-dhcp)
14. [Logging](#14-logging)
15. [Commit and verify](#15-commit-and-verify)
- [Appendix A — ESXi security settings](#appendix-a--esxi-security-settings)
- [Appendix B — Addressing plan](#appendix-b--addressing-plan)
- [Appendix C — vSRX2 delta summary](#appendix-c--vsrx2-delta-summary)
- [Appendix D — Failure modes worth knowing](#appendix-d--failure-modes-worth-knowing)

---

## 1. Design overview

**Firewall on a stick.** `ge-0/0/1` is a single 802.1Q trunk carrying every VLAN. The SRX
terminates each VLAN as an L3 subinterface and is the default gateway for it. There is no L2
switching on the box.

**VLAN roles.** VLAN 4092 is the internet/WAN VLAN (DHCP client to the ISP router). VLAN 255 is
the LAN management VLAN and the trunk native VLAN. VLAN 254 is WLAN management. VLANs
10/20/30/40/50/70/110/120/225/2054 are client and server segments. VLAN 3666 is an inter-SRX
point-to-point transit used during parallel-run cutover.

**Dedicated ICL.** `ge-0/0/0` is the interchassis link, IPsec-encrypted, on a host-only vSwitch
that never leaves the ESXi host. `fxp0` is out-of-band management on that same host-only vSwitch.

**Activeness is decided by SRG1**, using signal routes carried over an inter-node eBGP session —
not by anything on the WAN. This is the single most important thing to internalise about the
design; Section 10 explains why the WAN side needs no shared address.

### Three decisions that shape everything else

| Decision | Why |
|---|---|
| `deployment-type hybrid` | LAN VLANs present a floating default gateway (VIP) to clients — switching-style behaviour — while the WAN/ICL side is routed and activeness is driven by signal routes. Hybrid does both; pure switching or pure routing each gives you only one half. |
| **No `use-virtual-mac`** | The VIP resolves to the active node's own interface MAC. On failover the new active node sends a gratuitous ARP to re-point the VIP. GARP-based convergence avoids the synthetic `02:..` VMAC entirely, which removes a whole class of vSwitch forged-transmit drops. |
| **No WAN VIP** | MNHA cannot float a DHCP-learned address. Each node holds its own lease and source-NATs to its own WAN interface. See Section 10. |

### Lab caveat

Both nodes live on one ESXi host, and the ICL plus management ride a host-only vSwitch. That
discards the geographic-resilience premise of MNHA — if the host dies, both nodes die. Acceptable
for a lab. **Do not replicate the single-host ICL in production.**

---

## 2. Topology

```text
                                  INTERNET
                                     |
                          +--------------------+
                          |     ISP Router     |  serves DHCP (public IP)
                          +---------+----------+
                                    | VLAN 4092 (access)
                                    |
                     +------------------------------+
                     |        Aruba CX 6200M        |
                     |  1/1/1  : access VLAN 4092   |
                     |  1/1/49 : trunk              |
                     |           native 255         |
                     |           allowed ALL VLANs  |
                     +--------------+---------------+
                                    | 10G trunk (all VLANs, native = 255)
                                    |
 ===================================|====================================
 ESXi 8 host                        |
                          +---------+----------+
                          |      vSwitch0      |  uplink = 10G vmnic
                          |  PG "vSRX-Trunk"   |  VLAN 4095 (VGT)
                          |  Prom/MACchg/Forged = Accept
                          +----+----------+----+
                               |          |
                           ge-0/0/1   ge-0/0/1      <-- VGT trunk, both nodes
                               |          |
                        +------+----+ +---+-------+
                        |   vSRX1   | |   vSRX2   |
                        |  node 1   | |  node 2   |
                        | prio 200  | | prio 100  |
                        +--+-----+--+ +--+-----+--+
                           |     |       |     |
                        fxp0  ge-0/0/0  ge-0/0/0  fxp0
                           |     |       |     |
          +----------------+-----+-------+-----+----------------+
          |          vSwitch "vSRX Networks"  (NO uplink)       |
          |                                                     |
          |  PG "SRX ICL"        : ge-0/0/0 (172.16.254.2/.3)   |
          |  PG "SRX Management" : fxp0     (192.168.255.2/.3)  |
          |  Prom/MACchg/Forged = Accept                        |
          +-----------------------------------------------------+
 =========================================================================

 Control-plane overlay (entirely internal to the host or over the trunk):
   - ICL IPsec tunnel  : lo0 172.16.255.1 <-> 172.16.255.2 (via ge-0/0/0 transit)
   - OSPF area 0       : ge-0/0/0 (p2p) + ge-0/0/1.3666  -> advertises loopbacks
   - eBGP (multihop 2) : 172.16.255.1 <-> 172.16.255.2   -> carries signal routes
```

---

## 3. ESXi host networking

Free/standalone ESXi 8 provides Virtual Standard Switches (vSS) only — no vCenter, no vDS. That is
convenient here: the vDS L2-security-policy propagation defect (guest requests a MAC-change or
forged-transmit behaviour, port silently blocks; addressed in ESXi 8.0 U2b) cannot affect a vSS.

All of the following is done in the ESXi host client under **Networking**.

### 3a. The trunk — vSwitch0 and the VGT port group

`vSwitch0` already exists with your 10G vmnic uplinked to the 6200M trunk. Add one port group:

| Setting | Value |
|---|---|
| Name | `vSRX-Trunk` |
| VLAN ID | **4095** — Virtual Guest Tagging. 4095 means "all VLANs, pass 802.1Q tags up to the guest." The vSRX does the tagging. (4095 is reserved for VGT, which is why the highest usable real VLAN here is 4092.) |
| Promiscuous mode | Accept |
| MAC address changes | Accept |
| Forged transmits | Accept |

**Why all three Accept.** Juniper's documented vSRX requirement is all three set to Accept. For
this pure-L3 firewall-on-a-stick, the one that functionally matters is **Promiscuous** on the VGT
group, so the guest reliably receives every VLAN's frames. **Forged Transmits** and **MAC
Changes** become genuinely mandatory the moment `use-virtual-mac` is enabled on a VIP, because
the active node then sources frames from a synthetic `02:..` VMAC that is not the vNIC's assigned
MAC — and a vSS with Forged Transmits = Reject silently drops those. This build deliberately does
**not** use `use-virtual-mac` (GARP convergence instead, Section 9), so those two are
belt-and-suspenders. Set them anyway: it costs nothing and it is exactly the knob behind subtle
HA flaps that leave no trace in the Junos logs.

### 3b. The host-only switch — "vSRX Networks"

**Networking → Virtual switches → Add standard virtual switch:**

- Name: `vSRX Networks`
- Uplink: **NONE** — remove the default uplink; this switch is internal to the host
- MTU: 1500 is fine. Optionally 9000 to give the ICL IPsec tunnel header headroom — harmless on a
  host-only switch, since nothing physical constrains it.

Then two port groups on it:

| Name | VLAN ID | Security |
|---|---|---|
| `SRX ICL` | 0 (untagged) | All three = Accept |
| `SRX Management` | 0 (untagged) | All three = Accept |

Accept on all three matters *more* here than on the trunk. This is where the ICL IPsec endpoints
live, and a Forged Transmits = Reject on this switch produces an ICL that comes up and then drops
cold-sync in a way that presents as an IPsec problem but is not.

**On reaching `fxp0`:** this segment is host-only with no uplink. To actually reach `fxp0` you
either (a) place a management VM or jump host on the `SRX Management` port group, or (b) manage
in-band over the LAN VLANs and treat `fxp0` as console-adjacent. The `mgmt_junos` default route
(Section 5) points at `192.168.255.1`, which must be something that actually lives on that segment
if you want `fxp0` to route off-net.

### 3c. The AOS-CX 6200M trunk

The ESXi uplink lands on a 6200M trunk. The native VLAN must match the SRX `native-vlan-id 255`
end to end, or the management VLAN black-holes on untagged frames.

```text
vlan 10,20,30,40,50,70,110,120,225,254,255,2054,3666,4092

interface 1/1/49
    no shutdown
    description ESXi-host-vSwitch0
    no routing
    vlan trunk native 255
    vlan trunk allowed 10,20,30,40,50,70,110,120,225,254,255,2054,3666,4092

interface 1/1/1
    no shutdown
    description ISP-uplink
    no routing
    vlan access 4092
```

**The native-VLAN alignment chain:** 6200M sends VLAN 255 untagged → ESXi VGT(4095) passes
untagged up to the guest → SRX `native-vlan-id 255` interprets untagged as VLAN 255. Break any
link in that chain and *only* VLAN 255 misbehaves, which makes for a memorable afternoon.

---

## 4. Building the vSRX VMs

Deploy from the official **vSRX 3.0 OVA**. The OVA gets firmware (EFI), disk controller, and boot
order right for free; hand-building from the qcow/vmdk is where boot loops come from. If you must
hand-build: **EFI firmware**, and **VMXNET3** adapters — not E1000. vSRX 3.0 expects VMXNET3.

### Per-VM sizing

Without the L7 service daemons, the resource footprint is modest:

| Resource | Value |
|---|---|
| vCPU | 2 |
| RAM | 4 GB |
| Disk | ~20 GB thin |
| NICs | 3 × VMXNET3 (ordering below) |

> If you later add subscription services (IDP, UTM, AAMW), plan on 4 vCPU / 8 GB — those daemons
> are considerably hungrier once enrolled. This guide's build does not need it.

### NIC order is the whole ballgame

vSRX maps vNICs to interfaces strictly by PCI/adapter order. Add them in exactly this order:

| Adapter | Interface | Port group |
|---|---|---|
| Network adapter 1 | `fxp0` | `SRX Management` |
| Network adapter 2 | `ge-0/0/0` | `SRX ICL` |
| Network adapter 3 | `ge-0/0/1` | `vSRX-Trunk` |

Adapter 1 is **always** `fxp0` on vSRX 3.0. Attach them out of order and the box boots fine, then
nothing makes sense — the ICL never forms, the trunk is dead, and the config takes the blame.
Verify with `show interfaces terse` after first boot, before touching anything else.

Build vSRX2 identically. The only VM-level difference is the name; every per-node addressing
difference lives in Junos.

---

## 5. Base Junos onboarding

First boot drops you at the Linux shell as `root` with no password. Run `cli` for operator mode,
then add your license:

```text
request system license add terminal
```

Paste the key, press Enter for a newline, `Ctrl+D` to finish. Then `configure`.

```text
set system host-name vSRX1
set system root-authentication encrypted-password "<hash>"
set system login user admin uid 2000
set system login user admin class super-user
set system login user admin authentication encrypted-password "<hash>"
set system domain-name lab.example.net
set system time-zone US/Central
set system name-server 10.1.50.10
set system name-server 10.1.50.11
set system management-instance
set system services ssh protocol-version v2
set system services ssh max-sessions-per-connection 20
set system services ssh rate-limit 32
set system services netconf ssh rate-limit 32
set system services netconf rfc-compliant
set system license autoupdate url https://ae1.juniper.net/junos/key_retrieval
```

> Password hashes are placeholders — generate your own. Never publish real `$6$` or `$9$` hashes.

### NTP

Set time before anything else that cares about it — certificate validation, log correlation
across two nodes, and IPsec troubleshooting all degrade badly under clock skew.

```text
set system ntp server 10.1.50.10
set system ntp server 10.1.50.11
```

### Base syslog

```text
set system syslog file interactive-commands interactive-commands any
set system syslog file messages any any
set system syslog file messages authorization info
```

### Management routing instance

`mgmt_junos` is the reserved VRF enabled by `set system management-instance` above. Give it a
default route out of `fxp0`'s segment:

```text
set routing-instances mgmt_junos routing-options static route 0.0.0.0/0 next-hop 192.168.255.1
```

### IKE package

Pull the updated IKE package before building the IPsec profile — the ICL's modern DH groups
depend on it:

```text
request system software add optional://junos-ike.tgz
```

> **vSRX2 deltas:** hostname `vSRX2`; its own root and admin hashes. Everything else in this
> section is identical.

---

## 6. Interfaces

`ge-0/0/0` is the ICL transit. `ge-0/0/1` is the trunk — native VLAN 255, flexible tagging, one L3
unit per VLAN. `lo0` carries `127.0.0.1` plus the BGP/HA loopback.

```text
set interfaces ge-0/0/0 unit 0 family inet address 172.16.254.2/24

set interfaces ge-0/0/1 flexible-vlan-tagging
set interfaces ge-0/0/1 native-vlan-id 255

set interfaces ge-0/0/1 unit 10 description clients
set interfaces ge-0/0/1 unit 10 vlan-id 10
set interfaces ge-0/0/1 unit 10 family inet address 10.1.10.2/24

set interfaces ge-0/0/1 unit 20 description "wireless clients"
set interfaces ge-0/0/1 unit 20 vlan-id 20
set interfaces ge-0/0/1 unit 20 family inet address 10.1.20.2/24

set interfaces ge-0/0/1 unit 30 description "iot clients"
set interfaces ge-0/0/1 unit 30 vlan-id 30
set interfaces ge-0/0/1 unit 30 family inet address 10.1.30.2/24

set interfaces ge-0/0/1 unit 40 description "tunneled clients"
set interfaces ge-0/0/1 unit 40 vlan-id 40
set interfaces ge-0/0/1 unit 40 family inet address 10.1.40.2/24

set interfaces ge-0/0/1 unit 50 description servers
set interfaces ge-0/0/1 unit 50 vlan-id 50
set interfaces ge-0/0/1 unit 50 family inet address 10.1.50.2/24

set interfaces ge-0/0/1 unit 70 description "srx test"
set interfaces ge-0/0/1 unit 70 vlan-id 70
set interfaces ge-0/0/1 unit 70 family inet address 10.1.70.2/24

set interfaces ge-0/0/1 unit 110 description "lab clients"
set interfaces ge-0/0/1 unit 110 vlan-id 110
set interfaces ge-0/0/1 unit 110 family inet address 10.1.110.2/24

set interfaces ge-0/0/1 unit 120 description "lab wireless clients"
set interfaces ge-0/0/1 unit 120 vlan-id 120
set interfaces ge-0/0/1 unit 120 family inet address 10.1.120.2/24

set interfaces ge-0/0/1 unit 225 description "guest clients"
set interfaces ge-0/0/1 unit 225 vlan-id 225
set interfaces ge-0/0/1 unit 225 family inet address 10.1.225.2/24

set interfaces ge-0/0/1 unit 254 description "wlan mgmt"
set interfaces ge-0/0/1 unit 254 vlan-id 254
set interfaces ge-0/0/1 unit 254 family inet address 10.1.254.2/24

set interfaces ge-0/0/1 unit 255 description mgmt
set interfaces ge-0/0/1 unit 255 vlan-id 255
set interfaces ge-0/0/1 unit 255 family inet address 10.1.255.2/24

set interfaces ge-0/0/1 unit 2054 description "wlan lab mgmt"
set interfaces ge-0/0/1 unit 2054 vlan-id 2054
set interfaces ge-0/0/1 unit 2054 family inet address 10.20.54.2/24

set interfaces ge-0/0/1 unit 3666 description srx-ptp-cut
set interfaces ge-0/0/1 unit 3666 vlan-id 3666
set interfaces ge-0/0/1 unit 3666 family inet address 172.16.253.2/29

set interfaces ge-0/0/1 unit 4092 description internet
set interfaces ge-0/0/1 unit 4092 vlan-id 4092
set interfaces ge-0/0/1 unit 4092 family inet dhcp

set interfaces fxp0 unit 0 family inet address 192.168.255.2/24

set interfaces lo0 unit 0 family inet address 127.0.0.1/32
set interfaces lo0 unit 0 family inet address 172.16.255.1/32
```

VLAN 4092 is `family inet dhcp` — a DHCP **client** to the ISP. It is not a VIP and cannot be:
MNHA cannot float a DHCP-learned WAN address. Each node independently holds its own lease and
source-NATs to its own WAN interface (Section 10).

The `.x.2` addresses on the LAN units are each node's own real interface address. The `.x.1`
gateway that clients use is the floating VIP defined in Section 9.

> **vSRX2 deltas:**
> - `ge-0/0/0.0` → `172.16.254.3/24`
> - `ge-0/0/1.<unit>` → `10.1.X.3` / `10.20.54.3`
> - `ge-0/0/1.3666` → `172.16.253.3/29`
> - `fxp0` → `192.168.255.3/24`
> - `lo0` → `172.16.255.2/32`

---

## 7. Security zones

Three functional zones plus `junos-host`. `halink` is the ICL control zone, locked down to exactly
the protocols the ICL needs. `WAN` is the internet edge, screened. `LAN` holds every internal VLAN.

```text
set security zones security-zone halink host-inbound-traffic system-services ike
set security zones security-zone halink host-inbound-traffic system-services ping
set security zones security-zone halink host-inbound-traffic system-services high-availability
set security zones security-zone halink host-inbound-traffic system-services ssh
set security zones security-zone halink host-inbound-traffic protocols bfd
set security zones security-zone halink host-inbound-traffic protocols bgp
set security zones security-zone halink host-inbound-traffic protocols ospf
set security zones security-zone halink interfaces ge-0/0/0.0
set security zones security-zone halink interfaces lo0.0

set security zones security-zone WAN screen untrust-screen
set security zones security-zone WAN host-inbound-traffic system-services ike
set security zones security-zone WAN host-inbound-traffic system-services ping
set security zones security-zone WAN host-inbound-traffic system-services dhcp
set security zones security-zone WAN host-inbound-traffic protocols bfd
set security zones security-zone WAN interfaces ge-0/0/1.4092

set security zones security-zone LAN host-inbound-traffic system-services all
set security zones security-zone LAN host-inbound-traffic protocols all
set security zones security-zone LAN interfaces ge-0/0/1.10
set security zones security-zone LAN interfaces ge-0/0/1.20
set security zones security-zone LAN interfaces ge-0/0/1.30
set security zones security-zone LAN interfaces ge-0/0/1.40
set security zones security-zone LAN interfaces ge-0/0/1.50
set security zones security-zone LAN interfaces ge-0/0/1.70
set security zones security-zone LAN interfaces ge-0/0/1.110
set security zones security-zone LAN interfaces ge-0/0/1.120
set security zones security-zone LAN interfaces ge-0/0/1.225
set security zones security-zone LAN interfaces ge-0/0/1.254
set security zones security-zone LAN interfaces ge-0/0/1.255
set security zones security-zone LAN interfaces ge-0/0/1.2054
set security zones security-zone LAN interfaces ge-0/0/1.3666
```

`system-services dhcp` on the WAN zone is what allows `ge-0/0/1.4092`'s DHCP client to accept the
ISP's OFFER. Without it the lease never binds, and the symptom is a silent no-address condition
with nothing obviously wrong.

The WAN zone intentionally does **not** allow `bgp` or `ospf` — there are no routing neighbours out
there. `untrust-screen` (Section 11) is bound here.

> **vSRX2 deltas:** identical zone structure. Keep it symmetric — drifting zone or interface
> membership between MNHA nodes is the classic "works until failover" bug.

---

## 8. ICL transport: IPsec, OSPF, eBGP

The ICL is an IPsec tunnel between the two loopbacks, transported over `ge-0/0/0`. OSPF makes the
loopbacks reachable; eBGP (loopback-to-loopback, multihop) carries the activeness signal. Build in
this order so the loopbacks are routable before HA and BGP attempt to come up.

### 8a. IPsec profile for the ICL

```text
set security ike proposal MNHA_IKE_PROP description mnha_link_encr_tunnel
set security ike proposal MNHA_IKE_PROP authentication-method pre-shared-keys
set security ike proposal MNHA_IKE_PROP dh-group group14
set security ike proposal MNHA_IKE_PROP authentication-algorithm sha-256
set security ike proposal MNHA_IKE_PROP encryption-algorithm aes-256-cbc
set security ike proposal MNHA_IKE_PROP lifetime-seconds 3600

set security ike policy MNHA_IKE_POL description mnha_link_encr_tunnel
set security ike policy MNHA_IKE_POL proposals MNHA_IKE_PROP
set security ike policy MNHA_IKE_POL pre-shared-key ascii-text "<psk>"

set security ike gateway MNHA_IKE_GW ike-policy MNHA_IKE_POL
set security ike gateway MNHA_IKE_GW version v2-only

set security ipsec proposal MNHA_IPSEC_PROP description mnha_link_encr_tunnel
set security ipsec proposal MNHA_IPSEC_PROP protocol esp
set security ipsec proposal MNHA_IPSEC_PROP encryption-algorithm aes-256-gcm
set security ipsec proposal MNHA_IPSEC_PROP lifetime-seconds 3600

set security ipsec policy MNHA_IPSEC_POL description mnha_link_encr_tunnel
set security ipsec policy MNHA_IPSEC_POL proposals MNHA_IPSEC_PROP

set security ipsec vpn IPSEC_VPN_ICL ha-link-encryption
set security ipsec vpn IPSEC_VPN_ICL ike gateway MNHA_IKE_GW
set security ipsec vpn IPSEC_VPN_ICL ike ipsec-policy MNHA_IPSEC_POL
```

The PSK is stored as a per-node `$9$` hash, so the two configs **will** show different ciphertext.
That is expected. What matters is that the **cleartext** matches on both nodes. A cleartext
mismatch breaks `ha-link-encryption` silently — if the ICL will not form, rule this out first.

### 8b. OSPF — loopback reachability

```text
set protocols ospf area 0.0.0.0 interface ge-0/0/0.0 interface-type p2p
set protocols ospf area 0.0.0.0 interface lo0.0 passive
set protocols ospf area 0.0.0.0 interface ge-0/0/1.70 passive
set protocols ospf area 0.0.0.0 interface ge-0/0/1.3666
set protocols ospf export EXPORT-DEFAULT-AND-CONNECTED
```

OSPF here is inter-SRX only (`ge-0/0/0` plus the `.3666` transit); clients never see it.
`EXPORT-DEFAULT-AND-CONNECTED` originates a default plus connected routes, and it must be present
and identical on **both** nodes. If only one node originates the default, the inside loses its
learned default the moment the other node becomes active.

### 8c. eBGP — carrying the signal

eBGP between loopbacks. Native eBGP TTL is 1 and the loopbacks are one hop past the physical
interface, so `multihop ttl 2` is required.

```text
set routing-options autonomous-system 64512
set protocols bgp group EBGP type external
set protocols bgp group EBGP multihop ttl 2
set protocols bgp group EBGP local-address 172.16.255.1
set protocols bgp group EBGP peer-as 64513
set protocols bgp group EBGP neighbor 172.16.255.2
set protocols bgp group EBGP export MNHA_EXPORT
```

### 8d. Export policies and signal conditions

This is the heart of activeness. SRG1 installs the active signal route (`10.255.255.1/32`) on
whichever node is active and the backup signal route (`10.255.255.2/32`) on the other.
`MNHA_EXPORT` advertises with a better metric from the active node, so the peer always knows who
is hot.

```text
set policy-options policy-statement EXPORT-DEFAULT-AND-CONNECTED term ORIGINATE-DEFAULT from route-filter 0.0.0.0/0 exact
set policy-options policy-statement EXPORT-DEFAULT-AND-CONNECTED term ORIGINATE-DEFAULT then accept
set policy-options policy-statement EXPORT-DEFAULT-AND-CONNECTED term CONNECTED from protocol direct
set policy-options policy-statement EXPORT-DEFAULT-AND-CONNECTED term CONNECTED then accept
set policy-options policy-statement EXPORT-DEFAULT-AND-CONNECTED term REJECT-ELSE then reject

set policy-options policy-statement MNHA_EXPORT term active from protocol direct
set policy-options policy-statement MNHA_EXPORT term active from protocol static
set policy-options policy-statement MNHA_EXPORT term active from condition ACTIVE_SIG
set policy-options policy-statement MNHA_EXPORT term active then metric 10
set policy-options policy-statement MNHA_EXPORT term active then accept

set policy-options policy-statement MNHA_EXPORT term backup from protocol direct
set policy-options policy-statement MNHA_EXPORT term backup from protocol static
set policy-options policy-statement MNHA_EXPORT term backup from condition BACKUP_SIG
set policy-options policy-statement MNHA_EXPORT term backup then metric 20
set policy-options policy-statement MNHA_EXPORT term backup then accept

set policy-options policy-statement MNHA_EXPORT term fallback from protocol direct
set policy-options policy-statement MNHA_EXPORT term fallback from protocol static
set policy-options policy-statement MNHA_EXPORT term fallback then metric 30
set policy-options policy-statement MNHA_EXPORT term fallback then accept

set policy-options policy-statement MNHA_EXPORT term default then reject

set policy-options condition ACTIVE_SIG if-route-exists address-family inet 10.255.255.1/32
set policy-options condition ACTIVE_SIG if-route-exists address-family inet table inet.0
set policy-options condition BACKUP_SIG if-route-exists address-family inet 10.255.255.2/32
set policy-options condition BACKUP_SIG if-route-exists address-family inet table inet.0
```

> **vSRX2 deltas:**
> - `set routing-options autonomous-system 64513`
> - `set protocols bgp group EBGP local-address 172.16.255.2`
> - `set protocols bgp group EBGP peer-as 64512`
> - `set protocols bgp group EBGP neighbor 172.16.255.1`
> - PSK ciphertext differs; cleartext **must** match
>
> Optional parallel-run plumbing, if you are cutting over from an existing edge firewall — vSRX2
> points its default at vSRX1 across the `.3666` transit at a worse preference, and it is removed
> once cutover completes:
> ```text
> set routing-options static route 0.0.0.0/0 qualified-next-hop 172.16.253.2 preference 49
> ```

---

## 9. SRG1: HA chassis, VIPs, activeness

Peer identity, ICL interface and IPsec profile, liveness detection, then SRG1 with one VIP per LAN
VLAN.

```text
set chassis high-availability local-id 1 local-ip 172.16.255.1
set chassis high-availability peer-id 2 peer-ip 172.16.255.2
set chassis high-availability peer-id 2 interface ge-0/0/0.0
set chassis high-availability peer-id 2 vpn-profile IPSEC_VPN_ICL
set chassis high-availability peer-id 2 liveness-detection minimum-interval 400
set chassis high-availability peer-id 2 liveness-detection multiplier 5

set chassis high-availability services-redundancy-group 0 peer-id 2

set chassis high-availability services-redundancy-group 1 deployment-type hybrid
set chassis high-availability services-redundancy-group 1 peer-id 2
```

### VIP table

One VIP per LAN VLAN. Each VIP is the `.1` in its subnet — the address clients use as their
default gateway.

```text
set chassis high-availability services-redundancy-group 1 virtual-ip 1 ip 10.1.255.1/24
set chassis high-availability services-redundancy-group 1 virtual-ip 1 interface ge-0/0/1.255
set chassis high-availability services-redundancy-group 1 virtual-ip 2 ip 10.1.254.1/24
set chassis high-availability services-redundancy-group 1 virtual-ip 2 interface ge-0/0/1.254
set chassis high-availability services-redundancy-group 1 virtual-ip 3 ip 10.1.10.1/24
set chassis high-availability services-redundancy-group 1 virtual-ip 3 interface ge-0/0/1.10
set chassis high-availability services-redundancy-group 1 virtual-ip 4 ip 10.1.20.1/24
set chassis high-availability services-redundancy-group 1 virtual-ip 4 interface ge-0/0/1.20
set chassis high-availability services-redundancy-group 1 virtual-ip 5 ip 10.1.30.1/24
set chassis high-availability services-redundancy-group 1 virtual-ip 5 interface ge-0/0/1.30
set chassis high-availability services-redundancy-group 1 virtual-ip 6 ip 10.1.40.1/24
set chassis high-availability services-redundancy-group 1 virtual-ip 6 interface ge-0/0/1.40
set chassis high-availability services-redundancy-group 1 virtual-ip 7 ip 10.1.50.1/24
set chassis high-availability services-redundancy-group 1 virtual-ip 7 interface ge-0/0/1.50
set chassis high-availability services-redundancy-group 1 virtual-ip 8 ip 10.1.70.1/24
set chassis high-availability services-redundancy-group 1 virtual-ip 8 interface ge-0/0/1.70
set chassis high-availability services-redundancy-group 1 virtual-ip 9 ip 10.1.110.1/24
set chassis high-availability services-redundancy-group 1 virtual-ip 9 interface ge-0/0/1.110
set chassis high-availability services-redundancy-group 1 virtual-ip 10 ip 10.1.120.1/24
set chassis high-availability services-redundancy-group 1 virtual-ip 10 interface ge-0/0/1.120
set chassis high-availability services-redundancy-group 1 virtual-ip 11 ip 10.1.225.1/24
set chassis high-availability services-redundancy-group 1 virtual-ip 11 interface ge-0/0/1.225
set chassis high-availability services-redundancy-group 1 virtual-ip 12 ip 10.20.54.1/24
set chassis high-availability services-redundancy-group 1 virtual-ip 12 interface ge-0/0/1.2054
```

### Monitoring, signal routes, preemption

```text
set chassis high-availability services-redundancy-group 1 monitor interface ge-0/0/1
set chassis high-availability services-redundancy-group 1 active-signal-route 10.255.255.1
set chassis high-availability services-redundancy-group 1 backup-signal-route 10.255.255.2
set chassis high-availability services-redundancy-group 1 preemption
set chassis high-availability services-redundancy-group 1 activeness-priority 200
```

Note there is **no VIP bound to `ge-0/0/1.4092`**, and no VIP on the `.3666` transit. The WAN
carries a DHCP-learned address that cannot float; the transit is point-to-point between the nodes.

> **vSRX2 deltas:**
> - `set chassis high-availability local-id 2 local-ip 172.16.255.2`
> - `set chassis high-availability peer-id 1 peer-ip 172.16.255.1`
> - `set chassis high-availability peer-id 1 interface ge-0/0/0.0`
> - `set chassis high-availability services-redundancy-group 0 peer-id 1`
> - `set chassis high-availability services-redundancy-group 1 peer-id 1`
> - `set chassis high-availability services-redundancy-group 1 activeness-priority 100` (lower = backup; vSRX1 wins)
> - VIP table identical — same addresses, same units, same indices

---

## 10. NAT (and why the WAN needs no VIP)

Source NAT to the WAN interface, for LAN-outbound and for SRX-originated (`junos-host`) traffic:

```text
set security nat source rule-set LAN-TO-UPSTREAM from zone LAN
set security nat source rule-set LAN-TO-UPSTREAM to interface ge-0/0/1.4092
set security nat source rule-set LAN-TO-UPSTREAM rule SNAT-UPSTREAM match source-address 0.0.0.0/0
set security nat source rule-set LAN-TO-UPSTREAM rule SNAT-UPSTREAM then source-nat interface

set security nat source rule-set LAN-WAN from zone LAN
set security nat source rule-set LAN-WAN to zone WAN
set security nat source rule-set LAN-WAN rule RULE-LAN-WAN match source-address 0.0.0.0/0
set security nat source rule-set LAN-WAN rule RULE-LAN-WAN match destination-address 0.0.0.0/0
set security nat source rule-set LAN-WAN rule RULE-LAN-WAN then source-nat interface

set security nat source rule-set JUNOS-WAN from zone junos-host
set security nat source rule-set JUNOS-WAN to zone WAN
set security nat source rule-set JUNOS-WAN rule RULE-JUNOS-WAN match source-address 0.0.0.0/0
set security nat source rule-set JUNOS-WAN rule RULE-JUNOS-WAN match destination-address 0.0.0.0/0
set security nat source rule-set JUNOS-WAN rule RULE-JUNOS-WAN then source-nat interface
```

`then source-nat interface` is what makes the no-WAN-VIP design work. Each node NATs to whatever
its own `ge-0/0/1.4092` DHCP lease happens to be:

- Only the SRG1-active node forwards transit traffic — in hybrid mode the backup does not process
  transit packets — and it NATs clients to its own WAN lease.
- On failover, the new active node NATs to **its** own lease. Nothing about the WAN has to move,
  because there is no shared WAN address to fail over.

**ISP caveat.** Both nodes hold a DHCP client on VLAN 4092, so the upstream sees two MACs and
should hand out two leases. If your upstream permits only one lease, the backup simply has no
usable WAN until it becomes active and requests one — acceptable, but it makes the very first
failover slower to converge.

---

## 11. Security policies and screens

Global policies with `from-zone`/`to-zone` matches behave like zone-pair policies and are
evaluated top-down, first match wins. Default posture is **deny-all**. Inbound from WAN to LAN has
no explicit permit, so it falls through to the default deny.

These are pure L4 stateful policies — no application services are attached, and no
`dynamic-application` match conditions are used.

```text
set security policies global policy pLAN_WAN match from-zone LAN
set security policies global policy pLAN_WAN match to-zone WAN
set security policies global policy pLAN_WAN match source-address any
set security policies global policy pLAN_WAN match destination-address any
set security policies global policy pLAN_WAN match application any
set security policies global policy pLAN_WAN then permit
set security policies global policy pLAN_WAN then log session-close
set security policies global policy pLAN_WAN then count

set security policies global policy pJUNOS_ANY match from-zone junos-host
set security policies global policy pJUNOS_ANY match to-zone any
set security policies global policy pJUNOS_ANY match source-address any
set security policies global policy pJUNOS_ANY match destination-address any
set security policies global policy pJUNOS_ANY match application any
set security policies global policy pJUNOS_ANY then permit
set security policies global policy pJUNOS_ANY then log session-close
set security policies global policy pJUNOS_ANY then count

set security policies global policy pLAN_LAN match from-zone LAN
set security policies global policy pLAN_LAN match to-zone LAN
set security policies global policy pLAN_LAN match source-address any
set security policies global policy pLAN_LAN match destination-address any
set security policies global policy pLAN_LAN match application any
set security policies global policy pLAN_LAN then permit
set security policies global policy pLAN_LAN then log session-close
set security policies global policy pLAN_LAN then count

set security policies global policy pANY-WAN-icmp match from-zone any
set security policies global policy pANY-WAN-icmp match to-zone WAN
set security policies global policy pANY-WAN-icmp match source-address any
set security policies global policy pANY-WAN-icmp match destination-address any
set security policies global policy pANY-WAN-icmp match application junos-icmp-all
set security policies global policy pANY-WAN-icmp then permit
set security policies global policy pANY-WAN-icmp then log session-close
set security policies global policy pANY-WAN-icmp then count

set security policies global policy dWAN-IN match from-zone any
set security policies global policy dWAN-IN match to-zone WAN
set security policies global policy dWAN-IN match source-address any
set security policies global policy dWAN-IN match destination-address any
set security policies global policy dWAN-IN match application any
set security policies global policy dWAN-IN then deny
set security policies global policy dWAN-IN then log session-init
set security policies global policy dWAN-IN then count

set security policies default-policy deny-all
set security policies policy-rematch
```

`policy-rematch` causes existing sessions to be re-evaluated against a changed policy rather than
running to completion under the old one — worth having on in a lab where policies change often.

`pLAN_LAN` permits all inter-VLAN traffic. It is deliberately broad for a lab; in any real
deployment, replace it with per-pair policies (for example, isolating the IoT and guest segments
from everything else). Adjust before you copy this into production.

### Screens

Bound to the WAN zone in Section 7. Screens are a base platform feature and require no
subscription.

```text
set security screen ids-option untrust-screen icmp ping-death
set security screen ids-option untrust-screen ip source-route-option
set security screen ids-option untrust-screen ip tear-drop
set security screen ids-option untrust-screen tcp syn-flood alarm-threshold 1024
set security screen ids-option untrust-screen tcp syn-flood attack-threshold 200
set security screen ids-option untrust-screen tcp syn-flood source-threshold 1024
set security screen ids-option untrust-screen tcp syn-flood destination-threshold 2048
set security screen ids-option untrust-screen tcp syn-flood queue-size 2000
set security screen ids-option untrust-screen tcp syn-flood timeout 20
set security screen ids-option untrust-screen tcp land
```

> **vSRX2 deltas:** none. Policies and screens are identical on both nodes. Asymmetric policy is
> the most common cause of "traffic works, then failover breaks it."

---

## 12. Policy map

```text
 ZONES
 -----
   WAN         ge-0/0/1.4092  (internet, DHCP client)   screen: untrust-screen
   LAN         ge-0/0/1.{10,20,30,40,50,70,110,120,225,254,255,2054,3666}
   halink      ge-0/0/0.0, lo0.0  (ICL + control plane; ike/ssh/HA/bfd/bgp/ospf only)
   junos-host  SRX self-originated traffic


 GLOBAL POLICY EVALUATION  (top-down; first match wins)
 ------------------------------------------------------
  #   NAME             FROM  ->  TO      ACTION    APPLICATION      LOG
  1   pLAN_WAN         LAN   ->  WAN     permit    any              close
  2   pJUNOS_ANY       host  ->  any     permit    any              close
  3   pLAN_LAN         LAN   ->  LAN     permit    any              close
  4   pANY-WAN-icmp    any   ->  WAN     permit    junos-icmp-all   close
  5   dWAN-IN          any   ->  WAN     DENY      any              init
 --------------------------------------------------------------------------
  default-policy   <everything else, including WAN -> LAN>   DENY-ALL


 TRAFFIC FLOWS
 -------------
  client (10.1.X.0/24)
      --default-gw--> VIP 10.1.X.1 (active node)
      --> matches pLAN_WAN --> stateful L4 permit
      --> source-nat to the active node's own WAN lease --> ISP

  client --> other VLAN
      --> matches pLAN_LAN (inter-VLAN routed, permitted, logged)

  internet --> inside
      --> no permit exists -> default deny-all (DROP)

  SRX itself --> internet
      --> junos-host -> WAN via pJUNOS_ANY + JUNOS-WAN source-nat
```

A note on rule 5: `dWAN-IN` matches `any -> WAN` and denies. Because it sits *below* `pLAN_WAN`,
`pJUNOS_ANY`, and `pANY-WAN-icmp`, it functions as a catch-all for outbound traffic that none of
those permitted — not as an inbound filter. Inbound protection comes from the `default-policy
deny-all` at the bottom, since no `WAN -> LAN` permit exists anywhere in the list.

---

## 13. DHCP

Two roles: **relay** for the client VLANs, forwarded to the real DHCP servers on VLAN 50; and
**local server** for the two management VLANs (255, 2054) that are served on-box.

```text
set forwarding-options dhcp-relay server-group DHCP-SERVERS 10.1.50.10
set forwarding-options dhcp-relay server-group DHCP-SERVERS 10.1.50.11
set forwarding-options dhcp-relay active-server-group DHCP-SERVERS
set forwarding-options dhcp-relay group relay interface ge-0/0/1.10
set forwarding-options dhcp-relay group relay interface ge-0/0/1.20
set forwarding-options dhcp-relay group relay interface ge-0/0/1.30
set forwarding-options dhcp-relay group relay interface ge-0/0/1.40
set forwarding-options dhcp-relay group relay interface ge-0/0/1.70
set forwarding-options dhcp-relay group relay interface ge-0/0/1.110
set forwarding-options dhcp-relay group relay interface ge-0/0/1.120
set forwarding-options dhcp-relay group relay interface ge-0/0/1.225
set forwarding-options dhcp-relay group relay interface ge-0/0/1.254

set system services dhcp-local-server group GROUP1 interface ge-0/0/1.255
set system services dhcp-local-server group GROUP1 interface ge-0/0/1.2054

set access address-assignment pool POOL2054 family inet network 10.20.54.0/24
set access address-assignment pool POOL2054 family inet range RANGE2054 low 10.20.54.200
set access address-assignment pool POOL2054 family inet range RANGE2054 high 10.20.54.210
set access address-assignment pool POOL2054 family inet dhcp-attributes name-server 10.1.50.10
set access address-assignment pool POOL2054 family inet dhcp-attributes name-server 10.1.50.11
set access address-assignment pool POOL2054 family inet dhcp-attributes router 10.20.54.1

set access address-assignment pool POOL255 family inet network 10.1.255.0/24
set access address-assignment pool POOL255 family inet range RANGE255 low 10.1.255.200
set access address-assignment pool POOL255 family inet range RANGE255 high 10.1.255.220
set access address-assignment pool POOL255 family inet dhcp-attributes name-server 10.1.50.10
set access address-assignment pool POOL255 family inet dhcp-attributes name-server 10.1.50.11
set access address-assignment pool POOL255 family inet dhcp-attributes router 10.1.255.1
```

**Active-node-only DHCP is free from the SRG design** — there is nothing to configure explicitly.
The local server's pools hand out the VIP as the gateway (`router 10.1.255.1` and `10.20.54.1`),
and in hybrid mode the backup node does not process those broadcast Discovers, because the VIP
only lives on the active node. Whichever node is active answers, the backup stays quiet, and the
lease points clients at the floating gateway.

Pool selection is by subnet match, which is why binding the interface to `GROUP1` is all that is
needed — no explicit pool-to-interface reference.

Verify that only one node is leasing:

```text
show dhcp server binding
```

> **vSRX2 deltas:** none. Identical DHCP configuration on both nodes.

---

## 14. Logging

Session logging from the policies in Section 11 has to land somewhere. Two options, both
unlicensed:

**Option A — stream from the data plane to a local collector** (recommended; scales better and
keeps flow logs off the RE):

```text
set security log utc-timestamp
set security log mode stream
set security log source-interface ge-0/0/1.255
set security log stream local-collector format sd-syslog
set security log stream local-collector category all
set security log stream local-collector host 10.1.50.20
set security log stream local-collector host port 514
set security log stream local-collector transport protocol udp
```

**Option B — event mode**, which sends security logs to the routing engine's syslog and into the
`messages` file configured in Section 5. Simpler, fine at lab volumes, and it lets you read logs
with `show log messages` without standing up a collector:

```text
set security log utc-timestamp
set security log mode event
```

Pick one. Configuring both is a common source of "my logs disappeared" confusion.

> **vSRX2 deltas:** identical, except `source-interface` becomes the node's own
> `ge-0/0/1.255` address implicitly — the statement itself is the same.

---

## 15. Commit and verify

Commit on each node. Bring the ICL up first and confirm cold-sync, then check SRG, routing, and
data plane.

Use `commit confirmed` on anything that touches the ICL, zones, or SRG — locking yourself out of
a node mid-HA-build is unpleasant.

```text
show chassis high-availability information
show chassis high-availability services-redundancy-group 1
show security ipsec security-associations ha-link-encryption detail
show ospf neighbor
show bgp summary
show route 10.255.255.1
show route 10.255.255.2
show dhcp server binding
show interfaces ge-0/0/1.4092 terse
show security flow session summary
show security policies hit-count
```

### What "good" looks like

| Check | Expected |
|---|---|
| HA connection | `Conn State: UP`, `Cold Sync Status: COMPLETE` |
| SRG1 | vSRX1 `ACTIVE` / vSRX2 `BACKUP`; peer `HEALTHY`; Failover Readiness `READY` |
| SRG1 backup, hybrid mode | `Process Packet In Backup State: NO` |
| eBGP | Established between `172.16.255.1` and `172.16.255.2` |
| Signal route | Active node's route present with the better metric |
| DHCP | One node holds bindings; the other holds none |
| WAN | `ge-0/0/1.4092` shows a bound DHCP address on the active node |

### Failover test

```text
request chassis high-availability services-redundancy-group 1 manual-failover
```

Or pull the active node's monitored interface. Watch the VIPs and DHCP leases move, and confirm
the GARP re-points client gateways. A ping from a client through to the internet should drop
only a small number of packets.

---

## Appendix A — ESXi security settings

```text
 vSwitch / Port group          VLAN    Promisc   MAC-chg   Forged    Maps to
 ---------------------------   -----   -------   -------   -------   --------------------
 vSwitch0 / vSRX-Trunk         4095    Accept    Accept    Accept    ge-0/0/1 (VGT trunk)
 vSRX Networks / SRX ICL       0       Accept    Accept    Accept    ge-0/0/0 (ICL)
 vSRX Networks / SRX Mgmt      0       Accept    Accept    Accept    fxp0
```

**Defaults you are overriding:** on a vSS, Promiscuous defaults to Reject — you must flip the
trunk group to Accept for VGT to work. MAC-changes and Forged-transmits have historically
defaulted to Accept on a vSS, but set them explicitly so a future host rebuild, or a migration to
a vDS, does not silently change behaviour underneath you.

---

## Appendix B — Addressing plan

All addressing is RFC 1918 plus two 172.16 transit ranges. Substitute your own; nothing here is
load-bearing except the relationships (VIP = `.1`, vSRX1 = `.2`, vSRX2 = `.3`).

| VLAN | Role | Subnet | VIP (gateway) | vSRX1 | vSRX2 |
|---:|---|---|---|---|---|
| 10 | clients | 10.1.10.0/24 | 10.1.10.1 | 10.1.10.2 | 10.1.10.3 |
| 20 | wireless clients | 10.1.20.0/24 | 10.1.20.1 | 10.1.20.2 | 10.1.20.3 |
| 30 | IoT clients | 10.1.30.0/24 | 10.1.30.1 | 10.1.30.2 | 10.1.30.3 |
| 40 | tunneled clients | 10.1.40.0/24 | 10.1.40.1 | 10.1.40.2 | 10.1.40.3 |
| 50 | servers | 10.1.50.0/24 | 10.1.50.1 | 10.1.50.2 | 10.1.50.3 |
| 70 | SRX test | 10.1.70.0/24 | 10.1.70.1 | 10.1.70.2 | 10.1.70.3 |
| 110 | lab clients | 10.1.110.0/24 | 10.1.110.1 | 10.1.110.2 | 10.1.110.3 |
| 120 | lab wireless | 10.1.120.0/24 | 10.1.120.1 | 10.1.120.2 | 10.1.120.3 |
| 225 | guest clients | 10.1.225.0/24 | 10.1.225.1 | 10.1.225.2 | 10.1.225.3 |
| 254 | WLAN mgmt | 10.1.254.0/24 | 10.1.254.1 | 10.1.254.2 | 10.1.254.3 |
| 255 | LAN mgmt (native) | 10.1.255.0/24 | 10.1.255.1 | 10.1.255.2 | 10.1.255.3 |
| 2054 | WLAN lab mgmt | 10.20.54.0/24 | 10.20.54.1 | 10.20.54.2 | 10.20.54.3 |
| 3666 | inter-SRX transit | 172.16.253.0/29 | — | 172.16.253.2 | 172.16.253.3 |
| 4092 | internet / WAN | DHCP from ISP | — | DHCP | DHCP |

| Non-VLAN | Role | vSRX1 | vSRX2 |
|---|---|---|---|
| `ge-0/0/0.0` | ICL transit | 172.16.254.2/24 | 172.16.254.3/24 |
| `lo0.0` | BGP / HA loopback | 172.16.255.1/32 | 172.16.255.2/32 |
| `fxp0.0` | OOB management | 192.168.255.2/24 | 192.168.255.3/24 |
| signal routes | SRG1 activeness | active 10.255.255.1/32 | backup 10.255.255.2/32 |

---

## Appendix C — vSRX2 delta summary

Every legitimate difference between the two nodes, in one place. Anything **not** on this list
should be byte-identical between the configs.

| Category | vSRX1 | vSRX2 |
|---|---|---|
| Hostname | `vSRX1` | `vSRX2` |
| Auth hashes | own | own |
| LAN interface addresses | `.2` | `.3` |
| `ge-0/0/0.0` | 172.16.254.2/24 | 172.16.254.3/24 |
| `ge-0/0/1.3666` | 172.16.253.2/29 | 172.16.253.3/29 |
| `fxp0` | 192.168.255.2/24 | 192.168.255.3/24 |
| `lo0` | 172.16.255.1/32 | 172.16.255.2/32 |
| BGP AS | 64512 | 64513 |
| BGP local-address | 172.16.255.1 | 172.16.255.2 |
| BGP peer-as / neighbor | 64513 / 172.16.255.2 | 64512 / 172.16.255.1 |
| IKE PSK ciphertext | node-local `$9$` | node-local `$9$` (cleartext must match) |
| HA local-id / local-ip | 1 / 172.16.255.1 | 2 / 172.16.255.2 |
| HA peer-id / peer-ip | 2 / 172.16.255.2 | 1 / 172.16.255.1 |
| SRG0 / SRG1 peer-id | 2 | 1 |
| `activeness-priority` | 200 | 100 |
| VIP table | identical | identical |
| Zones, policies, NAT, DHCP, screens | identical | identical |

---

## Appendix D — Failure modes worth knowing

1. **NIC order wrong.** Adapter 1 must map to `fxp0`. Symptom: the ICL never forms and the trunk
   is dead, while the config looks correct.
2. **Trunk port group not VLAN 4095.** The guest never sees tags, so every VLAN is dead.
3. **Promiscuous = Reject on the trunk.** Intermittent, VLAN-dependent reachability — the most
   frustrating variant, because some things work.
4. **Native VLAN mismatch** between the 6200M trunk and the SRX `native-vlan-id 255`. Only VLAN
   255 breaks, which is exactly the VLAN you are managing from.
5. **ICL PSK cleartext mismatch.** `ha-link-encryption` will not form. Differing ciphertext
   between nodes is normal and expected; differing cleartext is not.
6. **Config drift between nodes** (zones, VIPs, export policy, security policy). Produces the
   classic "works until failover." Keep the configs symmetric; the only legitimate per-node
   differences are in Appendix C.
7. **Only one node originating the default** in `EXPORT-DEFAULT-AND-CONNECTED`. The inside loses
   its default the moment the other node becomes active.
8. **Adding `use-virtual-mac` later.** Forged Transmits = Accept then becomes mandatory on the
   trunk port group, or VIP traffic is silently dropped at the vSwitch. This build avoids the
   dependency by design; adding the knob reintroduces it.
9. **Upstream permitting only one DHCP lease on VLAN 4092.** The backup has no usable WAN until it
   becomes active, which slows the first failover.
10. **Both `security log mode stream` and `mode event` configured.** Pick one (Section 14).

---

## What this build deliberately does not do

For transparency, since the source design included them and they were removed here:

- **IDP / IPS** — requires an IDP signature subscription.
- **Application identification (AppID)** — requires an application signature subscription; the
  `dynamic-application` match conditions were removed from all policies along with it, and
  `pre-id-default-policy` with them, since it only applies to sessions awaiting app identification.
- **UTM** — anti-virus (Sophos), enhanced web filtering, and anti-spam all require subscriptions.
- **SecIntel** — command-and-control, DNS, and infected-host feeds require a subscription.
- **ATP Cloud / AAMW** — requires a cloud tenant and subscription, plus the associated PKI CA
  profiles and SSL initiation profiles.
- **Security Director Cloud** — requires a tenant; the `outbound-ssh` dial-home client, the
  `sdcloud-messages` syslog file, and the TLS log stream to the SDC collector were removed with it.

Everything remaining is base Junos and base SRX functionality plus MNHA.
