# Lessons Learned: Ubuntu 26 Login Loop After Adding Second RTX 3090

**Build:** MSI MEG X870E + 2× NVIDIA RTX 3090 (PNY), Ubuntu 26.04
**Result:** Fixed. Two independent root causes had to be resolved together.

## Symptoms
- POST and BIOS fine.
- Login screen appears (rendered by the secondary GPU — top card's ports were physically inaccessible after mounting).
- Correct password accepted → screen goes black → returns to login screen (classic **login loop**).
- `Ctrl+Alt+F3` TTY also unreachable at first / no useful login.

## The Two Actual Root Causes (both had to be fixed)

### Root cause A — `nvidia-drm.modeset=1` was not on the kernel command line
`/var/log/Xorg.0.log` showed:
```
(WW) NVIDIA(G0): Failed to set the display configuration
(WW) NVIDIA(G0):  - Setting a mode on head N failed: Insufficient permissions
(II) Server terminated successfully (0). Closing log file.
```
Without early NVIDIA KMS, `simpledrm`/`simplefb` grabs the framebuffer at boot and the nvidia driver can't take DRM master → Xorg exits → DM restarts → loop.

**Fix:**
```bash
# 1. kernel cmdline — APPEND (don't overwrite; keep existing params)
sudo cp /etc/default/grub /etc/default/grub.bak
sudo sed -i 's|\(GRUB_CMDLINE_LINUX_DEFAULT="[^"]*\)"|\1 nvidia-drm.modeset=1 nvidia-drm.fbdev=1"|' /etc/default/grub
sudo update-grub

# 2. modprobe option (belt & braces)
echo 'options nvidia-drm modeset=1 fbdev=1' | sudo tee /etc/modprobe.d/nvidia-kms.conf

# 3. force nvidia modules into initramfs so they win over simpledrm
printf 'nvidia\nnvidia_modeset\nnvidia_uvm\nnvidia_drm\n' | sudo tee -a /etc/initramfs-tools/modules
sudo update-initramfs -u

sudo reboot
```

Verify:
```bash
cat /proc/cmdline | grep -o 'nvidia-drm[^ ]*'         # must contain nvidia-drm.modeset=1
sudo cat /sys/module/nvidia_drm/parameters/modeset    # must print Y
```

### Root cause B — Wrong display manager (LightDM instead of GDM)
`journalctl -u gdm` showed no entries for the current boot. `cat /etc/X11/default-display-manager` returned `/usr/sbin/lightdm`. **LightDM was the active DM, not GDM** — installing `xrdp` had silently swapped GDM for LightDM.

LightDM launched `gnome-session --session=ubuntu` on Wayland, and `~/.xsession-errors` said:
```
ERROR: Failed to obtain session bus: The given address is empty
```
Available sessions on the box:
```
/usr/share/wayland-sessions/:  ubuntu.desktop  xfce-wayland.desktop
/usr/share/xsessions/:         lightdm-xsession.desktop  xfce.desktop
```
No `ubuntu-xorg` session existed → LightDM could only launch Ubuntu on Wayland → GNOME/Wayland failed D-Bus activation under LightDM → session died with return code 1 → loop.

**Fix — switch back to GDM (the DM Ubuntu GNOME expects):**
```bash
sudo rm -f /etc/lightdm/lightdm.conf.d/50-force-xorg.conf   # remove any bad overrides
sudo systemctl disable --now lightdm
sudo systemctl enable gdm3
echo "/usr/sbin/gdm3" | sudo tee /etc/X11/default-display-manager
sudo reboot                                                  # a clean reboot was required
```

Trying to start GDM without rebooting hit stubborn "another session is active" errors from leftover logind sessions — a full reboot was the reliable path.

## Diagnostic Path (in order)

### 1. Get in without a desktop session
- Hold **Shift** at POST fails on modern UEFI — **tap Esc** repeatedly, or force 2–3 hard power-offs to trigger GRUB recovery.
- GRUB → *Advanced options for Ubuntu* → kernel entry ending in **(recovery mode)** → **root — Drop to root shell prompt**.
- `mount -o remount,rw /` to enable writes.
- Ended up at `grub>` command line once — `normal` command loads the menu from there.

### 2. Verify the driver stack
```bash
dpkg -l | grep -i nvidia            # nvidia-driver-595 installed
lspci -k | grep -A3 -Ei 'vga|3d'    # both 3090s bound to `nvidia`
lsmod | grep -E 'nvidia|nouveau'    # nvidia modules loaded, nouveau NOT loaded
```
Driver was fine — `ollama` was even using both GPUs happily.

### 3. Bypass the GUI to debug remotely
```bash
sudo systemctl set-default multi-user.target   # boot to text mode
sudo systemctl enable --now ssh                # so you can SSH in
sudo systemctl enable --now NetworkManager     # WiFi doesn't auto-start on multi-user
```
Restore later with `sudo systemctl set-default graphical.target`.

### 4. Find the real error
```bash
sudo grep -E '\(EE\)|\(WW\)' /var/log/Xorg.0.log | tail -40
cat ~/.xsession-errors
sudo tail -80 /var/log/lightdm/lightdm.log      # or journalctl -u gdm
```
The `(WW) Insufficient permissions` line pointed at Root Cause A.
The `Failed to obtain session bus` line pointed at Root Cause B.

## Related Issue: No Mouse or Keyboard After GPU Install

After physically installing the second RTX 3090 and booting into the GUI, the display came up but **the mouse cursor never appeared and keyboard input did nothing** (a monitor image, but a totally dead session).

**Root cause:** the NVIDIA driver version that was already installed pre-dated proper KMS input-cursor support for this hardware. The kernel had `nvidia-drm.modeset=1` requested but the loaded driver couldn't wire it up, so libinput never got attached to a working DRM device and cursor rendering silently failed.

**Fix:** upgrade to a newer NVIDIA driver series that includes the required KMS + cursor plane support.

```bash
# See what's currently installed and what's available
dpkg -l | grep -i nvidia-driver
ubuntu-drivers devices                       # lists recommended driver for your GPUs

# Purge any stale driver bits, then install the recommended (or newest supported) series
sudo apt purge -y '^nvidia-.*' 'libxnvctrl*'
sudo apt autoremove -y
sudo apt update
sudo ubuntu-drivers install                  # or: sudo apt install nvidia-driver-595
sudo reboot
```

After the reboot, cursor and keyboard came back immediately. Verify:

```bash
nvidia-smi                                   # driver + both GPUs listed
sudo cat /sys/module/nvidia_drm/parameters/modeset   # must be Y
cat /proc/bus/input/devices | grep -iE 'name|handlers' | head   # mouse/keyboard listed
```

> Lesson: a monitor picture is **not** proof that the driver is fully working. If input is dead but the framebuffer paints, suspect the NVIDIA driver version, not the input stack.

## Things That Turned Out NOT to Be the Cause
- **Wayland vs Xorg via `/etc/gdm3/custom.conf`** — irrelevant because LightDM was actually the DM.
- **`nvidia-prime` in `on-demand` mode** — removed it, no change (there's no iGPU on this Ryzen 9000 system).
- **Custom user services in `~/.config/systemd/user/gnome-session.target.wants/`** (`no-mistakes-daemon`, `panorama-overlay`, `reed-tpse-daemon`, `gnome-remote-desktop-headless.service`) — worth parking to isolate, but not the cause here.
- **Home directory ownership** — was already correct.
- **GNOME shell state / dconf** — nuking `~/.cache`, `~/.config/dconf`, `~/.local/share/gnome-shell` didn't help.
- **Pinning Xorg to a specific GPU** with a `BusID` `xorg.conf.d` snippet — didn't help either.

## Gotchas Encountered
- **`hostname -I` returned nothing in text mode** because NetworkManager doesn't auto-start on `multi-user.target`. Fix: `sudo systemctl enable --now NetworkManager` then `sudo nmcli device wifi connect "SSID" password "PASS"`.
- **SSH `Connection refused` on `127.0.0.1`** — sshd wasn't running. Starting it in *recovery mode* only helps that session; must `systemctl enable ssh` to survive reboots.
- **`127.0.0.1` is loopback only** — LAN clients need the real `192.168.x.x` from `hostname -I` or `ip -4 addr`.
- **Shift at boot fails on UEFI** — Esc taps or forced-reboot recovery works better.
- **MSI X870E + dual 3090**: no BIOS change needed if the monitor is on the secondary card — BIOS defaults to whichever slot has an active display output.
- **`WaylandEnable=false` in `/etc/gdm3/custom.conf`** does nothing when LightDM is the DM — always confirm the DM first with `cat /etc/X11/default-display-manager`.
- **`tailscale up` "hangs"** — it's actually waiting for you to visit the auth URL it prints. Scroll up in the terminal to find it, or use `--authkey=`.
- **`grub>` command line** (not the menu) — type `normal` to load the menu.
- **Switching DMs without reboot** was unreliable — leftover logind sessions kept blocking. A full reboot was the clean move.

## Reusable Checklist for "Login Loop" on NVIDIA + Ubuntu
1. **Confirm which display manager is actually active:** `cat /etc/X11/default-display-manager` and `systemctl status display-manager`.
2. **Get a root shell** via recovery mode, or bypass via `systemctl set-default multi-user.target` + SSH.
3. **Confirm driver loaded:** `lspci -k`, `lsmod | grep nvidia`.
4. **Ensure NVIDIA KMS is active early** (`nvidia-drm.modeset=1` in cmdline, `Y` in `/sys/module/nvidia_drm/parameters/modeset`).
5. **Look at the actual error:** `/var/log/Xorg.0.log` (grep `(EE)`), `~/.xsession-errors`, `/var/log/lightdm/*` or `journalctl -u gdm`.
6. **Match DM to desktop:** GNOME belongs with GDM. If `xrdp` (or similar) swapped it to LightDM and there's no `ubuntu-xorg` xsession installed, switch back to GDM.
7. **When switching DMs, reboot** — don't fight leftover sessions.
8. **Fix ownership, if suspect:** `chown -R $USER:$USER /home/$USER`.
9. **Isolate user config** by creating a fresh test user (`sudo adduser testuser`) and trying to log in as them.

## Key Commands Cheat-Sheet
```bash
# Which DM is really running?
cat /etc/X11/default-display-manager
systemctl status display-manager

# NVIDIA KMS status
cat /proc/cmdline | grep -o 'nvidia-drm[^ ]*'
sudo cat /sys/module/nvidia_drm/parameters/modeset

# GPUs detected & driver bound
lspci -k | grep -A3 -Ei 'vga|3d'

# Xorg errors
sudo grep -E '\(EE\)|\(WW\)' /var/log/Xorg.0.log | tail -40

# LightDM logs
sudo tail -80 /var/log/lightdm/lightdm.log
sudo tail -80 /var/log/lightdm/x-0.log

# User session errors
cat ~/.xsession-errors

# Available session types
ls /usr/share/xsessions/ /usr/share/wayland-sessions/

# Bypass to text mode + enable SSH
sudo systemctl set-default multi-user.target
sudo systemctl enable --now ssh NetworkManager
```

## Bonus: Remote Desktop (RDP) Access on a Headless Machine

**Bottom line: use `xrdp` + `xorgxrdp` + an XFCE session.** GNOME Remote Desktop (`gnome-remote-desktop`, v50) sounds like the modern answer but it does **not** actually deliver a truly headless RDP path on Ubuntu 26.04 — see "Why not gnome-remote-desktop" below.

### Working setup: xrdp + XFCE (Xorg via xorgxrdp)

```bash
# 1. Install (both packages are required — xorgxrdp gives real Xorg-in-xrdp sessions)
sudo apt install -y xrdp xorgxrdp xfce4 xfce4-session

# 2. If xrdp was previously masked / disabled, un-mask and enable
sudo systemctl unmask xrdp xrdp-sesman
sudo systemctl enable --now xrdp xrdp-sesman

# 3. Force the RDP session to launch XFCE on Xorg (NOT gnome-session on Wayland — that black-screens)
mv ~/.xsession    ~/.xsession.bak    2>/dev/null || true
mv ~/.xsessionrc  ~/.xsessionrc.bak  2>/dev/null || true
cat > ~/.xsession <<'EOF'
#!/bin/sh
export XDG_SESSION_TYPE=x11
export XDG_CURRENT_DESKTOP=XFCE
unset DBUS_SESSION_BUS_ADDRESS
exec startxfce4
EOF
chmod +x ~/.xsession

sudo systemctl restart xrdp xrdp-sesman
```

Verify:

```bash
sudo systemctl is-active xrdp xrdp-sesman   # both active
ss -tlnp | grep 3389                        # LISTEN on *:3389
sudo journalctl -u xrdp -u xrdp-sesman -n 40 --no-pager
```

Connect from **Microsoft Remote Desktop / Windows App** on Mac:
- Host: Tailscale IP (`100.x.y.z`) or LAN IP (`192.168.x.y`)
- Username: your **Linux** username (e.g. `kc-llm`)
- Password: your **Linux** login password — xrdp authenticates through PAM, not any RDP-specific password

If UFW is on:
```bash
sudo ufw allow in on tailscale0 to any port 3389 proto tcp
```

### Critical: xrdp silently swaps your Display Manager

Installing `xrdp` on Ubuntu 26.04 **changes `/etc/X11/default-display-manager` from `gdm3` to `lightdm`**. This is what triggered the original login loop in this document (see Root Cause B). After installing xrdp, always re-check:

```bash
cat /etc/X11/default-display-manager
# If it says lightdm, put it back:
sudo systemctl disable --now lightdm
sudo systemctl enable gdm3
echo "/usr/sbin/gdm3" | sudo tee /etc/X11/default-display-manager
sudo reboot
```

GDM keeps running the local console/greeter on tty1; xrdp binds port 3389 independently. They coexist fine.

### Why not gnome-remote-desktop (`grdctl --headless`)?

We burned hours discovering this the hard way. The `--headless` daemon is not actually usable without a live GNOME session:

- **The daemon requires Mutter's DBus proxies.** In `grd-daemon-user.c`, `grd_daemon_user_is_ready()` blocks until `org.gnome.Mutter.RemoteDesktop` and `org.gnome.Mutter.ScreenCast` appear on the session bus. Those services only exist inside a running `gnome-shell` / Mutter instance — which requires a graphical session to already be logged in.
- Symptoms when this is missing: the systemd unit reports `active (running)`, the daemon holds its DBus name (`org.gnome.RemoteDesktop.Headless`), but the RDP server **never binds port 3389**. The DBus properties `Enabled=false, Port=-1` on `org.gnome.RemoteDesktop.Rdp.Server` are the giveaway. Logs show only the harmless `Init TPM credentials failed ... using GKeyFile as fallback` and then silence.
- **"Session creation inhibited"** in journald means an existing local graphical session is blocking a new remote one — they are mutually exclusive without `--system` mode.
- The `--headless` daemon in isolation also needs the caller to be in the `render` and `video` groups (see below); otherwise you get `libEGL warning: failed to open /dev/dri/renderD*: Permission denied`.

### Why not `grdctl --system` either?

The `--system` daemon is the "Remote Login" flow: it accepts a connection on 3389, PAM-authenticates the Linux user, spawns a fresh GDM/GNOME session, then hands the RDP socket off to the user-level `--handover` service. This is architecturally the right answer for headless — but on this box the **Microsoft Remote Desktop / Windows App on macOS** repeatedly failed to complete the handover. The daemon logs `[RDP] Sending server redirection` and immediately `ERRINFO_LOGOFF_BY_USER` from the client. Other clients (Windows mstsc, xfreerdp) may fare better, but the Mac client did not.

### GRD pitfalls we learned along the way

- `grdctl` operates on **three independent configuration namespaces**: user default (`grdctl ...`), headless (`grdctl --headless ...`), and system (`sudo grdctl --system ...`). Setting TLS cert / credentials / enable in one does **not** propagate to the others. Each daemon reads its own namespace from GSettings.
- The RDP username/password set with `grdctl set-credentials` is stored in `~/.local/share/gnome-remote-desktop/credentials.ini` as a plain GKeyFile. It is **not** your Linux password.
- Systemd **user manager caches supplementary groups at start**. After `sudo usermod -aG render,video $USER`, the daemon inside `user@1000.service` still sees the old groups until you `systemctl restart user@1000.service` (which kills your SSH session) or reboot. Verify with `cat /proc/$(pgrep -f gnome-remote-desktop-daemon)/status | grep Groups`.
- `/dev/dri/card*` and `/dev/dri/renderD*` normally get dynamic ACLs granting the *currently-logged-in graphical user* access. With no local session, only members of `video` (44) and `render` (990) can open them. `setfacl` fixes it temporarily but does **not** survive reboot — the group memberships do.
- **Windows App on macOS does not honour RDP server-redirection to alternate host/port pairs.** If your setup depends on the daemon telling the client to reconnect to `host:3390`, expect the Mac client to silently give up.
- Ignore the noise: `Init TPM credentials failed ... using GKeyFile as fallback` is harmless (no TPM present); `x509_utils_from_pem: BIO_new failed for certificate` before you've configured a cert path is harmless.

### xrdp pitfalls

- A `~/.xsession` that runs `exec gnome-session` under xrdp gives a black screen (`Wayland compositor does not support the Layer Shell protocol` in `~/.xsession-errors`). Use `exec startxfce4` instead.
- Only two session types were installed by default: `lightdm-xsession.desktop` and `xfce.desktop`. If you want GNOME-on-Xorg via xrdp, install a `ubuntu-xorg` session package first — otherwise stick with XFCE.
- The occasional `SSL_read: unexpected eof while reading` / `xrdp_sec_incoming failed` in `journalctl -u xrdp` is usually a Tailscale/mDNS/scanning probe hitting 3389 — not a real client failure. Cross-check with `journalctl -u xrdp-sesman` for actual login attempts.
