---
title: "THM – Active Reconnaissance"
platform: TryHackMe
room: https://tryhackme.com/room/activerecon
module: Network Security
date: 2026-10-21
status: completed
tags: [tryhackme, recon, active-recon, ping, traceroute, telnet, netcat, banner-grabbing, chapter-2]
related:
  - "[[Ch2 - Attack Targets on the Internet - Practice Session]]"
  - "[[THM - Passive Reconnaissance]]"
---

# THM – Active Reconnaissance

> [!abstract] One-line summary
> **Directly interacting** with the target: sending packets, making connections, probing ports. It reveals things passive recon can't (open ports, live services, software versions), but it's **detectable** and **requires authorization**.

Follows [[THM - Passive Reconnaissance]]. Many of these tools I already ran on my own network in [[Ch2 - Attack Targets on the Internet - Practice Session]].

> [!warning] Authorization
> Active recon makes contact with the target → it shows up in logs, IDS, and WAFs. **Never do it without signed legal authorization** (or your own lab / THM machines). This is the line I already respected when I reported the AITU router instead of poking it.

---

## The five core tools
| Tool | Job |
|---|---|
| **Web browser + DevTools** | server tech, headers, JS sources, certificate details, least suspicious |
| **ping** | is the host reachable? + TTL-based OS clue |
| **traceroute / mtr** | map the path, find intermediate routers and filtering points |
| **telnet** | connect to a port, grab its banner (legacy) |
| **netcat (nc)** | the flexible version of telnet, client or server |

---

## 1. Web browser + Developer Tools
- Most convenient and **least suspicious** active tool — looks like normal browsing.
- DevTools shortcut: **Ctrl+Shift+I** (Linux/Windows), **Option+Cmd+I** (macOS).
- Reveals: server technologies, request/response **headers**, **JavaScript source**, **certificate** details.

### Extensions that turn the browser into a recon tool
- **FoxyProxy**: switch between proxies (Burp Suite, ZAP, SOCKS5 tunnels) for intercepting/routing traffic.
- **User-Agent Switcher/Manager**: fake the User-Agent to look like mobile Safari or an old browser → find mobile-specific or version-specific endpoints. (Modern WAFs/CDNs flag rapid UA changes.)
- **Wappalyzer**: passively fingerprints the site's tech — CMS (WordPress), web server (Apache), OS (CentOS), analytics, CDN (Cloudflare), databases (MySQL). Runs as I browse.
- Alternatives: **BuiltWith**, **WhatRuns**, **Library Detector** (JS libraries).
- Even "just browsing" can trip behavioural rules: rapid page loads, modified headers, heavy DevTools use, odd User-Agents. **Goal: mimic legitimate user behaviour.**

---

