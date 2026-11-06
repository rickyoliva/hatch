# Shopping list — capability hypotheses, not a parts order

**Read this before buying anything.** Every line is a claim of the form "this part is the cheapest way to reach *this specific code path*." If you don't understand the path, the part becomes a drawer ornament. Each entry therefore carries what it unlocks, what it does **not** unlock, what must be verified first, and why it might not be worth it.

Tiers are purchase order, not importance. Nothing in T3 is urgent.

---

## T0 — Verify before spending

These cost nothing or nearly nothing and decide whether whole tracks exist.

### V1. Can the OLED take USB-C→A adapters and a third-party dock?
Third-party docks and adapters are widely reported to work but are not guaranteed on latest firmware. If host-mode peripherals don't enumerate on the OLED, the USB host track is v1-only and the "confirm on target" story weakens.

- Cost: one adapter you probably own
- Unlocks: confidence in the USB track
- Doesn't unlock: anything by itself
- Test: attach adapter, attach a Pico 2 as a LAN adapter, see if it enumerates

### V2. Can the Miyoo Mini Plus run `hostapd` in AP mode?
It's a Linux handheld with an 8188-class Realtek SDIO radio. As a delivery AP it's genuinely good — pocketable, battery powered, runs `dnsmasq` and nginx. As an *injection* radio it's weak: no reliable monitor mode, so no beacon/probe-response corpus replay.

- Cost: £0
- Unlocks: portable delivery for the WebKit track
- Doesn't unlock: 802.11 IE fuzzing, frame injection
- Test: `hostapd` up, phone associates, DHCP leases, captive page served

### V3. Does the nRF52840 sustain NTAG215 emulation?
Nordic's NFC peripheral does ISO14443A tag emulation, but amiibo wants a full 540-byte read at speed. If it times out, the amiibo track needs physical stickers.

- Cost: £0
- Unlocks: programmable amiibo, no stickers
- Doesn't unlock: reliable timing without testing
- Test: dump a real tag, replay it from the XIAO, see if the console reads it end to end

### V4. Does Nintendo's connectivity probe stay plaintext on 23.0.1?
Decides whether the Miyoo is an AP or a link-delivery device only.

- Cost: £0
- Unlocks: the whole network delivery story, or kills it
- Test: packet capture against the Miyoo AP during a stock console's network check

---

## T1 — Buy now

### 1. ESP32-S3 dev board (×2)
Wi-Fi + BLE 5, dual core, USB-OTG, plenty of RAM. This is the AP/injection and USB-persona board.

- Why: beacon and probe-response generation, malformed IE corpora, and a portable AP that can actually inject. Also a second USB device persona platform.
- Unlocks: `nifm` corpus work at frame level; AP delivery independent of the Miyoo
- **Does not unlock: Joy-Con or Pro Controller emulation.** The S3 has no Classic Bluetooth.
- Verify first: none, but confirm your toolchain supports raw 802.11 injection on it
- Why not: some S3 modules ship with an external antenna connector and some don't; pick one with a u.FL
- Cost: ~£8 each

### 2. ESP32 Classic dev board (×1) — only if the Bluetooth sub-track matters
BR/EDR is required for controller emulation. Plain ESP32-WROOM or WROVER has it; the S3 does not.

- Why: the only cheap path to a fake Pro Controller / Joy-Con
- Unlocks: `hid` Bluetooth and the pairing state machine
- Doesn't unlock: anything Wi-Fi-worthy
- Verify first: T0/V2 is unrelated; this is gated on the `hid` track being promoted above opportunistic
- Why not: `hid` is the lowest-scoring surface in the matrix. Buy only if you want the controller track
- Cost: ~£7

### 3. microSD extension ribbon (male→female, ×2)
The single part that makes the interposer buildable without a custom PCB. It gets the card bus out of the console's slot and into a breadboard.

