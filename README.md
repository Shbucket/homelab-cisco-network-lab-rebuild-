# Cisco Enterprise Network Lab — Rebuild (v2)

## Overview

A full rebuild of my physical Cisco homelab from a clean slate. The first build covered VLANs, trunking, STP, EtherChannel, inter-VLAN routing, OSPF, and ACLs, but it had some design shortcuts I wanted to fix:

- Management traffic lived on VLAN 1.
- My home network was bridged directly into the lab through a switch port.
- The STP root bridge was chosen by the lowest MAC address, not by design.
- Access-SW1 was daisy-chained behind Access-SW2 instead of uplinking to the core.

This rebuild fixes those problems and adds the CCNA 200-301 topics the first build didn't cover: NAT/PAT, DHCP, HSRP, router-on-a-stick, static and floating static routes, IPv6, port security, DHCP snooping, DAI, NTP, Syslog, and SNMP.

The rebuild is done **from memory**, without referring back to the first build's notes unless I get stuck. Anything I have to look up goes into a weak-topic log for exam review.

## Hardware

| Device | Model | Role | IOS Version |
|---|---|---|---|
| Core-Switch | Catalyst 3560G | Layer 3 collapsed core | 12.2(25)SED1 |
| Access-SW1 | Catalyst 2960-24PC-L | Access switch | 15.0(2)SE11 |
| Access-SW2 | Catalyst 2960-24PC-L | Access switch | 15.0(2)SE11 |
| Router1 | Cisco 1941 ISR | Edge router (NAT, internet) | 15.0(1)M6 |
| Router2 | Cisco 1941 ISR | Secondary router (DHCP, HSRP) | 15.7(3)M6 |
| Proxmox host | Dell OptiPlex 7060 | VM host on a VLAN trunk | Proxmox VE |

## Topology

![Lab topology](images/topology.svg)

### Design changes from v1

**Collapsed core instead of a daisy chain.** In v1, all of Access-SW1's traffic crossed Access-SW2 to reach the core, and losing SW2 cut SW1 off completely. Now each access switch uplinks directly to the core over a two-link Gigabit EtherChannel, and a single SW1↔SW2 cross-link serves as the backup path that STP blocks during normal operation. This is the two-tier (collapsed core) design most small and medium networks use.

**Gigabit uplinks.** The v1 EtherChannel ran over two FastEthernet ports (200 Mbps total) while the 2960s' Gigabit uplink ports sat shut down. Both bundles now use the Gigabit ports.

**Home network connects at Layer 3 only.** In v1, the home router (TP-Link Deco) plugged into a lab switch port, which bridged the home LAN into VLAN 1. That's why management had to stay on VLAN 1. Now the Deco connects only to Router1 G0/0, making Router1 the actual boundary between the home network and the lab. That makes NAT, the default route, and edge ACLs real instead of simulated.

**Household devices off the lab.** The PS5 previously ran through Access-SW2. It now connects directly to the Deco, so lab downtime never affects the household, and it avoids double NAT.

## Addressing Plan

### VLANs

| VLAN | Name | Subnet | Gateway |
|---|---|---|---|
| 10 | WORKSTATIONS | 10.10.10.0/24 | 10.10.10.1 (Core-Switch SVI) |
| 20 | SERVERS | 10.10.20.0/24 | 10.10.20.1 (Core-Switch SVI) |
| 30 | MGMT | 10.10.30.0/24 | 10.10.30.1 (Core-Switch SVI) |
| 50 | GUEST | 10.10.50.0/24 | 10.10.50.1 (HSRP VIP on Router1/Router2) |
| 60 | IPV6-LAB | 2001:db8:60::/64 | Router1/Router2 subinterfaces |
| 99 | NATIVE | none | Trunk native VLAN only, carries no traffic |
| 100 | TRANSIT | 10.0.100.0/24 | Core-Switch .1, Router1 .2, Router2 .3 |
| 999 | PARKING | none | Unused ports, administratively down |

VLAN 1 is not used for anything and is pruned from every trunk.

### Management addresses

| Device | Management address |
|---|---|
| Core-Switch | 10.10.30.1 (VLAN 30 SVI), Loopback0 10.255.0.1 |
| Router1 | Loopback0 10.255.0.2 |
| Router2 | Loopback0 10.255.0.3 |
| Access-SW1 | 10.10.30.11 (VLAN 30 SVI) |
| Access-SW2 | 10.10.30.12 (VLAN 30 SVI) |

---

## Phase 0 — Factory Reset and Recabling

### Wipe

Reset all five devices over the console. The switches need their VLAN database deleted separately, because VLANs and VTP settings are stored in `vlan.dat`, not in the startup configuration. `write erase` alone leaves them behind.

**Switches:**

```
enable
delete flash:vlan.dat
configure terminal
crypto key zeroize rsa
end
write erase
reload
```

**Routers:**

```
enable
configure terminal
crypto key zeroize rsa
end
write erase
reload
```

Answered **no** to saving the configuration before reload, and **no** to the initial configuration dialog afterward.

### Verification

- `show vlan brief` on all three switches: only VLAN 1 and 1002–1005 present.
- `show sdm prefer` on Core-Switch: "desktop default" template. The SDM template survives a config wipe, so it had to be checked separately. The default template supports 8 routed interfaces, enough for the planned SVIs.
- `show version | include register` on both routers: **0x2102**. This mattered most on Router2, which was stuck at 0x2142 during the first build and silently booted with a blank configuration on every restart.

### Recabling

| Link | From | To |
|---|---|---|
| Core ↔ Access-SW1 (2 cables) | Access-SW1 Gi0/1, Gi0/2 | Core Gi0/5, Gi0/6 |
| Core ↔ Access-SW2 (2 cables) | Access-SW2 Gi0/1, Gi0/2 | Core Gi0/1, Gi0/3 |
| Access-SW1 ↔ Access-SW2 (1 cable) | Access-SW1 Fa0/24 | Access-SW2 Fa0/24 |
| Router1 | Router1 G0/1 | Core Gi0/2 |
| Router2 | Router2 G0/1 | Core Gi0/4 |
| Home network | Deco | Router1 G0/0 (inactive until the NAT phase) |

### Topology verification with CDP

Before configuring anything, I verified the physical cabling with `show cdp neighbors` on Core-Switch, checking every link against the diagram. This came directly from a lesson in the first build, where OSPF configuration went onto the wrong physical ports because I trusted the plan instead of checking the cabling.

After the hostnames were set (Phase 1), CDP confirmed each port pair went to the correct switch:

```
Device ID     Local Intrfce   Platform       Port ID
Router2       Gig 0/4         CISCO1941      Gig 0/1
Router1       Gig 0/2         CISCO1941      Gig 0/1
Access-SW1    Gig 0/5         WS-C2960-24PC  Gig 0/2
Access-SW1    Gig 0/6         WS-C2960-24PC  Gig 0/1
Access-SW2    Gig 0/1         WS-C2960-24PC  Gig 0/2
Access-SW2    Gig 0/3         WS-C2960-24PC  Gig 0/1
```

Each pair is crossed (for example, Core Gi0/5 connects to SW1 Gi0/2), which doesn't matter for EtherChannel, since member ports don't need to match one-to-one.

### Lessons Learned / Troubleshooting — Phase 0

**Uplinks plugged into the wrong port type.** The first CDP check showed the access switches' Port ID as **Fas 0/1** and **Fas 0/2** instead of Gigabit. The uplinks had gone into FastEthernet ports 1 and 2 instead of the Gigabit ports. On the 2960-24PC-L, the two Gigabit uplinks sit apart at the far right end of the switch but are also labeled 1 and 2, which makes them easy to confuse with Fa0/1–0/2. Moving the cables fixed it, and the second CDP check showed **Gig 0/1** and **Gig 0/2**. It was only caught because I checked the far-end port ID, not just whether a neighbor appeared at all. A link on the wrong port still shows up in CDP.

