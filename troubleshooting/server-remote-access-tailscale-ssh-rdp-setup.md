# Remote Access Setup — Tailscale + SSH + RDP

**Date:** 2026-07-10  
**System:** MSI MEG X870E ACE MAX | Ubuntu 26.04  
**Hostname:** kc-llm-MS-7E85

---

## Overview

The server is accessible remotely via Tailscale (mesh VPN). Direct LAN IP or public IP won't work for team members on different networks — Tailscale handles NAT traversal.

---

## Access Credentials

| Method | Address | User | Password |
|--------|---------|------|----------|
| SSH | `ssh kc-llm@<tailscale-ip>` | kc-llm | kh@nn@11mb0x |
| RDP | `<tailscale-ip>` (port 3389) | kc-llm | kh@nn@11mb0x |
| Hostname (on Tailscale network) | kc-llm-ms-7e85 | — | — |

---

## Setup Performed on Server

1. **SSH server** — installed and enabled (`openssh-server`)
2. **RDP** — installed `xrdp` (GNOME's built-in remote desktop had issues)
3. **Tailscale** — installed and authenticated on Varun's Google account (`vkhanna529@gmail.com`)

---

## Connecting as a Team Member

### 1. Install Tailscale

Download from [tailscale.com](https://tailscale.com/download) for your OS.

### 2. Join the Tailnet

You need an invite link from Varun (the Tailnet admin). Once you get one:
- Click the invite link
- Log in with Google (use whatever account — the invite grants access to the Tailnet)
- Varun must **approve** your machine in the Tailscale admin console

### 3. Register Your Machine

After installing and accepting the invite, your machine must show up in `tailscale status`. If it doesn't, you haven't successfully registered on the Tailnet.

```bash
# Check your Tailscale status
tailscale status
# The server (kc-llm-ms-7e85) should appear in the list
```

### 4. Connect

```bash
# SSH
ssh kc-llm@<server-tailscale-ip>

# Or by hostname if DNS is working
ssh kc-llm@kc-llm-ms-7e85
```

For RDP, use Microsoft Remote Desktop (or any RDP client) pointed at the server's Tailscale IP on port 3389.

---

## Troubleshooting

### "Connection timed out"
- You're probably not on the Tailnet, or your machine isn't registered
- Run `tailscale status` — the server must appear in the list
- If it doesn't, your machine hasn't joined the same Tailnet (check invite/approval)

### "No such host is known"
- Hostname resolution requires both machines on the same Tailnet
- Fall back to the Tailscale IP address directly

### Can't RDP without HDMI connected
- **Known issue** (as of 2026-07-10) — RDP doesn't work unless a display is physically connected
- Workaround: use an HDMI dummy plug, or SSH in instead
- TODO: Fix headless RDP

### "Permission denied" on SSH despite correct password
- SSH password auth can fail when the connection is non-interactive (e.g. piped input, scripts)
- The server requires an interactive TTY for password entry — tools that pipe the password in will get rejected even with the correct credentials
- **Fix:** Use SSH key-based authentication instead (see below)

### Setting Up SSH Key Auth (Recommended)

Key-based auth is more secure and avoids password issues entirely. Your private key never leaves your machine.

**On your local machine:**
```bash
# If you don't have a key yet, generate one:
ssh-keygen -t ed25519

# Copy your public key to the server (requires someone already on the server to run this):
# On the server, run:
mkdir -p ~/.ssh && echo '<your-public-key-here>' >> ~/.ssh/authorized_keys && chmod 700 ~/.ssh && chmod 600 ~/.ssh/authorized_keys
```

**Then connect with:**
```bash
ssh -i ~/.ssh/id_ed25519 kc-llm@<server-tailscale-ip>
```

To get your public key, run `cat ~/.ssh/id_ed25519.pub` (or whatever key file you have) and send it to someone who already has server access.

### "User invite already completed" on Tailscale
- Invite links are single-use — if someone else used it, you need a fresh one from the admin
- Ask Varun to generate a new invite from `login.tailscale.com/admin/users`

### "User approval required" on Tailscale
- After joining the Tailnet, the admin must approve your device
- Ask Varun to go to `login.tailscale.com/admin/users` and approve your pending account

### Common Mistakes
- Using LAN IP (192.168.x.x) — only works if you're on the same physical network
- Using the Tailscale IP without being on the Tailnet — will time out
- Not getting admin approval after accepting the invite — machine won't be reachable

---

## Notes

- Each team member needs their own Tailscale invite link (single-use)
- Varun is the Tailnet admin and must approve new devices
- The server's sudo password is the same as the login password
