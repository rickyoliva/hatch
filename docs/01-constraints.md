# 01 — Hard constraints

These bound everything else. Anything in the dossiers that violates one of these is fantasy.

## 0. Firmware target

**23.0.1, on both consoles, frozen.** The v1 is on CFW at the same version as the stock OLED, so lab and target share identical parser builds. This is the best possible position and it is fragile: 23.0.1 is currently the newest release, so the diff corpus has only one end and a successor invalidates the baselines.

Consequences:

- Archive the 23.0.1 corpus before anything else (protocol `10-v1-prep.md` §12).
- Both consoles stay offline except during tests.
- Older versions must be sourced separately to give K0 a real range.
- 23.0.1 is also the firmware the current Switch 2 jailbreak targets. A chain here plausibly ports, which raises the value of every result and the cost of losing the version.

## 1. Memory-safety posture

Tegra X1 / Cortex-A57, ARMv8.0-A.

| Mitigation | Present? | Consequence |
| --- | --- | --- |
| ASLR (user processes) | Yes | Need a leak before a ROP chain |
| KASLR (kernel) | Yes | Need a kernel leak before kernel ROP |
| NX / DEP | Yes | Need ROP or JIT, not shellcode |
| W^X on JIT pages | Yes, via `svcSetProcessMemoryPermission` | JIT processes can toggle; others cannot |
| PAC | No | Return-address overwrite is a valid primitive |
| BTI / CFI | No | Indirect-call overwrite is a valid primitive |
| MTE / memory tagging | No | Use-after-free is trivially exploitable |
| Stack canaries | Per-binary, not platform | Assume absent, verify per target |

**Read this as: exploitation is *easy* once you have a bug. Bug discovery plus ASLR defeat is the whole game.**

## 2. Save data is not portable

This kills the naive version of "modded console crafts a save, jailed console eats it."

Switch saves are encrypted with a **console-unique `save` key** and integrity-protected with **HMAC-SHA256 over the save header and body using a console-unique `save_mac_key`**. Both derive from `PRODINFO` via `spl`. `fs` validates on mount/commit and treats a mismatch as corruption.

So a raw transplant needs the *target's* keys.

| Target | Keys obtainable? | Transplant viable? |
| --- | --- | --- |
| Unpatched v1 | Yes, `Lockpick_RCM` from RCM | Yes |
| Patched / iPatched / Lite / OLED | Only with homebrew or a leak | No, not initially |

Note the split. "Save transplant" is a strong route for v1 targets and a dead end for exactly the population that motivates the project. The previous analysis under-weighted this. **Verify the exact ordering of HMAC validation vs. game-visible read before building anything on it** — see open question Q1.

Consequences:

- The deliverable must be *in-game importable* (server-mediated sharing, in-game transfer), or *non-save* (media, ROMs, amiibo, network, USB, card).
- Any save exploit is really a *format* exploit. Prefer the formats that arrive without a MAC.

## 3. Process privilege model

| Class | Services | Syscall mask | JIT | Notes |
| --- | --- | --- | --- | --- |
| Kernel | — | full | — | Scarcest bugs, highest payoff |
| Sysmodule | broad, per-`npdm` | broad | some | `hid`, `nfp`, `usb`, `ldn`, `gc`, `bt`, `fsp` |
| Application (game) | `fsp-srv`, `sockets`, `hid`, `am`, `avm` | reduced | some (Unity/UE) | Most reachable-with-content class, most services |
| Applet | very narrow | heavily reduced | `web` only, for JS | Least post-exploitation value |
| Library applet | caller-defined | reduced | no | **Shares memory with the caller** |

The critical consequence: **a bug in the `web` applet does not give you a game's services.** It gives you an applet with a small handle table and a reduced syscall mask. Every applet entry needs an applet→something pivot before it is useful. Historically that pivot has been a kernel bug, which is why WebKit chains were always paired with one.

Library applets are the interesting exception: `ILibraryAppletAccessor` lets the caller push `IStorage` that is mapped writable in **both** processes. A bug in `swkbd`, `myPage`, `web` or `controllerSupport` therefore yields corruption in memory the *calling game* also maps. That is a cross-process primitive for free, and it is absent from the previous analysis. See `06-dossiers/peripherals.md`.

## 4. Delivery friction and what it costs

| Channel | User action | Target keys | Target network |
| --- | --- | --- | --- |
| SD insertion | Insert card | None | None |
| USB device plug | Plug in | None | None |
| Controller pairing | Pair | None | None |
| amiibo tap | Tap | None | None |
| Game card insert | Insert | None (cloned cert) | None |
| Local wireless | Be in range | None | None |
| Wi-Fi join | Join AP | None | Local only |
| In-game import | Paste an ID | None | Yes |
| Raw save transplant | Copy file | **Yes** | No |
| eShop / NSP | Download | None | Yes |

## 5. HTTPS is not MITM-able by default

Stock firmware trusts only the system CA store. You cannot add a CA without homebrew. So intercepting Nintendo HTTPS traffic needs either a genuine certificate for a Nintendo-controlled name, or a path where validation is weak.

Practical consequence: the captive-portal trick must target the **plaintext or weakly-validated** connectivity check, not arbitrary HTTPS. **Verify whether the `ctest.cdn.nintendo.net` probe on 20.x is still plaintext HTTP and whether its result is trusted enough to open a page.** Open question Q2. If it is validated, the fallback is eShop/NSO/news links.

## 6. Things you cannot do on a stock console

- Install an NSP or XCI. No eShop, no install.
- Add a trusted CA.
- Restore a raw save.
- Read `PRODINFO`.
- Mount an SD card for anything other than media (it mounts, but is not used for saves).
- Change the JIT/ASLR behaviour of a system process.

## 7. What you *can* count on

- Any file the user can place on an SD card, unauthenticated and unencrypted.
- Any USB descriptor or device class on plug-in.
- Any 802.11 frame, before association completes.
- Any controller report or SPI calibration blob.
- Any amiibo tag byte layout, with public amiibo keys.
- Any game-supplied library-applet argument blob.
- Any content the target game's own features accept from the internet.

## 8. Verification discipline

A crash is not a bug. A bug is not a primitive. A primitive is not a chain. A chain is not a result until it survives:

1. a cold boot,
2. a restored golden NAND image,
3. a second console of the same model/firmware,
4. someone who did not write it following the notes.
