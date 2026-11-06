# 00 — Thesis, tiers, and falsification

## The claim on the table

That the Picofly and its relatives are the only remaining route into a patched Switch.

Three separate propositions hide inside that sentence, and they fail differently:

1. **"There are no software entry points left."** Almost certainly false. Every firmware ships new parsers, new formats and new WebKit. Entry points are the cheapest thing in the world to create; the scene simply stopped shipping them because they stopped paying.
2. **"Even with an entry point, the chain is not completable."** This is the real claim, and it is about *kernel bug scarcity*, not parser scarcity.
3. **"Therefore the modchip is the only rational answer."** A cost argument, not a technical one. It is true for one console and one owner. It is false for a fleet, for a sealed unit, or for anyone unwilling to solder.

Most public discussion collapses 2 into 1. That collapse is the thing this programme exists to break.

## The chain, and which link is actually scarce

```
attack surface -> parser -> memory corruption -> info leak -> arbitrary R/W -> kernel -> persistence
      cheap         cheap        medium             medium        medium        SCARCE      medium
```

Every link except the last third is commodity. Switch 1 is a Cortex-A57 on ARMv8.0-A:

- **No PAC, no BTI, no CFI, no MTE, no memory tagging.**
- Stack canaries only in whatever library happened to compile with them — treat their presence as per-binary, never as a platform property.
- Mitigations are ASLR, NX, and W^X on JIT pages.

That is roughly a 2015 defensive posture. A stack smash here is still a straight line to ROP, and a heap overflow is still a straight line to a vtable or a function pointer. The hard part is not the corruption. The hard part is that after a userland bug you must *reach kernel*, and kernel bugs on 19.x/20.x are the single scarcest resource in the ecosystem precisely because nobody has needed one since 2018.

**So: the programme is not primarily a parser hunt. It is a parser hunt in service of a kernel hunt, and the kernel track starts on day one, not in week three.**

## Tiers

| Tier | Capability | Requires | Status |
| --- | --- | --- | --- |
| T0 | Session-only userland ROP, NRO loader, save editor | Entry + primitive + leak | A real result |
| T1 | T0 + kernel: full per-boot CFW with a re-trigger | T0 + kernel bug + KASLR leak | The goal |
| T2 | Persistence across cold boot with no re-trigger | T1 + a persist vector | Bonus |
| T3 | Deployment without solder, at fleet scale | A delivery channel with near-zero friction | The differentiator |

T0 alone is already worth having. A T0 chain that only works until reboot is still a jailbreak for that session, and it is enough to dump saves, extract keys, run homebrew and — crucially — *become a better lab*. Do not let T2-or-nothing thinking kill a T0 result.

## What we are actually betting on

Not "the parsers are buggy" — they are, but that is a commodity. The bet is narrower:

> **The kernel surface on 19.x/20.x is under-hunted because the modchip removed the economic motive, and firmware diffs will surface the validation work Nintendo has been doing quietly.**

Supporting reasoning:

- Patch diffing between adjacent firmwares ranks functions by "got shorter and sprouted a bounds check." That is a list of bugs Nintendo already knows about. It has never been cheaper to produce.
- Switch 1 is terminal. Attention is on Switch 2, backports are slower, and a bug found now lives longer than one found in 2019.
- The current Switch 2 jailbreak targets 23.0.1 — literally our firmware. A chain that lands here has a plausible port and is worth two consoles at once.
- The scene's kernel-bug supply is not exhausted, it is un-priced. Nobody has needed a kernel bug since the modchip became cheap.

## What would falsify the thesis

State these up front so we cannot move the goalposts later:

- **F1.** Across four consecutive firmware versions, `Kernel`, `spl`, `pm`, `ldr`, `ro` and `fs` show no security-relevant validation changes in the diff. If the kernel really is frozen-hardened, T1 is out of reach and the programme re-scopes to T0-only.
- **F2.** Every tier-A entry point reaches only a process with no useful service access and no pivot. Then we hold a pile of crashes and no chain.
- **F3.** Six weeks of harness work on a tier-A entry produces zero reproducible crashes. Then the surface is genuinely clean.
- **F4.** A delivery channel cannot be made cheaper than "solder a chip." Then T3 evaporates and only the intellectual case survives.

If F1 and F3 both fire, the honest conclusion is that the modchip really is the answer and this becomes a well-documented dead end. That is a publishable outcome too.

## The uncomfortable part

There is a real chance the scene is *correct*. Five years of no softmod is evidence. The strongest counter-argument is not "nobody looked" — some people did — it is that Nintendo shipped an unusually well-engineered OS: microkernel-shaped, small parsers, capability-gated services, no shell, no user-installable content. Horizon is genuinely better than iOS or Android at this, and by a lot.

The response to that is not bravado. It is to aim at the parts of Horizon that are *not* small — WebKit, the emulators, the media path, the wireless stack — and to accept that the realistic yield is T0 on a well-chosen target, not T1.

## Prior art lineage to study, not repeat

Wii Twilight Hack, Letterbomb. 3DS Ninjhax, oot3dhax, Soundhax, basehaxx. PSP GTA/Lumines. PS Vita HENkaku. PS4 WebKit. Switch Pegaswitch, Deja Vu, Nereba/Caffeine.

The consistent lesson across all of them: **the winning entry point is the parser the platform holder wrote casually, delivered by the channel with the least friction.** Not the most elegant bug, not the deepest one. The casual parser, the cheap delivery. That is the selection rule this programme uses.