**Routers missing from CDP after the wipe.** The first CDP output showed both access switches but neither router. Switch ports are enabled by default, but router interfaces are **administratively down by default** after a wipe, so CDP can't form over them. Running `no shutdown` on each router's G0/1 brought both routers into the neighbor table.

**Stale CDP entries after renaming.** After setting hostnames, the neighbor table showed the new names alongside the old default ones ("Router" and "Switch") on the same interfaces. These were cached entries from before the rename. CDP keeps an entry until its hold time (180 seconds by default) expires, so the old names aged out on their own. No action needed, but worth knowing so a duplicate entry isn't mistaken for a cabling problem.

**`crypto key zeroize rsa` is a global configuration command.** It failed when run from privileged EXEC mode (`Switch#`). It has to be run from `Switch(config)#`.

---

## Phase 1 — Base Configuration

All Phase 1 configuration was done over the console, since no device has a reachable management address yet.

### Hostnames

```
configure terminal
hostname <name>
no ip domain-lookup
end
```

`no ip domain-lookup` stops the device from treating a mistyped command as a hostname and freezing for about 30 seconds while it tries a DNS lookup.

### Local access security (all five devices)

```
configure terminal
enable secret <password>
username admin privilege 15 secret <password>
service password-encryption
banner motd # Authorized Access Only #
line console 0
 login local
 logging synchronous
 exec-timeout 15 0
end
write memory
```

Changes from v1:

- **Console uses `login local`** instead of a line password. In v1, the console was protected by a type 7 password, which is trivially reversible. Now it authenticates against the local user database, which uses hashed secrets.
- **`exec-timeout 15 0`** logs out idle sessions after 15 minutes. In v1, sessions never timed out.
- **`logging synchronous`** stops log messages from breaking up commands while I'm typing.

Tested logging out and back in as `admin` on one device before applying it to the rest, to avoid locking myself out of all five at once.

### SSH (all five devices)

```
configure terminal
ip domain-name lab.local
crypto key generate rsa modulus 2048
ip ssh version 2
line vty 0 15
 login local
 transport input ssh
 exec-timeout 15 0
 logging synchronous
end
write memory
```

- **`transport input ssh`** is set explicitly on every device. In v1, Access-SW2's VTY lines never had it set, which left Telnet open.
- SSH is configured now but not reachable yet. Each device becomes reachable once it has a management address the network can route to (see Management Access below).

### VTP and VLAN database (all three switches)

```
configure terminal
vtp mode transparent
vlan 10
 name WORKSTATIONS
vlan 20
 name SERVERS
vlan 30
 name MGMT
vlan 50
 name GUEST
vlan 60
 name IPV6-LAB
vlan 99
 name NATIVE
vlan 999
 name PARKING
end
write memory
```

Plus VLAN 100 (TRANSIT) on Core-Switch only, since it only connects the core to the routers and never crosses the access switches.

**VTP transparent mode** means each switch keeps its own VLAN database and never syncs with the others, so one switch can't overwrite another's VLANs.

### Unused port parking (all three switches)

Every port not in use was moved into VLAN 999 (PARKING) and shut down, so nothing plugged into a spare port can land on a live VLAN.

**Core-Switch** (in use: Gi0/1–0/6):

```
configure terminal
interface range gi0/7 - 28
 switchport mode access
 switchport access vlan 999
 shutdown
end
write memory
```

**Access-SW1 and Access-SW2** (in use: Gi0/1–0/2 and Fa0/24):

```
configure terminal
interface range fa0/1 - 23
 switchport mode access
 switchport access vlan 999
 shutdown
end
write memory
```

This goes further than v1, where unused ports were shut down but left in VLAN 1. A shut-down port in VLAN 1 is one `no shutdown` away from joining the default VLAN. A port in VLAN 999 lands in an isolated VLAN with no gateway, even if it's accidentally re-enabled. Ports get opened individually as devices are connected.

### Lessons Learned / Troubleshooting — Phase 1

**Old IOS rejects inline RSA modulus.** On Core-Switch (12.2(25)SED1, from 2005), `crypto key generate rsa modulus 2048` failed with `% Invalid input detected`. That image doesn't accept the modulus as part of the command. Running `crypto key generate rsa` without it brings up a prompt for the key size, and I entered **1024**. This is the same aging image that needed `diffie-hellman-group1-sha1` for SSH in v1, and it's a good example of why IOS version matters when planning a feature.

### Management access decision

My PC's only wired connection goes to the Deco, and I use it for gaming, so I didn't want to move it onto the lab or switch to Wi-Fi. Rather than dual-homing the PC, I'm staying on the console until Router1 is configured. After that, the PC will reach the lab management network over the home network, routed through Router1, the same way a real network is managed through its edge instead of by bridging the home LAN into it.

---

## Phase 2 — EtherChannel, Trunking, and Management

### EtherChannel Po1 — Core-Switch ↔ Access-SW1 (LACP)

**Core-Switch:**

```
configure terminal
interface range gi0/5 - 6
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport nonegotiate
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,20,30,50,60,99
 channel-group 1 mode active
interface port-channel 1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport nonegotiate
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,20,30,50,60,99
end
write memory
```

**Access-SW1:** the same configuration on Gi0/1–0/2, without `switchport trunk encapsulation dot1q`. The 2960 only supports 802.1Q, so the command doesn't exist on it.

### EtherChannel Po2 — Core-Switch ↔ Access-SW2 (PAgP)

**Core-Switch:**

```
configure terminal
interface range gi0/1 , gi0/3
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport nonegotiate
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,20,30,50,60,99
 channel-group 2 mode desirable
interface port-channel 2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport nonegotiate
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,20,30,50,60,99
end
write memory
```

**Access-SW2:** the same configuration on Gi0/1–0/2, without the encapsulation command.

Using **LACP on one bundle and PAgP on the other** was deliberate, so both negotiation protocols have been configured and verified on real hardware. LACP (IEEE 802.3ad) uses `active`/`passive`, while PAgP (Cisco proprietary) uses `desirable`/`auto`. The Core-Switch's member ports aren't consecutive, so the range uses comma syntax (`gi0/1 , gi0/3`).

### Cross-link trunk — Access-SW1 ↔ Access-SW2

Configured on Fa0/24 on both access switches:

```
configure terminal
interface fa0/24
 switchport mode trunk
 switchport nonegotiate
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,20,30,50,60,99
end
write memory
```

A single link, so no channel-group. It exists only as a redundant path for STP.

### Trunk design

- **Native VLAN 99** on every trunk. It carries no traffic and has no SVI, so untagged frames on a trunk land somewhere harmless.
- **VLAN 1 pruned from every trunk.** In v1, VLAN 1 had to stay on the trunks because management lived on it. With management on VLAN 30, VLAN 1 is gone entirely.
- **`switchport nonegotiate`** disables DTP, so trunk mode is always set by configuration, never negotiated.

### Management SVIs

| Device | Interface | Address | Gateway |
|---|---|---|---|
| Core-Switch | Vlan30 | 10.10.30.1/24 | — |
| Access-SW1 | Vlan30 | 10.10.30.11/24 | `ip default-gateway 10.10.30.1` |
| Access-SW2 | Vlan30 | 10.10.30.12/24 | `ip default-gateway 10.10.30.1` |

The access switches use `ip default-gateway` instead of a static route because they're Layer 2 switches. The gateway is only for their own management traffic.

### Verification

`show etherchannel summary`:

```
Core-Switch:
1      Po1(SU)         LACP      Gi0/5(P)    Gi0/6(P)

Access-SW2:
2      Po2(SU)         PAgP      Gi0/1(P)    Gi0/2(P)
```

`show interfaces trunk` on Core-Switch:

