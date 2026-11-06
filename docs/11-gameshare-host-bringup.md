# 11 — GameShare host bring-up without a Switch 2

## 0. Goal

Make the **v1 host a GameShare session to the stock OLED**, with no Switch 2 in the loop.

This is the first feasibility gate in `06-dossiers/gameshare.md`, moved off the critical path: if a Switch 1 can host, a PC host becomes a port rather than a discovery problem, and the whole track stops depending on borrowed hardware.

The immediate claim to test:

> A modded 23.0.1 Switch 1 can launch the GameShare host stack, advertise a room, and get `splay` launched on a stock Switch 1 guest.

No stream, no save, no media yet. Room and applet launch only.

## 1. Why this is credible

The Switch 1 23.0.1 firmware contains the complete host and guest stack. It is dormant, not absent.

Verified from the 23.0.1 IPC dumps:

| Fact | Evidence |
| --- | --- |
| `NGC_services` exists as a system program, title `0100000000000050` | `process_swipc.py` title map; `ns1/23.0.1/programs/NGC_services.json` |
| `NGC_services` provides `nn::streamplay::IUserService`, `IUserShimScopedObject`, `ISystemService`, `ISystemShimScopedObject`, `nn::ngc::*` | Same file |
| `NGC_services` also invokes `stpl:sys`, `stpl:u`, `ngc:u`, `ngct:u`, `fsp-srv`, `arp:r` | Same file, `_services` |
| `splay` applet exists, title `0100000000001048`, new in 22.0.0 | `process_swipc.py`; `ns1/23.0.1/applets/splay.json` |
| `splay` invokes `stpl:sys` and holds `fsp-srv`, `ldn:s/u`, `hid:sys`, `nifm`, `bsd`, `nvdrv`, `audout/audren` | Same file |
| `fsp` has `EncryptStreamPlaySaveData` (830) and `DecryptStreamPlaySaveData` (831) | `Filesystem_services.json` |
| `pctl` has `TryBeginFreeCommunicationForStreamPlay` / `EndFreeCommunicationForStreamPlay` | `Parental_Control_services.json` 1021/1022 |
| `hid` has `SendConnectionTriggerWithoutTimeoutEvent`, `CancelConnectionTrigger`, `SetConnectionLimitForSplay` | `HID_services.json` 1156–1159 |
| `ldn` exposes `OpenAccessPoint`, `CreateNetwork`, `CreateNetworkPrivate`, `SetAdvertiseData`, `SetStationAcceptPolicy`, `EnableActionFrame` | `LDN_services.json` 200–208, 500 |
| `lp2p` exposes group create/destroy/advertise/send/recv | `LDN_services.json`, `lp2p::ISfService` |

The pieces that make a host are all present on the target firmware. What is not present is the UI path that launches them on Switch 1.

## 2. Host stack model

`NGC_services` is the streamplay service provider used by both sides; it owns the save-slot, content-check and system-service interfaces. `splay` is the guest client. `fsp` handles streamplay save crypto. `hid` handles the connection trigger. `ldn`/`lp2p` are the transport. The game, when one is involved, is only a caller of `stpl:u`.

```
NGC_services (host program)
  provides stpl:u, stpl:sys, ngc:u, ngct:u
  uses fsp, ldn, lp2p, hid, pctl

splay (guest applet)
  uses stpl:sys, fsp, ldn, nvdrv, audout, hid

fsp 830/831  <-> streamplay save encrypt/decrypt
hid 1156-1159 <-> connection trigger, Splay limit
```

If `NGC_services` can be brought up on the v1, the host role is Nintendo's own code, not our reimplementation. That is the shortest path and the best possible fidelity.

## 3. Phases

### A0 — Static map (start here, no console needed beyond the dump)

1. Dump the v1 23.0.1 system partition and extract `NGC_services`, `splay`, `ldn`, `lp2p`, `fsp`, `hid`, `pctl`.
2. Hash and archive.
3. Ghidra:
   - `NGC_services`: NPDM permissions, `_srv` registration, `stpl:sys`/`stpl:u` command handlers, `ldn` call sequence, `CreateNetwork` parameters, any platform check (`nn::oe`, `set:sys`), launch assumptions.
   - `splay`: NPDM, launch flags, which `stpl:sys` commands it calls before and after the room, applet argument expectations.
   - `ns`/`am`: how `splay` is launched, and whether a room alone is enough.
   - `fsp` 830/831: buffer shapes and key source.
4. Produce a call table: host command, guest command, order, argument shape, blocking behaviour.

**Exit:** every `stpl` command has a named implementation and a caller, and the `splay` launch condition is identified.
**Kill:** `NGC_services` contains an explicit Switch 2 platform gate that cannot be satisfied on a v1 and cannot be patched. Record it and stop.

### A1 — Service registration probe

The cheapest dynamic test. A small homebrew:

1. `smGetService("stpl:sys")`, `stpl:u`, `ngc:u`, `ngct:u`.
2. Record which are already registered on a booted v1.

