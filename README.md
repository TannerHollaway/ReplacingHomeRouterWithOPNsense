# Home Network Domain Project — OPNsense Firewall/Router Deployment

Personal notes. This took quite a while with a lot of banging my head against the wall and research. In the end I simply didn't set the static IP correctly before attempting setup. 

---

## Project Overview

This project deploys OPNsense as a dedicated firewall and router on a Dell Optiplex 3040 SFF, replacing a consumer router and establishing the foundation for a fully segmented home network. The setup uses a double NAT topology with the existing ISP router upstream, and OPNsense managing all internal routing, DHCP, DNS, and firewall enforcement.

This is Part 1 of a multi-part home lab series:

- **Part 1 — OPNsense Firewall/Router Deployment** ← you are here
- Part 2 — VLAN Segmentation and Multi-SSID Wireless Setup
- Part 3 — Windows Server 2022 Active Directory Domain

**Skills demonstrated:** network architecture, firewall deployment, physical hardware troubleshooting, subnet planning, DHCP/DNS configuration, double NAT design, and real-world troubleshooting and root cause analysis.

---

## Hardware

| Component | Details |
|-----------|---------|
| OPNsense box | Dell Optiplex 3040 SFF |
| NIC (WAN) | Onboard Realtek (re0) |
| NIC (LAN) | Intel 82576 dual-port low-profile PCIe card (igb0) |
| Switch | TP-Link TL-SG108E 8-port gigabit smart switch |
| Access Point | TP-Link EAP225 Omada AC1350 |
| Domain Controller | Dedicated laptop, Windows Server 2022 Standard |
| Main PC | Dakota VLAN, connected via TP-Link AV1000 powerline adapter |
| Cabling | Cable Matters Cat6 5ft x4 |

---

## Network Topology

```
ISP Modem → Existing Router → Optiplex WAN (re0) → Optiplex LAN (igb0) → TL-SG108E Switch → EAP225 AP → Wireless devices
```

This is a **double NAT** setup. The existing router handles the ISP connection and provides internet to OPNsense via DHCP on the WAN port. OPNsense manages all internal routing, DHCP, DNS, and firewall rules. Client devices never interact directly with the existing router.

**Double NAT trade-off:** Port forwarding from the internet requires configuration on both the existing router and OPNsense. This is acceptable for a home lab focused on internal segmentation and Active Directory.

---

## IP Addressing

| Interface | Address | Notes |
|-----------|---------|-------|
| Existing router | 192.168.1.1/24 | Upstream router |
| OPNsense WAN (re0) | 192.168.1.119/24 | DHCP from existing router |
| OPNsense LAN (igb0) | 10.0.0.1/24 | Static |
| DHCP pool | 10.0.0.100 – 10.0.0.200 | Client devices |
| Reserved | 10.0.0.1 – 10.0.0.99 | Infrastructure (DC, AP, etc.) |

---

## VLAN Design (Implemented in Part 2)

| VLAN | Name | Purpose | Rules |
|------|------|---------|-------|
| 10 | Dakota | Main PC, personal devices | Full LAN and internet access |
| 20 | Domain Controller | DC laptop | Strict — only necessary ports |
| 30 | Family | Family PC, trusted devices | Internet and domain access |
| 40 | Guest | Visitors | Internet only, no LAN access |
| 50 | IoT | Smart devices | Internet only, isolated from LAN |

---

## Phase 1 — Hardware Preparation

The Intel 82576 dual-port low-profile NIC was installed into the PCIe x1 slot on the Optiplex 3040 SFF. The card was confirmed detected in BIOS on first boot, appearing as Intel Gigabit ET Dual Port Server Adapter.

![NIC installed in Optiplex](screenshots/01_NIC_installed_in_optiplex.jpg)

The BIOS boot order was updated to boot from the OPNsense USB installer first.

![BIOS boot sequence](screenshots/02_BIOS_boot_sequence_USB_first.jpg)

The TL-SG108E switch and EAP225 AP were unboxed and staged.

![Switch and AP unboxed](screenshots/03_switch_and_AP_unboxed.jpg)

---

## Phase 2 — OPNsense Installation

OPNsense 26.1 was installed from USB using the installer account. ZFS with stripe configuration was selected to allow for future snapshot and RAID capabilities. The SanDisk SD8SB8U-256G (ada0, 256GB) was selected as the installation target. The existing Windows installation was wiped and OPNsense was written to the drive.

![OPNsense installer task selection](screenshots/05_OPNsense_installer_task_selection.webp)

![ZFS stripe mode selected](screenshots/06_ZFS_stripe_mode_selected.webp)

![Disk selection](screenshots/07_disk_selection_SanDisk_256GB.webp)

A secure root password was set during installation.

![Root password configuration](screenshots/08_root_password_configuration.webp)

---

## Phase 3 — Interface Assignment and IP Configuration

After installation, OPNsense defaulted to LAN=re0 and WAN=igb0, which was backwards from the intended design. Console option 1 was used to reassign:

- **WAN = re0** (onboard Realtek, connects to existing router)
- **LAN = igb0** (Intel 82576 port 1, connects to switch)
- **igb1** (Intel 82576 port 2) — unassigned, unused

The LAN IP was set to **10.0.0.1/24** via console option 2 to avoid a subnet conflict with the existing router's 192.168.1.x range. DHCP pool configured for 10.0.0.100 to 10.0.0.200.

![Interface assignment console](screenshots/10_interface_assignment_console.jpg)

![Final correct console state](screenshots/33_console_final_correct_state.jpg)

