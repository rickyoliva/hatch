# 04 — Methods

Ordered by expected yield per hour. The top method is not fuzzing.

## 4.1 Method 1 — the diff oracle

Nintendo's patch notes are silent. The binaries are not. A function that got shorter and grew a bounds check is a function Nintendo knows is buggy. This is the highest-signal oracle available and it costs almost nothing.

Pipeline:

1. Acquire clean full dumps for four consecutive firmwares. Keep them read-only and hashed.
2. Extract `Kernel`, `spl`, `pm`, `ldr`, `ro`, `fs`, and the sysmodules of interest from `package2`/`ini1` and the system partitions.
3. Normalise: decompress, resolve script/relocation tables, apply `switchbrew` symbols where they exist.
4. Bindiff (Diaphora, BinDiff, Ghidra Version Tracking) and rank by: similarity drop, size delta, new conditional branches, new `memcpy`/`memmove` clamps, call-site churn.
5. Emit a **queue**: one entry per suspected patch, each with a fingerprint and the two versions' pseudocode.
6. Confirm by **reverting the check** in the newer binary (or in a harness shim) and replaying inputs that exercise the old path. If the reverted build faults and the patched one does not, that is a confirmed bug, not a guess.

Step 6 is the technique most people skip. Without it you have a list of guesses; with it you have bugs, in an afternoon, that Nintendo handed you.

Apply to: `svc` validation in `Kernel`, `spl` blob size fields, `pm`/`ldr`/`ro` NSO header and flag parsing, `fs` path canonicalisation and mount records, `fsp` cluster arithmetic, `nfp` APDU lengths, `usb` descriptor lengths, `nifm` IE lengths.

Guard against: compiler churn masquerading as security. A function can shrink because of inlining or a different optimisation level. Require a *semantic* change (new branch on a length or index) before a diff becomes a queue entry.

## 4.2 Method 2 — grammar from data

Extract every file of type *X* from a game or system partition and use it as a seed corpus. Humans hand-write twelve seeds; this produces twelve thousand. It finds the deep structural states that random mutation never reaches: valid FAT chains, valid MP4 box trees, valid course data.

Rules: dedupe on structure, not bytes. Keep a small curated tier for triage sanity so a crash in a 4 GB MP4 does not eat the batch.

## 4.3 Method 3 — symbolic gates

Formats die on invariants long before they reach the bug: CRC32, length-prefix consistency, magic values, structural nesting. Humans are bad at these; constraint solvers are good at them.

Use angr or Triton to model the gate and solve for a header that survives to the parser body. This is what makes a 4-byte overflow reachable without hand-counting. Feed mixed concrete/symbolic execution: concrete for the boring 90 %, symbolic only for the gate words.

Limit: Horizon's parsers are optimised ARM64, so lift to VEX with care and always confirm the solved header on hardware. A symbolic "solution" that segfaults the emulator is not a solution.

## 4.4 Method 4 — differential parser testing

Run the same input through the target parser and a reference parser (libexfat, exfatprogs, libjpeg-turbo, a second MP4 demuxer, the emulator's HLE). Divergence in acceptance, size accounting or error class marks the input as interesting. Cheap pre-filter before hardware.

## 4.5 Method 5 — LLM triage, and where it lies

Helpful:

- "Here is a decompiled function and its call sites. Rank the `memcpy` sinks by attacker control and say which length field to aim at."
- "Write a libFuzzer/AFL harness around this struct with a fixed 0x400-byte header."
- Naming, summarising a 300-line function, cross-referencing a CVE description to a binary shape.
- Bulk translation of Ghidra pseudocode into a fault table.

Not helpful:

- Understanding an optimised ARM64 function end to end. The output will be plausible and wrong.
- Deciding whether a bug is real. That is what the harness is for.
- Anything where it can confabulate an offset. Verify every offset against the binary.

Rule: **LLM output is a hypothesis generator, never an oracle.** Every claim it makes gets a `// VERIFY` until a harness or a disassembly confirms it.

## 4.6 Method 6 — fuzzing

Last, not first, because without a harness the rest is theory. AFL++ with persistent mode where possible, libFuzzer for in-process harnesses, structure-aware mutators once the grammar is known.

Non-obvious requirements:

- Differential oracle (4.4) as the crash filter, otherwise you drown in wrong-format noise.
- Deterministic seed ordering and a recorded seed ID in every crash directory.
- Batch caps and golden-image reset between batches (03 §3.3).
- One unit per campaign. Do not share a console between concurrent harnesses.

## 4.7 The kernel track

Runs continuously and mostly statically, because dynamic access needs a userland primitive first.

Focus, in order:

1. `svc` argument validation — bounds, alignment, overflow in size arithmetic, signedness, handle-vs-object confusion, TOCTOU on user pointers.
2. `svcMapProcessMemory` / `MapProcessCodeMemory` / `SetProcessMemoryPermission` — historically the richest seam.
3. `pm` / `ldr` / `ro` NSO header, flag and rela parsing at process creation.
4. `fs` path canonicalisation and mount-record parsing (mount points reachable from a user process).
5. `spl` blob and size-field handling.
6. `static` shared-memory and `nvservices`-style interfaces.
7. IPC raw-data paths where a variable-size buffer is copied before a length check.

Success is a validation gap that survives the revert test in 4.1 step 6. After that you need a KASLR leak — often the same bug, sometimes a sibling.

## 4.8 Triage discipline

- Severity ladder: crash → controlled fault → controlled length → controlled write → R/W → kernel.
- Never upgrade a bug on vibes. Each rung needs a demonstration.
- A bug is not "found" until it reproduces three times cold.
- The crash corpus is versioned alongside the notes that produced it.

## 4.9 Publication

- Notes written continuously, published when the chain is verified or the track is killed.
- Embargo a confirmed T1 chain long enough to give Nintendo the same courtesy the scene historically got, then publish regardless of response time.
- Publish the *harnesses* even when the bug is patched. The next person is the point.
