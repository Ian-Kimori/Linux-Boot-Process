# Linux-Boot-Process

---

## What happens after the Linux kernel boots

After the kernel boots, it hands off to an **init system** (systemd on Pop!_OS). Systemd then decides what **target** to reach — think of targets as runlevels:

- `multi-user.target` → boots into **pure text mode** (real TTYs, no GUI)
- `graphical.target` → boots into **GUI** (starts a display server + desktop environment on top of text mode)

So yes — the GUI is an **optional layer on top**. Text mode is the base.

---

## The relationship between real terminals and the GUI terminal

```
Hardware
    └── Linux Kernel
            └── TTY Subsystem (virtual consoles: tty1–tty6)  ← REAL terminals
                    └── Display Server (X11 / Wayland)        ← GUI layer
                            └── Desktop Environment (GNOME)
                                    └── gnome-terminal        ← EMULATED terminal
                                            └── Pseudo-TTY (pts/0, pts/1...)
```

- **Real terminals (TTYs)** — `/dev/tty1` through `/dev/tty6` — are virtual consoles managed directly by the kernel's TTY subsystem. No GUI involved at all.
- **gnome-terminal / xterm / terminator** — these are just GUI applications that create a **pseudo-TTY (pts)** and draw terminal behaviour inside a window. They are emulating what a real TTY does, but they depend on the display server being alive.

You can verify this yourself:

```bash
# In gnome-terminal:
tty
# Returns something like /dev/pts/0  ← pseudo-TTY (emulated)

# In a real TTY (tty2 for example):
tty
# Returns /dev/tty2  ← real virtual console
```

---

## Switching to a real TTY (leaving the GUI)

Linux keeps **tty1–tty6** as real virtual consoles and typically runs the GUI on **tty1** or **tty7** (varies by distro). On Pop!_OS with GDM:

| Action | Shortcut |
|---|---|
| Switch to real TTY2 | `Ctrl + Alt + F2` |
| Switch to real TTY3 | `Ctrl + Alt + F3` |
| ... | ... |
| Switch back to GUI | `Ctrl + Alt + F1` or `Ctrl + Alt + F7` |

Try `Ctrl + Alt + F2` — your screen will go black and present a plain login prompt. That is a **real TTY**, no GUI involved whatsoever.

---

## Fully stopping the GUI (closing it properly)

**Temporarily stop the display manager (GDM):**
```bash
sudo systemctl stop gdm
```
This kills the GUI entirely and drops you to a TTY. To bring it back:
```bash
sudo systemctl start gdm
```

**Boot into text mode permanently** (change default target):
```bash
sudo systemctl set-default multi-user.target
```
To go back to GUI default:
```bash
sudo systemctl set-default graphical.target
```

---

## Summary

| | Real TTY | Emulated Terminal |
|---|---|---|
| Device | `/dev/tty1`–`/dev/tty6` | `/dev/pts/0`, `pts/1`... |
| Depends on GUI? | No | Yes |
| Access | `Ctrl+Alt+F2`–`F6` | Open gnome-terminal |
| Kernel subsystem | TTY (direct) | PTY (pseudo, via display server) |
| Use case | Server admin, recovery, no-GUI systems | Day-to-day work on desktop |

The real TTY and the GUI are genuinely different interfaces — the GUI is a graphical process running in userspace, while the TTYs are a kernel-level facility that exists regardless of whether any GUI is running.

## TTY, Terminals, SSH — The Full Picture

---

### 1. Start from hardware — what is a TTY?

TTY stands for **TeleTYpewriter**. In the 1960s–70s, a TTY was a physical machine — a keyboard + printer — connected to a mainframe by a serial cable. You typed, the mainframe responded, it printed on paper.

```
[Physical TTY machine] ←serial cable→ [Mainframe]
```

Linux inherited this model. When Linux refers to a TTY today, it means **a communication channel between a user and the kernel** — the concept, not the physical machine.

---

### 2. The TTY subsystem — what Linux actually built

