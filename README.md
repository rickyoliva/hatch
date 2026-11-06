# HATCH

A research programme into software exploitation of the Nintendo Switch (1) as a viable alternative to modchip installation (Picofly, SX Core/Lite, hwfly, and clones).

> A modchip is a crowbar. We are looking for a hatch.

## Status

**Phase 0 — planning.** Documentation only. No code in this repo yet, by design.

Target firmware: **23.0.1**, both consoles, frozen. The v1 preparation protocol is in [docs/10-v1-prep.md](docs/10-v1-prep.md). Note that 23.0.1 is also the firmware the current Switch 2 jailbreak supports — a chain here may port.

## The claim under attack

"The Picofly is the only way to run code on a patched Switch."

This is an economic observation wearing technical clothing. A modchip collapses an N-stage exploit chain into one hardware stage, so nobody prices the chain any more. The parsers did not run dry; the hunting stopped. HATCH proceeds on the thesis that a full software chain remains findable on current firmware because the kernel-bug search was *abandoned*, not *exhausted*.

## What counts as a win

Defined in [docs/00-thesis.md](docs/00-thesis.md). Summary:

| Tier | Meaning | Value |
| --- | --- | --- |
| **T0** | Session-only userland. Enough to load homebrew NROs and edit saves on the target. | A real result |
| **T1** | T0 plus kernel. Full per-boot CFW with a re-trigger. | The goal |
| **T2** | Persistence without a re-trigger. | Bonus |
| **T3** | No-solder deployment at scale. | The actual differentiator vs. a modchip |

A modchip delivers T2/T3 for roughly £15 and a soldering iron. We are not trying to beat that on convenience for a single console — for one personal unit a modchip is genuinely the rational choice. We are trying to beat it on: no soldering, reversibility, mass deployment, and units you do not want opened.

## Non-goals

- Pirated content pipelines. We assume the reader can get their own dumps.
- Switch 2 internals, except where Switch 1 content is parsed by its compatibility layer.
- Emulator development for its own sake.
- Any claim that has not been reproduced cold on hardware.

## Principles

1. **Fidelity over throughput.** Hardware in the loop is the only truth. Emulators are a pre-filter and a liar.
2. **Diff before fuzz.** Nintendo's patches are a free bug oracle. Read them first.
3. **Falsifiable milestones with kill criteria.** Every track dies on a stated date unless it earns its keep.
4. **Publishable from day one.** Notes are written to be read by someone who was not there.
5. **No primitive is real until it survives a cold boot and a clean image.**

## Map

- [docs/00-thesis.md](docs/00-thesis.md) — the argument, the tier model, what would falsify us
- [docs/01-constraints.md](docs/01-constraints.md) — hard technical bounds
- [docs/02-attack-surface.md](docs/02-attack-surface.md) — scoring model and ranked matrix
- [docs/03-lab.md](docs/03-lab.md) — hardware, software, harness patterns, triage
- [docs/04-methods.md](docs/04-methods.md) — diffing, symbolic gates, LLM triage and its limits
- [docs/05-roadmap.md](docs/05-roadmap.md) — phases, milestones, kill criteria, risk register
- [docs/06-dossiers/](docs/06-dossiers/) — per-target working notes
- [docs/06-dossiers/gameshare.md](docs/06-dossiers/gameshare.md) — PC-hosted GameShare / `splay` receiver surface (reserve)
- [docs/07-open-questions.md](docs/07-open-questions.md) — must-verify facts before any commitment
- [docs/08-decisions.md](docs/08-decisions.md) — ADR log
- [docs/09-inventory.md](docs/09-inventory.md) — actual kit, blocked tracks, gaps
- [docs/10-v1-prep.md](docs/10-v1-prep.md) — hands-on v1 preparation and safety protocol
- [shopping_list.md](shopping_list.md) — capability hypotheses, costed, with the reasoning attached

## Legal posture

We work from our own console dumps and our own derived keys. No copyrighted blobs are committed. Toolchain is GPL/MIT and public. Nintendo has never litigated against a published softmod as such; the risk is commercial, not criminal.
