# Day 2: Firewall, Kali, Wazuh, and the domain controller (Thu, Oct 8, 2026)

Two sessions: an early-morning one (network and firewall) and an afternoon/evening one (SIEM and domain controller).

---

## Session 1 (early morning): network, firewall, attacker box

### What I did

1. Uploaded ISOs to `local` storage: OPNsense 26.7, Kali 2026.2 installer, Ubuntu Server, VirtIO Windows drivers, Windows 11 Enterprise Evaluation.
2. Created a second Linux bridge, **`vmbr1`**, with no IP and no physical port. This is the isolated lab network.
3. Built the **OPNsense VM** (ID 100): 2 vCPU, 4 GB RAM, 32 GB disk. First NIC on `vmbr0` (WAN), second on `vmbr1` (LAN), both VirtIO, with the Proxmox firewall flag off.
4. Installed OPNsense (UFS), and declined LAGG and VLAN setup at the console, since neither is needed.
5. Assigned interfaces: WAN = `vtnet0` (`vmbr0`), LAN = `vtnet1` (`vmbr1`).
6. Set the LAN to `10.10.10.1/24` and enabled DHCP for `10.10.10.100` to `10.10.10.200`. Declined IPv6 on the LAN so the lab does not receive the ISP's prefix. WAN picked up `192.168.1.247` over DHCP.
7. Built **Kali** (ID 101): 2 vCPU, 4 GB, 60 GB, NIC on `vmbr1` only. It received `10.10.10.119` from OPNsense.
8. Confirmed connectivity from Kali: ping to the firewall (`10.10.10.1`), to `8.8.8.8`, and to `google.com` all worked.
9. Logged into the OPNsense web UI from inside Kali (`https://10.10.10.1`), since the lab network is invisible to the main PC by design.
10. **Hardened the firewall:**
    - Created an alias `PrivateNets` containing `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`.
    - LAN rule 1: **Pass**, source `LAN net`, destination `LAN address` (so lab machines can still reach the firewall for DNS and the UI).
    - LAN rule 2: **Block**, source `LAN net`, destination `PrivateNets`.
    - Both rules sit above the default allow-to-any rule.

### Verification

| Test (from Kali) | Expected | Result |
|---|---|---|
| `ping 10.10.10.1` | success | success |
| `ping 8.8.8.8` | success | success |
| `ping google.com` | success | success |
| `ping 192.168.1.1` (home router) | fail | fail (before and after the rules, baseline succeeded first) |

### Problems and fixes

**1. VM would not start: `bridge 'vmbr1' does not exist`.**
The bridge had been created but not applied. In Proxmox, network changes stay pending until you click **Apply Configuration** on the node's Network page. Pending bridges still appear in the VM's bridge dropdown, so seeing it there is not proof it exists.

**2. OPNsense came up with WAN empty and LAN on `192.168.1.1`.**
The interfaces were assigned backwards: the NIC on the home bridge was acting as LAN, carrying OPNsense's default LAN address (the same address as my router) onto the home network. Fixed by re-running *Assign interfaces* with WAN = `vtnet0` and LAN = `vtnet1`, matching NIC order against the VM's hardware tab. **Lesson:** verify which virtual NIC is which before bringing up any DHCP server.

**3. Kali installer: "network autoconfiguration failed."**
The lab had no working DHCP server yet. Re-ran the LAN interface setup at the OPNsense console with DHCP enabled and retried.

**4. Kali login rejected.**
Linux usernames are case sensitive. (lol)

**5. Opened the wrong firewall.**
Proxmox and OPNsense both have "Firewall > Aliases." I started creating the alias in the Proxmox UI, which does not affect lab traffic. The rules belong in OPNsense, reached from inside the lab.

**6. Alias rejected: `" 172.16.0.0/12" is not a network`.**
A stray leading space in one entry. Removed it and re-entered the value.

**7. Firewall rules in the wrong order (twice).**
New rules landed below the default allow-all, where they never match. Fixed with the checkboxes and the "move selection before this rule" arrow. The second time, the Block rule sat above the Pass rule, which would have blocked access to the firewall itself because `10.0.0.0/8` includes it. Final order: Pass, Block, default allow. Firewall rules are first-match.

