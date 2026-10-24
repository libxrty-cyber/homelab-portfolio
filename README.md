# Blue Team Homelab

An isolated cybersecurity lab on a single mini PC, built to practice defensive security: SIEM monitoring, Active Directory, firewall segmentation, and detection engineering.

**Status:** In progress. The core infrastructure is running (hypervisor, firewall, SIEM, domain controller, attacker box). The Windows endpoint, monitoring agents, and first detections are next.

## Goals

- Build a safe, isolated environment where attacks can be run and detected without touching the home network
- Practice the blue-team workflow: collect logs, triage alerts, separate noise from real signal, write detections
- Learn the supporting skills along the way: virtualization, firewalling and segmentation, DNS/DHCP, Active Directory

## Hardware

| Item | Detail |
|---|---|
| Host | GMKtec M5 Ultra mini PC|
| CPU / RAM | Ryzen 7 7730U, 32 GB DDR4 (upgradeable) |
| Network | Dual 2.5 GbE ports (currently on a gigabit switch) |
| Storage | Proxmox reports ~876 GB usable VM storage (LVM-thin) |
| Placement | Basement, wired to the existing home switch |
| Management | From my main PC over Wi-Fi, through the Proxmox web UI |

## Architecture

```
Internet
   |
Home router (192.168.1.1)
   |
Basement switch ---- Main PC (Wi-Fi, browser console only)
   |
Mini PC: Proxmox VE 9.2  (192.168.1.254)
   |
 vmbr0  (bridged to the physical NIC, home side)
   |
OPNsense VM   WAN: 192.168.1.247 (DHCP)   LAN: 10.10.10.1/24
   |
 vmbr1  (internal bridge, no physical port: the isolated lab)
   |
   +-- Kali            (attacker)
   +-- Wazuh           (SIEM)               10.10.10.10
   +-- DC01            (Active Directory)   10.10.10.20
   +-- Windows 11      (endpoint, planned)
```

`vmbr1` has no physical port, so the only path in or out of the lab is through the OPNsense VM.

## Network plan

| Network | Subnet | Notes |
|---|---|---|
| Home LAN | 192.168.1.0/24 | Router .1, Proxmox host .254 (static), OPNsense WAN .247 (DHCP) |
| Lab LAN | 10.10.10.0/24 | OPNsense .1, Wazuh .10 (static), DC01 .20 (static), DHCP pool .100 to .200 |

Lab DNS is served by DC01 (`10.10.10.20`), pushed to lab clients through a DHCP option in Dnsmasq.

## VM inventory

| ID | Name | OS | vCPU / RAM / Disk | Role |
|---|---|---|---|---|
| 100 | opnsense | OPNsense 26.7 | 2 / 4 GB / 32 GB | Firewall and DHCP for the lab |
| 101 | kali | Kali Linux 2026.2 | 2 / 4 GB / 60 GB | Attacker box and lab console |
| 102 | wazuh | Ubuntu Server 24.04.5, Wazuh 4.14 | 4 / 8 GB / 100 GB | SIEM (all-in-one) |
| 103 | dc01 | Windows Server 2025 Evaluation | 2 / 4 GB / 60 GB | Domain controller for `lab.internal` |
| 104 | win11 | Windows 11 Enterprise Evaluation | planned | Domain-joined endpoint |

## Key design decisions

- **Bare-metal hypervisor, not dual boot.** Proxmox is the only OS on the hardware, and everything else is a VM. This lets several machines run at once and makes snapshots possible.
- **Isolated lab bridge behind a firewall VM.** Lab traffic can only leave through OPNsense.
- **Lab-to-home blocking.** OPNsense's default LAN rule allows everything, which would let the lab reach the home router and the Proxmox host. I added a block rule for all private address ranges above the default allow, plus a pass rule so lab machines can still reach the firewall itself for DNS and the web UI. I tested both sides.
- **IPv6 off on the lab side.** The lab does not receive an IPv6 prefix from the ISP, so nothing bypasses the IPv4 block rules.
- **Static addresses for infrastructure.** Wazuh and the domain controller sit outside the DHCP range so they never move.
- **Management is separate from study.** Real accounts are never logged into from lab VMs.

## Daily log

| Day | Date | Summary |
|---|---|---|
| 0 | Before Oct 7 | [Planning](logs/00-planning.md): hardware choice, architecture, cabling plan |
| 1 | Oct 7, 2026 | [Proxmox install](logs/01-2026-10-07-proxmox-install.md) |
| 2 | Oct 8, 2026 | [Firewall, Kali, Wazuh, and the domain controller](logs/02-2026-10-08-firewall-siem-ad.md) |

## Roadmap

- [ ] Windows 11 endpoint, joined to `lab.internal`
- [ ] Sysmon and the Wazuh agent on both Windows machines
- [ ] First Atomic Red Team test, found and triaged in Wazuh
- [ ] Suricata on OPNsense, with alerts forwarded to Wazuh
- [ ] Baseline snapshots of every VM
- [ ] Wired connection to my room (currently Wi-Fi)
- [ ] Possible later: VLAN segmentation of the lab, Security Onion

## Open items

- Router DHCP pool overlaps the Proxmox host address (`.254`); fix by shrinking the pool or adding a reservation
- DC01 DNS forwarder (`10.10.10.1`) so outside lookups work through the domain controller
- Isolation tests from DC01 (pings and port checks to home network)
