# 09 — Inventory and gaps

Reality check against `05-roadmap.md`. Some of the assumed budget is already covered; two tracks are blocked and need a part under £10 each.

## 9.1 Held

| Item | Verdict | Track |
| --- | --- | --- |
| Raspberry Pi Pico 2 (RP2350) | **Excellent.** USB 1.1 device via TinyUSB, PIO for SD SPI-mode and protocol capture, 150 MHz dual-core, 520 KB SRAM, 4 MB flash. Becomes LAN adapter, GC adapter, wired controller, and slow-bus sniffer. | `usb`, `sdmmc`, `hid` |
| Seeed XIAO nRF52840 Sense | **Good, one trick only.** NFC-A / ISO14443A tag emulation (amiibo), USB HID, BLE, IMU and PDM mic for motion/audio paths. No BR/EDR, so it cannot be a Pro Controller or Joy-Con. | `nfp`, `usb`, audio/motion edges |
| Switch OLED (jailed) | **Target, but cannot be imaged.** Mariko, stock, 23.0.1. Right family, wrong risk profile — see 9.4. | Confirmation only |
| Switch v1 RCM (jailbroken) | **The lab.** CFW, `dmnt`/`twili`, keys, NAND source, golden images, cold reproduction. Same parser builds as Mariko for a given firmware. Also the GameShare host bring-up platform: 23.0.1 carries `NGC_services` and `splay`. | All, GameShare |
| macOS / Windows / Ubuntu | Ubuntu is the rig host. macOS is analysis and second opinion. Windows is the second enumeration host for USB differential work. | `usb` differential |
| Miyoo Mini Plus (OnionOS) | **Delivery AP, not a fuzzing radio.** Linux, `hostapd`/`dnsmasq`/nginx, pocketable, battery powered. Realtek SDIO part: weak monitor mode, no reliable injection. | WebKit delivery, AP setup |
| AI API credits | Triage and harness generation. Allocation in 9.6. | `04-methods` §4.5 |
| `nxnative` | Conditional. Useful only if it beats `nxlink` + `sys-ftpd` for staging and launch. It does not replace `twili` for crash capture. | Harness QoL |

## 9.2 Hard-blocked tracks

| Track | Blocked by | Cheap unblocker |
| --- | --- | --- |
| `nifm` frame-level fuzzing | The Miyoo is a delivery AP only: no monitor mode, no injection | ESP32-S3 (~£8). Delivery itself is *not* blocked |
| `hid` Bluetooth | No BR/EDR anywhere. nRF52840 is BLE-only and Joy-Con/Pro Controller are Classic. **ESP32-S3 is also BLE-only** | Plain ESP32-WROOM/WROVER (~£7) |
| `sdmmc` L1 interposer | No microSD breakout, no level shifters, no bus mux | ~£30 in parts plus the Pico 2 |
| `gc` gamecard | No card, and the card protocol needs an ASIC handshake the Pico can't fake | MIG-class card, ~£50, or defer |
| Cold-reproduce rigs | No relay or inline USB switch | ~£12 of relay boards |
| Bus visibility | No logic analyser | Pico 2 PIO capture covers ≤25 MHz; a £120 analyser covers SD UHS |

**Get the ESP32-S3 pair first.** It unblocks `nifm` and gives a second USB persona platform for under £20. The Classic board is a separate, smaller buy gated on the `hid` track.

## 9.3 Missing at every priority

| Priority | Item | Why |
| --- | --- | --- |
| P0 | 2× spare SD cards, imaged and hashed | Fuzz media, never real saves |
| P0 | USB-C to USB-A adapters / a spare dock | USB host path needs a real A-port |
| P0 | Relay / inline switch boards | Unattended plug cycles |
| P0 | Second NAND backup medium | One backup is not a backup |
| P1 | ESP32-S3 dev board ×2 | Wi-Fi delivery and injection; second USB persona board |
| P1 | ESP32 Classic ×1 | BR/EDR only, gated on the `hid` track |
| P1 | microSD extension ribbon ×2, level shifters, µSD breakout | `sdmmc` interposer |
| P1 | NTAG215 stickers ×20 | Amiibo baseline; nRF emulation will be flakier |
| P2 | HDMI capture | Watch `fatal` remotely without a camera pointed at a screen |
| P2 | Spare patched unit (Lite) | The OLED is a daily driver |
| P2 | Second dock | Keep one wired permanently to the rig |

