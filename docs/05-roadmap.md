# 05 — Roadmap, milestones, kill criteria

Clock runs from phase-0 exit. Weeks are effort-weeks for two people; one person roughly doubles them.

## Phases

### P0 — Lab (weeks 1–2)

Execute `10-v1-prep.md` in full on the v1: two verified NAND images, keys, `PRODINFO` ×2, emuMMC, golden image, reset rig. Then build the client side per `03-lab.md` and `09-inventory.md`: Pico 2 personas on inline switches, XIAO as tag writer/emulator, Miyoo as delivery AP.

Source the earlier firmware versions here. Only 23.0.1 is held; K0 needs a range, so four consecutive versions ending at 23.0.1 become the corpus, hashed and read-only.

**Exit:** a faulting input is injected, captured and cold-reproduced with no human for a 30-minute run on the v1; the reset rig restores golden unattended; the corpus is archived.
**Kill:** none — P0 is mandatory. Nothing is fuzzed before the two-image check passes.

**OLED:** prepared but not imaged. It enters at P3 as a confirmation target only.

### K0 — Kernel track, static (starts week 1, parallel, never stops)

The diff oracle (`04-methods.md` §4.1) over `Kernel`, `spl`, `pm`, `ldr`, `ro`, `fs`. Build the patch-queue and the revert rig.

**Exit:** ≥20 confirmed validation changes with reproductions, or F1 fires. F1 now needs sourced historical versions as well as 23.0.1.
**Kill (F1):** no security-relevant validation change across four versions → re-scope to T0-only and say so publicly.

### P1 — Surface mapping (weeks 3–4)

Ghidra pass over `fsp`/`sdmmc` and `usb`, plus `nifm`. Map entry functions, length fields, copies, and the fault handlers. Output is a *target map*, not a bug list: which functions are reached by which bytes, in what order.

**Exit:** every tier-A entry point has a named function, a call graph to the copy, and a harness design.
**Kill:** a tier-A surface turns out to be a thin passthrough to a hardened shared library with no Nintendo-written parsing. Drop it.

### P2 — Harnesses and crash corpus (weeks 3–7)

Build in this order: filesystem mount → USB descriptors → `nifm` IEs. Run nightly. Triage daily. Tracks whose parts have not arrived stay in mapping until they do.

**Exit:** ≥5 cold-reproducible crashes per active track, or a track is killed.
**Kill (F3):** six weeks, zero reproducible crashes on a track → park it, publish the negative result.

### P3 — Primitive (weeks 6–10)

Take the best three crashes. Push each up the ladder in §4.8: controlled fault → controlled length → controlled write → leak.

**Exit:** one leak (ASLR break) in a tier-A process, or one controlled write of ≥16 bytes at a chosen offset.
**Kill:** all crashes stall at controlled fault for three weeks → the surface has bugs but no depth; publish and rotate.

### P4 — Chain (weeks 9–14)

Entry + leak + write + K0 bug. Confirm on a second unit and a third firmware.

**Exit:** T1 on two units, cold, from golden image, following written notes someone else can execute.
**Kill:** K0 has no confirmed bug by week 14 → ship the best T0 chain instead.

### P5 — Publish (weeks 14–16)

Write-up, harnesses, corpus, disclosure note to Nintendo, coordinated release.

## Milestones

| ID | Week | Falsifiable statement |
| --- | --- | --- |
| M0 | 2 | Unattended 30-min run, two units, cold replay |
| M1 | 4 | All tier-A entries mapped with harness designs |
| M2 | 7 | 5 cold crashes per live track |
| M3 | 10 | One leak or one ≥16-byte controlled write |
| M4 | 12 | T0 chain end-to-end on one unit |
| M5 | 14 | T1 on two units, both 23.0.1 |
| M6 | 16 | Published |

## Risk register

| Risk | Likelihood | Impact | Mitigation |
| --- | --- | --- | --- |
| Kernel bug scarcity (F1) | High | Kills T1 | K0 starts week 1; T0 fallback pre-agreed |
| Emulator/HLE mismatch wastes weeks | High | Medium | Hardware confirmation gate; emulator-only flag |
| Nintendo patches mid-campaign | High | High | 23.0.1 is frozen; archive the corpus now; both consoles offline except during tests |
| HTTPS/CA blocks network delivery | Medium | Medium | Physical channels are the primary story; network is upside |
| Interposer build eats the schedule | Medium | Medium | Ribbon + level shifters on a breadboard first; no custom PCB until a fault appears |
| Cert/key dependency (gamecard, amiibo) | Low | Low | Both are public or cloneable; document the constraint |
| Fuzz noise with no depth | High | Medium | Differential oracle as filter; kill criteria are real |
| Single-person burnout | Medium | High | Publish negative results; the programme is legible even if it fails |
| OLED is the only patched target and cannot be imaged | High | High | Confirm only on it; buy a spare Lite at P2 |
| Nintendo ships a 23.0.1 successor mid-campaign | High | High | Archive the 23.0.1 corpus now; both consoles offline except during tests; watch the release feed |

## Budget (revised against held kit, `09-inventory.md`)

| Item | Status | Approx |
| --- | --- | --- |
| v1 RCM (lab) | Held | — |
| OLED (confirmation) | Held | — |
| Pico 2 ×4 | Held | — |
| XIAO nRF52840 Sense | Held | — |
| ESP32-S3 dev board ×2 | **Buy, P1, injection and delivery** | £16 |
| ESP32 Classic ×1 | Buy only if `hid` is promoted | £7 |
| microSD ribbon, level shifters, µSD breakout | **Buy, P1** | £30 |
| NTAG215 ×20 | **Buy, P1** | £10 |
| Relay boards, inline switches | Buy, P0 | £12 |
| Spare SD cards | Buy, P0 | £15 |
| USB-C→A adapters, cables, spare dock | Buy, P0 | £35 |
| Logic analyser | Buy or substitute with Pico PIO | £0–120 |
| HDMI capture | Defer to P2 | £35 |
| Spare patched Lite | Defer to P2 | £120 |
| MIG-class card | Defer, `gc` only | £50 |
| **Committed now** | | **~£125** |

Committed spend drops from ~£790 because the boards and one target are already owned. The S3 pair plus the interposer parts are the only purchases that unblock a live track.

## Sequencing note

K0 is deliberately first and constant. Everything else is a delivery mechanism for it. If only one track can run, it is K0 plus whichever entry track already has a crash.
