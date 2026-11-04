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

### `thm-writeups/`
TryHackMe room notes (concepts and approach — no flags or step-by-step answer
keys).
- **Passive Reconnaissance** — whois, dig/nslookup, CT logs, Shodan.
- **Active Reconnaissance** — ping, traceroute, telnet/netcat banner grabbing.
- **SOC Simulator: Repossessed (my first CTF)** — tracing a full intrusion
  through a SIEM: phishing → execution → C2 → credential access → lateral
  movement → collection → exfiltration, plus MITRE ATT&CK and case reporting.

### `lab-sessions/`
Notes from the hands-on cybersecurity course I'm taking alongside university.
- **L4 – Linux: SSH** — Kali VM on a bridged network, installing and reading
  `sshd_config` (what commented defaults mean), failing logins on purpose and
  finding them in `journalctl`, source ports vs the SSH port, and how a blue
  team spots and responds to SSH brute force.

### Projects
Hands-on builds that grew out of my studies (each in its own repo).
- **[Fedora–Android Voice Monitor](https://github.com/arm21n/fedora-android-voice-monitor)**
  — built while learning SSH in my cybersecurity course. A Python script reads
  my Fedora laptop's CPU temperature and, when it gets too hot, uses key-based
  SSH into Termux to make my Android phone speak a warning. Along the way I
  debugged a firewalld zone mismatch and set up non-interactive Ed25519 auth.

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
`firewalld` · `ssh`/`sshd_config`/Ed25519 keys · `journalctl` · VirtualBox/Kali · Termux · `mokutil`/Secure Boot · Shodan · TryHackMe · Obsidian (for the
original linked notes).

---

*Work in progress — I add to this as I learn.*
