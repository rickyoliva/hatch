# 03 — The lab

Fidelity over everything. This document is the part most write-ups skip, and it is the part that decides whether the programme produces anything.

## 3.1 Hardware

Current holdings and gaps live in `09-inventory.md`. This is the target state.

| Item | Purpose | Status |
| --- | --- | --- |
| Unpatched v1 | Primary lab. CFW, dumps, debug, golden source. | **Held** |
| OLED (Mariko, stock 23.0.1) | Target-population confirmation | **Held** — no image, daily driver |
| Jig + payload injector | RCM into the v1 | **Held** |
| Host PC (Ubuntu) | `nxlink`, `twili`, Ghidra, fuzzing, serial | **Held** |
| Raspberry Pi Pico 2 ×4 | USB personas, SD SPI-mode card, PIO sniffer | **Held** |
| XIAO nRF52840 Sense | NFC tag emulation, writing | **Held** |
| Miyoo Mini Plus | Delivery AP (`hostapd`, `dnsmasq`, nginx) | **Held** — delivery only, no injection |
| ESP32-S3 ×2 | Beacon/probe injection, second AP, USB persona | Buy, P1 |
| ESP32 Classic ×1 | BR/EDR controller emulation | Buy, gated on `hid` |
| SD interposer (ribbon, shifters, breakout) | `sdmmc` L1 | Buy parts, P1 |
| NTAG215 ×20 | amiibo baseline | Buy, P1 |
| Relay / inline switches | Unattended plug cycles | Buy, P0 |
| Logic analyser | SD and USB capture, ≥100 MS/s | Defer; PIO covers ≤25 MHz |
| USB meter, spare dock, SD cards | Rig infrastructure | Buy, P0 |
| MIG-class card | `gc` transport | Defer |

## 3.2 Software

- **CFW:** Atmosphere, Hekate. Hekate is also the partition/NAND tool.
- **Extraction:** `Lockpick_RCM` (v1 only), `NxNandManager`, `TegraExplorer`, `hactool`.
- **Debug:** `twili`, `dmnt`, `nxlink`, `sys-ftpd`, `ftpd`. Core dumps and `fatal` capture.
- **RE:** Ghidra plus `switchbrew`/IPSwitch symbol data where available. Never IDA-with-no-symbols as the only tool.
- **Emulators:** as pre-filters only. See 3.4.

## 3.3 Golden image discipline

1. Full NAND dump of every lab unit, raw, hashed, stored twice.
2. Boot, do the bare minimum, dump again — that is the **golden image** for that unit.
3. Every faulting run restores the golden image before the next.
4. Never let a fuzz run dirty a unit for longer than one batch.

A fuzz campaign that cannot be reset is a campaign that produces unreproducible results, and unreproducible results are the same as no results.

## 3.4 The fidelity ladder, and what emulators lie about

```
emulator (fast, wrong)  ->  reimplemented HLE (medium)  ->  hardware (slow, true)
```

Current emulators reimplement `hid`, `nfp`, `fsp-srv`, `ldn` and `usb` **in host code** — C#, Rust, C++. That means:

- Fuzzing `nfp` in an emulator finds bugs in *the emulator's* `nfp`, not in `nfp.nss`.
- Bug *classes* and protocol coverage transfer. Byte-exact behaviour and exact crash offsets do not.
- The guest ARM64 is JITed, so game-side parsers are faithful; sysmodule-side parsers are not.

Rules:

- Emulator is for **corpus generation, harness prototyping, and coverage mapping.** No crash is real until hardware reproduces it.
- Keep a "confirmed / emulator-only" flag on every crash. Never let them mix in a report.
- Hardware runs carry the Atmosphere patch profile. A patched `fsp` is a different binary; record the profile or the result is not comparable (ADR-014).

## 3.5 Harness patterns

| Target | Harness | Throughput |
| --- | --- | --- |
| Filesystem | NRO that takes an `.img` path, writes it to a loop-mounted FAT image or a real SD, calls `fs` mount, parses result. Auto-reboot via `twili`. | ~dozens/min |
| Album | Push `.jpg`/`.mp4` to `sdmc:/Nintendo/Album`, launch Album, poll `fatal` screen | ~2–5/min |
| USB | RP2040 presents a descriptor set from a slot in flash; plug-cycle via inline switch; capture bus with analyser | ~10/min |
| `nifm` | ESP32 AP replays a beacon/probe corpus from flash; console auto-joins a saved network | ~5/min |
| `nfp` | NTAG215 writer loop + tap jig, or an emulator tag | ~20/min |
| NSO | Stage ROM into the app's save dir, launch, capture | ~2/min |
| Kernel | Static only at first: diff, decompile, and count. Dynamic needs a userland primitive to call through. | n/a |

Cheap and dumb beats clever. The winning harness is the one that runs unattended overnight and leaves a numbered directory of crashes.

## 3.6 Crash triage

1. Capture: `fatal` report (title, program, PC/LR, stack, module offsets) + screen photo.
2. Dedupe on (module, PC, fault type, top 3 stack frames).
3. Classify: read/write, in-bounds of what, near which length field.
4. Reproduce cold: golden image, same input, three times.
5. Only then write it up.

A crash with no module and offset is a screenshot. A crash with a PC in `fsp` and a controllable write is a lead.

## 3.7 Throughput reality

The reboot cost dominates everything. Budget for:

- A hardware plug/reboot rig (relay or inline switch) on **every** unit.
- Boot-to-harness under 20 s via a launch config, not a menu.
- Parallel units. One unit is a hobby; six units is a lab.
- Batch size caps so a unit can be reset and hashed between batches.

If a campaign needs a human every ten minutes, it is not a campaign.

## 3.8 Safety

- Units sit on a VLAN with no route out during fuzzing.
- SD cards are imaged, never used for real saves.
- Two physically separate golden backups per imageable unit. SD cards die, and so do cheap eMMC.
- The OLED is offline except during tests and never receives persistence work early.
- Both consoles are on frozen 23.0.1. No update is accepted mid-batch.

## 3.9 What this lab cannot do

It cannot manufacture a kernel bug. It can only make one cheap to pursue once found. The kernel track (see roadmap, phase K) runs on static analysis and diffs and does not wait for a chain.
