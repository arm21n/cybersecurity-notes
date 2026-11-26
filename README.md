# Cybersecurity Learning Notes

My notes and write-ups as I learn cybersecurity, working through the book
*How Cybersecurity Really Works* (Sam Grubb) and rooms on TryHackMe. I'm a
first-year student working toward penetration testing, and I keep these to
review what I've learned and to track my progress.

Everything here is written in my own words from hands-on practice — translating
the book's Windows/macOS exercises to my Fedora Linux setup, then turning each
one into a practical mission.

## Contents

### `book-notes/`
Notes from *How Cybersecurity Really Works*.
- **Ch2 – Attack Targets on the Internet** — networking fundamentals through
  practice: subnetting, NAT/CGNAT, DNS & GeoDNS, IP geolocation vs whois,
  traceroute, TTL fingerprinting, auditing open ports, hardening Fedora and
  Windows, Secure Boot with a self-enrolled key, Shodan, and a responsible
  disclosure.
- **Ch3 – Phishing Tactics** — analyzing a real phishing email (fake Brazilian
  bank "points expiring today" scam): reading headers with MX Toolbox,
  SPF/DKIM/DMARC and alignment, decoding a base64 body on Fedora, VirusTotal
  (and why 0 detections ≠ safe), tracing the sender to a rented cloud VPS, and
  the social-engineering tricks used.

### `thm-writeups/`
TryHackMe room notes (concepts and approach — no flags or step-by-step answer
keys).
- **Passive Reconnaissance** — whois, dig/nslookup, CT logs, Shodan.
- **Active Reconnaissance** — ping, traceroute, telnet/netcat banner grabbing.
- **SOC Simulator: Repossessed (my first CTF)** — tracing a full intrusion
  through a SIEM: phishing → execution → C2 → credential access → lateral
  movement → collection → exfiltration, plus MITRE ATT&CK and case reporting.
- **CyberHeroes** — a login page that checks credentials in client-side
  JavaScript; finding them in DevTools and why that's broken.
- **Neighbour** — an IDOR: changing a username in the URL to view the admin's
  page, and why being logged in isn't the same as being allowed.
- **Corridor** — the same IDOR idea with the IDs hashed: the doors are
  `md5(1..13)`, so hashing the missing value (0) reaches an unlinked page.
- **Linux Fundamentals Part 3** — moving files (`wget`, `scp`,
  `python3 -m http.server`), processes and signals, systemd/`systemctl`,
  background jobs, cron, apt repositories with GPG keys, and `/var/log`.
  The `http.server` task became my HTTP Request Logger project.

### `lab-sessions/`
Notes from the hands-on cybersecurity course I'm taking alongside university.
- **L4 – Linux: SSH** — Kali VM on a bridged network, installing and reading
  `sshd_config` (what commented defaults mean), failing logins on purpose and
  finding them in `journalctl`, source ports vs the SSH port, and how a blue
  team spots and responds to SSH brute force.
- **L9 – Packet Tracer: build and fix a company network** — building an office
  LAN in Cisco Packet Tracer (switch, 4 PCs, server), static IPv4 and subnet
  masks, ping/ARP, hosting an internal site over HTTP, DNS A records and
  `nslookup`, then troubleshooting a broken DNS record (works by IP, not by
  name).

### `concepts/`
Short explanations of security ideas I've picked up.
- **Network segmentation & printer security** — why printers and other IoT
  devices are common weak spots, subnets/VLANs, firewall rules between
  segments, limiting lateral movement, and how a pentester checks for a flat
  network.

### `tool-notes/`
Short comparisons of everyday tools.
- **Process monitors & cross-platform shells** — htop vs Windows Task Manager
  vs Xfce Task Manager (and why htop is what you'll have over SSH on a
  target), PowerShell 7 (`pwsh`) on Kali, and running Linux on Windows with
  WSL.

### Projects
Hands-on builds that grew out of my studies (each in its own repo).
- **[Fedora–Android Voice Monitor](https://github.com/arm21n/fedora-android-voice-monitor)**
  — built while learning SSH in my cybersecurity course. A Python script reads
  my Fedora laptop's CPU temperature and, when it gets too hot, uses key-based
  SSH into Termux to make my Android phone speak a warning. Along the way I
  debugged a firewalld zone mismatch and set up non-interactive Ed25519 auth.
- **[caesar-ctf](https://github.com/arm21n/caesar-ctf)** — grew out of a
  Caesar cipher problem on my programming end-term prep. A C++ toolkit: an
  encrypter, a brute-force cracker (all 25 keys), an automatic cracker using
  letter-frequency analysis (and where it fails on short messages), and a
  CTF-style challenge served over the network with `ncat`, which I attacked
  from my phone in Termux to capture the flag. Along the way I found that
  Fedora's default firewall zone leaves ports 1025–65535 open.
- **[HTTP Request Logger](https://github.com/arm21n/http-request-logger)**
  — built after TryHackMe's *Linux Fundamentals Part 3*. A Python web server
  (extending `http.server`) that logs every request's IP, path and headers. I
  compared curl, wget2, Firefox and my Android phone: why the User-Agent can't
  be trusted, how header count and `Sec-Fetch-*` headers still separate scripts
  from humans, what `Accept-Language` leaks, the wget2 IPv6 fallback
  (`errno=111`), and why ports below 1024 need root.

## A note on privacy & ethics

- Personal values (my public IP, MAC addresses, key fingerprints, exact subnet,
  and the specific network/organization) are redacted or replaced with
  placeholders like `<PUBLIC_IP>`. The techniques and lessons are intact.
- The TryHackMe notes cover **ideas and methodology only** — no flag values or
  full walkthroughs.
- Core principle throughout: only test systems you **own** or have **written
  permission** to test. Legal practice targets like `scanme.nmap.org`,
  TryHackMe, and CTFs are where the hands-on work happens.

## Tools I'm learning with

Fedora Linux · `ip`, `ss`, `dig`, `nslookup`, `whois`, `traceroute`, `nmap` ·
`firewalld` · `htop` · `pwsh` · WSL · `ssh`/`sshd_config`/Ed25519 keys · `journalctl` · VirtualBox/Kali · Termux · `mokutil`/Secure Boot · Shodan · MX Toolbox · VirusTotal · `base64` · C++/`g++` · `ncat`/`nc` · `curl`/`wget` · `scp` · `python3 -m http.server` · `tail -F` · TryHackMe · Obsidian (for the
original linked notes).

---

*Work in progress — I add to this as I learn.*
