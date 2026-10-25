---
title: "Ch2 – Attack Targets on the Internet – Practice Session"
book: "How Cybersecurity Really Works – Sam Grapp"
chapter: 2
pages: 25-34
machine: Lenovo IdeaPad · Fedora + Windows 11 (dual boot)
network: University campus Wi-Fi
tags: [cybersecurity, networking, fedora, linux, shodan, hardening, secure-boot, subnetting, dns, traceroute, recon, chapter-2]
---

# Ch2 – Attack Targets on the Internet – Practice Session

> [!abstract] Summary
> Translated the book's Windows/macOS exercises to Fedora, then turned them into a full hands-on session: mapped my own network, learned subnetting, NAT, DNS/GeoDNS, IP geolocation, routing, TTL fingerprinting, audited every open port on my laptop, hardened Fedora **and** Windows, audited my firmware (found a leftover test certificate), enabled Secure Boot with my own signing key, looked myself up on Shodan, and sent a responsible-disclosure report about an exposed service on a network I use.

> [!warning] Ethics rule (applies to everything below)
> Only look at / test systems you **own** or have **written permission** to test. Viewing Shodan results is fine. **Connecting to, logging into, or trying passwords on** devices you find there is illegal unauthorized access, even if the password is printed on screen. **Authorization** is what separates a pentester from a criminal.
> Legal practice targets: `scanme.nmap.org`, TryHackMe, CTFs.

> [!note] Note on redaction
> Personal values (my public IP, MAC addresses, key fingerprints, exact subnet, the specific network/org) are replaced with placeholders like `<PUBLIC_IP>` or `203.0.113.x`. The techniques and lessons are unchanged.

---

## 1. Windows / macOS → Fedora command cheat sheet

| Task | Windows | macOS | Fedora |
|---|---|---|---|
| Show IP config | `ipconfig` | `ifconfig` | `ip addr` (`ip a`) |
| Full config | `ipconfig /all` | `ifconfig` | `ip addr` + `nmcli device show` |
| Default gateway | in `ipconfig` | `netstat -nr` | `ip route` → `default via …` |
| DNS servers / suffix | in `ipconfig` | `scutil --dns` | `resolvectl status` |
| Ping | `ping host` (stops after 4) | `ping -c 4 host` | `ping -c 4 host` (otherwise forever, Ctrl+C) |
| Trace route | `tracert` | `traceroute` | `traceroute` (`sudo dnf install traceroute`) or `tracepath` (preinstalled) |
| DNS lookup | `nslookup` | `nslookup`/`dig` | `nslookup` / `dig` (`sudo dnf install bind-utils`) |
| Listening ports | `netstat -an` | `netstat -an` | `ss -tuln` (listening) / `ss -tun` (active) / `sudo ss -tulnp` (with process) |
| ARP table | `arp -a` | `arp -a` | `ip neigh` |
| Routing table | `route print` | `netstat -nr` | `ip route` |

**Fedora notes**
- `ifconfig` exists on Fedora (`sudo dnf install net-tools`) but is **deprecated** → use `ip`.
- macOS `lo0` = Fedora `lo`; macOS `en0` (Wi-Fi) = Fedora `wlp*` (e.g. `wlp0s20f3`).
- Fedora interface names are based on hardware location (`wlp0s20f3`, `enp7s0`), not `eth0`/`wlan0`.
- Linux `ping` sends **56 data bytes → 64-byte replies** (56 + 8 header). Windows sends **32 bytes**. Linux shows lowercase `ttl`.
- Linux `traceroute` uses **UDP** by default; Windows `tracert` uses **ICMP**. If many `* * *`, try `sudo traceroute -I <ip>`. Add `-n` to skip hostname lookups (faster, cleaner).
- If `dnf` doesn't have a tool installed, it almost always has the package.

---

## 2. What the book covers (pp. 25–34)

**Exercise: Analyzing Your Network** — understand what you're posting to the internet; attackers use what you've unknowingly left visible. Use command-line tools to learn about your own network, then search **Shodan** ("a search engine for IP addresses") to see what services you leave open.