**8. Skipped snapshots that night.** Took them afterward.

---

## Session 2 (afternoon and evening): Wazuh and Active Directory

### What I did: Wazuh (SIEM)

1. Built the **Wazuh VM** (ID 102): 4 vCPU (`host` CPU type), 8 GB RAM, 100 GB disk, NIC on `vmbr1`.
2. Installed **Ubuntu Server 24.04.5 LTS** (standard install, not minimized) with a static address, `10.10.10.10/24`, gateway and DNS `10.10.10.1`. The address is outside the DHCP pool so it never changes.
3. Grew the root volume: the installer's guided LVM layout used only about half the disk (48 GB), so I extended it with `lvextend -l +100%FREE -r` to about 98 GB.
4. Updated the OS and rebooted.
5. Installed Wazuh 4.14 using the all-in-one installation assistant from the official quickstart.
6. Logged into the dashboard from Kali at `https://10.10.10.10`. Credentials come from the installer output (and `wazuh-install-files.tar`).

**Observation:** In the first few minutes the dashboard showed about 300 alerts (0 critical, 0 high, 133 medium, 170 low) with nothing deployed yet. That is the expected baseline burst: configuration assessment, vulnerability detection, and the first inventory pass on the Wazuh server itself. A real lesson in noise versus signal before writing any detections.

### What I did: Domain controller

1. Built **DC01** (ID 103): Windows Server 2025 Standard Evaluation (Desktop Experience), 2 vCPU, 4 GB RAM, 60 GB disk on VirtIO SCSI, `q35` machine type, with the VirtIO driver ISO attached as a second CD drive. NIC on `vmbr1`.
2. During setup, loaded the VirtIO storage driver (`vioscsi`, `2k25`, `amd64`) so the installer could see the disk.
3. After install, ran `virtio-win-gt-x64.msi` from the driver CD to get the network driver.
4. Set a static address: `10.10.10.20/24`, gateway `10.10.10.1`, and renamed the machine `DC01`.
5. Installed the **AD DS** role and promoted the server to a domain controller in a new forest, **`lab.internal`** (NetBIOS name `LAB`). I used `.internal`, which is reserved for private use, to avoid the mDNS clash that `.local` can cause. The "DNS delegation cannot be created" warning is expected for a standalone lab.
6. Verified: `nslookup lab.internal` on DC01 returns `10.10.10.20`, answered by the local DNS server.
7. Pointed the lab at the DC for DNS: in OPNsense's Dnsmasq service, added a DHCP option on the LAN (`dns-server`, `10.10.10.20`).
8. On Kali, renewed the lease and confirmed `resolv.conf` shows `10.10.10.20` and `nslookup lab.internal` resolves through the DC.

### Problems and fixes

**1. Ubuntu 26.04 selected by default, but unsupported by Wazuh.**
The Ubuntu download page defaults to the newest release, and Wazuh's quickstart lists up to 24.04. Switched to the 24.04.5 LTS server ISO via the releases page. **Lesson:** check the vendor's supported-OS list before building the VM.


**2. Windows installer showed no disks.**
Windows has no driver for VirtIO storage. Loaded `vioscsi` from the VirtIO CD (`2k25` folder), after which the disk appeared.

**3. No `dhclient` on Kali.**
Current Kali uses NetworkManager. Renewed the lease with `nmcli device disconnect eth0 && nmcli device connect eth0`.

### Verification still to do

- Isolation checks from DC01: `ping 192.168.1.1` and `Test-NetConnection 192.168.1.254 -Port 8006` should fail, while `ping 10.10.10.1` and `ping 8.8.8.8` succeed.
- DNS forwarder on DC01 (`10.10.10.1`) so `nslookup google.com` resolves through the DC.

## Result

A working isolated lab: firewall, attacker box, SIEM, and an Active Directory domain with DNS, all behind OPNsense. The Windows 11 endpoint is the next build.

## Next

1. Windows 11 endpoint (UEFI, TPM 2.0, VirtIO drivers), joined to `lab.internal`
2. Sysmon and the Wazuh agent on both Windows machines
3. First Atomic Red Team test, then find it in Wazuh
4. Baseline snapshots of every VM