When Linux boots, the kernel creates **virtual consoles** — software simulations of those old physical TTY machines. These are called **TTYs** (`tty1` through `tty6`).

```
Kernel boots
    └── TTY Subsystem initialises
            ├── /dev/tty1   ← virtual console 1
            ├── /dev/tty2   ← virtual console 2
            ├── /dev/tty3   ← virtual console 3
            ├── /dev/tty4   ← virtual console 4
            ├── /dev/tty5   ← virtual console 5
            └── /dev/tty6   ← virtual console 6
```

These exist **at the kernel level**, before any GUI, before any network, before anything. They are always there as long as the kernel is running. Each one is an independent login session on your machine.

This is **text mode** — raw kernel TTYs with no graphical layer whatsoever.

---

### 3. What is text mode vs graphical mode?

| Mode | What it means |
|---|---|
| **Text mode** | Kernel TTYs only. Characters on screen rendered directly by kernel. No display server. |
| **Graphical mode** | A display server (X11/Wayland) is running on top of text mode, rendering pixels. |

The GUI is not a separate mode — it is a **process running inside text mode**. Text mode is always the foundation. The GUI is an optional layer on top.

```
Text mode (always present)
    └── Display server (X11/Wayland) ← optional process running on top
            └── Desktop Environment (GNOME, KDE)
                    └── GUI apps, windows, etc.
```

Systemd controls which layer you boot into:

```bash
# Boot to text mode only
sudo systemctl set-default multi-user.target

# Boot to GUI (text mode + display server + desktop)
sudo systemctl set-default graphical.target
```

---

### 4. What is a terminal emulator?

Once the GUI is running, you want a terminal inside it. But the GUI has no direct connection to the kernel TTYs — it runs inside a display server in userspace.

So GUI terminal apps (gnome-terminal, xterm, Terminator, Alacritty) create a **Pseudo-TTY (PTY)** — a fake TTY that behaves exactly like a real one but exists entirely in software.

```
gnome-terminal (GUI app)
    └── Creates /dev/pts/0  ← Pseudo-TTY (PTY)
            └── Bash runs inside it
                    └── Your commands execute here
```

The PTY has two ends:
- **Master** — gnome-terminal reads/writes here (draws what you see)
- **Slave** — bash thinks it's talking to a real TTY via `/dev/pts/0`

Bash doesn't know or care that it's in a PTY vs a real TTY. It just talks to whatever `/dev/pts/N` it was given.

---

### 5. The full picture — everything in one diagram

```
Hardware (keyboard, screen)
    └── Linux Kernel
            └── TTY Subsystem
                    ├── /dev/tty1  ← Real TTY (text mode, virtual console 1)
                    ├── /dev/tty2  ← Real TTY (text mode, virtual console 2)
                    ├── /dev/tty3  ← Real TTY (you can log in here)
                    │
                    └── /dev/tty1 or tty7 → Display Server (X11/Wayland)
                                                └── GNOME Desktop
                                                        └── gnome-terminal
                                                                └── /dev/pts/0  ← PTY (fake TTY)
                                                                        └── bash
```

---

### 6. Terminology map — all the words, clarified

| Term | What it actually means |
|---|---|
| **TTY** | Any terminal interface — real or emulated. Historically a physical machine, now a kernel concept. |
| **Virtual console** | One of tty1–tty6. A software-simulated TTY at kernel level. Text mode only. |
| **Text mode terminal** | Using a virtual console directly (tty1–tty6). No GUI involved. |
| **PTY / pts** | Pseudo-TTY. A fake TTY created by terminal emulators and SSH. Lives in userspace. |
| **Terminal emulator** | A GUI app (gnome-terminal, Terminator) that creates a PTY and draws a terminal window. |
| **Shell** | The program running inside a TTY or PTY (bash, zsh). Takes your commands, returns output. |

---

### 7. Switching between modes

**Switch to a real TTY (leave GUI entirely):**