**Network command-line tools (Windows)**
- Open **CMD** → Command Prompt.
- `ipconfig` → IP address, **default gateway**, DNS info.
  - **Default gateway** = the first router your computer sends traffic to in order to leave your network. A router creates a single network devices can join.
  - IPv4 address = your **private** IP. Gateway also has a private IP. At home it's usually the ISP's modem/router.
- `ipconfig` doesn't help with Shodan → you need **public** IPs. Use **`nslookup google.com`** → public IPs attached to a domain. Output varies by location.
- `ping <ip>` → sends small packets, waits for replies → tells you whether you can communicate. Unreachable → packets shown as lost.
- `tracert` → shows **how** packets get there: each router (**hop**) on the path, using **TTL (Time to Live)**. Each router decrements TTL by 1; at 0 the packet "dies" and the last router sends a message back. Each hop gets **3 probes**.

**macOS**
- Terminal app. `ifconfig` = more detailed `ipconfig`. Main adapter usually **`en0`**.
- `traceroute` — 64 hops max, 52-byte packets, `*` = no reply.
- `nslookup`/`ping` nearly identical, but macOS pings **forever** → use **`-c`** (count).

**Using Shodan** (free account; limits downloads/reports)
- **Explore** tab → categories (Video Games/Minecraft, Default Password, Telnet).
- **Default Password** → systems Shodan verified use default creds (e.g. Cisco `cisco`/`cisco`).
- **Telnet** → systems allowing Telnet — remote admin control, **unencrypted** → dangerous.
- **Closing exposure:** identify the exposed device, then control the exposure (remove from network, vendor security recommendations, forums).

**Conclusion:** Understand how black hats find targets → defend better. **First step: know what information you're making public.**

---

## 3. Mission 1 – My network card

Commands: `ip addr`, `ip route`, `resolvectl status`

| Item | Value (redacted) |
|---|---|
| Interface | `wlp0s20f3` (Wi-Fi, `state UP`) |
| Private IP | `10.x.y.22` |
| Subnet | **/21** (`255.255.248.0`) |
| Gateway | `10.x.y.1` |
| DNS server | gateway (router does DNS) |
| Private? | ✅ yes – RFC 1918 range |

**Other interfaces**
- `lo` – loopback `127.0.0.1/8`, `::1/128` (machine talking to itself).
- `enp7s0` – Ethernet port, `NO-CARRIER`, `state DOWN` = no cable plugged in. Normal when on Wi-Fi.

> [!tip] MAC address randomization
> `link/ether <random>` ≠ `permaddr <real>`. Fedora shows Wi-Fi networks a **fake MAC** instead of the real hardware one → harder to track between networks. Privacy feature (and MAC spoofing is also used offensively).

> [!note] Big network
> A /21 (~2,046 hosts) usually means a big shared network (campus/dorm/office), not a home router.

> [!warning] Correction
> "Same first three numbers = same network" only works for **/24**. On a /21, IPs can differ in the 3rd octet and still be in the same network.

---

## 4. Subnetting

### Method (range)
1. **Find the interesting octet**: prefix = 8 + 8 + *n* → the octet where the prefix ends.
2. **Mask value** for that octet: *n* network bits → e.g. 4 bits = `11110000` = 240.
3. **Block size = 256 − mask value**. Cross-check: 2^(host bits in that octet). Block sizes are **always powers of 2**.
4. Find which block your number falls in. **Block ends at start + block size − 1.**

### Method (host count)
- Host bits = **32 − prefix**
- Total addresses = **2^(host bits)**
- **Usable = 2^(32 − prefix) − 2** (first = **network address**, last = **broadcast address**)

| Prefix | Mask | Host bits | Total | Usable |
|---|---|---|---|---|
| /24 | 255.255.255.0 | 8 | 256 | 254 |
| /22 | 255.255.252.0 | 10 | 1,024 | 1,022 |
| /21 | 255.255.248.0 | 11 | 2,048 | 2,046 |
| /20 | 255.255.240.0 | 12 | 4,096 | 4,094 |