## 2. ping
Confirms reachability + gives a TTL OS clue (my Ch2 [[Ch2 - Attack Targets on the Internet - Practice Session#10. Mission 5 – TTL fingerprinting|TTL fingerprinting]] note covers the details).

### Setting packet size with `-s`
```
ping -c 5 -s PACKET_SIZE MACHINE_IP
ping -c 5 -s 64 MACHINE_IP      # 64 bytes of data
man ping                         # look up -s to confirm
```
`-s` sets the **data bytes** in the ICMP echo request. (Recall from Ch2: Linux default is 56 data bytes → 64-byte packets.)

### Interpreting results
| Result | Most likely meaning | Next step |
|---|---|---|
| Fast replies, low/no loss | Target online, allows ICMP | Proceed to **port scanning** |
| "Destination Host Unreachable" | Target down or no route | Check if machine is powered on |
| 100% loss, **no error** | ICMP **filtered/blocked** | Try TCP/UDP host discovery with **Nmap** |
| High latency / heavy loss | Congestion, distance, or filtering | Investigate the path with **traceroute** |

> [!note] This table is a mini decision tree. "100% loss but no error" ≠ host is down — it often means a firewall is silently dropping ICMP, which is exactly why Nmap has TCP/UDP ping options (next rooms).

---

## 3. traceroute / mtr
- `traceroute` maps the path hop by hop (my Ch2 route went router → CTC Astana → Transtelecom → **MSK-IX Moscow** → Google).
- **mtr** = traceroute + ping combined, running continuously → **real-time path monitoring**, good for spotting where loss/latency starts.

---

## 4. telnet — banner grabbing
Connect to a port and read what the service announces.
```
telnet 10.113.133.62 80
Trying 10.113.133.62...
Connected to 10.113.133.62.
GET / HTTP/1.1
host: telnet
          ← blank line sends the request

HTTP/1.1 200 OK
Server: nginx/1.6.2      ← the prize
...
```
- **`Server: nginx/1.6.2`** reveals the web server **type and version** → cross-reference against **CVE** and **Exploit-DB** for known vulnerabilities. This is the core goal of active recon.
- **Banner grabbing** works on any TCP service:
  - HTTP → type HTTP commands (`GET / HTTP/1.1`)
  - Mail → SMTP / POP3 commands
  - **FTP (port 21)** → sends its banner immediately, no command needed
- Principle: **connect to the port, read the banner, optionally send protocol commands.**

> [!warning] Telnet can't do encryption
> For TLS-wrapped services use proper tools:
> - HTTPS (443): `curl --head https://10.113.133.62` or `openssl s_client -connect 10.113.133.62:443`
> - SMTPS (465) and other TLS services: `openssl s_client` or `ncat --ssl`
>
> (Ties back to [[Ch2 - Attack Targets on the Internet - Practice Session#21. Telnet, SSH, reverse shells|Telnet vs SSH]] — Telnet is plaintext.)

---

## 5. netcat (nc) — the swiss army knife
```
nc MACHINE_IP PORT_NUMBER     # client: connect / banner grab
nc -lvnp PORT_NUMBER          # server: listen on a port
```
- `-l` listen, `-v` verbose, `-n` no DNS, `-p` port.
- More flexible and secure than telnet; `ncat --ssl` handles TLS.
- As a listener it's also how **reverse shells** are caught (the attacker side of the concept from my Ch2 note).

---

## Quick reference (Task 7)
| Command | Example |
|---|---|
| ping (Linux/macOS) | `ping -c 10 10.113.157.184` |
| ping (Windows) | `ping -n 10 10.113.157.184` |
| ping (IPv6) | `ping -6 MACHINE_IPV6` or `ping6 MACHINE_IPV6` |
| traceroute (Linux/macOS) | `traceroute 10.113.157.184` |
| tracert (Windows) | `tracert 10.113.157.184` |
| traceroute (IPv6) | `traceroute -6 MACHINE_IPV6` or `traceroute6 MACHINE_IPV6` |
| mtr | `mtr 10.113.157.184` |
| telnet (legacy) | `telnet 10.113.157.184 PORT` |
| netcat client | `nc 10.113.157.184 PORT` |
| netcat server | `nc -lvnp PORT` |
| netcat (IPv6) | `nc -6 MACHINE_IPV6 PORT` |
| curl HTTP banner | `curl -I http://10.113.157.184` or `curl -I https://10.113.157.184` |
| DevTools | Ctrl+Shift+I (Linux/Win) · Option+Cmd+I (macOS) |

---

## Putting it together (a recon flow)
1. **ping** → is the host alive?
2. **traceroute** → understand the path and filtering points.
3. **nc / curl -I** → probe specific ports, grab banners, identify services + versions.
4. Prefer `curl -I` or `nc` over telnet for HTTP banner grabbing (safer, more flexible).
5. Cross-reference versions against CVE / Exploit-DB.

Then move on to automated scanners (**Nmap**), which extend host discovery and port scanning far beyond ping and nc.

---

## Connections to my earlier work
| THM concept | Where I saw it |
|---|---|
| ping reachability + TTL | Ch2 Missions 4–5 (700 ms mystery, TTL fingerprinting) |
| traceroute path mapping | Ch2 Mission 6 (route through Moscow) |
| `ping -s` packet size | Ch2 cheat sheet (Linux 56→64 bytes) |
| Telnet plaintext / banner | Ch2 §21; book's Shodan Telnet list |
| curl -I / headers | Ch2 Shodan showed my router's `Server:` header + `X-Detail` |
| Authorization before probing | Ch2 Mission 9 (reported AITU router, didn't touch it) |

---

## Practice ideas
- [ ] On a THM target: `ping` → `traceroute` → `nc <ip> 80` with `GET / HTTP/1.1` → read the `Server:` banner
- [ ] Grab an FTP banner (port 21) — no command needed, it greets you
- [ ] `curl -I` the same target and compare to the nc banner
- [ ] Install Wappalyzer, browse a site, note the detected stack
- [ ] Look up whatever `Server:` version you find on **CVE** / **Exploit-DB**
- [ ] Next rooms: **Nmap Live Host Discovery → Basic → Advanced → Post Port Scans**