```
Port        Mode         Encapsulation  Status        Native vlan
Po1         on           802.1q         trunking      99
Po2         on           802.1q         trunking      99

Port        Vlans allowed on trunk
Po1         10,20,30,50,60,99
Po2         10,20,30,50,60,99
```

All VLANs are forwarding on both port-channels from the core, which suggests the blocked STP port is on the SW1↔SW2 cross-link. This gets confirmed and controlled deliberately in Phase 3.

**End-to-end management test:** From Core-Switch, pinged 10.10.30.11 and 10.10.30.12 successfully, proving VLAN 30 crosses both bundles. Then SSHed from Core-Switch to both access switches (`ssh -l admin <ip>`) and confirmed the hostnames. My PC can't reach the lab yet, so using the core as the SSH client confirmed the access switches' SSH servers, local authentication, and VTY configuration work, without waiting on the routing phases.

### Lessons Learned / Troubleshooting — Phase 2

**Trunk settings before `channel-group`.** In v1, one EtherChannel member got suspended because its operational trunk mode was still dynamic even though the running config showed `switchport mode trunk`. This time the trunk settings were applied to the member ports in the same block, before the `channel-group` line. Both bundles came up with all members **(P)** on the first attempt, with no suspended ports and no interface bounce needed.

**Applied the wrong switch's configuration.** I accidentally applied Access-SW1's Po1/LACP configuration to Access-SW2's uplink ports. No outage resulted, because the Core-Switch end of that link wasn't bundled yet, so SW2's ports had nothing to negotiate with. The clean way to undo it:

```
configure terminal
no interface port-channel 1
default interface range gi0/1 - 2
end
```

`no interface port-channel 1` deletes the logical bundle and removes the `channel-group` lines from its members. `default interface range` then returns the physical ports to factory settings, clearing the leftover trunk configuration. `show etherchannel summary` confirmed zero channel-groups before I applied the correct Po2/PAgP configuration.

---

## Phase 3 — Spanning Tree (Rapid PVST+)

Every STP change in this phase followed the same method: **gather the facts, predict the result, then verify with `show` output.** The CCNA blueprint (topic 2.5) says to *interpret* Rapid PVST+ operation, so reading output and predicting behavior mattered as much as the configuration.

### Baseline: predicting the default election

Before changing anything, I collected each switch's base MAC address (`show version | include MAC`) to predict the election.

| Switch | Base MAC | VLAN 10 priority |
|---|---|---|
| Core-Switch | 00:18:18:1B:F2:80 | 32778 (default) |
| Access-SW2 | DC:A5:F4:83:1D:80 | 32778 (default) |
| Access-SW1 | DC:A5:F4:E6:57:00 | 32778 (default) |

**Prediction:**

- **Root bridge:** All priorities tie at 32768 + VLAN ID, so the lowest MAC wins. The Core-Switch's `00` beats `DC` immediately.
- **Root ports:** Each access switch reaches the root through its EtherChannel (cost 3). That beats the cross-link path (19 + 3 = 22).
- **Blocked port:** On the Fa0/24 cross-link, both ends are cost 3 from the root, so the tie goes to the lower bridge ID. SW1 and SW2 match through `DC:A5:F4`, then `83` (SW2) beats `E6` (SW1), so **SW2 gets the designated port and SW1's Fa0/24 blocks.**

**Verification** on Access-SW1, which matched the prediction exactly:

```
VLAN0010
  Spanning tree enabled protocol ieee
  Root ID    Priority    32778
             Address     0018.181b.f280
             Cost        3
             Port        64 (Port-channel1)
  Bridge ID  Priority    32778  (priority 32768 sys-id-ext 10)
             Address     dca5.f4e6.5700

Interface           Role Sts Cost      Prio.Nbr Type
Fa0/24              Altn BLK 19        128.24   P2p
Po1                 Root FWD 3         128.64   P2p
```

### Legacy PVST+ vs. Rapid PVST+ failover

**Legacy PVST+ (802.1D):** Shut down Po1 on Access-SW1 and repeatedly checked Fa0/24. It moved through **BLK → LIS → LRN → FWD** and took about **30 seconds**: Forward Delay (15 seconds) for listening plus 15 seconds for learning. SW1 detected the failure directly because Po1 was its own link. An indirect failure would also wait out Max Age (20 seconds), for up to 50 seconds total.

**Rapid PVST+ (802.1w):**

```
configure terminal
spanning-tree mode rapid-pvst
end
write memory
```

Applied on all three switches. `show spanning-tree` then reported `protocol rstp`. Repeated the same test: Fa0/24 went from **Altn BLK** to **Root FWD immediately**, with no listening or learning. In RSTP, an alternate port is a pre-calculated backup root port, so when the root port fails, it's promoted right away. It became a **Root** port rather than Designated because it was now SW1's only path to the root.

| | Legacy PVST+ | Rapid PVST+ |
|---|---|---|
| Failover time (direct failure) | ~30 seconds | Immediate |
| States seen | BLK → LIS → LRN → FWD | BLK → FWD |

### Deterministic root and per-VLAN load balancing

The Core-Switch was root only because it happened to have the lowest MAC. I made that deliberate:

```
! Core-Switch
spanning-tree vlan 10,20,30,50,60,99 root primary

! Access-SW1
spanning-tree vlan 10,30 root secondary

! Access-SW2
spanning-tree vlan 20,50 root secondary
```

- **`root primary`** set the Core-Switch's priority to 24576. For VLAN 10 that's **24586** with the sys-id-ext, which I predicted correctly before checking. It uses 24576 only if that beats the current root; otherwise it goes 4096 below the existing root. Priorities must be multiples of 4096.
- **`root secondary`** sets 28672, a backup root if the core fails. IOS saves both commands as `spanning-tree vlan X priority Y`, not as written.

Because the Core-Switch is root for every VLAN, the secondary priorities also decide the **cross-link tiebreaker per VLAN**:

| VLAN | SW1 priority | SW2 priority | Fa0/24 blocks on |
|---|---|---|---|
| 10, 30 | 28682 / 28702 | 32778 / 32798 | **SW2** |
| 20, 50 | 32788 / 32818 | 28692 / 28722 | **SW1** |
| 60, 99 | default | default | **SW1** (tie, SW2 has the lower MAC) |

**Verification:** VLAN 10 showed Fa0/24 **Altn BLK** on SW2, and VLAN 20 showed Fa0/24 **Altn BLK** on SW1. It's the same physical cable, but a different end blocks depending on the VLAN. That's what the "per VLAN" in PVST+ means in practice. `show spanning-tree summary` confirmed the pattern in the Blocking column on both switches.

### PortFast and BPDU Guard

On both access switches:

```
configure terminal
spanning-tree portfast default
spanning-tree portfast bpduguard default
end
write memory
```

The global forms apply to every access port automatically, including ports opened later, so a new port can't be forgotten. Trunks aren't affected. (The per-interface equivalents are `spanning-tree portfast` and `spanning-tree bpduguard enable`.)

**Live test:** Opened Fa0/1 and Fa0/2 on Access-SW1 and connected them to each other with a patch cable, simulating an unauthorized switch on an access port:

```
%SPANTREE-2-BLOCK_BPDUGUARD: Received BPDU on port FastEthernet0/2 with BPDU Guard enabled. Disabling port.
%PM-4-ERR_DISABLE: bpduguard error detected on Fa0/2, putting Fa0/2 in err-disable state
%SPANTREE-2-BLOCK_BPDUGUARD: Received BPDU on port FastEthernet0/1 with BPDU Guard enabled. Disabling port.
%PM-4-ERR_DISABLE: bpduguard error detected on Fa0/1, putting Fa0/1 in err-disable state
```

```
Access-SW1#show interfaces status err-disabled
Port      Name               Status       Reason
Fa0/1                        err-disabled bpduguard
Fa0/2                        err-disabled bpduguard
```