If absent, launch `NGC_services` by title ID via `pm:shell` (or `ns:am2` if `pm:shell` is denied) and probe again. Then `twili`-trace its startup.

**Exit:** all four services open and answer a trivial command.
**Kill:** `NGC_services` crashes or exits before registration, and the failure is a hard platform check, not a missing permission.

### A2 — Room bring-up

If `NGC_services` creates the room itself, trace the `ldn` calls and copy them. If not, call `ldn` from the host NRO:

1. `OpenAccessPoint`.
2. `SetAdvertiseData` with the GameShare room payload.
3. `CreateNetwork` or `CreateNetworkPrivate` with the parameter block.
4. `SetStationAcceptPolicy`, `EnableActionFrame`.
5. `SetConnectionLimitForSplay` via `hid`.

The parameter values come from A0. Start with the `NetworkType`/scene and game ID used by a GameShare title, and vary one field at a time.

**Exit:** the OLED scan list shows the room with the expected name and game icon/metadata.
**Kill:** the room appears but the OLED refuses to select it for two weeks, with the refusal pinned to a specific field or crypto mismatch.

### A3 — Guest launch

Accept the room on the OLED. Watch for `splay`.

- Capture before, during and after: 802.11, `ldn` state, `appletAE`/`ns:am2` activity on the host, and the OLED screen.
- If `splay` never launches, check whether the guest is waiting for an applet argument or a `stpl:sys` command from the host. Diff against `splay`'s static call table from A0.

**Exit:** `splay` appears on the guest and reaches its first network state.
**Kill:** `splay` launches but immediately exits with a fixed error code that maps to a host-side precondition we cannot satisfy.

### A4 — Session skeleton

Drive `stpl:sys` from the host in the order A0 produced: memory sizing, display creation, session open. No media yet. Log every call and the guest's response.

**Exit:** applet waits for media instead of timing out.
**Kill:** command order is unlearnable from static analysis and the applet requires host state we cannot infer.

### A5 — First media and save

Only after A4:

1. Replay a short captured stream if one exists, else a synthetic keyframe.
2. Exercise `fsp` 830/831 with a tiny save slot.
3. Record the first successful frame and the first successful commit.

Then hand back to the dossier's G6 pressure matrix.

## 4. What to record

- Title IDs and hashes of every module used.
- NPDM permissions for `NGC_services` and `splay`.
- Full `stpl` call table with argument sizes and direction.
- `ldn` parameter block, advertising payload, and every field change.
- pcap and screen recording for each run.
- Error codes verbatim, with the guest's on-screen text.

## 5. If NGC_services will not run

Ordered fallbacks:

1. **Permission gap.** Launch it as a system program, not an application; check `pm:shell`/`ns:am2` path and NPDM.
2. **Platform gate.** Patch the check in the v1 copy (CFW lab only) and record the revert.
3. **Missing caller.** Some fields may only be filled by a game process. Run a GameShare-capable Switch 1 title on the v1 and let it call `stpl:u` instead of calling it ourselves.
4. **PC host with v1 as radio.** If the service logic works but the applet launch does not, split: PC speaks `stpl` over IP, v1 does `ldn`. This still removes the Switch 2.
5. **Park.** Borrow a Switch 2, capture one real room, and diff.

## 6. If splay will not launch on the guest

1. Confirm the room advertises the exact GameShare scene, not a generic private network.
2. Confirm `hid` `SetConnectionLimitForSplay` was called.
3. Check `ns:am2`/`appletAE` on the host for a launch request and its result.
4. Check the guest's `splay` NPDM for a required applet argument or a title-ID allowlist.
5. Capture a real Switch 2 room and diff the advertisement byte-for-byte.

## 7. Kill criteria and timebox

- **A0:** one week. No call table, no track.
- **A1:** one week after A0. No service registration, fall back to patch analysis or park.
- **A3:** two weeks after A1. No `splay` launch, capture a real room and diff; if still nothing, park.
- **Overall:** six weeks to a waiting `splay`. Past that, publish the negative and the call table.

## 8. Do not

- Do not buy a Switch 2 for this.
- Do not patch the OLED. It is stock and it is the confirmation target.
- Do not fuzz before A4.
- Do not assume `splay` is guest-only; verify on both sides.
- Do not commit Nintendo blobs; hashes and offsets only.

## 9. Deliverables

- Module hash manifest.
- `NGC_services` and `splay` target maps.
- Service registration result.
- `ldn` room parameter table.
- pcap and timeline for A2/A3.
- A minimal host NRO that reproduces A1–A3 from a clean boot.

## 10. Relationship to the dossier

This document is Phase A of `06-dossiers/gameshare.md`. The dossier's G0–G6 ladder becomes Phase B and is renumbered B0–B6 once A3 passes. H-GS0 (a Switch 1 can host) is the gate that decides whether the PC-host track is worth building at all.
