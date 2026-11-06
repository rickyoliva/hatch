# 10 — v1 preparation and safety protocol

Hands-on, in order. Do not skip; do not reorder. The v1 is the lab, the only image source, and (with one NAND) a single point of failure.

Target firmware: **23.0.1**, both consoles. Freeze it. See §0.

## 0. Preconditions

- [ ] Both consoles confirmed on 23.0.1, recorded in `09-inventory.md`.
- [ ] Auto-update disabled on both. The OLED sleeps and will otherwise jump to 23.0.2 mid-campaign.
- [ ] OLED kept offline except during tests.
- [ ] Battery above 60 % on the v1. Charge during long phases.
- [ ] Jig and a payload injector ready.
- [ ] Two blank SD cards, FAT32, labelled `V1-WORK` and `V1-GOLDEN`.
- [ ] Linux host with `nxlink`, `twili`, `sys-ftpd`, `hactool`, `NxNandManager`, Ghidra.
- [ ] Two separate backup destinations, different physical drives, encrypted.

## 1. Record

- [ ] Serial, model, region, purchase date, battery health, existing wear.
- [ ] Exact firmware string and build ID.
- [ ] Hekate and Atmosphere versions installed later, recorded to the notes file.
- [ ] Photograph the console and the label.

## 2. Payload card

- [ ] `hekate.bin` at the card root.
- [ ] `Lockpick_RCM.bin` on the card.
- [ ] `TegraExplorer.bin` on the card.
- [ ] Verify each payload hash against upstream releases before booting anything.

## 3. First boot into RCM

- [ ] Enter RCM, inject Hekate.
- [ ] Do **not** create emuMMC yet. Do not mount anything read-write.
- [ ] Note the eMMC health and the partition table.

## 4. Full raw dump — twice

- [ ] Full raw NAND/eMMC image to `V1-GOLDEN-DUMP-A`.
- [ ] Hash it (`sha256sum`), record the hash in the notes file.
- [ ] Dump again to destination B on a different drive.
- [ ] Hash B and confirm equal.
- [ ] Verify A and B both end in the same trailing bytes and the same size.
- [ ] Copy both off the console and make them read-only.

**Do not proceed until two independently verified images exist.**

## 5. Keys

- [ ] Boot `Lockpick_RCM` from RCM (payload, not from CFW).
- [ ] Dump `prod.keys`, `title.keys`, `partialaes.keys` if offered.
- [ ] Store them in the encrypted backup, not in the repo.
- [ ] Confirm `prod.keys` contains the `save` and `save_mac_key` families — needed for `fs` work on this unit.

## 6. Partition dumps

- [ ] `BOOT0`, `BOOT1`, `PRODINFO`, `PRODINFO_BAK`, `SAFE`, `SYSTEM`, `USER`.
- [ ] Hash each and record.
- [ ] Keep a second copy of `PRODINFO` specifically. It is the one partition whose loss is unrecoverable.

## 7. Partition-level verification

- [ ] Restore `BOOT0` from the dump and boot Hekate.
- [ ] Confirm the console still boots normally.
- [ ] Only then continue.

## 8. CFW layout decision

EmuMMC on SD for all fuzzing. SysMMC stays clean and matches the OLED.

- [ ] Create emuMMC (file-based, on `V1-WORK` unless space forces partition-based).
- [ ] Install Atmosphere + Hekate config.
- [ ] `sys-ftpd`, `nxlink`, `twili`, `dmnt` available.
- [ ] Confirm emuMMC boots and reports 23.0.1.
- [ ] Keep sysMMC offline; never fuzz it.

### 8a. Patch fidelity

Atmosphere's `fs` patches modify `fsp` **in memory** — signature-check bypass, `nofs`, GC bypass. Fuzzing `fsp` through a patched build means testing a binary that is not the one on the OLED.

- [ ] Record every loader/`fs` patch state in the notes file.
- [ ] For `fsp`/`sdmmc` campaigns, disable patches that alter mount, save or path behaviour.
- [ ] Re-run the key crashes once with the stock patch profile to confirm the offset holds.
- [ ] Never report a crash without the patch profile attached.

## 9. Golden image

- [ ] Boot the fresh emuMMC once, minimal config, no user profiles.
- [ ] Power off cleanly.
- [ ] Dump the emuMMC file, hash it, label `GOLDEN-V1-23.0.1`.
- [ ] Verify a restore from golden boots to the same state.

## 10. Reset rig

- [ ] Inline power relay or switch on the dock/USB-C path.
- [ ] Test a scripted off/on cycle ten times with no human.
- [ ] Confirm golden restore + boot completes unattended.

## 11. Harness smoke test

- [ ] Push one benign file by `nxlink`/ftpd.
- [ ] Launch one NRO.
- [ ] Trigger a deliberate fault; confirm the `fatal` report is captured and readable.
- [ ] Restore from golden; confirm clean boot.
- [ ] Repeat the whole sequence three times.

## 12. Diff corpus

- [ ] Archive the 23.0.1 `Kernel`, `spl`, `pm`, `ldr`, `ro`, `fs`, `usb`, `nifm` binaries extracted from this unit.
- [ ] Hash the corpus and mark it read-only.
- [ ] Note that older versions must be sourced separately — 23.0.1 is the newest, so the corpus currently has one end only.
- [ ] Build the K0 baseline table for 23.0.1 before the first update lands.

## 13. Sign-off

- [ ] Two verified full images.
- [ ] Keys stored twice, encrypted.
- [ ] `PRODINFO` ×2.
- [ ] Golden image verified by a restore.
- [ ] Unattended reset proven.
- [ ] Fault capture proven.
- [ ] Corpus archived and hashed.
- [ ] Firmware frozen and recorded.

## Do not

- Do not fuzz sysMMC.
- Do not install persistence on sysMMC until T1 has survived ten cold boots.
- Do not update either console until the 23.0.1 corpus is archived.
- Do not connect emuMMC to Nintendo without deciding the ban posture first.
- Do not reuse the `V1-WORK` card for real saves.
- Do not proceed past a failed hash check.
- Do not run a batch on a battery below 40 %.

## Recovery quick reference

| Symptom | Action |
| --- | --- |
| Boot loop in emuMMC | Restore golden, retry input |
| Boot loop in sysMMC | Restore `BOOT0`/`SYSTEM` from partition dumps |
| No RCM entry | Check jig, cable, battery; RCM is in bootrom and cannot be lost |
| `PRODINFO` corrupt | Restore `PRODINFO_BAK`, then `PRODINFO` |
| SD corruption | Reimage, restore golden, rehash |

## Open item

Q16/Q17 resolved: both on 23.0.1, parity achieved. The remaining risk is a Nintendo update, not a mismatch. Watch the release feed; on first sight of 23.0.2, archive the 23.0.1 corpus and freeze both consoles offline until the diff is complete.
