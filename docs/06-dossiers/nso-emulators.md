# Dossier — NSO emulators

Highest ceiling in the programme. Not a tier-A score, a tier-A upside.

## 1. Why

A hostile ROM is a file. Opening it is a normal user action, one menu deep. Inside, the target is an **Application** process with `fsp-srv`, `sockets`, `hid`, `am` and `avm` — the richest handle table on the console short of a sysmodule. NSO N64 in particular carries an HLE RSP/RDP, a MIPS→ARM64 recompiler, and a per-title save-type layer written for convenience rather than safety.

This is the Ninjhax/Ocarina model with a 1996 attack surface: every ROM is attacker-controlled, and the emulator's whole job is to trust it.

## 2. Attack surface, ordered

1. **Save states** — the densest serialisation in the stack if they are exposed or persisted anywhere. Verify. Free-form struct dumps, restored without validation, often with pointers or indices inside.
2. **ROM header and bootcode** (`0x40` region in a `.z64`): country, magic, entry point, boot CRC fields.
3. **Byte-order detection** (`.z64`/`.n64`/`.v64`) and the swapping routine that runs *before* anything is validated.
4. **Save-type selection** from the ROM's save ID: SRAM, EEPROM (4k/16k), FlashRAM, Controller Pak, none. Each has its own size assumptions.
5. **RSP microcode detection** — F3DEX/F3DEX2/F3DZEX/S2DEX selection and the HLE dispatch that follows. Picking an HLE task by a header field from an untrusted blob is a classic.
6. **Boot CRC / IPL3 checksum** — often a fast-path that skips verification when the value "looks valid".
7. **Recompiler** — block cache sizing, delay-slot handling, `cache` and unaligned ops, register allocation. A recompiler bug emits wrong ARM64 into a JIT page.
8. **GBA / SNES / Genesis equivalents** — smaller cores, separate save-type and link-cable surfaces.
9. **Controller Pak and Rumble Pak** — save-media parsing and the accessory detection state machine.

## 3. Bug classes

- Integer overflow in save-media sizing (`save_type` chosen by ROM, length chosen by content).
- Off-by-one in EEPROM/Controller Pak page arithmetic.
- OOB read in RSP microcode detection (scan for magic ucode words over a fixed window, no length bound).
- Recompiler miscompilation that escapes the emulator's assumptions about guest isolation.
- State restore with an unchecked count or index → arbitrary write in the host process.
- Header-swap routine reading `0x40` bytes from a 1-byte file.

## 4. Harness

- Stage the ROM into the app's save directory, launch, poll `fatal`.
- Vary one field per run from a structured corpus: header bytes, save ID, ucode magic, file length, byte order.
- Two-file differential: same bytes, different declared save type.
- Emulator first for corpus and coverage; hardware for confirmation only.

## 5. Privilege note

An Application is not a sysmodule, but it is far better than an applet. `sockets` alone makes a beacon possible; `fsp-srv` makes save exfiltration possible; `am` makes a launcher possible. Do not waste a successful exploit on a "hello world".

## 6. Known-unknowns

- Whether save states are persisted to disk and whether they are importable.
- Whether the byte-order swap runs before length checks.
- Whether the boot CRC is enforced or advisory.
- Whether HLE ucode selection is bounded by ROM length.
- Build provenance per title and whether the cores are Mupen/mGBA-derived with local patches (diff against upstream is then a queue).

## 7. First three experiments

1. `.z64` truncated at 0x50, 0x42, 0x41, 0x40, and 1 byte. Record where it faults.
2. Save ID set to each value in turn on an otherwise valid ROM, with and without a matching save blob.
3. ucode magic replaced with each HLE-selected value on a ROM whose real ucode differs.

## 8. Status

Tier A on upside. Begin after a tier-A track produces its first crashes, or immediately if the filesystem track turns out to be a passthrough.
