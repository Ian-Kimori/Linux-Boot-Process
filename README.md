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
