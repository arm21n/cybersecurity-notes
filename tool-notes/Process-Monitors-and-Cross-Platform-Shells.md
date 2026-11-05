---
title: "Process monitors (htop vs Task Manager) & cross-platform shells (pwsh, WSL)"
series: Tool notes
date: 2026-11-05
status: completed
tags: [tool-notes, linux, windows, htop, task-manager, process-monitoring, powershell, pwsh, wsl, kali]
related:
  - "[[L4 – Linux: SSH]]"
---

# Process monitors & cross-platform shells

> [!abstract] One-line summary
> GUI task managers are for checking your own machine. **htop** works in any terminal, including over SSH on a box you've gotten into. Shells cross OS lines too: **PowerShell 7 (`pwsh`)** runs on Kali, and **WSL** gives Windows a real Linux terminal.

---

## 01. htop vs Task Manager

Both show what's running and how much CPU/RAM it uses. The difference is **where** and **for whom**.

### Windows Task Manager
- GUI, opened with `Ctrl+Shift+Esc`.
- Processes, CPU/RAM/disk/network/GPU graphs, startup apps, services.
- Mostly for "find what's slow, end task." Easy to use, but it can't be scripted and hides low-level detail.

### htop (Linux/Unix)
- Interactive tool that runs **in the terminal**, an upgraded version of `top`.
- Per-core CPU bars, memory/swap meters, load average, process list you can sort, filter and search, plus a **tree view** that shows which process started which.
- Install on Fedora: `sudo dnf install htop` (on Kali: `sudo apt install htop`).

| | Task Manager | htop |
|---|---|---|
| Platform | Windows | Linux/Unix |
| Interface | GUI, mouse | Terminal, keyboard (mouse works too) |
| Works over SSH | No | Yes |
| Process tree | Limited | Yes (`F5`) |
| Process details | Name, PID, user, resources | + full command line, nice/priority, state, threads |
| Signals | Basically "End task" | Any signal: SIGTERM, SIGKILL, SIGSTOP… (`F9`) |
| Change priority | Yes, via right-click menu | `F7` / `F8` (renice) |
| Startup apps / services | Yes | No (on Linux that's `systemctl`) |
| GPU / disk / network graphs | Yes | Mostly CPU/RAM by default (configurable in `F2`) |

### htop keys worth remembering
| Key | Does |
|---|---|
| `F6` | Sort by column |
| `F4` | Filter |
| `/` | Search |
| `F5` | Tree view |
| `F9` | Send a signal (kill) |
| `u` | Show one user's processes |
| `H` | Hide/show threads |
| `F2` | Setup (meters, columns) |

### Why it matters for pentesting
- htop runs **over SSH** on a target, where there's usually no desktop.
- Tree view + full command lines help spot weird stuff: a **reverse shell spawned by a web server process**, an odd cron job, a process running as **root** that shouldn't be.

### Windows equivalents with more detail
- **Process Explorer** (Sysinternals): closest to htop, with a tree view, full command lines and much more detail.
- CLI: `tasklist` / `taskkill`, or `Get-Process` in PowerShell.

---

## 02. Kali has a "Task Manager" too

Kali's default desktop is **Xfce**, which ships **Xfce Task Manager** (`xfce4-taskmanager`).
- Menu → System → Task Manager, or run `xfce4-taskmanager`.
- Same basic job as htop: list processes, sort, search, kill, renice, tree view, CPU/memory graphs at the top, right-click menus.

| | Xfce Task Manager | htop |
|---|---|---|
| Needs a desktop session | Yes | No (terminal, SSH, TTY, headless servers) |
| Detail | Simpler | More: per-core bars, load avg, states, threads, any signal |
| Best for | Quick look at my own Kali VM | Everything else, especially remote/target machines |

On **GNOME** (Fedora's default desktop) the equivalent is **GNOME System Monitor** (`gnome-system-monitor`).

**Rule of thumb:** GUI task manager on my own desktop, **htop everywhere else**.

---

## 03. Shells across operating systems

### PowerShell on Linux
- Kali can run **PowerShell 7** with `pwsh`. It's cross-platform: Linux, macOS, Windows.
- Windows also ships the older **Windows PowerShell 5.1** (`powershell`).
- Some Windows-only cmdlets (much of the AD and registry stuff) **don't work on Linux**.

### A Linux terminal on Windows
Yes, mainly through **WSL (Windows Subsystem for Linux)**.

- **WSL 2** runs a real Linux kernel in a lightweight VM, so you get real Bash, `apt`/`dnf` and Linux tools.
- Install (admin PowerShell):
  ```powershell
  wsl --install                    # Ubuntu by default
  wsl --list --online              # see available distros
  wsl --install -d kali-linux      # Kali
  ```
- Kali on WSL has **Win-KeX** for the full Kali Xfce desktop on Windows.
- Windows files appear inside Linux at `/mnt/c/`, and Linux files can be opened from Windows Explorer.

### Other options
- **Git Bash**: small Bash shell + common Unix commands (`ls`, `grep`, `ssh`). Not a full Linux system.
- **Cygwin / MSYS2**: older Unix-like environments, mostly replaced by WSL now.
- **SSH into a Linux machine** from Windows Terminal (Windows has a built-in `ssh` client).
- **Full VM** (VirtualBox, VMware, Hyper-V): what most pentesters still use for Kali.

### Pentesting caveat
WSL is great for quick tools and scripts. For anything needing **raw network access** (Wi-Fi attacks, some Nmap scan types, packet sniffing), a **Kali VM or bare-metal install** is more reliable, because WSL's networking sits behind Windows.

---

## Takeaways
- Task managers (Windows, Xfce, GNOME) = GUI, for my own machine.
- **htop** = terminal, works over SSH, more detail. It's the one I'll actually have on targets.
- `pwsh` runs on Linux; **WSL** puts real Linux on Windows. For serious network work, still use a proper Kali VM.