- Why: there is no way to interpose on an SD bus without breaking it out
- Unlocks: `sdmmc` L1 (CID/CSD/SCR, CMD sequencing, tuning)
- Doesn't unlock: L2 filesystem work, which needs no hardware
- Verify first: pinout matches the console's slot and the ribbon is thin enough to seat
- Why not: fragile and rated for few insertions; buy two
- Cost: ~£8

### 4. Level shifters (TXS0108E or 74LVC245, ×4) + µSD breakout (×2)
3.3 V bus, but the console and card sides need isolation and the analyser needs taps. Without these, the ribbon is just a cable.

- Why: electrical safety and capture points
- Unlocks: the rest of the interposer
- Verify first: voltage tolerance of the console's SD lines at 1.8 V during UHS — **1.8 V is the trap, check it**
- Cost: ~£10

### 5. NTAG215 stickers (×20)
Physical tags, written by the v1 with public amiibo keys.

- Why: nRF emulation will be flaky until proven; stickers are the baseline
- Unlocks: the `nfp` track immediately, with no timing risk
- Doesn't unlock: fast iteration — each write is a physical tag
- Verify first: none
- Cost: ~£10

### 6. Relay boards / inline USB switches (×4)
Unattended plug cycles. Without them every USB or card experiment needs a human.

- Why: throughput is the whole game and a human is the bottleneck
- Unlocks: overnight batches on `usb`, `sdmmc`, `nfp`
- Doesn't unlock: SD power cycling through the console's own slot — that needs the interposer
- Cost: ~£12

### 7. Spare SD cards (×4, 32–64 GB, decent endurance)
Fuzz media only, never real saves.

- Why: fuzz writes destroy cards
- Unlocks: parallel campaigns
- Cost: ~£15

### 8. USB-C→A adapters (×2) and one spare dock
Host-mode peripherals need a real A-port, and one dock should live on the rig permanently.

- Why: the rig must not be disassembled for normal use
- Unlocks: USB host track
- Verify first: T0/V1
- Cost: ~£35

---

## T2 — Buy when a track earns it

### 9. HDMI capture dongle
- Why: watch `fatal` screens from the host without a camera on a tripod
- Unlocks: unattended fault capture on the OLED, which has no debug channel
- Why not yet: the v1 has `dmnt`/`twili`; the OLED is confirmation-only
- Cost: ~£35

### 10. Spare patched console (Lite)
- Why: the OLED is a daily driver and cannot be imaged
- Unlocks: a confirmation target you can afford to break
- Why not yet: fix the safety protocol first; the OLED is adequate while the v1 carries development
- Cost: ~£120

### 11. Logic analyser (≥100 MS/s)
- Why: a hang with no bus capture is unexplainable, and Pico PIO capture tops out around 25 MHz
- Unlocks: SD UHS and USB timing visibility
- Why not yet: PIO capture is adequate for the 12.5 MHz work at the start of the interposer track
- Cost: ~£120

### 12. MIG-class card
- Why: the only realistic route to `gc`
- Unlocks: gamecard interface parsing
- Why not yet: `gc` is tier B and needs a cloned cert
- Cost: ~£50

---

## T3 — Nice to have

- Bench supply with current limit — catches a shorted interposer before it costs a console.
- USB power meter — brownout and current profiling.
- Second v1 — parallel campaigns without sharing a unit.
- Extra Pico 2 — one per persona, no reflashing between runs.

---

## Deliberately not on the list

| Thing | Why not |
| --- | --- |
| Modchip | The point of the exercise, and it would mask every result |
| Flipper Zero | Covers NFC and sub-GHz, but the nRF52840 already does the NFC we need |
| SD-mux ICs | The console-side bus is not a clean mux target; interpose instead |
| A dedicated fuzzing server | Ubuntu on existing hardware is sufficient at this scale |
| ESP32-CAM | Camera is worse than a phone and the pins are consumed |

---

## The rule

Buy in the order the roadmap kills or promotes tracks, not in the order the list is written. If the filesystem track turns out to be a passthrough, the interposer parts become urgent and the ESP32s do not. If `nifm` proves to be the richest surface, the reverse.

Every purchase should be traceable to a named function you want to reach.