Both ports shut down within milliseconds of each other.

**Recovery:** An err-disabled port stays down even after the cause is removed. I recovered manually (removed the cable, then `shutdown` / `no shutdown`), then configured automatic recovery:

```
errdisable recovery cause bpduguard
errdisable recovery interval 60
```

With this, a port retries after 60 seconds and gets err-disabled again if the rogue device is still connected. Both test ports were parked again afterward.

### Root guard

Applied to the Core-Switch's downlinks, so neither access switch can ever become root:

```
interface port-channel 1
 spanning-tree guard root
interface port-channel 2
 spanning-tree guard root
```

**Live test:** Set Access-SW1's VLAN 10 priority to 0, beating the Core-Switch's 24576. I predicted which core port-channels would react before checking:

```
Core-Switch#show spanning-tree inconsistentports
Name                 Interface              Inconsistency
VLAN0010             Port-channel1          Root Inconsistent
VLAN0010             Port-channel2          Root Inconsistent
```

**Both** port-channels went root-inconsistent, not just Po1. SW2 received SW1's superior BPDU over the Fa0/24 cross-link, accepted SW1 as root for VLAN 10, and relayed that claim up Po2. Root guard chose to isolate VLAN 10 from the access layer rather than let the root move. Other VLANs, including management on VLAN 30, were unaffected.

Restoring SW1's VLAN 10 priority to 28672 cleared both ports **automatically**, with no action needed on the Core-Switch. Setting `priority 0` had replaced the earlier secondary priority, so it had to be set again rather than simply removed.

| | BPDU Guard | Root guard |
|---|---|---|
| Trigger | Any BPDU on a PortFast port | A superior BPDU (better root) |
| Result | Err-disabled | Root-inconsistent (blocking) |
| Recovery | Manual, or errdisable recovery timer | Automatic once superior BPDUs stop |
| Placement | Access ports | Downlinks toward access switches |

### Loop guard

Enabled globally on both access switches:

```
spanning-tree loopguard default
```

Loop guard protects non-designated ports (root and alternate), which in this topology are the access switches' uplinks and the cross-link. Those ports stay in their role *because* they keep receiving BPDUs. If BPDUs stop because of a one-way link failure, normal STP would move the port to forwarding and create a loop. Loop guard puts it into **loop-inconsistent** (blocking) instead, and recovers automatically when BPDUs return. A one-way failure is hard to simulate on copper, so this was configured and verified in `show spanning-tree summary` but not failure-tested.

**BPDU filter** was deliberately **not** configured. It stops a port from sending or processing BPDUs, effectively disabling STP on that port. A filtered port that gets looped won't detect the loop, and when BPDU filter and BPDU Guard are on the same port, the filter wins and BPDU Guard never triggers. BPDU filter *suppresses* BPDUs, while BPDU Guard *reacts* to them.

### Lessons Learned / Troubleshooting — Phase 3

**A wrong prediction caught a misconfiguration.** After configuring the secondary roots, I predicted SW2's Fa0/24 would block in VLAN 10, but the output showed **Desg FWD**. Rather than doubting the prediction, I checked its inputs. `show spanning-tree vlan 10 | include Priority` showed SW2 at 28682, the priority meant for SW1. Then `show run | include spanning-tree vlan` showed exactly what happened:

```
Access-SW2#show run | include spanning-tree vlan
spanning-tree vlan 10,20,30,50 priority 28672
```

Both switches' `root secondary` commands had been entered on SW2. The fix was `no spanning-tree vlan 10,30 priority` on SW2 and `spanning-tree vlan 10,30 root secondary` on SW1, after which both predictions verified. The prediction was right all along. The mismatch between expected and actual output is what exposed the configuration error.

**`include` filters are case-sensitive.** `show spanning-tree vlan 10 | include f0/24` returned nothing, because the output shows `Fa0/24`. The filter matches the exact text in the output, not the interface name as you'd type it in a command.

**Unset clock.** The BPDU Guard log messages were timestamped `*Mar 1`. The asterisk means the clock was never set and isn't trusted. Log timestamps are meaningless until NTP is configured in Phase 8.

---

## Phase 4 — Inter-VLAN Routing and Remote Management

### SVIs on Core-Switch

```
configure terminal
ip routing
interface vlan 10
 ip address 10.10.10.1 255.255.255.0
 no shutdown
interface vlan 20
 ip address 10.10.20.1 255.255.255.0
 no shutdown
interface vlan 100
 ip address 10.0.100.1 255.255.255.0
 no shutdown
interface loopback 0
 ip address 10.255.0.1 255.255.255.255
end
write memory
```

The VLAN 30 SVI already existed from Phase 2. `ip routing` turns the 3560G into a router as well as a switch. Without it, SVIs only work as management addresses and nothing routes between them. Loopback 0 never goes down and will serve as the OSPF router ID in Phase 6.

**Predicting SVI state:** Before checking, I predicted that VLANs 10, 20, and 30 would come up and VLAN 100 would stay down. An SVI only comes up when its VLAN has at least one active, forwarding port on the switch. VLANs 10, 20, and 30 ride the trunks to the access switches, but no port was in VLAN 100 yet. `show ip interface brief` confirmed it: VLAN 100 was the only SVI that wasn't up.

### Trunks to the routers

Core-Switch Gi0/2 (Router1) and Gi0/4 (Router2) became trunks, so each router can use subinterfaces for several VLANs on one physical port:

```
configure terminal
interface range gi0/2 , gi0/4
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport nonegotiate
 switchport trunk native vlan 99
 switchport trunk allowed vlan 50,60,99,100
 no shutdown
end
write memory
```

As predicted, the VLAN 100 SVI came up immediately, **before the routers had any subinterfaces configured**. The SVI only needs a forwarding port on the switch, and it doesn't verify the far end. That's a good reminder that an up SVI proves nothing about end-to-end connectivity, which is why every step here was confirmed with a ping.

### Router subinterfaces (router-on-a-stick)

**Router1:**

```
configure terminal
interface g0/1.50
 encapsulation dot1Q 50
 ip address 10.10.50.1 255.255.255.0
interface g0/1.100
 encapsulation dot1Q 100
 ip address 10.0.100.2 255.255.255.0
interface g0/1.99
 encapsulation dot1Q 99 native
end
write memory
```

**Router2:** G0/1.100 at 10.0.100.3 and G0/1.99 as native. Router2 joins VLAN 50 in Phase 8 for HSRP.

- `encapsulation dot1Q` must come before the IP address on a subinterface, or IOS rejects the address.
- G0/1.99 with `native` matches the switch's native VLAN, so untagged frames are handled consistently. It has no IP because VLAN 99 carries no traffic.
- Subinterface numbers don't have to match VLAN IDs. Matching them is just convention for readability.

### Remote management from my PC

In v1, the home network was bridged into VLAN 1, so my PC was on the same subnet as every device. In v2, the home network connects only to Router1 G0/0, so SSH from my PC is a routing problem. It needed routes in both directions:

**Router1 joins the home network** (10.0.0.0/24, Deco at 10.0.0.1):

```
interface g0/0
 description UPLINK-TO-HOME-DECO
 ip address 10.0.0.15 255.255.255.0
 no shutdown
```

**Lab routes back toward the home network** (temporary static routes, to be replaced by OSPF in Phase 6):

```
! Core-Switch: send anything unknown to Router1
ip route 0.0.0.0 0.0.0.0 10.0.100.2

! Router1: one summary route covers every lab VLAN (10, 20, 30, 50)
ip route 10.10.0.0 255.255.0.0 10.0.100.1

! Router2: send anything unknown to Router1
ip route 0.0.0.0 0.0.0.0 10.0.100.2
```

The access switches already pointed at 10.10.30.1 with `ip default-gateway`, so their replies follow the Core-Switch's default route to Router1.

