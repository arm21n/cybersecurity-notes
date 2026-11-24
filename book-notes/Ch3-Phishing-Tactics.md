---
title: "Ch3 – Phishing Tactics – Analyzing a Phishing Email"
book: "How Cybersecurity Really Works – Sam Grubb"
chapter: 3
pages: 43-53
machine: Lenovo Legion · Fedora Linux
sample: "phishing_pot (rf-peixoto/phishing_pot) – real phishing email, Sept 2023"
tags: [cybersecurity, phishing, email-headers, spf, dkim, dmarc, base64, mxtoolbox, virustotal, joe-sandbox, social-engineering, chapter-3]
---

# Ch3 – Phishing Tactics – Analyzing a Phishing Email

> [!abstract] Summary
> The book teaches a three-step method for analyzing a phishing email: **spot the indicators → analyze the header → analyze the URL**. My spam folder was empty, so I used a real sample from the **phishing_pot** repo: a fake **Banco Bradesco / Livelo** "your points expire today" email. I parsed the header with **MX Toolbox**, decoded the **base64** body on Fedora, translated it from Portuguese, and scanned the link on **VirusTotal**. Verdict: a **mass credential-phishing** email sent from a rented **DigitalOcean** server, with every email-authentication check failing.

> [!warning] Safety rules
> - **Never click** links or open attachments in a suspected phish. Copy the link address instead.
> - Open `.eml` files in a **text editor**, not a mail client. Read decoded HTML as **text**, not in a browser.
> - VirusTotal / Joe Sandbox submissions are **public**. Never submit anything with personal info (your own email address is in every real header).
> - Write malicious links **defanged** in notes: `hxxps://example[.]com`.

---

## 1. What the book covers (pp. 43–53)

**Why analyze, not just delete?** Recognizing a phish only tells you to delete it. Analyzing it tells you **where it came from, whether it's malicious, and what attack was attempted**. In an organization that matters: other users may have received it, and someone may have clicked. The sender address and URL behaviour are what the email admin / security team needs.

**The book's example:** a fake Apple iCloud "your Apple ID was used to sign in" email, compared with a real one. They look almost identical. The tells:
1. **Display name** "iCloud Notice" in quotes = a *friendly name*, which can be anything. The real sender address is hidden behind it.
2. **"To" field isn't my address** → sent via **BCC** to many victims at once.
3. **No account name**, grammar errors, and a **threat** ("disabled permanently", "under 24 hours").
4. **Link text ≠ real URL**: hovering shows a different destination.
5. Footer "links" aren't links at all, just an **image** copied from the real email.

**Tools:** Gmail → ⋮ → **Show original** (full header) → **MX Toolbox** Analyze Headers → **VirusTotal** (URL scan with ~90 engines) → **Joe Sandbox** (runs the link in a disposable VM and shows a behavior graph and screenshots).

**Conclusion:** it only takes **one click**. Phishing comes through email, phone (**vishing**), DNS tricks (**pharming**) and targeted messages (**spear phishing**). Slow down and think about what's being asked of you.

---

## 2. Email authentication: SPF, DKIM, DMARC

| Check | Question it answers | How |
|---|---|---|
| **SPF** (Sender Policy Framework) | Is this **server/IP allowed** to send mail for this domain? | Domain publishes a list of allowed IPs in a DNS TXT record; receiver compares the sending IP |
| **DKIM** (DomainKeys Identified Mail) | Was this email **signed by the domain**, and is it **unchanged**? | Sender signs the message with a private key; receiver checks with the public key in DNS |
| **DMARC** (Domain-based Message Authentication, Reporting & Conformance) | Do SPF/DKIM pass **for the same domain shown in From**, and what should happen if not? | Domain publishes a policy: `none` / `quarantine` / `reject` + reports |

> [!warning] Correction (my first answer mixed these up)
> I wrote that DKIM checks the **IP address**. It doesn't. **SPF checks the IP**; **DKIM checks a cryptographic signature**. DMARC isn't "SPF + DKIM combined". It's a **policy layer** on top that also requires **alignment** (the authenticated domain must match the visible From domain).

> [!note] Not proof either way
> Failed SPF/DKIM is a strong phishing indicator, but not proof: many legit domains are misconfigured. And **passing** isn't proof of safety: attackers can register their own domain and set up perfect SPF/DKIM for it.

