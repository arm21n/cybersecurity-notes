---
title: "THM SOC Simulator – Repossessed (My First CTF)"
platform: TryHackMe
event: "Strange & Unusual Activity (blue-team CTF, 22–26 Oct 2026)"
room: SOC Simulator → scenario "Repossessed"
date: 2026-10-25
status: partial — investigated the full attack, closed/reported first alerts, stepped back to build foundations
tags: [tryhackme, ctf, blue-team, soc, dfir, incident-response, siem, mitre-attack, first-ctf]
related:
  - "[[Ch2 - Attack Targets on the Internet - Practice Session]]"
  - "[[THM - Active Reconnaissance]]"
---

# THM SOC Simulator – Repossessed (My First CTF)

> [!abstract] What this was
> My **first CTF ever** — a blue-team (defensive) event aimed at working SOC analysts. I played the SOC Simulator scenario "Repossessed": a game studio (TryHatMe Studios) breached, its unreleased game **NeuralNexus** stolen. I traced the whole attack from phishing to exfiltration, understood every stage, and wrote a case report. Didn't finish all alerts — chose to build foundations first (book + Nmap) and come back. That was a deliberate call, not a failure.

> [!note] Honest reflection
> This event is built for professional analysts. As a first-year student I needed heavy guidance. That's rep one of something hard — not evidence I can't do it. I now know *what a real investigation looks like from the inside*, which will make the rest of my learning click faster. Coming back after: Nmap module + more of the book + blue-team prep rooms (SOC Sim: Intro to Phishing, Threat Hunting: Health Hazard).

---

## The big idea: what a SOC analyst does
- **SOC** = Security Operations Center. Analysts watch **alerts** and decide: real attack (**True Positive / TP**) or false alarm (**False Positive / FP**), then document and escalate.
- **Logs vs alerts:** logs = raw event records in the SIEM (you search them, never close them). Alerts = the flagged items in the queue (you action + close those).
- **SIEM** = the searchable log database (Security Information and Event Management).
- The core skill isn't knowing the answer — it's **reading each alert carefully and justifying the verdict**. Blindly clicking TP on everything is how you'd miss a real FP and waste the team's time.

> [!tip] #1 lesson
> **Don't trust a process by its name — read what it actually did.** Attackers name malware after legit things and abuse real signed programs. Inspect path, publisher, and command line before judging.

---

## The attack, stage by stage (the kill chain)

```
Phishing .docm → macro → ai.exe (LOLBin) → PowerShell C2 (45.147.230.131:8067)
  → vaultcmd dumps credential vault (stole svc-mktshare)
  → net user /domain (Discovery)
  → lateral movement: SvcUpdate service created on THM-SREYES (hidden PS as SYSTEM)
  → 7-Zip archives C:\Dev\NeuralNexus\Build → nx_build_beta.7z (Collection)
  → exfiltration: archive sent to 104.18.32.47:443 AND back to C2 45.147.230.131:8067
```

Two hosts compromised: **THM-MSMITH** (patient zero) and **THM-SREYES**. The stolen prize: the unreleased game **NeuralNexus**.

### Alert-by-alert
| ID | Alert | MITRE tactic | What it showed |
|---|---|---|---|
| 6352 | Suspicious Child Process Spawned | Execution | `WINWORD.EXE` → `ai.exe` from macro doc `Q3_PressKit_Feedback.docm` |
| 6353 | Suspicious Child Process Spawned | Execution | next execution step (ai.exe → powershell) |
| 6354 | Suspicious Outbound Connection | Command & Control | `powershell.exe` → `45.147.230.131:8067` |
| 6355 | Anomalous Credential Vault Access | Credential Access | `vaultcmd.exe /listcreds` dumps saved creds |
| 6356 | Rapid Domain/Host Enumeration | Discovery | `net user /domain` maps the domain |
| 6357 | Suspicious Remote Logon + Service Creation | Lateral Movement 🔴 | service `SvcUpdate` created on THM-SREYES using stolen `svc-mktshare` |
| 6358 | Suspicious External Connection | Exfiltration 🔴 | archive → `104.18.32.47:443` (HTTPS — blends in!) |
| 6359 | Suspicious Archive Utility Execution | Collection | `7z.exe` archives `C:\Dev\NeuralNexus\Build` |
| 6360 | Suspicious External Connection | Exfiltration 🔴 | archive → back to C2 `45.147.230.131:8067` |

All **True Positive** — one coordinated attack, no false positives in this scenario.

---

## Concepts I learned tonight

### `.docm` macros (initial access)
- `.docm` = Word doc **with macros** (VBA scripts). The `m` = macro. A normal `.docx` **can't** run code; a `.docm` can, automatically on open.
- Classic attack: **phishing → malicious macro → code execution**. MITRE: *Spearphishing Attachment* → *User Execution: Malicious File*.
- Red flag here: a macro doc in **Downloads** (`Q3_PressKit_Feedback.docm` — believable lure for a marketing person) spawned a process.

### LOLBins (Living Off the Land Binaries)
- Attackers abuse **legitimate, signed system programs** to do malicious work, so antivirus doesn't flag them.
- Seen tonight:
  - `ai.exe` — real Microsoft "Local AI Host", abused via the macro
  - `vaultcmd.exe` — real MS tool, abused to **dump saved credentials** (`/listcreds`)
  - `7z.exe` — real 7-Zip, abused to **archive stolen data**
  - `net.exe` — real Windows tool, abused for **domain enumeration**
- Lesson: "signed by Microsoft" ≠ "safe to ignore." Check *what it's doing*.

