<div align="center">
  <img src="logo.svg" width="160" alt="SciGym Logo"/>
  <h1>SciGym</h1>
  <p><strong>Scientific Research Automation Workflow — AI Agent Skill</strong></p>
  <p>
    <a href="README.md">中文</a>
  </p>
</div>

---

## What is SciGym?

SciGym is a **research automation workflow document set** that guides AI agents in helping researchers build 24/7 autonomous experiment loops.

Core pipeline: **Benchmark (independent validation) → Research Environment (CLI / Gym) → AutoRun (autonomous execution)**

Without Benchmark, RL will fool itself. Without a good research environment, exploration costs too much. Without AutoRun, researchers stay chained to their machines.

## Installation

Just send this link to your Agent and let it read the docs and install:

```
https://github.com/Osgood001/scigym-skill
```

The Agent will read the relevant documents, confirm details with you interactively, then execute the setup.

## Documents

| Document | Content |
|----------|---------|
| [skills/scigym/SKILL.md](skills/scigym/SKILL.md) | Full pipeline overview & entry point |
| [phase1-benchmark.md](skills/scigym/phase1-benchmark.md) | Extract paper/hardware data, build validation scripts |
| [phase2-gym.md](skills/scigym/phase2-gym.md) | CLI + logging first; Gymnasium / FastAPI optional |
| [phase3-interface.md](skills/scigym/phase3-interface.md) | Mock / Hardware / HPC backend switching |
| [phase4-autorun.md](skills/scigym/phase4-autorun.md) | Cryochamber setup, CLAUDE.md task description, Zulip |
| [submit.md](skills/scigym/submit.md) | scigym.json manifest + GitHub Topics + registry |

## Core Design

**Research Environment — three layers (different use cases, not maturity levels)**

- **CLI + logging** — fastest to ship; Agent calls via subprocess; most flexible
- **FastAPI server** — stateful sessions; multi-step experiments share context
- **Gymnasium env** — maintains a running environment state; ground truth hidden from Agent (prevents reward hacking); ideal for constrained experiments, RL training, or theory-experiment hybrid research

Every step logs `--motivation` ("why are we running this experiment") + obs + result — a fully auditable research trail.

**Benchmark Principles**
1. **Independent** — never use the same model for both training and validation
2. **Auditable** — data sources must be traceable (paper DOI / experiment records)
3. **Quantified error** — provide confidence bounds

**AutoRun Safety Rules** (written in CLAUDE.md)
- Parameters must have boundary checks
- N consecutive runs with no improvement → hibernate

## Links

- [scigym-registry](https://github.com/Osgood001/scigym-registry) — auto-discovery registry
- [GiggleLiu/cryochamber](https://github.com/GiggleLiu/cryochamber) — AI agent persistence framework

---

<div align="center">
  <sub>Powered by Cryochamber · Indexed by scigym-registry</sub>
</div>