### Worked problems
**`10.0.5.77/20`** → 20 = 8+8+**4** → `11110000` = 240 → block 16 → 5 is in 0–15
- Network `10.0.0.0`, broadcast `10.0.15.255`, usable `10.0.0.1`–`10.0.15.254`, **4,094 hosts**.
- *Lesson:* you **can't** know gateway/DNS from an IP alone.

**`172.20.100.9/22`** → 22 = 8+8+**6** → `11111100` = 252 → block 4 → 100 is in 100–103
- Network `172.20.100.0`, broadcast `172.20.103.255`, usable 2^10 − 2 = **1,022**.
- *Mistake 1:* block = 255 − 252 = 3 ❌ → correct: **256 − 252 = 4** (check: 8−6 = 2 host bits → 2² = 4).
- *Mistake 2:* range "100–104" ❌ → 104 starts the next block → **100–103**.

**Why it matters:** in pentests/CTFs you need to know how big a target network is.

---

## 5. Mission 2 – Private vs public IP, NAT, CGNAT

- `curl ifconfig.me` → **public IP** (different from the private one).
- Tip: curl output has no newline, so it gets glued to the prompt — normal.

### NAT (Network Address **Translation**)
> [!warning] Common misconception (corrected)
> "If an attacker learns my private IP I'm cooked; NAT hides it."
> Reality: **private IPs aren't secret.** Millions of networks use `192.168.x.x`. A private IP only means something inside the local network and **can't be reached from the internet at all.**

- **Why NAT exists:** IPv4 has only ~4.3 billion addresses — not enough. The **ISP** assigns the router **one public IP**; all devices behind it share it.
- Router keeps a **translation table** to deliver replies back.
- **Security side effect (not its purpose):** unsolicited inbound traffic gets dropped. **Port forwarding** and **UPnP** punch holes in this — exactly what Shodan finds.

### CGNAT (Carrier-Grade NAT)
- If public IP or early traceroute hops are in **`100.64.0.0/10`** → ISP uses CGNAT (you share a public IP with other customers).
- In my case: **no CGNAT** — hop 2 (ISP gateway) was in the same range as my public IP → the router holds a real public IP directly.

---

## 6. Mission 3 – DNS & GeoDNS

### nslookup (Fedora)
```
nslookup google.com
Server: 127.0.0.53   Address: 127.0.0.53#53
```
- `127.0.0.53` = Fedora's **local DNS stub** (systemd-resolved). Real upstream DNS is in `resolvectl status`.

### dig comparison
```
dig google.com +short           → one set of Google IPs
dig google.com @1.1.1.1 +short  → a DIFFERENT set of Google IPs
dig github.com +short           → a single IP
```
- A first-query timeout to `127.0.0.53` that retries and succeeds is fine; frequent timeouts = slow router DNS.

**Why different answers? → GeoDNS**
- Your normal DNS asks on behalf of a resolver in your country → Google picks servers best for that location.
- **Cloudflare (1.1.1.1)** asks from *its own* location and, for privacy, **doesn't tell Google where you are** → Google guesses based on Cloudflare's location.
- Different asker → different guess → different IPs. Both are real Google servers.
- Answers also **change over time**.

> [!tip] Pentest angle
> The same domain can show different infrastructure from different places. During recon, query from several resolvers to find servers you'd otherwise miss.

---

## 7. IP geolocation vs whois

> [!warning] whois ≠ physical location
> whois shows **who owns** the IP block and where they're **registered**. Google registers all its ranges under HQ in **Mountain View, California** — even servers physically in Finland or Singapore.

### Physics as a geolocation tool
- Light in fiber ≈ **200 km per millisecond**.
- A ~70 ms round trip can't reach a server >10,000 km away → latency gives a **hard upper limit** on distance.
- Example: a Google server pinged at ~70 ms from Central Asia resolved (via `curl ipinfo.io/<ip>`) to **Hamina, Finland** (~3,500 km, min RTT ≈ 35 ms) — database + physics agree.