```
Ctrl + Alt + F2   → tty2 (real text mode terminal)
Ctrl + Alt + F3   → tty3
Ctrl + Alt + F1   → back to GUI (Pop!_OS)
Ctrl + Alt + F7   → back to GUI (some distros)
```

You will see a plain login prompt. That is the raw kernel TTY — no display server, no GNOME, nothing.

**Stop/start the GUI manually:**

```bash
sudo systemctl stop gdm    # kills GUI, drops to TTY
sudo systemctl start gdm   # brings GUI back
```

---

### 8. How SSH fits into all of this

When you SSH into a server, you are connecting over a network. The server has no physical keyboard or screen attached to your session — so it cannot give you a real TTY.

Instead, SSH creates a **PTY on the server** for your session:

```
Your machine                          Remote server
─────────────────                     ─────────────────────────────
SSH client (terminal)                 sshd (SSH daemon)
    └── your pts/0        ←network→       └── creates pts/1 for you
                                                  └── bash runs here
```

So when you SSH in, you land in a **PTY** (`/dev/pts/N`) on the server — same mechanism as gnome-terminal, just delivered over a network instead of locally.

This is exactly why closing SSH kills your script — your bash shell is attached to that PTY, and when the SSH connection drops, the PTY is destroyed, SIGHUP is sent, and everything dies.

---

### 9. How tmux/nohup fix the SSH problem

**nohup** — tells the process to ignore SIGHUP:

```bash
nohup bash scan.sh > scan.log 2>&1 &
```

The process detaches from the PTY and keeps running even after SSH disconnects.

**tmux** — creates its own persistent PTY layer:

```
sshd
    └── pts/1 (your SSH session)
            └── tmux server  ← runs as its own process, independent of your SSH PTY
                    └── tmux creates its own pts/2
                            └── bash + your script run here
```

When you disconnect SSH, `pts/1` dies — but the tmux server is its own independent process and keeps running. When you reconnect and run `tmux attach`, you get a new PTY connected to the same tmux session, and your script is still running exactly where it left off.

This is the key insight: **tmux decouples your shell session from the SSH connection**.

Great question. Here's the full logic:

---

### Why closing SSH kills your script

When you SSH into a server, your session creates a **controlling terminal (TTY)**. Every process you start (like your nmap script) is a **child of that shell**. When you close SSH:

1. The SSH client disconnects
2. The server sends a **SIGHUP** (hangup signal) to your shell
3. Your shell forwards SIGHUP to all child processes
4. Your script gets killed

So **yes, you need to keep the session alive** — unless you detach the process from the terminal.

---

### Solutions

**Option 1 — `nohup` (simplest)**

Detaches the script from the terminal so SIGHUP is ignored:

```bash
nohup bash scan.sh > scan.log 2>&1 &
```

- `nohup` — ignore hangup signal
- `> scan.log 2>&1` — redirect all output to a log file
- `&` — run in background

You can now close SSH. Check progress later:

```bash
ssh leo_audit@10.10.10.10
tail -f scan.log
```

---

**Option 2 — `tmux` (recommended for your use case)**

Creates a persistent terminal session that survives disconnection:

```bash
# Start a named session
tmux new -s scan

# Run your script inside it
bash scan.sh

# Detach without killing it (keep it running)
Ctrl+B then D
```

Reconnect later:

```bash
ssh leo_audit@10.10.10.10
tmux attach -t scan
```

You'll see your script exactly where it left off, live output and all.

---

**Option 3 — `screen` (similar to tmux)**

```bash
screen -S scan
bash scan.sh

# Detach
Ctrl+A then D

# Reattach later
screen -r scan
```

---

### Which to use

| Situation | Use |
|---|---|
| Quick one-off, don't need to watch output | `nohup` |
| Want to reconnect and see live progress | `tmux` |
| tmux not installed | `screen` |

**`tmux` is the best fit for you** — long nmap scans benefit from being able to reconnect and watch live output, and tmux is standard on most servers.