**Alignment = why "passing" can still be a lie.** An attacker can pass SPF for `attacker.com` while showing `From: bank.com`. DMARC alignment catches that mismatch, but only if `bank.com` actually publishes a DMARC record.

---

## 3. My analysis – fake Bradesco / Livelo email

### Indicators of compromise (IOCs)

| Item | Value |
|---|---|
| Subject | `CLIENTE PRIME - BRADESCO LIVELO: Seu cartão tem 92.990 pontos LIVELO expirando hoje!` ("your card has 92,990 Livelo points expiring today!") |
| From (display) | `BANCO DO BRADESCO LIVELO <banco.bradesco@atendimento[.]com[.]br>` |
| Return-Path | `root@ubuntu-s-1vcpu-1gb-35gb-intel-sfo3-06` |
| Sending IP | `137.184.34.4` (DigitalOcean, AS14061) |
| First hop | Postfix on `ubuntu-s-1vcpu-1gb-35gb-intel-sfo3-06`, `from userid 0` |
| Message-ID | `<20230919183549.39DEA3F725@ubuntu-s-1vcpu-1gb-35gb-intel-sfo3-06>` |
| Malicious URL | `hxxps://blog1seguimentmydomaine2bra[.]me/` |
| Date | 19 Sep 2023, 18:35 UTC |

### 3.1 Where did it really come from?