**My PC's routes to the lab** (PowerShell as Administrator):

```
route -p add 10.10.0.0 mask 255.255.0.0 10.0.0.15
route -p add 10.0.100.0 mask 255.255.255.0 10.0.0.15
```

Only lab traffic goes to Router1. Everything else, including gaming, still goes straight to the Deco, so the gaming connection never depends on the lab. `-p` makes the routes survive a reboot.

**SSH config (`~/.ssh/config`)**, updated for the v2 addresses:

| Host alias | Address | Key exchange override |
|---|---|---|
| core-switch | 10.10.30.1 | diffie-hellman-group1-sha1 |
| access-sw1 | 10.10.30.11 | diffie-hellman-group14-sha1 |
| access-sw2 | 10.10.30.12 | diffie-hellman-group14-sha1 |
| router1 | 10.0.0.15 | diffie-hellman-group14-sha1 |
| router2 | 10.0.100.3 | diffie-hellman-group14-sha1 |

All hosts also add `HostKeyAlgorithms +ssh-rsa`, `Ciphers +aes128-cbc`, and `MACs +hmac-sha1`.

**Result:** SSH from my PC to all five devices, including **Core-Switch**, which never fully worked over SSH in v1 and needed a Telnet fallback. Telnet is now disabled on every device.

### Proxmox host on a trunk (single NIC)

In v1, moving one VM to a different VLAN with an access port took the entire Proxmox host offline, because the host's own management shared that port. The v2 fix makes the host's switch port an **802.1Q trunk** and the Proxmox bridge **VLAN-aware**: the host's management lives in VLAN 30, and each VM is tagged into its own VLAN, all over one NIC.

**Order mattered** to avoid locking myself out. The Proxmox network changes were staged in the web UI (while the host was temporarily on the home network) but **not applied**:

```
auto vmbr0
iface vmbr0 inet manual
	bridge-ports nic0
	bridge-stp off
	bridge-fd 0
	bridge-vlan-aware yes
	bridge-vids 2-4094

auto vmbr0.30
iface vmbr0.30 inet static
	address 10.10.30.50/24
	gateway 10.10.30.1
```

Then I shut down the host, configured the switch port, recabled, and booted. The pending changes applied at startup.

**Access-SW2 Fa0/20:**

```
interface fa0/20
 description PROXMOX-HOST
 switchport mode trunk
 switchport nonegotiate
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,20,30,50,60
 spanning-tree portfast trunk
 no shutdown
```

`spanning-tree portfast trunk` is needed because the global PortFast default only applies to access ports, and the Proxmox host is an end device, not a switch. `bridge-stp off` on the Proxmox side means it sends no BPDUs, so BPDU Guard doesn't trigger.

**Result:** Proxmox web UI reachable from my PC at `https://10.10.30.50:8006`, routed through Router1 and Core-Switch and down the trunk. The host has no internet access until NAT is configured in Phase 8, because the Deco has no return route to 10.10.x.x.

### End-to-end verification with a real VM

**SVI routing (VLAN 10):** Tagged a Windows VM into VLAN 10 in Proxmox, with static IP 10.10.10.50/24 and gateway 10.10.10.1:

| Ping from VM | Result | Proves |
|---|---|---|
| 10.10.10.1 | Success | VM reaches its own gateway |
| 10.10.20.1 | Success | Core-Switch routes between VLANs |
| 10.10.30.11 | Success | Routed all the way to Access-SW1's management |

This is the end-to-end inter-VLAN test that had to be done in Packet Tracer in v1 because of the single-NIC problem.

**Router-on-a-stick (VLAN 50):** Moved the same VM to VLAN 50 (10.10.50.50/24, gateway 10.10.50.1 on Router1). Before testing, I predicted the path of a ping to 10.10.10.1:

1. VM → Access-SW2 → Core-Switch, tagged VLAN 50. The Core-Switch has no VLAN 50 SVI, so it only **switches** the frame up the trunk.
2. Router1 receives it on G0/1.50 and **routes** it using the 10.10.0.0/16 static route.
3. Router1 sends it back down **the same physical cable**, tagged VLAN 100.
4. Core-Switch receives it on its VLAN 100 SVI and replies, since 10.10.10.1 is its own address.

Both pings to 10.10.50.1 and 10.10.10.1 succeeded. This hairpin, the same cable carrying the traffic twice under different tags, is the main drawback of router-on-a-stick compared with SVIs on a Layer 3 switch.

### Lessons Learned / Troubleshooting — Phase 4

**Password recovery left an interface shut down.** Router2 couldn't reach anything after its subinterfaces were configured. `show ip interface brief` showed G0/1 and every subinterface **administratively down**. Earlier, I had done ROMMON password recovery on Router2, which reloads the config with `copy startup-config running-config`. That merges the saved config, but interfaces stay shut down, because `no shutdown` isn't stored in the config as a command. One `no shutdown` on the physical G0/1 brought it up, and the subinterfaces followed automatically.

**A one-digit typo in an SVI address.** After Router2 was fixed, Router1 could ping Router2, but the Core-Switch couldn't. Router-to-router traffic crossing the Core-Switch proved Layer 2 in VLAN 100 was fine, so I checked the Core-Switch's ARP table:

```
Internet  10.10.100.1   -   0018.181b.f2c4  ARPA   Vlan100
```

The VLAN 100 SVI was **10.10.100.1**, not **10.0.100.1**. That put the Core-Switch in a different subnet from the routers, so it treated 10.0.100.x as a remote network with no route and never even sent an ARP request. The missing ARP entries for both routers were the clue.

**Wrong subnet mask on a summary route.** Router1 couldn't reach Access-SW1 after the static routes were added. `show ip route static` showed:

```
S        10.10.0.0/24 [1/0] via 10.0.100.1
```

The mask had gone in as 255.255.255.0 instead of 255.255.0.0. A /24 only covers 10.10.0.0–10.10.0.255, so none of the lab VLANs matched it. Removing the route and re-adding it as /16 fixed it immediately. A summary route is only as good as its mask.

**SSH host key warnings after the rebuild.** SSH from my PC failed with `REMOTE HOST IDENTIFICATION HAS CHANGED`. Expected, because `crypto key zeroize rsa` and the new key generation gave every device a new identity, while my PC still had the v1 keys saved. Since I knew why the keys changed, it was safe to remove the old entries with `ssh-keygen -R <address>` and accept the new keys. In production, this exact warning would be a reason to stop and investigate before connecting.

**Windows blocks inbound ping.** Router1 could ping the Deco but not my PC. Windows Firewall drops inbound ICMP echo by default, so a failed ping *to* a Windows machine doesn't mean it's unreachable. Testing in the other direction (PC → Router1) confirmed connectivity.

**Forgot to change the VM's VLAN tag.** Pings from the VM in VLAN 50 failed, even to its own gateway. Rather than guessing, I used the MAC address tables to find where the frames stopped:

- Core-Switch `show mac address-table vlan 50`: **no learned MACs at all**, so no VLAN 50 frames from either direction.
- After pinging the VM from Router1, Access-SW2 learned Router1's MAC (0007.7d78.7701) on Po2, proving the Router1 → Core-Switch → SW2 path in VLAN 50 worked.
- No MAC on **Fa0/20** (the Proxmox port), so the VM wasn't sending anything in VLAN 50.

That pointed straight at the VM side, where the Proxmox VLAN tag was still set to 10. I had changed the IP inside Windows but not the tag in Proxmox. Changing it to 50 fixed it. The MAC address table is the right tool for Layer 2 problems: it shows exactly which devices a switch has heard from, and on which port.

---

## Phase 5 — Static Routing

Static routing comes before OSPF on purpose, so I understand exactly what a dynamic routing protocol replaces. Three static routes already existed from Phase 4 (added to get remote SSH working). This phase extended them and focused on reading the routing table and predicting reachability.