### Better location tools
```
dig -x <ip> +short     → reverse DNS (IP → name); Google uses 1e100.net
curl ipinfo.io/<ip>    → geolocation database (still an estimate)
```
- **`1e100.net`** = Google's server domain (1e100 = 10^100 = a googol).
- Router names often hide city/airport codes (e.g. `dfw` = Dallas).

### Bogons
`curl ipinfo.io/<private-ip>` → `"bogon": true`
- **Bogon** = address that should never appear on the public internet (private RFC 1918, reserved, unassigned). No location possible — thousands of networks reuse it.

### Geolocating your own public IP
`curl ipinfo.io/<PUBLIC_IP>` → usually correct **city** and **ISP**; exact point is often the ISP's office/equipment or an arbitrary default.
- **False precision:** 4-decimal coordinates suggest ~11 m accuracy; real accuracy is often several km.
- **Privacy takeaway:** every website sees your public IP → learns roughly your **city + ISP**, not your home. IP → person needs ISP records (normally a court order).

---

## 8. Typosquatting (did it by accident)

- Typed `curl ipconfig.me` instead of `ifconfig.me` → `Moved Permanently. Redirecting to https://ifconfig.me/`.
- Someone **owns the typo domain**. That technique = **typosquatting**: attackers register misspellings (`gooogle.com`, `paypa1.com`) for phishing/malware.

**Was it dangerous? No.**
- curl only **downloaded and printed text**; nothing ran.
- It **didn't follow** the redirect (curl only follows with **`-L`**).

**When typosquatting hurts:** in a **browser** (fake login pages) and with **`curl … | bash`** (runs whatever the server sends). Never pipe a URL to bash without checking it.

