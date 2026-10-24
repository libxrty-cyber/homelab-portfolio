# Day 1: Proxmox install (Wed, Oct 7, 2026)

## Goal

Get a working hypervisor on the mini PC and manage it from the main PC's browser.

## What I did

1. Downloaded the Proxmox VE 9.2 ISO to the main PC.
2. Wrote it to a USB stick (see problem 1 below).
3. Booted the mini PC from the stick, with a monitor, keyboard, and an Ethernet cable to the basement switch.
4. Management network settings in the installer, taken from the router's DHCP offer: `192.168.1.254/24`, gateway `192.168.1.1`, DNS `192.168.1.1`.
5. Installed to the internal drive and removed the USB stick on first reboot.
6. Opened the web UI from the main PC at `https://192.168.1.254:8006` and logged in as `root` (Linux PAM realm).
7. Removed the enterprise repositories and added the no-subscription repository so updates work without a paid subscription.
8. Checked storage: `local` (directory, for ISOs) and `local-lvm` (thin pool, for VM disks).

## Problems and fixes

**1. ISO copied onto the USB stick, but it would not boot.**
Copying an ISO file onto a drive does not make it bootable. The image has to be written to the drive with an imaging tool in DD mode. Reflashed the stick and the installer booted.

**2. Hostname field would not accept edits in the installer.**
Left the default, since it is only a label on a standalone node. <!-- verify: final hostname -->

**3. "No valid subscription" popup on every login.**
Expected on the free edition. Every feature works without a subscription. It only means the enterprise update repository is unavailable.

**4. Static IP chosen by the installer sits at the very end of the router's DHCP pool.**
The installer saves the address it was offered as a static address, and `.254` is the last address in the router's DHCP range (`.2` to `.254`). Conflict risk is low (routers hand out from the bottom of the range), but it is real. Planned fix: shrink the router's DHCP pool to end at `.253`, or add a reservation for the host's MAC address. <!-- verify: confirm this was done -->

## Observations

- The Proxmox thin pool showed about **876 GB** free, so the SSD is larger than the 512 GB I had planned around. That makes storage a non-issue for the lab.
- The Proxmox UI uses a self-signed certificate, so browsers show a warning. Expected.

## Result

Proxmox running, reachable from the main PC over Wi-Fi, storage confirmed, updates configured.