## 9.4 The OLED problem

It is stock. Stock means:

- **No NAND dump.** You cannot golden-image a console without CFW. There is no restore path for a corrupted system state, only maintenance mode and a re-update.
- **It is a daily driver.** Every fault is on the console someone wants to play.

Policy:

1. **v1 develops, OLED confirms.** No track is called working until it reproduces on the OLED *cold*, but nothing destructive is tried there first.
2. **No persistence work on the OLED** until T1 is stable on the v1 across ten cold boots.
3. For a userland-only chain the brick risk is genuinely low — a crash is a reboot. Say so plainly, but do not spend the console on curiosity.
4. Buy a cheap patched Lite when the budget allows. Confirmation on a unit that is not your daily driver is worth more than a faster analyser.

## 9.5 Firmware — resolved

Both consoles are on **23.0.1**. Parity holds, so the fidelity model is intact: one parser build, one diff baseline, and the OLED is a valid confirmation target.

Two facts follow, and the second is the better one:

- 23.0.1 is currently the newest release, so the corpus has only a near end. Older versions must be sourced and the K0 baseline built before the first update lands.
- 23.0.1 is what the current Switch 2 jailbreak supports. A chain found here plausibly ports, which raises both the payoff and the cost of losing the version.

Freeze both. `10-v1-prep.md` §0 is the operational rule.

## 9.6 AI credit allocation

| Use | Share | Gate |
| --- | --- | --- |
| Decompiled-function triage and sink ranking | 45 % | Every offset marked `VERIFY` against the binary |
| Harness and mutator generation | 30 % | Must compile and run before it counts |
| Diff queue labelling (is this a security change?) | 15 % | Human adjudicates every accept |
| Corpus synthesis and format description | 10 % | Differential oracle filters output |

Rules: batch, cache prompt prefixes, never send a whole binary, keep a pinned model per task so results are comparable across weeks.

## 9.7 `nxnative`

Conditional yes. Two tests decide it:

1. Does it stage a file and launch an NRO over the network faster than `nxlink` + `sys-ftpd` + a launch config?
2. Does it survive an unattended reboot loop without a human?

If either is no, it is not worth the shelf space. It is QoL for P2, never a dependency, and it is not a substitute for `dmnt`.

## 9.8 Fleet layout

| Item | Role | Modded | Reset rig |
| --- | --- | --- | --- |
| v1 #1 | Development, dumps, keys | Yes | Relay |
| v1 #2 (when bought) | Parallel campaign | Yes | Relay |
| OLED | Confirmation only | No | Manual |
| Pico #1–#3 | USB personas (LAN, GC, HID) | n/a | Inline switch |
| Pico #4 | SD SPI-mode card, then sniffer | n/a | n/a |
| XIAO #1 | NFC tag emulation | n/a | n/a |
| Miyoo Mini Plus | Delivery AP | n/a | n/a |
| ESP32-S3 (to buy) | Frame injection, second AP, USB persona | n/a | Inline switch |
| ESP32 Classic (gated) | BR/EDR controller | n/a | n/a |

## 9.9 Firmware position

Both consoles are on **23.0.1** and frozen. Parity is achieved, so the lab/target split is clean and the diff baselines are identical.

Two consequences:

1. 23.0.1 is currently the newest release, so the diff corpus has only a *near* end. Earlier versions must be sourced and the K0 baseline table built before the first update.
2. 23.0.1 is also the firmware the current Switch 2 jailbreak targets. A chain that lands here is plausibly portable, which raises the value of every result.

Aging firmware is now the main schedule risk: on first sight of a successor, archive the corpus and take both consoles offline.

## 9.10 Verdict

The boards are well chosen and cover the top three tracks. The console pair is thin, not because the OLED is wrong — it is the right family — but because it cannot be imaged and doubles as a daily driver. Firmware parity means the fidelity model holds; the OLED risk remains a *recovery* risk, not a matching one.

Nothing here is a reason to stop. The ESP32-S3 pair, the interposer parts and a spare patched unit close every gap that currently blocks a tier-A track, in that order.