---

## Troubleshooting — Full Account

This section documents every issue encountered during setup and how each was resolved. Real-world troubleshooting is a core part of this project and demonstrates diagnostic methodology.

---

### Issue 1 — Web UI unreachable after interface reassignment

**Symptom:** After reassigning interfaces and reloading services, the OPNsense web UI at https://192.168.1.1 was unreachable. Devices connected to the AP received APIPA addresses (169.254.x.x), indicating no DHCP response from OPNsense.

![Browser cannot reach 192.168.1.1](screenshots/12_browser_cannot_reach_192.168.1.1.jpg)

![ipconfig showing APIPA address](screenshots/13_ipconfig_APIPA_address.jpg)

**Steps taken:**
- Reloaded all services via console option 11 — no change
- Disabled packet filter via `pfctl -d` in shell — still unreachable, ruling out firewall rules
- Plugged laptop directly into switch — also received APIPA, ruling out the AP

**Root cause:** Physical — see Issue 2 below.

---

### Issue 2 — Switch cable in wrong NIC port

**Symptom:** Running `ifconfig igb0` from the shell showed `status: no carrier`, meaning no physical link on igb0. Running `ifconfig igb1` showed `status: active`.

![ifconfig showing igb0 no carrier, igb1 active](screenshots/18_ifconfig_igb1_active_igb0_fixed.jpg)

**Root cause:** The Intel 82576 card has two physically adjacent ports (igb0 and igb1). The switch cable was plugged into igb1 instead of igb0, which was assigned as LAN.

**Resolution:** Cable moved from igb1 to igb0. Running `ifconfig igb0` again confirmed `status: active`. Web UI became accessible from the wired laptop.

![OPNsense web UI login page](screenshots/19_OPNsense_webUI_login_page.jpg)

---

### Issue 3 — Subnet conflict causing no internet

**Symptom:** After accessing the web UI and completing the setup wizard, wired devices had no internet access.

![Console showing subnet conflict](screenshots/26_console_subnet_conflict.jpg)

**Root cause:** Both WAN and LAN were on the 192.168.1.x/24 subnet. WAN received 192.168.1.119 from the existing router, and LAN defaulted to 192.168.1.1. OPNsense cannot route between two interfaces on the same subnet.

**Resolution:** LAN changed to 10.0.0.1/24 via Interfaces > LAN in the web UI.

![LAN interface changed to 10.0.0.1](screenshots/27_LAN_interface_changed_to_10.0.0.1.jpg)

---

### Issue 4 — DHCP server not updated after LAN subnet change

**Symptom:** After changing the LAN subnet, devices received APIPA addresses. Kea DHCPv4 (OPNsense 26.1's default DHCP server) was not enabled and had no subnets configured. Dnsmasq was still serving the old 192.168.1.x range, causing a conflict.

![dnsmasq old DHCP range](screenshots/32_dnsmasq_old_DHCP_range.jpg)

**Resolution:** Kea DHCPv4 enabled and bound to LAN. Subnet 10.0.0.0/24 added with pool 10.0.0.100–10.0.0.200. Conflicting dnsmasq DHCP range deleted.

![Kea DHCP enabled](screenshots/29_Kea_DHCP_enabled.jpg)

![Kea DHCP subnet configured](screenshots/31_Kea_DHCP_subnet_list.jpg)

---

### Issue 5 — AP not forwarding wireless traffic

**Symptom:** Even after fixing DHCP, wireless devices could not get IP addresses. Kea DHCP leases table was empty — DHCP requests from wireless clients were not reaching OPNsense at all.

**Root cause:** The EAP225 had a stale DHCP lease from the old 192.168.1.x subnet, placing it on a different subnet than the switch and OPNsense, breaking layer 2 bridging for wireless clients. Accessing the AP management page was not possible because the laptop (10.0.0.x) and AP (192.168.1.x) were on different subnets.

---

### Resolution — Factory Reset and Correct Startup Sequence

Both OPNsense (console option 4) and the EAP225 (8-second reset button hold) were factory reset. On the fresh OPNsense boot, the correct sequence was followed **before** accessing the web UI:

1. Console option 1 — reassign interfaces: WAN=re0, LAN=igb0
2. Console option 2 — set LAN IP to 10.0.0.1/24, DHCP pool 10.0.0.100–10.0.0.200
3. Only then connect devices and access the web UI

**Key lesson learned:** In a double NAT setup where the existing router uses 192.168.1.x, OPNsense's default LAN of 192.168.1.1 creates a subnet conflict that breaks routing and causes cascading DHCP and AP issues. Setting the LAN to a non-conflicting subnet from the console before touching anything else resolves all downstream problems. The EAP225 received a 10.0.0.x IP on reboot and wireless connectivity was immediately restored.

---

## Outcome

- OPNsense 26.1 deployed and running on Dell Optiplex 3040 SFF
- WAN: 192.168.1.119 (DHCP from existing router)
- LAN: 10.0.0.1/24 (static)
- DHCP serving 10.0.0.100–10.0.0.200 to client devices
- Wired and wireless internet access confirmed
- Web UI accessible at https://10.0.0.1
- Foundation in place for VLAN segmentation (Part 2) and Active Directory (Part 3)

---

## Next Steps

Still in progress

- [Part 2 — VLAN Segmentation and Multi-SSID Wireless Setup](../Part2-VLANs/README.md)
- [Part 3 — Windows Server 2022 Active Directory Domain](../Part3-ActiveDirectory/README.md)
