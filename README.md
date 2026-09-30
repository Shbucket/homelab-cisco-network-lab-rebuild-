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
| Proxmox host | Dell OptiPlex 7060 | VM host (to be connected later) | Proxmox VE |

## Topology

```
                  Home network (Deco)
                          |
                        G0/0
       [Router1]                     [Router2]
         G0/1                          G0/1
           |                             |
         Gi0/2                         Gi0/4
        +-------------------------------------+
        |        Core-Switch (3560G)          |
        +-------------------------------------+
         Gi0/5, Gi0/6                Gi0/1, Gi0/3
         (Po1 - LACP)                (Po2 - PAgP)
              |                           |
          Gi0/1, Gi0/2               Gi0/1, Gi0/2
         [Access-SW1] ---Fa0/24---- [Access-SW2]
                    (cross-link, STP backup path)
```

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

## Next Steps

- **Phase 4:** Inter-VLAN routing with SVIs on Core-Switch, and router-on-a-stick on Router1 for VLAN 50.
- **Phase 5:** Static routing before OSPF, and routing table analysis.
- **Phase 6:** Single-area OSPF with a deliberately chosen DR and a default route originated from Router1.
- **Phase 7:** IPv6 addressing (EUI-64, link-local, SLAAC) and static routing on the routers.
- **Phase 8:** DHCP with relay, NAT/PAT through the home network, NTP, Syslog, SNMP, and HSRP.
- **Phase 9:** Redesigned ACLs, port security, DHCP snooping, and Dynamic ARP Inspection.
- **Phase 10:** Netmiko automation, and a full power-cycle sign-off.