### Loopbacks on the routers

```
! Router1
interface loopback 0
 ip address 10.255.0.2 255.255.255.255

! Router2
interface loopback 0
 ip address 10.255.0.3 255.255.255.255
```

Core-Switch already had Loopback 0 (10.255.0.1). These will become the OSPF router IDs in Phase 6.

### Predicting reachability: "there and back"

Before testing, I predicted each ping using two questions:

1. Does the **sender** have a route to the destination?
2. Does the **destination** have a route back to the sender's **source address**, which is the IP of the interface the ping leaves from, not the loopback?

Static routes in place at the time:

| Device | Static route |
|---|---|
| Core-Switch | 0.0.0.0/0 → 10.0.100.2 (Router1) |
| Router1 | 10.10.0.0/16 → 10.0.100.1 (Core-Switch) |
| Router2 | 0.0.0.0/0 → 10.0.100.2 (Router1) |

| Ping | Prediction | Reasoning | Actual |
|---|---|---|---|
| Core-Switch → 10.255.0.2 | Works | Default route there; reply to 10.0.100.1 is directly connected on Router1 | Works |
| Router1 → 10.255.0.1 | Fails | 10.255.0.1 is outside 10.10.0.0/16, and Router1 had no default route | Fails |
| Router1 → 10.255.0.3 | Fails | No matching route, no default | Fails |
| Router2 → 10.255.0.2 | Works | Default route there; reply to 10.0.100.3 is directly connected | Works |

All four matched.

### Host routes

Router1 got /32 host routes to the other two loopbacks:

```
ip route 10.255.0.1 255.255.255.255 10.0.100.1
ip route 10.255.0.3 255.255.255.255 10.0.100.3
```

Both previously failed pings then succeeded.

### Reading the routing table

```
Router1#show ip route static
Gateway of last resort is not set

      10.0.0.0/8 is variably subnetted, 10 subnets, 3 masks
S        10.10.0.0/16 [1/0] via 10.0.100.1
S        10.255.0.1/32 [1/0] via 10.0.100.1
S        10.255.0.3/32 [1/0] via 10.0.100.3
```

- **`[1/0]`** is [administrative distance / metric]. Static routes have an AD of 1, so they're trusted over any routing protocol (OSPF is 110).
- **"variably subnetted, 10 subnets, 3 masks"**: the router knows prefixes of three lengths (/16, /24, /32).
- **"Gateway of last resort is not set"**: no default route, which is exactly why the two pings failed.

### Default route and longest-prefix match

As the edge router, Router1 got a default route to the Deco:

```
ip route 0.0.0.0 0.0.0.0 10.0.0.1
```

Router1 now had two routes matching 10.255.0.1: the 0.0.0.0/0 default and the 10.255.0.1/32 host route. I predicted correctly that the **/32 wins**. The longest (most specific) prefix match always takes priority, so the default route is only used when nothing more specific matches.

### Directly connected networks need no route

I predicted both of these pings would fail, and got one wrong:

| Ping | Source address | Prediction | Actual |
|---|---|---|---|
| Router1 → 8.8.8.8 | 10.0.0.15 | Fails | **Works** |
| VM (VLAN 50) → 8.8.8.8 | 10.10.50.50 | Fails | Fails |

My reasoning was that the Deco had no route back to either source. But 10.0.0.15 is in **10.0.0.0/24, the Deco's own home network**, so the Deco is directly connected to it and needs no route. To the Deco, Router1 is just another home device, like my PC. 10.10.50.50 is a network the Deco has never heard of, so the reply had nowhere to go.

That's the gap NAT on Router1 will close in Phase 8: it rewrites internal source addresses to 10.0.0.15, an address the Deco already knows how to reach.

### What Phase 5 covered

- Default routes (Core-Switch and Router2 to Router1, Router1 to the internet)
- Network routes, including a summary (10.10.0.0/16 covering every lab VLAN)
- Host routes (/32 loopbacks)
- Routing table interpretation: codes, [AD/metric], gateway of last resort
- Longest-prefix match
- Why directly connected networks never need a static route

The static routes stay in place going into Phase 6. OSPF will be enabled alongside them to show that static routes (AD 1) beat OSPF (AD 110), and then they'll be removed one at a time so OSPF takes over without losing remote access. Floating static routes are deferred to Phase 8, when a real backup path exists.

---

## Phase 6 — OSPF (Single Area, Built Deliberately)

In v1, OSPF worked, but several outcomes were accidental: the DR was decided by router ID, every link cost the same, and hellos went out on every interface. This time each of those was a design decision, and the static routes from Phase 5 stayed in place until OSPF had proven it could replace them.

### Core-Switch

```
configure terminal
interface vlan 100
 ip ospf priority 255
router ospf 1
 router-id 10.255.0.1
 auto-cost reference-bandwidth 10000
 passive-interface default
 no passive-interface Vlan100
 network 10.0.100.0 0.0.0.255 area 0
 network 10.10.10.0 0.0.0.255 area 0
 network 10.10.20.0 0.0.0.255 area 0
 network 10.10.30.0 0.0.0.255 area 0
 network 10.255.0.1 0.0.0.0 area 0
end
write memory
```

- **`ip ospf priority 255`** on VLAN 100 makes the Core-Switch the intended DR on the transit segment. It was set **before** the routers joined, because DR elections aren't preemptive.
- **`router-id`** is set manually to match Loopback 0, so it never changes on its own.
- **`auto-cost reference-bandwidth 10000`** (10 Gbps), set on all three devices. With the default 100 Mbps reference, every link at 100 Mbps or faster costs 1, so OSPF can't tell gigabit from FastEthernet. The SVIs now cost 10.
- **`passive-interface default`** with **`no passive-interface Vlan100`**: hellos only go out where OSPF neighbors exist. The user VLANs are still advertised, but no OSPF packets reach end devices.
- **Wildcard masks** are the inverse of subnet masks: `0.0.0.255` matches a /24, and `0.0.0.0` matches one exact address.

### Router1 and Router2

```
! Router1
router ospf 1
 router-id 10.255.0.2
 auto-cost reference-bandwidth 10000
 passive-interface default
 no passive-interface g0/1.100
 network 10.0.100.0 0.0.0.255 area 0
 network 10.10.50.0 0.0.0.255 area 0
 network 10.255.0.2 0.0.0.0 area 0
 default-information originate

! Router2
router ospf 1
 router-id 10.255.0.3
 auto-cost reference-bandwidth 10000
 passive-interface default
 no passive-interface g0/1.100
 network 10.0.100.0 0.0.0.255 area 0
 network 10.255.0.3 0.0.0.0 area 0
```

Router1's G0/0 (home network) is deliberately left out of OSPF. Instead, **`default-information originate`** advertises Router1's existing static default (to the Deco) into OSPF. It only advertises a default route the router already has. Adding `always` would advertise one regardless.

### DR/BDR election, predicted at each step

| Event | Prediction | Result |
|---|---|---|
| Core-Switch alone on VLAN 100 | DR (after the 40-second WAIT) | WAIT → DR |
| Router1 joins | BDR (DR already exists) | BDR (after fixing the issue below) |
| Router2 joins | DROTHER, despite a higher router ID than Router1, because BDR also doesn't preempt | DROTHER |

```
Router2#show ip ospf neighbor
Neighbor ID     Pri   State           Dead Time   Address         Interface
10.255.0.1      255   FULL/DR         00:00:35    10.0.100.1      GigabitEthernet0/1.100
10.255.0.2        1   FULL/BDR        00:00:36    10.0.100.2      GigabitEthernet0/1.100
```

Router2 forms FULL adjacencies with both the DR and BDR. Two DROTHERs on the same segment would stop at **2WAY** with each other and only fully synchronize with the DR and BDR. Reducing the number of full adjacencies is the reason DRs exist.