**Investigation:** `whois ipconfig.me | grep -i -E "registrar|creation|org"` → registered through a brand-protection registrar → likely a **defensive registration** (owner bought the typo so attackers couldn't).

---

## 9. Mission 4 – Ping & diagnosing latency

### Reading ping stats (format: min / avg / max / mdev)
- **avg** = typical delay. **mdev** = how much times jump around (high = unstable → hurts calls/gaming more than plain slowness).
- One outlier (e.g. a single 52 ms reply among 4–10 ms ones) drags the average up. **Averages hide outliers** — look at min and individual lines.
- Healthy Wi-Fi router replies in **< ~10 ms**.

### My "slow ping" mystery, solved by layer
- ❌ Not my Wi-Fi (router ~5–10 ms)
- ❌ Not my ISP (first few hops single-digit ms)
- ✅ Route goes through a distant **Internet Exchange** → adds delay
- ✅ **Which Google server DNS hands you** decides the rest (near data center ~60 ms vs far one 200–550 ms)

---

## 10. Mission 5 – TTL fingerprinting

- Starting TTLs: **Linux = 64**, **Windows = 128**, many routers/network gear = **255**.
- TTL counts **down** by 1 per router → **higher remaining TTL = fewer hops = closer**.
- The TTL in a reply is set by the **responding device**, not you. `ttl=64` from a router → the router runs Linux-based firmware.

| Server | Received TTL | Likely start | Hops used |
|---|---|---|---|
| Router | 64 | 64 | 0 |
| Near server | 109 | 128 | 19 |
| Far server | 104 | 128 | 24 |

> [!info] Asymmetric routing
> Traceroute measures the path **you → server**. TTL in a reply measures **server → you**. These paths can differ. TTL hop counts = estimates, usually within 1–2 hops.

> [!tip] Pentest use
> Basic **OS fingerprinting**. A strong *hint*, not proof — admins can change default TTLs. Combine with other clues (port scans).

---

## 11. Mission 6 – Traceroute + whois route map

### Reading traceroute
- **Several IPs on one hop** = load balancing (parallel paths).
- **`* * *`** = router didn't answer. Common, not broken.
- First hop = your router; first public hop = ISP.

### whois each hop
```
whois <ip> | grep -i -E "netname|descr|country"
```
- Hop 1: `PRIVATE-ADDRESS-CBLK-RFC1918` (private, owned by nobody)
- Hop 2: my ISP
- Then a national backbone/transit provider
- Then a distant **Internet Exchange (IX)** — a meeting point where networks swap traffic. Traffic from my region passed through a far IX to reach Google → the big latency jump.

### Two traces side by side
- Same first ~6 hops (router → ISP → backbone → IX → into Google).
- **Split inside Google's network:** one path stays ~60 ms, the other jumps to 150+ ms at a specific hop → I could point to the **exact hop** where extra latency starts.

---

## 12. Mission 7 – What my laptop exposes

Commands: `ss -tuln`, `sudo ss -tulnp` (adds process), `sudo firewall-cmd --list-all`

### Address patterns
- `127.x.x.x` / `::1` / `[::ffff:127.0.0.1]` → **loopback, local only**
- `0.0.0.0` / `*` / `[::]` → **all interfaces, reachable from the network**
- `224.x`–`239.x` / `ff02::` → **multicast groups** (not an open door)

### Port audit (before hardening)
| Port | Owner | Exposure | What it is |
|---|---|---|---|
| 53 (127.0.0.53/54) | systemd-resolved | 🟢 local | DNS stub |
| 323 (127.0.0.1) | chronyd | 🟢 local | clock sync |
| 631 (127.0.0.1) | cupsd | 🟢 local | printing |
| **27500** | **passimd** | 🟡 network | firmware cache sharing |
| 5355 | systemd-resolved | 🟡 network | **LLMNR** |
| 5353 | avahi-daemon | 🟡 network | **mDNS** |
| 3702 + others | wsdd | 🟡 network | **WS-Discovery** |
| high UDP | firefox | client | browser traffic |

> [!note] Guess vs reality
> I guessed a high UDP port was a "discovery service" — it was actually **Firefox**. **Verify, don't trust the port number.** Any program can use any port.

### How to identify ports
1. **Well-known ports** (22 SSH, 23 Telnet, 53 DNS, 80/443 web, 123 NTP, 631 printing). `grep -w -E "631|5353" /etc/services`. Official list: IANA registry.
2. **Clues from earlier output** (connecting outputs = core recon skill).
3. **Address patterns**, **distro defaults**.
4. **Verify:** `sudo ss -tulnp` (process), `sudo lsof -i :<port>`.

### Initial firewall state
```
FedoraWorkstation (default)
ports: 1025-65535/udp 1025-65535/tcp   ← !!
```
> [!danger] Fedora Workstation's default zone allows **every port above 1024**
> So the firewall did **not** block the network-facing services above. On a big shared network, thousands of devices could reach them.

---

## 13. Deep dive – passim & firmware

### What is passim?
- Small **local caching server** shipped with Fedora alongside **fwupd** (firmware updater). Shares downloaded firmware from **LVFS** with other computers on the LAN.
- **Do I need it? No** — one laptop; fwupd downloads directly. On a shared network you'd be a mini file server for strangers.
- **Close passim, keep fwupd.**

### Verifying an unknown service is legit
`rpm -qi passim` → **Signature** (Fedora's key), **Packager/Vendor: Fedora Project**, official GitHub source.
`systemctl status passim` → runs as its own user (`passim`, not root = **least privilege**); unit is **`static`** (no enable/disable → use **mask**).

> [!tip] Lesson
> **Signature + vendor + source** = how to tell a real system service from malware pretending to be one.

### Closing it (attack surface reduction)
```
sudo systemctl mask --now passim
→ Created symlink '/etc/systemd/system/passim.service' → '/dev/null'.
```
- **Masking** = service file replaced by a link to `/dev/null` → systemd can't start it. Undo: `sudo systemctl unmask passim`.
- **"Don't need it" + "exposed" = close it.** "Harmless today" ≠ "no vulnerability tomorrow".

---

## 14. Hardening Fedora – LLMNR, avahi, passim

### LLMNR – "Hey everyone, who is X?"
- When DNS can't find a name, the computer **shouts to the whole LAN**.
- **Anyone can answer "that's me!"** → attacker tool **Responder** answers everything → victim may hand over login info; on Windows often an **NTLM hash** (crackable).
```
sudo mkdir -p /etc/systemd/resolved.conf.d
printf '[Resolve]\nLLMNR=no\n' | sudo tee /etc/systemd/resolved.conf.d/no-llmnr.conf
sudo systemctl restart systemd-resolved
```

### avahi (mDNS) – "Hey everyone, I'm here!"
- Announces the laptop's name + services on every network. Risk: device fingerprinting for anyone on the same Wi-Fi.
```
sudo systemctl disable --now avahi-daemon.socket avahi-daemon.service
```
- Both **socket and service**, because the socket can wake the service. Turn on when needed: `sudo systemctl start avahi-daemon`.

### Verification
- Ports 27500 / 5355 / 5353 gone; `resolvectl` shows **`-LLMNR`**; scopes `DNS` only.
- systemd-resolved PID changed → proof it restarted and loaded the setting.
- **4 exposed services → 0.** Full cycle: **discover → identify → decide → harden → verify.**

---

## 15. Finding devices on the network

- **mDNS devices:** `sudo systemctl start avahi-daemon` → `avahi-browse -a -t`. Only finds devices that **choose** to announce. Quiet devices need **active scanning (nmap)** — only on networks you own / have permission for.
- **Windows** usually won't show in avahi-browse — it uses **WS-Discovery** (wsdd), **NetBIOS**, **LLMNR**.
- Find Windows shares from Fedora: Files → Other Locations → Windows Network, or `smbclient -L //<windows-ip> -N`.

> [!warning] Placeholder lesson
> Typed `<windows-ip>` literally → `bash: windows-ip: No such file or directory`. `<` = "read input from a file" in bash. **Replace the whole `<placeholder>`, brackets included**, with a real value.

---

## 16. Firmware audit & Secure Boot

### A leftover test certificate in firmware
`fwupdmgr get-devices` showed a certificate (`CRT` in the GUID) named like a test sample, issued by **"Root Agency"** (the default issuer of Microsoft's old **makecert** test tool). → Likely an engineer's test cert that **shipped by mistake**.

> [!danger] PKfail (2024 context)
> Researchers found many laptops shipped with test platform keys labeled "DO NOT TRUST", weakening Secure Boot. My **Platform Key was a legit vendor key** → not affected. One questionable cert in `db` is a weakness but not an emergency → enable Secure Boot anyway (far better than none), check vendor for a BIOS update, don't edit `db` yourself.

Key Secure Boot concepts:
- **PK / KEK / db / dbx** = Platform Key / Key Exchange Key / allowed certs / blocklist.
- **dbx** blocks known-bad bootloaders (stops **bootkits**).
- **Secure Boot off** = laptop boots **anything** → bootkits + **evil maid** attacks.

### Enabling Secure Boot with NVIDIA proprietary drivers
Unsigned NVIDIA modules won't load with Secure Boot on. Fedora's **akmods** signs them with *your own* key (**MOK — Machine Owner Key**); you enroll that key so firmware trusts it.

```
sudo ls /etc/pki/akmods/certs/                      # find public_key.der
modinfo nvidia | grep -i -E "signer|sig_key"        # confirm the driver is signed with it
sudo mokutil --test-key /etc/pki/akmods/certs/public_key.der   # enrolled?
sudo mokutil --import /etc/pki/akmods/certs/public_key.der     # set a one-time password
sudo reboot
# blue MOK Manager screen → Enroll MOK → Continue → Yes → password → Reboot
# then BIOS → Security → Secure Boot → Enabled → save
mokutil --sb-state                                   # → SecureBoot enabled
lsmod | grep nvidia                                  # still loaded ✅
```

> [!tip] Verify the right key
> The pending key's **serial number** must match the driver's **sig_key** fingerprint — confirms you're trusting *that* key, not just *a* key.
> MOK Manager uses a **US keyboard layout** → use a Latin-only password.
> **Fallback if the display breaks:** BIOS → Secure Boot Disabled → back to before (no brick).

---

## 17. Windows side – BitLocker & LLMNR

### BitLocker check before enabling Secure Boot
- Many Windows 11 laptops have **Device Encryption (BitLocker)** on by default; it watches Secure Boot → may demand a **48-digit recovery key** after changes.
- Check `https://account.microsoft.com/devices/recoverykey`, and in **admin** PowerShell: `manage-bde -status C:`.
- If **Protection On**: save the key (`manage-bde -protectors -get C:`), then `Suspend-BitLocker -MountPoint "C:" -RebootCount 1` before enabling Secure Boot.
- (In my case BitLocker was off → safe to proceed.)

> [!warning] Unencrypted Windows disk
> If the disk isn't encrypted, a thief can read files by booting a USB stick. → future task.

### LLMNR off on Windows (where Responder does the real damage)
```powershell
New-Item -Path "HKLM:\Software\Policies\Microsoft\Windows NT\DNSClient" -Force
Set-ItemProperty -Path "HKLM:\Software\Policies\Microsoft\Windows NT\DNSClient" -Name EnableMulticast -Value 0 -Type DWord
```
→ `EnableMulticast : 0`. (Windows has LLMNR **on by default**. **NetBIOS** is a separate future task.)
- Note: admin PowerShell is required — "access denied" without it is Windows' version of forgetting `sudo`.

---

## 18. Mission 8 – Shodan (looking myself up)

- **My public IP** showed a **web service on port 80** returning `403 Forbidden` → likely a router admin page exposed to the internet. 403 = refusing, but it still advertises an admin interface over plain HTTP → one firmware bug from access (how botnets like **Mirai** spread).
- **An upstream gateway** answered **NTP (123/UDP)** and leaked an internal `refid` address → **information disclosure**; open NTP has been abused for **amplification DDoS**.
- A locked-down router showed **"No information available"** → absence of evidence ≠ evidence of absence.

---

## 19. Mission 9 – Router exposure → responsible disclosure

- The router / public IP belonged to a **network I use but don't own** (a campus network), not my own device.
- **Rule: don't touch it** — no logging in, no default passwords, no "just checking."

### Why an internet-exposed router admin page is dangerous
- Router = **gatekeeper for everyone** on the network.
- Bots scan the whole internet constantly → probed within hours.
- Ways in: **weak/default passwords**, **firmware bugs** that skip login, **plain HTTP** (password readable).
- With the router, an attacker could change **DNS** (send everyone to fake sites), **watch/redirect traffic**, **open doors inward** (NAT protection gone), or make it a **DDoS bot**.

### Responsible disclosure (what I did)
- Reported it to the network's **IT/helpdesk**, not managers. Checked for a `/.well-known/security.txt` first (standard place for a security contact).
- The report stated the finding, the source (public Shodan record), suggested fixes (disable remote management/WAN access, update firmware, non-default password), and explicitly: **"I have not tried to access or log into the device. I only viewed the public Shodan record."**

---

## 20. Firewall zones

```
nmcli -t -f NAME,DEVICE connection show --active
sudo nmcli connection modify "<connection name>" connection.zone public
sudo nmcli connection up "<connection name>"
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --zone=public --list-all
```
- **Quotes matter** if the connection name has a space.
- Public zone: `ports:` empty → the 1025-65535 range no longer open to the shared network.

### What this accomplished
- **Before:** any program opening a port 1025–65535 was reachable by everyone on the shared network.
- **After:** the firewall **blocks all incoming** except a short service list.
- **Two kinds of protection:** closing services = **locking individual doors**; firewall zone = **a fence around the house**. Doors can reopen (updates, new apps).
- **Unchanged:** outgoing traffic works (**stateful firewall** remembers connections you start); the setting is **per connection**, so other networks keep their own.

### Defense in depth: remove ssh from the public zone
```
sudo firewall-cmd --zone=public --remove-service=ssh --permanent
sudo firewall-cmd --reload
```

---

## 21. Telnet, SSH, reverse shells

- **Telnet** (~1969, **port 23**): remote shell, **everything plain text**. Still on old routers, cheap IoT, industrial gear, often with **default passwords**. **Mirai** spread via Telnet.
- **SSH** (**port 22**) replaced it: **encrypts everything**, supports **key** login.
- Pentester view: Telnet open in a scan = red flag.

**Can anyone SSH into my laptop?** No.
- **Real reason:** no SSH server running (nothing listening on 22). Check: `systemctl is-active sshd`.
- **Firewalls block incoming, not outgoing.** Malware could connect *out* to an attacker → **reverse shell** → the human layer (not running untrusted downloads) matters.

---

## 22. Threat model – what's protected now

> [!quote] Nothing is ever fully "safe" — you **reduce specific risks**. Security = layers.

### ✅ Protected / much harder
LLMNR poisoning (off on both OSes) · attacks on my network services (passim/avahi/wsdd closed) · accidentally exposed future ports (public firewall zone) · mDNS fingerprinting (avahi off) · bootkits (Secure Boot on) · known firmware bugs (up to date).

### ⚠️ Still exposed
- **Physical (biggest gap):** no BIOS password → someone can disable Secure Boot; unencrypted disk → USB-boot file theft.
- **Network:** ARP spoofing / MITM on shared Wi-Fi (HTTPS protects most traffic); compromised router → DNS redirect (**never click past HTTPS cert warnings**); NetBIOS still on in Windows.
- **Human:** phishing, typosquatting, `curl | bash`, weak/reused passwords, no 2FA.
- **Software bugs:** keep updating.

---

## 23. Mistakes & debugging lessons

| Mistake | Lesson |
|---|---|
| `ping -c <ip>` (no count) | `-c` needs a **number**: `ping -c 4 <target>` |
| `who is <ip>` | ran the `who` command. One space changes everything |
| `whois <ip> grep …` (missing `\|`) | grep args sent to the whois server |
| `dig -x <typo-ip>` | typo → empty result |
| `curl ipconfig.me` | typosquatting — read commands before Enter |
| Running a failing command twice unchanged | **read the error, change something** |
| PowerShell not as admin | Windows' "forgot sudo" |
| Thought private IP = secret | meaningless outside the LAN |
| Thought whois = server location | whois = owner's registration |
| Thought ping TTL = my OS | reply TTL = **responder's** OS |
| Thought higher TTL = farther | TTL counts **down** → higher = closer |
| 255 − 252 = block size | **256** − mask |

**General habits:** when output looks weird, **re-read the command first** · verify, don't trust guesses · predict first, then test · averages hide outliers.

---

## 24. Glossary
Private IP / RFC 1918 · Public IP · Default gateway · Subnet mask / prefix · Network / broadcast address · NAT · CGNAT · Bogon · Loopback · Multicast · MAC randomization · DNS stub · GeoDNS · Reverse DNS · whois · ASN · Internet Exchange (IX) · Hop · TTL · Asymmetric routing · Load balancing · OS fingerprinting · Latency / RTT / mdev · Typosquatting · LLMNR · NetBIOS · mDNS / Avahi · WS-Discovery / wsdd · Responder · NTLM hash · Attack surface reduction · Least privilege · Defense in depth · systemd mask · Firmware / fwupd / LVFS · Secure Boot · PK / KEK / db / dbx · PKfail · shim · MOK · akmods · Bootkit / evil maid · BitLocker · Stateful firewall · Telnet / SSH · Reverse shell · Information disclosure · NTP amplification · Mirai · ARP spoofing / MITM · Responsible disclosure / security.txt.

*(Full definitions kept in my private notes.)*

---

## 25. Recap questions
1. Difference between a private and public IP, and why we need both?
2. What does the default gateway do?
3. How does traceroute use TTL to discover hops?
4. Why is Telnet dangerous, and what replaced it?
5. Why does defense start with "knowing what attackers can see"?
