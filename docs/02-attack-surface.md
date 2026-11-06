# 02 — Attack surface

## Scoring model

Each vector is scored 1–5 and multiplied. The composite is a *sort key*, not a truth.

| Dimension | 5 means | 1 means |
| --- | --- | --- |
| **Friction** | Zero user interaction | User must navigate a menu |
| **Privilege** | Kernel | Applet / sandboxed |
| **Keys** | None required | Target console keys required |
| **Density** | Enormous attacker-controlled complexity | Tiny fixed protocol |
| **Unfuzzed** | No public work at all | Fuzzed for a decade |
| **Verify cost** | Crash in minutes | Needs bespoke hardware and a second console |
| **Pivot** | Compromise already outranks the process that could stop it | No path out of the process |

`Pivot` is the one the previous analysis lacked, and it matters more than density. A bug in a sysmodule is often *worth more than a kernel bug* for our purposes, because the sysmodule already holds capabilities the kernel would have to be re-exploited to grant (`fsp` can write partitions, `ldn` owns the radio, `hid` owns input). Rank by **chain contribution**, not by how interesting the bug is.

## Matrix

| # | Vector | Fric | Priv | Keys | Dens | Unfz | Verify | Pivot | **Score** |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | SD card FAT/exFAT mount | 5 | 4 | 5 | 3 | 5 | 5 | 3 | **22500** |
| 2 | `usb` host-class descriptors (LAN, GC adapter) | 5 | 4 | 5 | 3 | 5 | 4 | 3 | **18000** |
| 3 | `sdmmc` low-level card registers (CID/CSD/SCR, CMD seq, UHS) | 5 | 4 | 5 | 3 | 5 | 4 | 3 | **18000** |
| 4 | `usb` device mode (console plugged into hostile host) | 4 | 4 | 5 | 3 | 5 | 4 | 3 | **14400** |
| 5 | `nfp` / amiibo tag and APDU handling | 4 | 4 | 4 | 3 | 5 | 5 | 3 | **14400** |
| 6 | `nifm` / 802.11 IEs / DHCP / EAP config | 3 | 4 | 5 | 4 | 4 | 3 | 4 | **11520** |
| 7 | NSO emulators (ROM, save types, states, RSP HLE, JIT) | 3 | 3 | 5 | 5 | 4 | 3 | 4 | **10800** |
| 8 | `gc` gamecard interface (MIG-class card) | 5 | 4 | 3 | 3 | 4 | 3 | 4 | **10800** |
| 9 | Web applet / WebKit | 3 | 2 | 5 | 5 | 2 | 4 | 2 | **2400** |
| 10 | Album media parsers (JPEG/MP4/PNG) | 4 | 2 | 5 | 5 | 4 | 4 | 2 | **6400** |
| 11 | `hid` wired controller reports | 4 | 4 | 5 | 2 | 5 | 4 | 2 | **6400** |
| 12 | Server-mediated game formats (HOME, SMM2, ACNH) | 3 | 3 | 5 | 5 | 4 | 2 | 3 | **5400** |
| 13 | `ldn` local wireless + game netcode | 3 | 3 | 5 | 4 | 5 | 2 | 3 | **5400** |
| 14 | Library applets (`swkbd`, `myPage`, `controllerSupport`) | 2 | 2 | 5 | 4 | 5 | 3 | 4 | **4800** |
| 15 | `hid` Joy-Con over Bluetooth | 3 | 4 | 5 | 2 | 5 | 3 | 2 | **3600** |
| 16 | Raw save transplant | 4 | 3 | 2 | 4 | 2 | 4 | 1 | **1536** |
| 17 | GameShare / `splay` guest (PC-hosted) | 2 | 2 | 3 | 4 | 5 | 1 | 2 | **480** |

## Tier assignments

**Tier A — active tracks, in build order:**

1. **SD mount (filesystem).** Zero interaction, sysmodule privilege, no keys, never fuzzed, trivially verified, and the harness is the cheapest one in the whole matrix. If the exFAT driver has anything, it pays immediately.
2. **USB host classes.** Same friction, same privilege. Two parsers (`usb` core descriptors, then the AX88179/RTL8153 driver logic) in series, both reached by plugging in a £4 RP2040. Pairs well with #1 because a single gadget can present as both.
3. **`nifm` / wireless.** Sysmodule, local-only, and the only tier-A entry that can be delivered without physical contact at all. Adds the malware-like deployment story.
4. **NSO emulators.** Not tier-A on score, tier-A on upside: an Application process with `fsp-srv` and `sockets`, and the largest genuinely new parser complex in the stack. Highest ceiling per unit of luck.