### Migrating from static routes to OSPF

OSPF was enabled with the Phase 5 static routes still in place.

**Static routes beat OSPF for the same prefix.** After Router1's `default-information originate`, the Core-Switch's table still showed:

```
S*   0.0.0.0/0 [1/0] via 10.0.100.2
```

OSPF had learned the default, but a static route (AD 1) beats OSPF (AD 110), so the OSPF route stayed hidden. Only the winning route for a prefix is installed.

**Removing the static revealed the OSPF route:**

```
O*E2 0.0.0.0/0 [110/1] via 10.0.100.2, 00:00:24, Vlan100
```

- **E2** (external type 2): the route came from outside OSPF, Router1's static route injected by `default-information originate`.
- **[110/1]**: E2 routes keep a fixed metric of 1, no matter how far away the advertising router is.
- The gateway of last resort didn't change, and SSH stayed up throughout. The OSPF route took over the instant the static was removed.

Router2's static default was removed next, then Router1's summary and host routes, keeping only Router1's default to the Deco.

**Prediction:** I predicted correctly that Router1 would learn the lab VLANs as **separate /24s**, not one /16. Each SVI network is advertised individually, and OSPF doesn't auto-summarize. Summarization is only possible at area borders, and this is a single area.

```
O        10.10.10.0/24 [110/20] via 10.0.100.1, 00:55:21, GigabitEthernet0/1.100
O        10.10.20.0/24 [110/20] via 10.0.100.1, 00:55:21, GigabitEthernet0/1.100
O        10.10.30.0/24 [110/20] via 10.0.100.1, 00:55:21, GigabitEthernet0/1.100
O        10.255.0.1/32 [110/11] via 10.0.100.1, 00:00:23, GigabitEthernet0/1.100
O        10.255.0.3/32 [110/11] via 10.0.100.3, 00:00:10, GigabitEthernet0/1.100
```

**The route timers told a second story.** The VLAN routes had been installed for **55 minutes**, the whole time the static /16 existed. Administrative distance only compares routes for the **exact same prefix**. A /16 and a /24 are different prefixes, so both were installed, and longest-prefix match meant traffic was already using the OSPF /24s. The loopback routes were only seconds old, because they had the same /32 prefix as the static host routes, which blocked them until they were removed.

**Cost check:** loopbacks show 11 (10 for the VLAN 100 link plus 1 for the loopback), and the remote VLANs show 20 (10 plus 10).

After the migration, SSH from my PC to all five devices still worked, now entirely over OSPF. The lab's only static route is Router1's default to the internet.

### Break-it tests

**Hello timer mismatch.** I set Router2's hello interval to 5 seconds on G0/1.100, which automatically makes the dead interval 20 seconds. The Core-Switch and Router1 stayed at 10/40.

```
%OSPF-5-ADJCHG: Process 1, Nbr 10.255.0.2 on GigabitEthernet0/1.100 from FULL to DOWN, Neighbor Down: Dead timer expired
```

The adjacency didn't drop immediately. Each side silently discarded the other's hellos because the timers didn't match, and the neighbors only went down once the dead timer expired (Router2's 20 seconds first). There's no explicit error. The failure only shows up when the dead timer runs out, so the diagnosis is to compare `show ip ospf interface` on both ends.

**Area mismatch.** I moved Router2's transit network to area 1:

```
%OSPF-4-ERRRCV: Received invalid packet: mismatched area ID from backbone area from 10.0.100.2, GigabitEthernet0/1.100
```

This time IOS logged an explicit error, naming the problem, repeating with every hello.

| | Timer mismatch | Area mismatch |
|---|---|---|
| Error logged | None | `ERRRCV: mismatched area ID` |
| How it shows up | Dead timer expires | Immediately, repeating every hello |
| Diagnosis | Compare `show ip ospf interface` on both ends | The log message names it |

Settings that must match for an adjacency: area ID, hello/dead timers, subnet and mask, authentication, and stub flag. Router IDs must be unique. An MTU mismatch lets neighbors get past 2WAY but leaves them stuck in EXSTART/EXCHANGE.

### Lessons Learned / Troubleshooting — Phase 6

**Wrong device again, with a hidden side effect.** Router1's OSPF commands were pasted into the Core-Switch, and the error only appeared at `no passive-interface g0/1.100`, an interface the Core-Switch doesn't have. The lines before it had been accepted, so I checked `show run | section router ospf` and removed the stray `router-id 10.255.0.2` and Router1's network statements. A router ID change doesn't take effect until the OSPF process restarts, so the Core-Switch never actually ran as 10.255.0.2.

The less obvious damage showed up later. Router1 sat as **DR with 0 neighbors** after its own OSPF config, meaning it wasn't hearing hellos from the Core-Switch. The cause was the pasted `passive-interface default`: **entering it again resets every interface to passive**, silently undoing the earlier `no passive-interface Vlan100`. The Core-Switch had stopped sending hellos on the transit network. Re-entering `no passive-interface Vlan100` brought the adjacency up.

**DR non-preemption, live.** When the adjacency formed, the Core-Switch came up as **FULL/BDR**, despite priority 255:

```
10.255.0.1      255   FULL/BDR        00:00:38    10.0.100.1      GigabitEthernet0/1.100
```

While the Core-Switch was accidentally passive, Router1 had been alone on VLAN 100 and became DR. When the Core-Switch returned, it found an existing DR, and a higher priority doesn't take over. `clear ip ospf process` on Router1 forced a new election: the Core-Switch (the BDR) was immediately promoted to DR, which is the BDR's purpose, and Router1 rejoined as BDR. It was a real demonstration of why `ip ospf priority` has to be set before neighbors form.

**Breaking OSPF cut off Router2's own management.** The hello timer test dropped my SSH session to Router2. After Phase 6, Router2 had no static routes, so its **only** route back to my PC (10.0.0.88) was the OSPF default from Router1. When the adjacency dropped, Router2 lost that route. My SSH packets still reached it (Router1 is directly connected to 10.0.100.0/24), but Router2 had no route for the replies. It's the "there and back" rule from Phase 5 again.

Recovery didn't need the console. I SSHed to Router1 and hopped to Router2 (`ssh -l admin 10.0.100.3`). That session came from 10.0.100.2, on Router2's own subnet, so Router2 could reply with no routes at all. The area-mismatch test was run from that hop session for the same reason. The takeaway: a router that relies entirely on a dynamic routing protocol also relies on it for its own management, so when breaking a protocol on purpose, keep a path in that doesn't depend on it.

---

## Phase 7 — IPv6

IPv6 runs on the two routers. The Core-Switch's 12.2(25)SED1 image isn't used for IPv6 routing; it only carries VLAN 60 at Layer 2 over the existing trunks, which already allowed it. The CCNA IPv6 objectives are addressing, address types, and static routing, all of which fit on the routers.

### Enabling IPv6 routing

On both routers:

```
ipv6 unicast-routing
```

Without it, a router can hold IPv6 addresses but won't forward IPv6 between interfaces or send Router Advertisements, which SLAAC depends on.

### EUI-64, calculated by hand first

Router1's G0/1 MAC is `0007.7d78.7701`. Before configuring anything, I built the EUI-64 interface ID manually:

| Step | Result |
|---|---|
| Split the 48-bit MAC in half | `0007 7d` · `78 7701` |
| Insert `FFFE` in the middle | `0007:7DFF:FE78:7701` |
| Flip the 7th bit of the first byte (U/L bit): `00000000` → `00000010` | `0207:7DFF:FE78:7701` |
| Add the /64 prefix | **`2001:DB8:60::207:7DFF:FE78:7701`** |

My first attempt changed the wrong byte (`0017`) by adding 16 instead of 2. The 7th bit has a value of 2, and the flip only ever touches the second hex digit of the address: 0↔2, 1↔3, C↔E, and so on.

