---
title: "THM – Passive Reconnaissance"
platform: TryHackMe
room: https://tryhackme.com/room/passiverecon
module: Network Security
date: 2026-10-19
status: completed
tags: [tryhackme, recon, passive-recon, dns, whois, osint, shodan, chapter-2]
related:
  - "[[Ch2 - Attack Targets on the Internet - Practice Session]]"
---

# THM – Passive Reconnaissance

> [!abstract] One-line summary
> Gathering intelligence from **public sources only**, never touching the target. It's the stealthiest phase of recon: no alerts and minimal legal risk, but it often reveals forgotten subdomains, outdated services, and misconfigurations.

Builds directly on [[Ch2 - Attack Targets on the Internet - Practice Session]], where I already used whois, dig, nslookup, and Shodan on my own network.

---

## Passive vs active
| | Passive | Active |
|---|---|---|
| Contact with target | **None**, public data only | Direct (packets, connections) |
| Detectable? | No | Yes (logs, IDS, WAF) |
| Examples | WHOIS, DNS via public resolvers, crt.sh, Shodan | ping, traceroute, telnet, nmap, visiting the site |
| Authorization | Still needs to be **in scope** | **Signed authorization required** |

> [!question] Why is querying DNS "passive"?
> I ask a **public DNS server** (like `1.1.1.1`), not the target's own machines, so the target never sees my traffic. The same goes for WHOIS servers and Shodan, which did the scanning *earlier*.

---

## Tools & techniques

### 1. WHOIS: who registered the domain
- Shows the **registrar**, **creation/expiry dates**, and **name servers**.
- Most personal details are now **redacted** for privacy.
- I used it in Ch2: `whois ipconfig.me` → MarkMonitor, created 2013 → a defensive registration. And whois on IPs showed **owners** (CTC Astana, MSK-IX), not physical locations.

### 2. DNS lookups: record types
| Record | What it holds |
|---|---|
| **A** | IPv4 address |
| **AAAA** | IPv6 address (like Google's `2607:f8b0:...` in Ch2) |
| **MX** | Mail servers for the domain |
| **TXT** | Free-text records, used for **SPF**, **DMARC**, and domain-ownership verification |

- **SPF** (Sender Policy Framework): lists which servers may send email for the domain.
- **DMARC**: tells receivers what to do with mail failing SPF/DKIM checks (reject, quarantine, none). A weak or missing DMARC record = easier email spoofing and phishing.
- Recon value: MX and TXT records reveal **which email provider and third-party services** an organization uses.

### 3. Subdomain enumeration
- **DNSDumpster**: aggregates DNS data and draws a **graph** of a domain's infrastructure.
- **crt.sh**: searches **Certificate Transparency (CT) logs**. Every public SSL/TLS certificate is logged publicly, and certificates list the subdomains they cover. This is the **most effective passive way to find subdomains**.
  - Search `%.example.com` (`%` = wildcard).
- Why it matters: forgotten subdomains (old test servers, staging sites) are often unpatched and weakly protected.

### 4. Exposed services
- **Shodan**: device banners, open ports, hosting info, without me scanning anything.
- Ch2 example: my campus public IP showed **port 80, 403 Forbidden** (exposed router admin page), and the ISP gateway leaked `refid 10.0.0.1` over NTP.
- Alternative: **Censys** (similar internet-wide scan database).

---

## Command quick reference
| Purpose | Command |
|---|---|
| WHOIS record | `whois tryhackme.com` |
| A record (legacy) | `nslookup -type=A tryhackme.com` |
| MX at specific server (legacy) | `nslookup -type=MX tryhackme.com 1.1.1.1` |
| TXT record (legacy) | `nslookup -type=TXT tryhackme.com` |
| A record (**recommended**) | `dig tryhackme.com A` |
| MX at specific server (**recommended**) | `dig @1.1.1.1 tryhackme.com MX` |
| TXT record (**recommended**) | `dig tryhackme.com TXT` |
| Short output | add `+short` |
| Reverse lookup | `dig -x <IP> +short` |
| Passive subdomain discovery | browse **https://crt.sh** → search `%.tryhackme.com` |

`dig` is preferred over `nslookup`: more detail, scriptable. (`sudo dnf install bind-utils` on Fedora.)

---

## Tips & takeaways
- **Use DoH/DoT resolvers** (DNS over HTTPS / DNS over TLS, e.g. `1.1.1.1`) to keep my own queries private. Normal DNS is plaintext, like Telnet. (In Ch2, `resolvectl` showed `-DNSOverTLS` on my laptop, so it's currently **off**.)
- **Defender view:** monitor my own footprint by setting **Shodan/Censys alerts**, watching **CT logs** for unexpected new certificates (could mean someone issued a cert for my domain), and tracking **DNS changes** for **subdomain takeover** risks.
  - **Subdomain takeover:** a subdomain still points (via DNS) to a cloud service the owner deleted, so an attacker claims that service and controls the subdomain.
- **Passive ≠ automatically allowed:** the whole engagement must still be **authorized and in scope**.
- **Results change over time:** IPs rotate (e.g. **Cloudflare anycast**: the same IP served from many locations), subdomains come and go, and redactions increase. That's the same lesson as GeoDNS in Ch2, where Google gave me different IPs on different days and from different resolvers.

---

## Connections to my Ch2 session
| THM concept | Where I saw it already |
|---|---|
| whois | ISP / MSK-IX / Google ownership, typo domain registrar |
| dig with a specific resolver | `dig google.com @1.1.1.1` → different IPs (GeoDNS) |
| Shodan | my public IP, ISP gateway, MSK-IX "no info" |
| Results change over time | Google IPs changed between nslookup and dig |
| Plaintext DNS | `-DNSOverTLS` in resolvectl |

---

## Practice ideas
- [ ] `dig tryhackme.com MX` and `dig tryhackme.com TXT` → which email provider? Is there a DMARC record? (`dig _dmarc.tryhackme.com TXT`)
- [ ] crt.sh → `%.tryhackme.com`: how many subdomains? Any that look like test/staging?
- [ ] Turn on DNS over TLS on Fedora, then confirm `+DNSOverTLS` in `resolvectl status`
- [ ] Next room: **Active Reconnaissance**
