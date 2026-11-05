---
title: "Network segmentation & why printers are a weak spot"
series: Concepts
date: 2026-11-05
status: completed
tags: [concepts, networking, segmentation, subnet, vlan, firewall, iot, printers, lateral-movement, blue-team, pentesting]
related:
  - "[[THM - SOC Sim - Repossessed (First CTF)]]"
  - "[[Ch2 – Attack Targets on the Internet]]"
---

# Network segmentation & printer security

> [!abstract] One-line summary
> Put different kinds of devices in **separate subnets/VLANs** with **firewall rules between them**. Then if one device gets hacked (often a forgotten printer), the attacker is stuck in that segment and can't move freely to the rest of the network.

---

## 01. Why printers are a classic weak spot

A network printer is **a computer**. It has its own OS, a web admin panel, and services like FTP, SNMP and raw printing on port **9100**.

Common problems:
- **Firmware almost never gets updated.**
- **Default admin creds** (e.g. `admin/admin`) never get changed.
- **Nobody monitors them**, so an attacker can stay there unnoticed.
- They **store data**: copies of scanned/printed documents, and sometimes saved email or LDAP passwords for scan-to-email.

> [!note] Correction to my first assumption
> Printers aren't *the* most vulnerable devices. They belong to the same group as **IP cameras, smart TVs, VoIP phones and other IoT gear**, which all share the "never patched, never watched" problem. Printers are just a very common example in offices.

---

## 02. What segmentation is

- **Subnet:** a range of IPs that forms its own part of the network.
- **VLAN:** a way to split one physical network into separate logical networks. Each VLAN usually gets its own subnet.
- **Segmentation:** grouping devices by type or trust level, for example:

| Segment | Devices |
|---|---|
| Staff VLAN | Employee laptops/desktops |
| Server VLAN | File servers, domain controller, databases |
| Printer / IoT VLAN | Printers, cameras, smart devices |
| Guest VLAN | Visitors' Wi-Fi (internet only) |

### The part that actually protects you: firewall rules
A **firewall or router between segments** decides what can talk to what. For example:
- Laptops → printers **on the print port only**: allowed.
- Printers → laptops / servers (starting new connections): **blocked**.
- Guests → anything internal: **blocked**.

> [!important]
> The subnet alone isn't the protection. The **rules between subnets** are. If every segment can talk to every other one, splitting them does nothing (it's still a "flat" network).

### Why it works
If an attacker takes over a printer, they're **contained** in the printer segment and can't jump straight to the file server or domain controller. Moving from one machine to another inside a network is called **lateral movement**, and stopping it is the main point of segmentation. (I saw lateral movement in the Repossessed SOC sim.)

---

## 03. Defender checklist
- Put printers and IoT gear in their **own VLAN**.
- **Default-deny** between segments, then allow only what's needed.
- **Update firmware** regularly.
- **Change default passwords**, and turn off services that aren't used (FTP, Telnet, old SNMP versions).
- **Log and monitor** these devices too.

---

## 04. Pentester's view
- After getting a foothold, one of the first questions is: **how flat is the network?** What can I reach from where I landed?
- Printers and IoT devices are a favourite **first foothold** or pivot point.
- Example finding for a report:
  > *From the printer VLAN, the tester could reach the domain controller on SMB.*
- Only test networks I **own** or have **written permission** to test.

---

## Takeaways
- "Pot set" → the word is **subnet**; the idea is **network segmentation**.
- Printers are weak because they're unpatched, use default creds, nobody watches them, and they hold data.
- Segmentation + **firewall rules between segments** limit **lateral movement**.
