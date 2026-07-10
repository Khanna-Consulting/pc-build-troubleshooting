# PC Build Troubleshooting Log

**System:** MSI MEG X870E ACE MAX | AMD Ryzen 9 9950X | Corsair Vengeance RGB DDR5 6000MHz (4x sticks) | PNY GeForce RTX (XLR8) | Corsair RM1200e PSU | HYTE case with HYTE AIO cooler

**Symptom:** No display output on first boot (motherboard HDMI and GPU HDMI both tried), debug display showing various early POST error codes, system never completing POST.

---

## Root Cause (found at the end)

**A bent CPU socket pin.** One of the pins in the AM5 socket was physically bent, preventing proper electrical contact with the CPU. This caused the system to hang very early in POST — before memory training, before CPU initialization could complete — no matter what else was changed (RAM, cables, BIOS).

---

## Troubleshooting Timeline

### 1. Initial symptom: No display, debug code "C5"
- Tried HDMI into motherboard, then into GPU — neither produced video.
- Debug display showed **C5**, which pointed to early memory initialization stage.
- **Action taken:** Recommended clearing CMOS, reseating RAM, checking CPU power connector.

### 2. Debug code progressed: "15" → "0A"
- Code changed over subsequent boot attempts, suggesting some progress through POST stages, but system still never displayed video or fully booted.
- **0A** was identified (via AMI POST code reference) as *"initialization after microcode loading"* — a very early CPU/SEC-phase stage, occurring **before** memory training begins.
- This shifted suspicion away from RAM and toward CPU/motherboard/power issues.

### 3. Ruled out: RAM configuration
- Tested with all 4 sticks installed.
- Tested with a single stick in slot A2 (correct single-stick slot) — still hung at 0A.
- Tested with a single stick in slot A1 (incorrect slot) — produced **EC** code with CPU (red) and DRAM (yellow) EZ Debug LEDs lit, which is expected/normal behavior for an unsupported single-stick configuration, not a new fault.
- Tested with zero RAM sticks installed — produced code **10** ("pre-memory CPU initialization started"), consistent with expected behavior for no RAM present.
- **Conclusion:** RAM sticks, RAM compatibility (Corsair Vengeance RGB DDR5 6000MHz — confirmed well within the board's supported spec, up to 8400+ MT/s OC), and slot choice were not the cause.

### 4. Ruled out: Cabling
- Verified CPU EPS (8-pin) cable was correctly identified and distinct from PCIe cables (the Corsair RM1200e uses a shared "PCIe/CPU" PSU-side port label, but the CPU and PCIe cables themselves are different at the device end — 8-pin solid EPS vs. 6+2 splittable PCIe).
- Confirmed correct cables were running to CPU and GPU.
- **Conclusion:** Cabling was correct and not the cause.

### 5. Ruled out: CPU power delivery
- Checked that both CPU_PWR connectors on the board (it has two 8-pin EPS connectors) were populated.
- **Conclusion:** CPU power delivery was not the cause.

### 6. BIOS Flashback performed
- Given the early-stage hang pattern (CPU/microcode level) persisting across multiple RAM configurations, suspected the board's out-of-box BIOS didn't have full microcode support for the Ryzen 9 9950X.
- Downloaded latest BIOS from MSI's support page, renamed to `MSI.ROM`, placed on a FAT32 USB drive.
- Located the Flash BIOS Button and its dedicated USB-A port on the rear I/O panel.
- Performed BIOS Flashback: held button, waited for LED blink cycle to complete (several minutes).
- **Result:** Flash completed successfully, but the system still hung at 0A afterward with the DRAM light lit — **BIOS was not the actual root cause**, though it was a reasonable and necessary step to rule out.

### 7. Explored and ruled out: Multi-BIOS switch (BIOS_SW1) and other jumpers
- Confirmed BIOS_SW1 physically exists on the board (per manual, located near the onboard Power/Reset buttons), but could not visually confirm "A" vs "B" position from photos, and since the system never reached BIOS setup, this couldn't be confirmed in software either.
- Given the system wasn't reaching BIOS at all, this lead was deprioritized as unlikely to be checkable or relevant.

### 8. Considered and set aside: Case standoff short, thermal paste, cooler mounting
- Discussed case standoff shorts and bench-testing outside the case as a way to isolate case-related wiring issues — not ultimately needed once the real cause was found.
- Confirmed thermal paste and a minor fingerprint on the CPU contact pad were not likely causes of a no-POST condition on their own, but cleaned the CPU contact pads with 99% isopropyl alcohol as good practice regardless.

### 9. Root cause found: Bent CPU socket pin
- Close visual inspection of the AM5 socket (under magnification/bright light) revealed a bent pin near the socket's edge (marked "3C" corner).
- This explains the entire symptom pattern: the bent pin prevented reliable electrical contact between the CPU and socket, causing POST to hang consistently at early CPU-initialization stages regardless of RAM configuration, cabling, or BIOS version.

### 10. Fix
- The bent pin was carefully straightened using a non-conductive tool (wooden toothpick recommended over a mechanical pencil with lead, since graphite is conductive and could cause a short).
- After straightening the pin and reseating the CPU, the system successfully **POSTed** (debug code reached **00**, indicating a clean POST).
- System then displayed a **BOOT** LED (green), indicating no OS/bootable device was detected — expected behavior for a fresh build with no OS installed yet, not a hardware fault.

---

## Follow-up / Next Steps Discussed
- Install an OS (Windows) via bootable USB installer to resolve the "no boot device" state.
- Investigate a GPU overheating warning that appeared once the system began booting (cause not yet isolated — need to confirm whether it's a real thermal issue, a sensor/software glitch, or fan/airflow related).
- Add 3 additional HYTE FA12 case fans, connected via chained 1-to-2 PWM splitters into a single SYS_FAN2 header on the motherboard.

---

## Key Lesson
A hang at very early POST codes (pre-memory, CPU-initialization stage) that persists across different RAM configurations, confirmed-correct cabling, confirmed CPU power, and even a full BIOS reflash should raise suspicion toward a physical CPU/socket issue (bent pins, poor seating) rather than continuing to chase software/firmware causes. A close, well-lit visual inspection of the socket pins is a relatively quick check that can save a lot of troubleshooting time if done earlier in the process.
