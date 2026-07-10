# WiFi 7 — MediaTek MT7927 on Ubuntu 26.04

**Date:** 2026-07-10  
**System:** MSI MEG X870E ACE MAX | Ubuntu 26.04 | Kernel 7.0.0-27-generic  
**Adapter:** MediaTek MT7927 802.11be 320MHz 2x2 PCIe (Filogic 380) — PCI ID `14c3:7927`

---

## Symptom

After installing Ubuntu 26.04, the WiFi adapter was detected by PCI but no wireless interface appeared. `ip link` showed only Ethernet, `iw dev` returned nothing.

**Initial false lead:** We first assumed it was a missing WiFi antenna (the antenna ships in the motherboard box and screws onto the gold connectors on the rear I/O). The antenna was installed, but the issue persisted — it was a driver problem, not physical.

---

## Root Cause

The stock `mt7925e` kernel driver (loaded by default) does not include PCI ID `14c3:7927` in its device table. The driver stack was present and loaded, but never claimed the MT7927 hardware.

Confirmed via:
```bash
modinfo mt7925e | grep alias
# Only showed 14c3:0717 and 14c3:7925 — no 7927
```

Firmware was already present at `/lib/firmware/mediatek/mt7925`, so it was purely a driver binding issue.

---

## Fix: Install MT7927 DKMS Driver

### Prerequisites

Connect via Ethernet or USB tethering first, then:

```bash
sudo apt update

sudo apt install -y \
  build-essential \
  dkms \
  linux-headers-$(uname -r) \
  git \
  python3 \
  curl \
  zstd
```

### Clone and Install the DKMS Driver

```bash
cd ~
git clone https://github.com/jetm/mediatek-mt7927-dkms.git
cd mediatek-mt7927-dkms
```

### Register, Build, and Install

```bash
sudo dkms add -m mediatek-mt7927 -v 2.13
sudo dkms build -m mediatek-mt7927 -v 2.13
sudo dkms install -m mediatek-mt7927 -v 2.13
```

Verify:
```bash
sudo dkms status
# Should show: mediatek-mt7927/2.13, 7.0.0-27-generic, x86_64: installed
```

### Load the New Driver

```bash
sudo modprobe -r mt7925e mt7925_common mt792x_lib mt76_connac_lib mt76
sudo modprobe mt7925e
```

---

## Verification

```bash
# Confirm driver is bound
lspci -nnk -s 0e:00.0
# Should show: Kernel driver in use: mt7925e

# Confirm wireless interface exists
ip link
# Should show wlp14s0 or similar

# Confirm wireless device
iw dev
# Should show: Interface wlp14s0, type managed

# Connect
nmcli dev wifi list
nmcli dev wifi connect "SSID" password "PASSWORD"

# Verify connection
iw dev wlp14s0 link
```

---

## Diagnostic Commands (if issues persist)

```bash
# RF kill state
rfkill list

# NetworkManager status
nmcli device

# Kernel logs
sudo dmesg | grep -iE 'mt792|mt76'
sudo dmesg | grep -i firmware
sudo dmesg | grep -iE 'wifi|wlan|firmware|mt79'
```

---

## Notes

- Ubuntu 26.04 has known WiFi issues generally — this is specific to the MT7927 but other adapters may also need manual driver work on 26.04.
- The WiFi antenna is a plastic fin that screws onto the gold SMA connectors on the motherboard's rear I/O panel — it ships in the motherboard box, not the case box.
- The USB drive that ships with the motherboard contains Windows drivers only, not Linux.
