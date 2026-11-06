# 07 — Open questions

Facts we must establish before committing build time. Each has an owner action and a thing it blocks.

| ID | Question | How to answer | Blocks |
| --- | --- | --- | --- |
| Q1 | Does stock `fs` verify the save HMAC *before* the game can read the save, or only on commit? | Instrument a `fs` mount on a v1, or read `fsp` in Ghidra | Any save-transplant design |
| Q2 | ~~Is the `ctest.cdn.nintendo.net` probe plaintext?~~ Superseded by Q21/Q22 | — | — |
| Q3 | Does a stock console mount a foreign exFAT card read-only, and does it mount before or after any UI? | Insert an image card, capture `fs` activity | SD track viability (assumed yes) |
| Q4 | Are `fs` mount points reachable from an unprivileged process, or gated behind `fsp-pr`? | `npdm` capability read on `fsp`/`fsp-srv` | Harness design |
| Q5 | Is the album's extended header (`nn::album`) parsed before or after the JPEG/MP4 decoder? | Ghidra on the Album applet | Album track ordering |
| Q6 | Do NSO save states persist to disk and are they importable? | Filesystem survey after a state save | NSO priority |
| Q7 | Does the N64 core's byte-order swap run before any length validation? | Truncated-ROM run on hardware | NSO first experiment |
| Q8 | Is `nfp`'s app-area HMAC checked before or after the tag struct is handed to the game? | Read `nfp` in Ghidra | Whether amiibo payloads need a valid HMAC at all |
| Q9 | Is gamecard certificate reuse rate-limited or revoked server-side? | Two cards, same cert | `gc` track |
| Q10 | Can a clone MAC complete Joy-Con/Pro pairing without a Nintendo-signed link key? | ESP32 Classic pairing attempt | `hid` Bluetooth sub-track |
| Q11 | Which `svc` validations changed across the versions leading to 23.0.1? | K0 diff, first pass | Falsification test F1 |
| Q12 | Does the Switch 2 compatibility layer reuse Switch 1 `fsp`/`nfp`/`ldn` code? | Binary comparison | Whether a chain is worth two consoles |
| Q13 | Does the web applet still hold `fsp-srv` for cache? | `npdm` capability read | Post-exploitation value of a WebKit bug |
| Q14 | Are `SharedArrayBuffer` and JIT intrinsics enabled for applet content? | Page probe | JSC primitive availability |
| Q15 | Are stack canaries present in `fsp`, `usb`, `nifm` on 23.0.1? | Per-binary check | Exploit shaping |
| Q16 | ~~What firmware is the v1 on, and is it pinned deliberately?~~ | **Answered: 23.0.1, frozen** | — |
| Q17 | ~~What firmware is the OLED on, and will it be updated to match?~~ | **Answered: 23.0.1, frozen, parity with v1** | — |
| Q18 | Does the OLED accept USB-C→A adapters and third-party docks for host-mode peripherals? | Insert adapter, attach Pico 2 | USB host track reachability |
| Q19 | Can the nRF52840 sustain NTAG215 emulation for a full amiibo read without timing out? | Nordic NFC tag example + a real tag dump | Whether the amiibo track needs stickers |
| Q20 | Does `nxnative` beat `nxlink` + `sys-ftpd` on staging and launch time? | Timed comparison | Whether it earns a place |
| Q21 | Does the Miyoo's stock firmware permit `hostapd` AP mode, or does it need a cross-built binary? | OnionOS shell, `iw`/`nl80211` check | Whether it works as the delivery AP |
| Q22 | Is 23.0.1's connectivity probe still plaintext, and does a 200 open the portal? | Capture on the Miyoo AP | Entire network delivery story |
| Q23 | Are the SD bus lines 1.8 V during UHS, and do the chosen shifters tolerate it? | Analyser + level-shifter datasheet | Interposer design |
| Q24 | Does Atmosphere's 23.0.1 support include `dmnt`/`twili` and emuMMC without caveats? | Build notes, smoke test | Lab viability |
| Q25 | Which Atmosphere `fs`/loader patches alter mount, save or path behaviour on 23.0.1? | Diff patched vs stock text, one by one | Fidelity of every `fsp` result |
| Q26 | Does the guest `splay` applet receive save/state data, and through which interface? | Ghidra on `splay` plus a captured session | H-GS1 and the dual-stream hypothesis |
| Q27 | Is the media stream a standard container/codec, and can a PC encode it? | Capture and parse; compare with `nvdrv`/`codec` call sites | PC-host feasibility |
| Q28 | Does the guest system launch `splay` from room parameters alone, or is there a host applet-arg path? | G1/G2 replay | PC-host feasibility |
| Q29 | Does `GetRequiredMemorySize` trust a host-supplied guest count above three? | Host-side stub mutation at G3 | H-GS4 |
| Q30 | Which `stpl:sys` commands are reachable before a session exists? | Static call table plus G1 | G1/G2 ordering |
| Q31 | Is the Switch 1 guest a compatibility subset of the Switch 2 protocol? | Diff `splay` vs `SplayApplet` service and command use | Whether a Switch 1-only host is possible |
| Q32 | Can a PC complete the LDN join with AP-mode-only injection? | Rerun the TX10LED path on a known-good adapter | G1/B1 |
| Q33 | Can `NGC_services` be launched on a 23.0.1 v1 and register `stpl`/`ngc`? | `pm:shell`/`ns:am2` launch, then `smGetService` | H-GS0, A1 |
| Q34 | Does a v1-hosted room launch `splay` on a stock guest? | A2/A3 with the OLED | H-GS0 |
| Q35 | Do `fsp` 830/831 encrypt/decrypt streamplay saves on the guest? | Trigger a save slot and trace `fsp` on the host | H-GS1, dual-stream |

## Answered

- **Q16 / Q17** — both consoles on 23.0.1, frozen, parity achieved. Recorded in `09-inventory.md` §9.9 and `10-v1-prep.md` §0. The remaining risk is a Nintendo update, not a mismatch.
- **Q2 / Q21 / Q22** — collapsed into Q21 and Q22. The probe question only matters once the AP itself is proven.
