# Linux platform research — index

Research that shaped the Linux build of NEXUS (Pop!_OS 24.04, COSMIC, Wayland, PipeWire). The platform-constraints, voice-stack, distribution and Linux-gap studies from October 2026 are already in [`../2026-10-competitive-audit/`](../2026-10-competitive-audit/00-executive-summary.md) (`01-platform-constraints.md`, `06-computer-use-linux-gap.md`, `08-distribution-linux-packaging.md`); this folder adds the computer-control studies and the round-by-round history.

| File | What it answers |
|---|---|
| [01-computer-control-architecture-rethink.md](01-computer-control-architecture-rethink.md) | How NEXUS should see and act on a Linux desktop: semantic tiers (AT-SPI, CDP) before pixels |
| [02-computer-control-full-comparison.md](02-computer-control-full-comparison.md) | Every perception/actuation option compared |
| [03-perception-actuation-codebase-audit.md](03-perception-actuation-codebase-audit.md) | What the code actually did, file by file |
| [04-coverage-and-guardrails.md](04-coverage-and-guardrails.md) | Coverage by rendering engine and the safety guardrails |
| [05-every-browser-engine.md](05-every-browser-engine.md) | Browser engines and how each can be driven |
| [history/round-1-popos.md](history/round-1-popos.md) | First Pop!_OS round |
| [history/rounds-2-3-popos.md](history/rounds-2-3-popos.md) | Rounds 2–3 |
| [history/rounds-4-7-voice-loop.md](history/rounds-4-7-voice-loop.md) | Rounds 4–7: the Linux voice loop |

The October 2026 port that put this research onto the Windows codebase: [`../../features/58-linux-port-windows-base-and-linux-layer.md`](../../features/58-linux-port-windows-base-and-linux-layer.md).

Note: files 01–05 were written in the Linux repository as `docs/features/research/20–24`; their internal cross-references use those numbers.
