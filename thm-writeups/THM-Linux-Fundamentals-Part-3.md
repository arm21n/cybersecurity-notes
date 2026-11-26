---
title: "THM – Linux Fundamentals Part 3"
platform: TryHackMe
room: https://tryhackme.com/room/linuxfundamentalspart3
module: Linux Fundamentals
date: 2026-11-27
status: completed
tags: [tryhackme, linux, wget, scp, http-server, processes, systemd, systemctl, cron, apt, logs]
related:
  - "[[http-request-logger]]"
  - "[[caesar-ctf]]"
---

# THM – Linux Fundamentals Part 3

> [!abstract] One-line summary
> The everyday admin toolkit: **moving files** between machines (`wget`, `scp`, `python3 -m http.server`), **seeing and controlling processes** (`ps`, `top`, `kill`, `systemctl`, `&` / `fg`), and **keeping a system running on its own** (cron, apt repositories, logs in `/var/log`).

This room turned straight into a project: after the `http.server` task I built [http-request-logger](https://github.com/arm21n/http-request-logger), which extends the same module to log everything a client sends.

---

## Contents
| # | Topic | Key commands |
|---|---|---|
| 1 | Downloading files | `wget` |
| 2 | Transferring files between hosts | `scp` |
| 3 | Serving files from your host | `python3 -m http.server` |
| 4 | Processes 101 | `ps`, `ps aux`, `top`, `kill` |
| 5 | How processes start | namespaces, systemd (PID 1) |
| 6 | Services on boot | `systemctl start/stop/enable/disable/status` |
| 7 | Background & foreground | `&`, `Ctrl+Z`, `fg` |
| 8 | Automation | `crontab -e`, `crontab -l` |
| 9 | Packages & repositories | `apt`, `add-apt-repository`, GPG keys |
| 10 | Logs | `/var/log`, rotation |

---

## 1. Downloading files: `wget`
Downloads a file over **HTTP(S)**, like a browser would. Give it the URL:
```bash
wget https://assets.tryhackme.com/additional/linux-fundamentals/part3/myfile.txt
```
The file is saved in the **current directory** under its original name.

---

## 2. Transferring files between hosts: `scp`
- Like `cp`, but **between two computers**, running over **SSH** → authenticated **and** encrypted.
- Needs valid credentials for the remote user.
- Always **`scp SOURCE DESTINATION`**. The remote side is written `user@IP:/path`.

| Direction | Command |
|---|---|
| Local → remote (upload) | `scp important.txt ubuntu@192.168.1.30:/home/ubuntu/transferred.txt` |
| Remote → local (download) | `scp ubuntu@192.168.1.30:/home/ubuntu/documents.txt notes.txt` |

> [!tip]
> To reverse the direction, swap SOURCE and DESTINATION. Whichever side has `user@IP:` is the remote one.

---

## 3. Serving files from your host: `python3 -m http.server`
- Python 3's built-in module turns the machine into a quick web server.
- Serves the **directory you run it from**, on port **8000** by default.

```bash
python3 -m http.server
# Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
```
From the other machine (or another terminal), **include the port**:
```bash
wget http://<server-ip>:8000/<filename>
```

> [!warning] Gotchas
> - The server **holds its terminal** until `Ctrl+C`, so `wget` needs a second terminal.
> - The room says there's **no indexing**, so you must know the exact filename. In fact `http.server` shows a directory listing when the folder has no `index.html`; what it lacks is search. (Worth testing myself.)
> - **Updog** is a more advanced lightweight alternative.

> [!example] What I built from this
> [http-request-logger](https://github.com/arm21n/http-request-logger) extends `BaseHTTPRequestHandler` to log every request's IP, path and headers. Comparing curl, wget2, Firefox and my phone showed how much a client reveals (OS, browser version, languages) and how scripts can be told apart from humans even when they fake their User-Agent.

---

## 4. Processes 101
A **process** is a running program. The **kernel** manages processes and gives each one a **PID**. PIDs increase in start order: the 60th process gets PID 60. Running `ps` twice shows `ps` itself with a new PID each time (204, then 205).

| Command | Shows |
|---|---|
| `ps` | Processes in **my current session**: PID, TTY, CPU time, command |
| `ps aux` | **All** processes from **every user**, including system ones with no terminal (`TTY ?`) |
| `top` | **Live**, refreshing view of load, memory and per-process CPU/MEM |

Useful `ps aux` columns: `USER`, `PID`, `%CPU`, `%MEM`, `STAT` (`S` sleeping, `R` running, `T` stopped), `COMMAND`.

> [!note] Correction
> The room says `top` refreshes every 10 seconds. The default on most systems is about **3 seconds** (change it with `d`, quit with `q`).

### Signals with `kill`
`kill <PID>` sends a signal, e.g. `kill 1337`.

| Signal | Number | Effect |
|---|---|---|
| `SIGTERM` | 15 | Stop, but let the process **clean up first** (the default) |
| `SIGKILL` | 9 | Kill **immediately**, no cleanup |
| `SIGSTOP` | 19 | **Pause / suspend** |

Specific signal: `kill -9 <PID>` or `kill -SIGKILL <PID>`.

---

## 5. How processes start
- **Namespaces** split resources (CPU, RAM, priority) into slices, like a cake. Processes only see others **in the same namespace**: isolation that's good for security, and the basis of containers.
- On boot the **init system** starts first. On Ubuntu (and on my Fedora) that's **systemd**. Every program started afterwards is a **child process** of systemd.

> [!note] Correction
> The room says the boot process has ID **0**, but its own `top` screenshot shows systemd as **PID 1**. PID 1 is correct; PID 0 is the kernel's internal idle task, not a process you'll see in `ps`.

---

## 6. Services on boot: `systemctl`
`systemctl` talks to systemd to control **services** (daemons): web servers, databases, file servers.

Syntax: **`systemctl [option] [service]`**

| Option | Effect | Example |
|---|---|---|
| `start` | Run it **now** | `systemctl start apache2` |
| `stop` | Stop it **now** | `systemctl stop apache2` |
| `enable` | Start it **at every boot** | `systemctl enable apache2` |
| `disable` | Don't start at boot | `systemctl disable apache2` |
| `status` | Running or not, plus recent log lines | `systemctl status apache2` |

> [!warning] `start` ≠ `enable`
> `start` doesn't survive a reboot. `enable` doesn't start it now. For both at once: `systemctl enable --now apache2`.

---

## 7. Background & foreground
| Action | How | Result |
|---|---|---|
| Start in background | `command &` | Job number + PID (`[1] 16889`), prompt returns immediately |
| Suspend the current process | `Ctrl+Z` | Paused (`T` in `ps aux`), shown as `[1]+ Stopped ./background.sh` |
| Bring it back | `fg` | Resumes and takes over the terminal again |

- `echo "Hi THM" &` prints the job/PID first; the output appears whenever the job runs.
- `&` is handy for long tasks (big copies, scans) so the shell stays usable.
- `Ctrl+Z` **pauses**, it doesn't kill. To stop it for real: `fg` then `Ctrl+C`, or `kill`.

---

## 8. Automation: cron
**cron** is a process started at boot that runs scheduled tasks (**cron jobs**). They're defined in a **crontab**, a file where each line is one job.

| Command | Purpose |
|---|---|
| `crontab -e` | Edit **my** crontab (choose an editor such as nano) |
| `crontab -l` | **List** my crontab |
| `sudo crontab -u <user> -l` | List another user's crontab |
| `cat /etc/crontab`, `ls /etc/cron.d/` | System-wide cron jobs |

### Line format: 5 time fields + the command
```text
MIN  HOUR  DOM  MON  DOW  CMD
```
| Field | Meaning |
|---|---|
| MIN | Minute (0–59) |
| HOUR | Hour (0–23) |
| DOM | Day of month (1–31) |
| MON | Month (1–12) |
| DOW | Day of week (0–6, 0 = Sunday) |
| CMD | Command to run |

- `*` = "any / every".
- `*/12` in HOUR = **every 12 hours** (a step value).
- Lines starting with `#` are comments, so a crontab can look empty except for its help text.

| Example | Runs |
|---|---|
| `0 */12 * * * cp -R /home/cmnatic/Documents /var/backups/` | Every 12 hours, on the hour |
| `0 5 * * 1 tar -zcf /var/backups/home.tgz /home/` | 05:00 every Monday |

Instead of five fields, a line can start with a **special keyword** that is the whole schedule: `@reboot`, `@hourly`, `@daily`, `@weekly`, `@monthly`, `@yearly`.

### `>/dev/null 2>&1`
Generated cron lines often end with this:
- `>/dev/null` throws away normal output (stdout).
- `2>&1` sends errors (stderr) to the same place.

Without it, cron **emails** the output to the crontab's owner.

> [!tip] Tools
> **Crontab Generator** and **Cron Guru** turn schedules into cron syntax and show the next run times.

> [!note] What tripped me up
> I was stuck for ~20 minutes on the cron question. Two lessons: `crontab -l` is the quick way to read a crontab, and each user has their **own** crontab, so check *whose* job you're looking for.

---

## 9. Packages & repositories: `apt`
- Developers publish software to **apt repositories**. Ubuntu's list lives in `/etc/apt/sources.list`, and third-party repos go in **separate files** in `/etc/apt/sources.list.d/`.
- `deb` lines are binary package sources; `deb-src` lines (usually commented out) are source code.
- Add repos with `add-apt-repository` or manually.
- **GPG keys** prove a package really comes from its developer. If the key doesn't match what the system trusts, apt refuses the download.
- Advantage over installing a `.deb` with `dpkg`: repo software gets **updated** with the rest of the system.

### Adding a repo manually (Sublime Text example)
```bash
# 1. Trust the developer's GPG key
wget -qO - https://download.sublimetext.com/sublimehq-pub.gpg | sudo apt-key add -

# 2. One file per third-party repo
sudo nano /etc/apt/sources.list.d/sublime-text.list
#    deb https://download.sublimetext.com/ apt/stable/

# 3. Refresh the package lists, then install
sudo apt update
sudo apt install sublime-text
```

### Removing
```bash
add-apt-repository --remove ppa:PPA_Name/ppa   # or delete the .list file
apt remove sublime-text
```

> [!note] Modern note
> `apt-key` is **deprecated**. The current way is to save the key in `/etc/apt/keyrings/` and reference it with `signed-by=` in the `.list` file, so the key only applies to that one repo.

> [!info] On my Fedora
> Fedora uses **dnf** instead of apt. Repos live in `/etc/yum.repos.d/`, and GPG checking works the same way (`gpgcheck=1`).

---

## 10. Logs: `/var/log`
Logs for the OS, applications and services live in **`/var/log`**.

### Rotation
The OS manages logs automatically by **rotating** them: `syslog` is current, `syslog.1` is the previous one, and `syslog.2.gz`, `syslog.3.gz`… are older and compressed. Read compressed logs with `zcat` / `zgrep` without unpacking them.

### Logs worth knowing
| Log | Contains |
|---|---|
| `apache2/access.log` | **Every request** to the web server |
| `apache2/error.log` | Web server errors |
| `auth.log` | Authentication: logins, `sudo`, SSH attempts |
| `fail2ban.log` | Brute-force detection and bans |
| `ufw.log` | Firewall (UFW) activity |
| `syslog` | General system messages |

Web server logs let admins diagnose performance problems **and investigate an intruder's activity**.

> [!info] On my Fedora
> Many of these messages go to the systemd journal instead. Read it with `journalctl` (e.g. `journalctl -u sshd`), as in my voice-monitor troubleshooting.

---

## Pentest relevance
| Concept | Why it matters |
|---|---|
| `python3 -m http.server` + `wget` | Classic way to move tools onto a compromised machine |
| `scp` | File transfer when I already have SSH credentials |
| `ps aux` | Shows what runs **as root**, a common privilege-escalation lead |
| `systemctl enable` / cron | How admins automate, and how attackers get **persistence** |
| Writable scripts run by root's cron | A classic privilege-escalation path |
| `/var/log`, `auth.log`, `access.log` | Where my activity gets recorded; also what defenders read |

---

## Connections to my earlier work
| THM concept | Where I saw it |
|---|---|
| `python3 -m http.server` | Became [http-request-logger](https://github.com/arm21n/http-request-logger) |
| Ports below 1024 need root | Hit `PermissionError` binding port 80 in http-request-logger |
| Serving a program on the network | `ncat -l -k 4444 -e ./challenge` in [caesar-ctf](https://github.com/arm21n/caesar-ctf) |
| `tail -F` on a log | Watched `requests.log` live in http-request-logger |
| systemd / `journalctl` | Debugging SSH in [fedora-android-voice-monitor](https://github.com/arm21n/fedora-android-voice-monitor) |
| `scp` over SSH | Same Ed25519 key-based SSH as the voice monitor |

---

## Practice ideas
- [ ] Serve a folder with `python3 -m http.server` and pull a file from my phone with `wget` in Termux
- [ ] `scp` a file between Fedora and my Kali VM, in both directions
- [ ] Find a process in `ps aux`, pause it with `kill -SIGSTOP`, resume it with `kill -SIGCONT`
- [ ] Write a cron job that appends `date` to a file every minute, then remove it
- [ ] Run http-request-logger as a **systemd service** that starts on boot
- [ ] `zgrep "Failed password" /var/log/auth.log*` on an Ubuntu box to spot brute-force attempts
- [ ] Next rooms: **Bash Scripting**, **Regular Expressions**
