# 08 — Decision log

One ADR per structural decision. Append-only; supersede rather than rewrite.

## ADR-001 — Phase 0 is documentation only

**Status:** accepted.
**Context:** the user asked for discussion and planning before coding.
**Decision:** no harness or exploit code until P0 exit. Phase 0 produces the thesis, matrix, lab spec, dossiers and roadmap.
**Consequence:** the first code in the repo will be harnesses, not exploits.

## ADR-002 — Hardware in the loop is mandatory for confirmation

**Status:** accepted.
**Context:** emulators reimplement sysmodules in host code and lie about offsets.
**Decision:** every crash carries an `emulator-only` flag until a cold hardware reproduction exists. No report mixes the two.
**Consequence:** emulator throughput is a pre-filter, never evidence.

## ADR-003 — Kernel track starts at week 1, not after an entry

**Status:** accepted.
**Context:** the bottleneck is kernel bug scarcity, not entry-point scarcity. Everyone else's sequencing (entry first, kernel later) leaves the long pole to the end.
**Decision:** K0 runs in parallel from week 1 and never stops.

## ADR-004 — Rank by chain contribution, not bug density

**Status:** accepted. Supersedes the earlier "WebKit first" ordering.
**Context:** WebKit has the highest density and the lowest privilege; sysmodule bugs outrank applet bugs even at lower density.
**Decision:** the `pivot` and `privilege` dimensions carry the ranking. WebKit becomes a reserve.
**Consequence:** tier A is SD, USB, `nifm`, NSO. This is deliberately contrarian.

## ADR-005 — No raw save transplant in the first phase

**Status:** accepted.
**Context:** per-console `save` key and `save_mac_key` make transplant target-key-dependent, and patched units cannot give keys without homebrew.
**Decision:** deprioritise saves. Treat save formats as interesting only via in-game import or server mediation.
**Revisit:** if Q1 shows validation happens late.

## ADR-006 — Physical channels are the primary story

**Status:** accepted.
**Context:** stock trusts only the system CA store, so HTTPS MITM needs a Nintendo name or a weak validator.
**Decision:** SD, USB, amiibo, card and wireless are the road. Network delivery is upside, not a dependency.

## ADR-007 — Each lab unit has a golden image and a reset rig

**Status:** accepted.
**Context:** unreproducible crashes are worthless.
**Decision:** NAND dump, boot-minimum dump, hash, duplicate storage, restore between batches.

## ADR-008 — Negative results get published

**Status:** accepted.
**Context:** the scene's main failure mode is silent abandonment, which reads as "nothing there".
**Decision:** a killed track produces a write-up with the harness and the negative result.

## ADR-009 — Tier-A shortlist: SD, USB, `nifm`, NSO

**Status:** accepted.
**Context:** composite scores plus build cost.
**Decision:** build order SD → USB → `nifm`; NSO on standby until a track yields or dies.
**Supersede if:** F1 or F3 fires.

## ADR-010 — Two-person programme, not a crowd

**Status:** accepted.
**Context:** harness quality collapses as contributor count rises and coordination cost tracks the square.
**Decision:** no more than two active writers on the harness tree. External help arrives at triage or on a disjoint docket.

## ADR-011 — Firmware 23.0.1 is frozen on both consoles

**Status:** accepted.
**Context:** both are on 23.0.1, which is also the current Switch 2 jailbreak target. A successor invalidates every diff baseline.
**Decision:** archive the 23.0.1 corpus before any fuzzing, keep both consoles offline except during tests, and never accept an update mid-batch.
**Consequence:** `10-v1-prep.md` §0 and §12 are mandatory before hardware work.

## ADR-012 — The Miyoo is a delivery AP; ESP32-S3 is the injecting radio

**Status:** accepted.
**Context:** the Miyoo runs `hostapd` and is pocketable, but its Realtek SDIO part has no reliable monitor mode, so it cannot replay malformed 802.11 frames.
**Decision:** use the Miyoo for delivery and for the Q21/Q22 probe tests. Buy an ESP32-S3 for frame-level work. The S3 is Wi-Fi + BLE only, so BR/EDR controller emulation needs a separate Classic board and stays gated on the `hid` track.

## ADR-014 — Patch profile is part of every result

**Status:** accepted.
**Context:** Atmosphere patches `fsp` in memory. A crash found through a patched build may not exist on the stock OLED.
**Decision:** record the loader/`fs` patch profile with every crash, disable mount-affecting patches for `fsp`/`sdmmc` campaigns, and re-run any promoted crash under the stock profile.
**Consequence:** no crash report is valid without its profile attached.

## ADR-013 — The shopping list is rationalised, not itemised

**Status:** accepted.
**Context:** the user does not want a drawer of unexplained parts.
**Decision:** every entry in `shopping_list.md` carries what it unlocks, what it does not, what must be verified first, and why it may be deferred. Purchase order follows track deaths and promotions, not list order.

## ADR-015 — GameShare is a reserve, not a purchase trigger

**Status:** accepted.
**Context:** PC-hosted GameShare is the only no-content, host-driven surface on a stock 23.0.1 console, but it needs a Switch 2, a compatible title and an injecting radio. No public work exists.
**Decision:** add `06-dossiers/gameshare.md`, keep the track reserve, and borrow a Switch 2 before buying one. Start only if a tier-A track dies or a no-cartridge delivery story is needed. G1 (room replay from a PC) is the only gate worth funding until it passes.
**Consequence:** no new budget line; G0/G1 are the whole feasibility question.