**Tier B — pull on when a track stalls:** `nfp` (best modded→stock bus that exists, limited to games with amiibo payloads), `gc` (same shape, needs a card), Album media, server-mediated game formats, `ldn`.

**Tier C — opportunistic:** library applets, Bluetooth HID, save transplant.

**Reserve — WebKit.** Ranked ninth on chain contribution and first on density. It is the *best complement* to any kernel bug, because then its low privilege stops mattering and the chain completes instantly. Keep a WebKit track warm, and if a kernel bug lands first, WebKit becomes tier A overnight.

**Reserve — GameShare.** The only no-content, host-driven entry: no cartridge, no save install, no user navigation, and the attacker owns the host. Low privilege and high verification cost, but a PC impersonating the host would remove the Switch 2 from the loop and turn a social feature into an offline fuzz target. Keep cold until a no-cartridge delivery story is needed. See `06-dossiers/gameshare.md`.

## Why WebKit drops from first to ninth

Three corrections to the earlier ranking:

1. **It is the most fuzzed code on Earth.** Being un-fuzzed scores 2. The bug you find will be a variant of a public one, and Nintendo backports WebKit security patches selectively but genuinely.
2. **Privilege is applet, and the applet has almost no services.** It is not a game. `fsp-srv`, `sockets` and `am` belong to the *other* process. You inherit a reduced syscall mask and a small handle table.
3. **The pivot is the hard part, not the bug.** Every historical Switch WebKit chain (Pegaswitch, Deja Vu, Nereba/Caffeine) was really a kernel-bug delivery vehicle. WebKit supplied the entry, the kernel bug supplied everything that mattered.

Cyberpunk charm is not a criterion. Chain contribution is.

## Why the filesystem track leads

- **Mount happens on insertion, before any UI.** Zero interaction.
- **Nintendo wrote the exFAT driver themselves** for >32 GB cards in 8.0.0. This is not Microsoft's code path with 30 years of testing behind it.
- **FAT/exFAT is a spec built out of footguns**: FAT chains and loops, TexFAT, upcase table, allocation bitmap, `0xE5` deleted entries, LFN reassembly, timestamps, cluster run arithmetic, `..` canonicalisation, boot-sector backup divergence.
- **There is no public fuzzing of this driver that we can find.** That is the single strongest signal in the matrix.
- **The harness is free**: a `.img` file, a mount call, a crash.

Below the filesystem sits `sdmmc` itself — CID/CSD/SCR registers, command sequencing, tuning, UHS negotiation — all attacker-controlled by a card that lies. Same delivery, one layer deeper, slightly higher hardware cost.

## Explicit rejections

- **eShop / NSP / XCI install.** Requires network and eShop access; also the least attacker control. Revisit only if we need a delivery channel for a *game* payload.
- **News / BCAT / `prepo` / `npns`.** Server-mediated, so blocked on the HTTPS/CA problem (§5 of constraints). Revisit if we obtain a certificate path.
- **`spl` keyblobs, `package1`, safe mode.** Real targets but only reachable at boot with a modified NCA or package, i.e. not from stock. They become relevant *after* a kernel primitive, not before it.
- **HDMI / dock / `avm`.** No console-controlled parser worth the effort.
- **`es` tickets.** Needs install path first.

## Hardware gating

Scores assume the kit in `09-inventory.md`. Three entries are blocked or degraded by missing parts, not by the software:

- **`nifm`** is split. *Delivery* is unblocked — the Miyoo Mini Plus runs `hostapd`. *Frame-level fuzzing* is blocked until an injecting radio exists (ESP32-S3); the Miyoo has no reliable monitor mode.
- **`gc`** needs a card. Blocked on a MIG-class device.
- **`sdmmc` L1** needs an interposer. Degraded to L2-only (filesystem) until the parts arrive.

## Anti-goal

Do not collect crashes. Collect **reproducible crashes in tier-A processes with a stated path to a leak.** A pile of 200 Album JPEG crashes is worth less than three `fsp` mounts that fault deterministically.