**Router1 (EUI-64):**

```
interface g0/1.60
 encapsulation dot1Q 60
 ipv6 address 2001:db8:60::/64 eui-64
interface loopback 0
 ipv6 address 2001:db8:1::1/128
```

**Router2 (manual):**

```
interface g0/1.60
 encapsulation dot1Q 60
 ipv6 address 2001:db8:60::3/64
interface loopback 0
 ipv6 address 2001:db8:2::1/128
```

**Verification on Router1** — the global address matched the hand calculation exactly:

```
GigabitEthernet0/1.60      [up/up]
    FE80::207:7DFF:FE78:7701
    2001:DB8:60:0:207:7DFF:FE78:7701
```

### Link-local addresses

Every IPv6 interface automatically gets an **FE80::/10 link-local** address. IOS builds it with EUI-64 from the MAC unless one is set manually with `ipv6 address <addr> link-local`. Configuring a manual *global* address doesn't change that, as Router2 showed:

```
GigabitEthernet0/1.60  [up/up]
    FE80::7E0E:CEFF:FE47:6A61
    2001:DB8:60::3
Loopback0              [up/up]
    FE80::7E0E:CEFF:FE47:6A60
    2001:DB8:2::1
```

- Router2's link-local starts `7E0E`, so its MAC starts `7C0E`: the U/L flip turned C (1100) into E (1110).
- Loopbacks have no MAC, so IOS borrows one from a physical interface. Router2's loopback link-local ends in `6A60` (G0/0's MAC) while G0/1.60's ends in `6A61`.

Link-local addresses are never routed. IPv6 uses them for Neighbor Discovery, Router Advertisements, and as next hops.

### Neighbor Discovery instead of ARP

After pinging Router2 from Router1:

```
Router1#show ipv6 neighbors
IPv6 Address                              Age Link-layer Addr State Interface
FE80::7E0E:CEFF:FE47:6A61                   3 7c0e.ce47.6a61  DELAY Gi0/1.60
2001:DB8:60::3                              0 7c0e.ce47.6a61  REACH Gi0/1.60
```

IPv6 has no ARP. ICMPv6 Neighbor Solicitation and Advertisement messages resolve addresses to MACs, and each IPv6 address gets its own entry, so Router2 appears twice with the same MAC. **REACH** means confirmed reachable within the last 30 seconds; **DELAY** means the entry went stale, traffic was just sent, and the router is waiting briefly before probing.

### Static routes: global vs. link-local next hop

```
! Router1 — global next hop
ipv6 route 2001:db8:2::1/128 2001:db8:60::3

! Router2 — link-local next hop plus exit interface
ipv6 route 2001:db8:1::1/128 g0/1.60 FE80::207:7DFF:FE78:7701
```

A global next hop resolves through the routing table: 2001:db8:60::/64 is connected on G0/1.60, so the router knows which interface to use (a recursive lookup). A link-local next hop can't be resolved that way. Every interface has an FE80:: address, and the same link-local address can exist on several links, so the **exit interface is required**.

**Verification** — pinging with the loopback as the source forces the reply to use the other router's static route, testing both directions at once:

```
Router1#ping 2001:db8:2::1 source loopback0
Packet sent with a source address of 2001:DB8:1::1
!!!!!
Success rate is 100 percent (5/5)
```

### IPv6 default route and longest-prefix match

```
! Router2
ipv6 route ::/0 2001:db8:60::207:7dff:fe78:7701
```

`::/0` is the IPv6 equivalent of 0.0.0.0/0. With both the /128 host route and the default in place, I predicted correctly that the **/128 wins** (longest-prefix match works the same as IPv4), and that removing the /128 would leave the loopback ping working through the default. It did.

### SLAAC with a Windows VM

I tagged the Windows VM into VLAN 60 in Proxmox and left IPv6 on "obtain automatically." With no DHCP server and no manual addressing, it configured itself from Router Advertisements:

```
IPv6 Address. . . . . . . . . . . : 2001:db8:60:0:438f:f91b:4bb8:fc3e
Temporary IPv6 Address. . . . . . : 2001:db8:60:0:bd5d:6b77:7b62:c0d3
Link-local IPv6 Address . . . . . : fe80::f65a:4cc5:52c2:b4a3%11
Default Gateway . . . . . . . . . : fe80::207:7dff:fe78:7701%11
```

- **The prefix came from the RA; the interface ID didn't use EUI-64.** There's no `FFFE` in the middle. Windows generates a random interface ID so the address doesn't expose the MAC.
- **Temporary address**: a second, short-lived random address (RFC 4941 privacy extensions) that Windows uses for outgoing connections and rotates regularly.
- **Default gateway is Router1's link-local address**, the one calculated by hand. Gateways learned from RAs are always link-local.
- **`%11`** is the zone ID (the interface index). It exists for the same reason the static route needed an exit interface: a link-local address doesn't identify its link on its own.

Router1's neighbor table confirmed which address Windows actually used:

```
2001:DB8:60:0:BD5D:6B77:7B62:C0D3           0 bc24.115b.ad33  STALE Gi0/1.60
FE80::F65A:4CC5:52C2:B4A3                   0 bc24.115b.ad33  REACH Gi0/1.60
```

The VM's **temporary** address was the one learned, confirming Windows uses it for outbound traffic. `BC:24:11` is the MAC prefix Proxmox assigns to virtual NICs.

From the VM, pings to Router2 (2001:db8:60::3) and both router loopbacks succeeded.

### Multicast groups and address types

```
Router1#show ipv6 interface g0/1.60
  Joined group address(es):
    FF02::1
    FF02::2
    FF02::1:FF78:7701
  ND DAD is enabled, number of DAD attempts: 1
  ND router advertisements are sent every 200 seconds
  Hosts use stateless autoconfig for addresses.
```

| Address | Meaning |
|---|---|
| `FF02::1` | All nodes on the link. Every IPv6 interface joins it. |
| `FF02::2` | All routers. Joined only because `ipv6 unicast-routing` is on; hosts send Router Solicitations here. |
| `FF02::1:FF78:7701` | Solicited-node multicast: `FF02::1:FF` plus the last 24 bits of a unicast address. Neighbor Solicitations go here instead of a broadcast, so only interfaces that might own the address process them. |

Router1 joins only **one** solicited-node group because its link-local and global addresses share the same last 24 bits (`78:7701`). Router2 would join two: `FF02::1:FF00:3` for its manual `::3` address and `FF02::1:FF47:6A61` for its link-local.

The rest of the output ties back to what the VM did: **DAD** (Duplicate Address Detection) checks an address is unused before using it, RAs go out every 200 seconds, and "stateless autoconfig" tells hosts to use SLAAC rather than DHCPv6.

### What Phase 7 covered

| IPv6 topic | How it was verified |
|---|---|
| EUI-64 | Calculated by hand, matched the router's output |
| Manual addressing | Router2 `::3` |
| Link-local vs. global unicast | Both present on every interface, link-local always auto-generated |
| Neighbor Discovery | `show ipv6 neighbors` replacing ARP |
| Static routes (global and link-local next hop) | Loopback-sourced pings in both directions |
| Default route and longest-prefix match | `::/0` took over when the /128 was removed |
| SLAAC | Windows VM self-configured from RAs |
| Multicast and solicited-node addresses | `show ipv6 interface` |

---

## Next Steps

- **Phase 8:** DHCP with relay, NAT/PAT through the home network, NTP, Syslog, SNMP, and HSRP.
- **Phase 9:** Redesigned ACLs, port security, DHCP snooping, and Dynamic ARP Inspection. The VTY access-class must permit my home network (10.0.0.0/24) as well as VLAN 30, since I now manage the lab from 10.0.0.88.
- **Phase 10:** Netmiko automation, and a full power-cycle sign-off.
