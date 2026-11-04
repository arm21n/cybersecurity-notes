---
title: "L4 – Linux: SSH (setup, config, log investigation)"
series: Lab sessions
lesson: 4
date: 2026-11-04
status: completed
tags: [lab-session, linux, ssh, sshd_config, journalctl, auth-log, brute-force, blue-team, kali, virtualbox]
related:
  - "[[THM - Active Reconnaissance]]"
  - "[[Fedora–Android Voice Monitor]]"
---

# L4 – Linux: SSH

> [!abstract] One-line summary
> I put my Kali VM on the real home network, ran an SSH server on it, read its config, then **failed a login on purpose from my Fedora host and found myself in the logs**. SSH encrypts remote access, and it also **records every attempt**. Those records are what a defender watches.

**Setup:** Fedora host → Kali Linux VM in VirtualBox (Bridged Adapter on my Wi-Fi interface).

---

## 01. VM networking + SSH server

### NAT vs Bridged
- **NAT** (default): Kali hides behind the host's IP, so other devices can't reach it directly.
- **Bridged**: Kali gets its **own IP on the home network**, like a separate device. That's what lets my Fedora host SSH into it.
- VirtualBox: Settings → Network → Adapter 1 → *Attached to: Bridged Adapter* → pick the real Wi-Fi interface (`wlp…`).
- (On KVM / virt-manager, bridging over Wi-Fi usually doesn't work. VirtualBox handles it fine.)

### Find the IP
```bash
ip a          # look at eth0's "inet" line, NOT 127.0.0.1 (loopback)
```
My Kali got `<KALI_IP>/21`.

### Install + start the server
```bash
sudo apt install openssh-server -y
sudo systemctl start ssh      # run now
sudo systemctl enable ssh     # run at every boot
sudo systemctl status ssh     # want: active (running) + enabled
```
`Server listening on 0.0.0.0 port 22` = listening on all interfaces, port 22.

### Why SSH?
SSH gives **secure remote access to a server's command line** over the network. That matters because most servers have no monitor or keyboard. Older tools like **Telnet** sent everything, passwords included, in **plaintext**, so anyone sniffing the traffic could read it. SSH **encrypts the whole session**. It also supports **key-based auth**: the private key never leaves my machine, so it's much harder to steal or guess than a password.

---

## 02. Reading `/etc/ssh/sshd_config`

Filter only the lines I care about instead of scrolling the whole file:
```bash
sudo grep -nE "^\s*#?\s*(Port|PermitRootLogin|PasswordAuthentication|PermitEmptyPasswords)\b" /etc/ssh/sshd_config
```
Output on my Kali:
```
14:#Port 22
34:#PermitRootLogin prohibit-password
57:#PasswordAuthentication yes
58:#PermitEmptyPasswords no
```

> [!important] Commented lines = defaults
> A line starting with `#` is **not active**. SSH uses its **built-in default**, and the commented line just shows what that default is. A commented `#Port 22` does **not** mean "any port". It means "the default port, 22". Remove the `#` to set a value explicitly (e.g. `Port 2222`).

| Directive | Value | Meaning |
|---|---|---|
| `Port` | 22 (default) | The port SSH listens on. |
| `PermitRootLogin` | `prohibit-password` | root may log in **only with a key**, never a password. Blocks bots guessing root's password. |
| `PasswordAuthentication` | `yes` | Passwords are **allowed** (not required, since keys still work). `no` = key-only login. |
| `PermitEmptyPasswords` | `no` | Accounts with an empty password can't log in over SSH. |

---

## 03. Fail on purpose, then find myself in the logs

From Fedora:
```bash
ssh kali@<KALI_IP>     # wrong password ×3, then correct password, then exit
```
After the 3rd wrong password: `Permission denied (publickey,password)`. The server accepts a key or a password, and neither worked.

### Where are the logs?
Recent Kali **doesn't write `/var/log/auth.log` by default** (no rsyslog), so I went through **journald**:
```bash
sudo journalctl -u ssh
sudo journalctl -u ssh --since "10 min ago" --no-pager | grep -E "Failed|Accepted"
```

### What I found (IPs redacted)
```
sshd-session[3011]: Failed password for kali from <HOST_IP> port 36204 ssh2
sshd-session[3011]: Failed password for kali from <HOST_IP> port 36204 ssh2
sshd-session[3011]: Connection closed by authenticating user kali <HOST_IP> port 36204 [preauth]
sshd-session[3011]: PAM 2 more authentication failures; ... rhost=<HOST_IP> user=kali
sshd-session[3769]: Accepted password for kali from <HOST_IP> port 57198 ssh2
sshd-session[3769]: pam_unix(sshd:session): session opened for user kali(uid=1000)
```

Reading a `Failed password` line: **time · username tried · source IP · source port · protocol (ssh2)**.

- The IP is my **Fedora host**, because that's where I connected from. Every attempt is logged with its source.
- Host and VM are on the same network: the `/21` mask covers a whole block of 8 /24s, so a different third octet doesn't mean a different network.
- `[preauth]` = SSH hung up **before** authentication succeeded, because I hit the attempt limit.
- On the next login, the banner said `Last login: ... from <HOST_IP>`. That's the same fact from the other side.

> [!warning] The port in the log is NOT the SSH port
> `port 36204` / `port 57198` are **random source ports** my SSH client opened on Fedora, one per connection (that's why they differ). The server side is still port 22.

---

## Blue-team view: spotting brute force

**Scenario:** 500 `Failed password` lines from one IP within a minute. That's a **brute-force / password-guessing attack**: an automated tool trying passwords.

What I'd do:
1. **Block the IP now** with a firewall rule, or **fail2ban**, which reads these same logs and bans an IP automatically after N failures.
2. Check whether any `Accepted password` line came from **that same IP**. If it did, the password has already been guessed → treat it as a compromise.
3. Set up **SSH keys first**, *then* set `PasswordAuthentication no`. Doing it in the other order locks me out too.

---

## Connected project
While studying SSH I built **[Fedora–Android Voice Monitor](https://github.com/arm21n/fedora-android-voice-monitor)**. It's a Python script on Fedora that reads the CPU temperature and, when it gets too hot, SSHes into Termux on my phone to make it speak a warning. It puts this lesson into practice:
- **Key-only, non-interactive auth**: a dedicated Ed25519 key + `BatchMode=yes`.
- **Non-default port**: Termux's sshd listens on **8022**, the `Port` directive in action.
- **Debugging**: the connection first failed because my Wi-Fi interface was in the `public` firewalld zone while SSH was allowed in a different zone.

---

## Takeaways
- Bridged networking gives a VM a real address on the LAN. NAT hides it.
- In `sshd_config`, `#` = default. Know the defaults before changing anything.
- SSH logs = **who tried, as whom, from where, and whether it worked**. That's the core evidence for detecting brute force.
- The strongest setup is **keys on, passwords off**, but only after keys work.

## Commands cheat sheet
```bash
ip a                                   # find my IP
sudo systemctl {start|enable|status} ssh
sudo grep -nE "<pattern>" /etc/ssh/sshd_config
ssh user@<IP>                          # connect (-p <port> for non-22)
sudo journalctl -u ssh                 # SSH logs on systemd distros
sudo tail -50 /var/log/auth.log        # if rsyslog is installed
```