### Reading malicious PowerShell flags
The lateral-movement service ran:
```
cmd.exe /c start /b powershell.exe -NoP -NonI -W Hidden -ExecutionPolicy Bypass -File C:\Windows\Temp\svchost_update.ps1
```
| Flag | Meaning | Why suspicious |
|---|---|---|
| `-NoP` (-NoProfile) | skip PS profile | fewer traces |
| `-NonI` (-NonInteractive) | no prompts | automated, not a person |
| `-W Hidden` | no window | hiding from user |
| `-ExecutionPolicy Bypass` | ignore script-block safety | defeats a control |
| `C:\Windows\Temp\...ps1` | runs from Temp | real software doesn't live in Temp |
| service name `SvcUpdate`, acct `LocalSystem` | masquerades as legit, max privilege | |
- Any one = yellow flag. **All together = unambiguous malice.**

### Key Windows event IDs
- **Sysmon Event 1** = Process Create (parent/child, command line, hashes) — the backbone of execution analysis.
- **Sysmon Event 3** = Network Connection (source/dest IP + port, which process).
- **Security Event 4697** = *A service was installed* — classic lateral-movement / persistence artifact.

### The process tree (parent → child)
- Every "suspicious child process" alert = read **ParentImage** vs **Image**. The *story* is in who spawned whom.
- Example: `WINWORD.EXE` (parent) → `ai.exe` (child) = "a document launched a program."

### Correlation (the real skill)
- Alerts aren't isolated — you join them into one story on a **timeline**.
- Proved cause→effect: **6355 (creds dumped, 14:08) → 6357 (those creds reused to move to SREYES, 14:16)**. Steal keys, then open doors.
- The same script `svchost_update.ps1` appeared across multiple alerts and both hosts → one attacker, one toolkit. A single "search all hosts for svchost_update.ps1" would reveal full breach scope.

### Context beats the connection in isolation
- The C2 (6354) used a weird port **8067** → obviously suspicious.
- But exfil (6358) used port **443 (HTTPS)** and a **Cloudflare IP** → looks like normal web traffic!
- Why still malicious: a **high-privilege PowerShell** (not a browser) opened it, **right after** a `.7z` of source code was made, on a **freshly compromised host**. The *story* convicts it, not the port.
- Ties back to Ch2: an innocent-looking port (443) can hide bad traffic — judge by context.

---

## How to work an alert (the loop)
1. **Assign** yourself the alert (person icon, Action column).
2. **Open** and *read every field* — don't judge yet.
3. Check the **Playbook** (official decision steps for that alert type).
4. Use the **SIEM** to pivot (search the host/process to see what happened next).
5. Decide **TP or FP**, with a reason you can justify.
6. **Close** the alert and **write a case report**.

---

## The case report (the deliverable)
Findings nobody can read are useless. Fields and what goes in each:
- **Time of activity** — the alert's timestamp.
- **Affected entities** — host(s), user(s), process(es), file(s), grouped by type.
- **Reason for TP** — the evidence, in plain words (most important box).
- **Reason for escalating** — the impact / blast radius.
- **Remediation** — concrete actions: isolate host, reset account, block IPs, hunt the script, user awareness.
- **IOCs** — the forensic breadcrumbs (type them *exactly* — see lesson below).

> [!danger] A report is evidence — precision matters
> If I write a C2 IP as "45.127.230.131" when it was `45.147.230.131`, the team blocks the wrong address and the attacker stays connected. A typo in an IOC = the difference between stopping an attack and not. (Same lesson as my `dig` typos in Ch2 — read before you submit.)

---

## IOCs from this incident
- Phishing doc: `Q3_PressKit_Feedback.docm`
- Malicious script: `svchost_update.ps1`
- Malicious service: `SvcUpdate`
- Stolen data archive: `nx_build_beta.7z` (from `C:\Dev\NeuralNexus\Build`)
- Abused account: `svc-mktshare`
- C2 server: `45.147.230.131:8067`
- Exfil destination: `104.18.32.47:443`
- Compromised hosts: `THM-MSMITH`, `THM-SREYES`

---

## MITRE ATT&CK tactics seen (the attack lifecycle)
Initial Access → Execution → Command & Control → Credential Access → Discovery → Lateral Movement → Collection → Exfiltration.
Reading the alert "Type" column top-to-bottom literally spelled out the attack lifecycle. **MITRE ATT&CK** = the standard map of how attacks unfold.

---

## Connections to earlier learning
| Tonight | Where it links |
|---|---|
| Private IPs 10.60.10.x | [[Ch2 - Attack Targets on the Internet - Practice Session]] RFC 1918 |
| Innocent-looking port 443 hiding bad traffic | Ch2 — judge by context |
| LOLBins / abusing signed tools | new, but echoes "verify, don't trust the name" from Ch2 |
| Reading command-line flags carefully | Ch2 habit: read the command before running/judging |
| Precision in IOCs | Ch2 `dig`/`curl` typo lessons |

---

## What I'm doing next (my plan)
1. **Finish the book** (How Cybersecurity Really Works) — foundations, breadth
2. **THM Nmap module** — the recon depth I already enjoy
3. **Later:** come back to blue-team with prep rooms first (SOC Sim: Intro to Phishing, Threat Hunting: Health Hazard), then retry events like this

> [!success] Takeaway
> First CTF = done *something real*, even if not finished. I followed a full intrusion end to end and understood why each step mattered. The discomfort was the learning. Foundations first, then back to the deep end — on purpose.
