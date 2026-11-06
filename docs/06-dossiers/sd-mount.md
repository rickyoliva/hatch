# Dossier — SD mount, filesystem and card layer

Tier A. Zero interaction, sysmodule privilege, no keys. Lead target.

## 1. Why this is the top track

The console mounts an inserted card before any UI exists to ask about it. The driver is Nintendo's own (exFAT added for >32 GB in 8.0.0), and there is no public fuzzing of it we can find. Everything below the filesystem is also attacker-controlled by a card that lies.

Two layers, deliberately separate:

- **L1 — `sdmmc`.** CID/CSD/SCR registers, CMD sequencing, tuning, UHS negotiation, timeouts.
- **L2 — `fsp`/exFAT.** Boot sector, FAT chains, bitmap, upcase, directory entries, cluster runs.

L1 is reachable by a card that answers commands correctly then returns hostile register data. L2 is reachable by a card that mounts and returns hostile structure. L2 is far cheaper to build for.

## 2. Target map to produce (P1)

For `sdmmc`: the CMD dispatch table, the CSD decode (`c_size`, `c_size_mult`, block length), SCR, the tuning loop, the UHS switch path, and every place a card-supplied length feeds a copy.

For `fsp`/exFAT: the mount entry, boot-sector validation, sector-size and cluster-count arithmetic, FAT chain walk, bitmap lookup, upcase-table load, LFN reassembly, `..` resolution, and the directory iterator.

## 3. Prime bug classes

| Structure | Failure mode |
| --- | --- |
| `c_size` / cluster count | Integer overflow → tiny allocation, huge walk |
| FAT chain | Cycle → infinite loop; out-of-range link → OOB read |
| Allocation bitmap | Disagreement with FAT → double-allocate, UAF on write |
| Upcase table | Offset past table end → OOB read in name comparison |
| LFN entries | Count/checksum mismatch, reassembly length overflow |
| `0xE5` deleted entries | Treated as live, or skipped with pointer not advanced |
| Cluster run arithmetic | `cluster_count * cluster_size` overflow on 32-bit |
| Volume label / OEM field | Fixed-size copy with an attacker length |
| Boot-sector backup | Divergence between primary and backup used inconsistently |
| TexFAT | Two allocation bitmaps, transaction state machine, rollback |

The last two are the ones nobody fuzzes, because nobody boots TexFAT volumes.

## 4. Harness

```
.img file  ->  write to a loop-mounted FAT/exFAT image or a real SD  ->  fs mount  ->  readdir/stat walk  ->  capture
```

- `fs` mount via NRO, then a recursive `stat`/`open` walk so the iterator and name paths get exercised, not just the mount.
- Auto-reboot on fault via `twili`; golden image between batches.
- Seed corpus: a real card image, then structural mutators (not byte mutators) for the fields above.
- Differential oracle: `exfatprogs`/`libexfat` and `fsck.exfat`. Divergence = interesting.

## 5. Card interposer (L1)

RP2040 or small FPGA with level shifters, µSD breakout, pass-through to the console's reader. Goals:

- Lie on CID/CSD/SCR with a corpus of adversarial values.
- Replay valid CMD sequences, then inject malformed data phases and bad CRCs.
- Force UHS modes and tuning failure paths.
- Time out mid-transfer to hit error unwinding.

Capture the bus with the analyser at every run; a hang with no capture is unexplainable.

## 6. Known-unknowns

- Does the driver validate the boot sector against the backup, or pick one?
- Is the upcase table length checked against the declared table?
- Are cluster counts 32-bit or 64-bit in the internal arithmetic?
- Does mount happen read-only first, or mount-and-repair?
- Is the card's claimed capacity trusted for the FAT geometry, or is the last cluster derived from the card?
- Does the system mount a card that fails `fsck`-style invariants and continue, or refuse?

## 7. First three experiments

1. Card with a valid boot sector and a FAT chain that loops back two clusters in. Walk the root.
2. Card with `c_size` claiming exactly `0xFFFFFFFF` clusters and a sparse FAT.
3. Card with an LFN run whose count byte is larger than the entries present before the terminator.

## 8. Delivery notes

Mount-on-insert means the attacker needs the user to insert a card once. That is a lower bar than "launch a game". A card that also contains a normal-looking `Nintendo/Album` tree is indistinguishable to the user.

## 9. Status

Tier A. Build order 1. Harness is the cheapest in the matrix, which is why it starts first.