- The display name says **Banco do Bradesco**, but the first `Received:` hop is a machine called **`ubuntu-s-1vcpu-1gb-35gb-intel-sfo3-06`**.
- That's not a laptop. It's **DigitalOcean's default droplet naming**: `s-1vcpu-1gb` = cheapest size (1 CPU, 1 GB RAM), `sfo3` = **San Francisco datacenter 3**. The IP `137.184.34.4` is in DigitalOcean's `137.184.0.0/16` range.
- → The attacker **rented a cheap cloud VPS** for a few dollars, installed **Postfix**, and blasted emails from it. A bank sends from its own mail infrastructure, never from a throwaway cloud server.
- **Return-Path `root@…`** and `from userid 0` → sent as the **root user** with zero setup: no real domain, no hostname. Sloppy and disposable.
- **`To: phishing@pot`** = the honeypot mailbox that collected the sample, not a personal address → mass mailing (same idea as the book's BCC trick).

### 3.2 Authentication results (MX Toolbox)

`Authentication-Results: spf=temperror … dkim=none (message not signed) … dmarc=temperror … compauth=fail`

- **SPF: temperror.** The envelope sender was just the server's hostname (no real domain), so there was **no SPF record to look up** → DNS timeout. Nothing authorized this IP.
- **DKIM: none.** The message **wasn't signed at all**. That's not even a "bad signature", just no signature.
- **DMARC: no record found** for `atendimento.com.br` → the domain publishes **no policy**, so receivers don't know to reject spoofed mail claiming to be from it. That makes it easy to impersonate.
- **compauth=fail** = Microsoft's overall "composite authentication" verdict: **failed**.

### 3.3 The body (base64)

- `Content-Transfer-Encoding: base64` → the HTML body is encoded, so I couldn't read it raw. Decoded it in the Fedora terminal and read it as **plain text**.
- Translated from Portuguese: "You have Livelo points with your Bradesco card that **expire TODAY**, avoid losing them by redeeming **right now**… exchange for air miles, up to 35% off your card bill". Big **92.990** number, **"Resgatar Agora" (Redeem Now)** button.
- Both the "click here to view images" link and the button point to the same domain: `hxxps://blog1seguimentmydomaine2bra[.]me/`. It's a random `.me` domain with **nothing to do with Bradesco** (whose real site is `bradesco.com.br`).
- Images use **relative paths** (`header.png`, `icone-superior.png`) → they'd never load in an inbox. So the "click here to view images" line is itself bait.

> [!warning] Lesson: I rendered it
> I previewed the decoded HTML in an online HTML compiler. Here it only loaded Google Fonts, but a real phish often has a **tracking pixel** (a 1×1 image from the attacker's server). Rendering it tells the attacker the address is **live and someone opened it**. Reading the source as text is enough.

### 3.4 URL analysis (VirusTotal)

- **0 / 93 engines** flagged it, "Status: –", no content.
- **This does NOT mean it's safe.** The email is from **2023**; phishing domains live for hours or days before they're taken down or abandoned (the book's own Joe Sandbox example hit "link not found"). Scanners judge the domain **as it is now**: dead and empty.
- Everything else (unrelated domain, failed auth, bank impersonation) already proves intent. **Clean on VirusTotal ≠ safe; one detection = assume malicious.**

### 3.5 Psychology (social engineering)

- **Urgency / scarcity:** "expira HOJE", "agora mesmo", "before they expire".
- **Greed / reward:** 92,990 points, air miles, 35% off.
- **Authority / trust:** bank name and colours, "CLIENTE PRIME", "Visa Infinite" (a premium card tier, which flatters the victim).

### 3.6 Bonus – why it reached the inbox anyway

- `X-MS-Exchange-Organization-SCL: 5` → Microsoft's **Spam Confidence Level** rated it spam; `BCL:9` = maximum **bulk** score.
- But `X-Microsoft-Antispam-Mailbox-Delivery: … OFR:TrustedSenderList … dest:I` → it was delivered to the **Inbox** because of a **trusted sender list** override. (Probably on purpose here, so the honeypot collects everything.) User/admin allow-lists can override the filter. Another reason the **human is the last line of defense**.

### 3.7 Verdict

| Question | Answer |
|---|---|
| Where did it come from? | A rented DigitalOcean VPS (`137.184.34.4`, San Francisco) running Postfix as root, **not** Bradesco |
| Is it malicious? | **Yes.** Brand impersonation, unrelated link domain, all auth failed |
| What attack? | **Mass credential phishing** (the book calls it *credential hijacking*) targeting Brazilian bank customers. The link most likely led to a fake Bradesco/Livelo login page to steal bank credentials and card data. Generic, no victim name → **not** spear phishing |
| What would I do in an org? | Block the sender domain/IP and URL, search mail logs for the Message-ID / sending IP to find other recipients, check proxy logs for anyone who visited the domain, reset credentials for anyone who did |

---

## 4. Fedora workflow

| Step | Book (Gmail/Windows) | What I did |
|---|---|---|
| Get a phish | Spam folder | Spam was empty → `phishing_pot` sample (`.eml`) |
| Full header | Gmail → Show original | Opened the `.eml` in a text editor |
| Parse header | MX Toolbox | MX Toolbox Analyze Headers |
| Read body | Email client | Body was **base64** → decoded in the terminal, read as text |
| Check URL | VirusTotal | VirusTotal (link copied from decoded HTML, never visited) |
| Detonate | Joe Sandbox | Skipped (domain dead; results public) |

---

## 5. Mistakes & lessons

| Mistake | Lesson |
|---|---|
| "DKIM checks the IP address" | **SPF = IP allow-list. DKIM = signature.** DMARC = policy + alignment |
| "Sent from an Ubuntu laptop" | Hostname pattern = **DigitalOcean droplet** → read hostnames, they leak infrastructure |
| Rendered the HTML in an online preview | Read phishing HTML as **text**. Rendering can fire tracking pixels |
| VirusTotal 0/93 | **Absence of detection ≠ safe.** Old phish domains are dead |
| Attack type "phishing" | Be specific: **mass credential phishing**, impersonating a bank |

---

## 6. Glossary
Phishing · Spear phishing · Vishing · Pharming · Credential hijacking / harvesting · Display (friendly) name · BCC · Email header · Received hop · Return-Path · Envelope sender vs header From · Message-ID · X- headers · SPF · DKIM · DMARC · Alignment · compauth · SCL / BCL · Trusted sender list · base64 / Content-Transfer-Encoding · MIME · Tracking pixel · IOC · Defanging · VPS / droplet · ASN · MX Toolbox · VirusTotal · Sandbox / Joe Sandbox · Behavior graph.

---

## 7. Recap questions
1. Why is a display name worthless as proof of who sent an email?
2. SPF vs DKIM: which one checks the IP, and which one checks a signature?
3. Why can an email **pass** SPF and DKIM and still be phishing?
4. A URL scores 0/93 on VirusTotal. Why might it still be malicious?
5. What does a hostname like `ubuntu-s-1vcpu-1gb-…-sfo3-06` tell you about the attacker?
