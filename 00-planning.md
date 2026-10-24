# Day 0: Planning

*Date: before Oct 7, 2026* <!-- verify: exact planning dates -->

## Hardware choice

Chose the **GMKtec M5 Ultra** (Ryzen 7 7730U, 32 GB DDR4, 512 GB listing, ~$550 from Amazon). The deciding factors were upgradeable RAM and dual 2.5 GbE ports, with room under an ~$800 budget for later upgrades.

## Architecture decisions

- **Proxmox VE installed directly on the hardware**, with Kali, Windows, and the firewall as VMs. A dual boot was rejected because only one OS could run at a time.
- **Main PC stays untouched.** It is only a console: a browser pointed at the Proxmox web UI. No Windows reinstall.
- **Firewall VM between home and lab.** OPNsense sits between the home-facing bridge and a private lab bridge. This keeps malware samples and attack tools off the family network.
- **Keep personal logins off lab machines.** Study and accounts stay on the main PC. Lab VMs use throwaway credentials only.

## Build order

1. Install Proxmox
2. Build the network (home bridge, isolated lab bridge)
3. OPNsense as the lab gateway
4. Attacker VM (Kali)
5. SIEM (Wazuh)
6. Domain controller and Windows endpoint
7. Agents and detections

## Cabling plan (deferred)

Planned an Ethernet run from the basement switch to my room: solid bare-copper Cat6, riser (CMR) rated for the vertical run, with a spare cable pulled alongside and firestopping at floor penetrations. The lab does not depend on it, so for now the mini PC sits in the basement and the main PC reaches it over Wi-Fi.

## Pre-purchase shopping list

USB stick for the installer, a Cat6 patch cable, a surge protector, a spare keyboard and monitor for the one-time install, and an optional UPS.
