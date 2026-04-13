<div align="center">
  <img src="logo.svg" width="160" alt="SciGym Logo"/>
  <h1>SciGym</h1>
  <p><strong>Automated Research Machine — Claude Code Skill</strong></p>
  <p>
    <a href="README.md">中文</a>
  </p>
</div>

---

## What is SciGym?

SciGym is a **Claude Code skill** that guides researchers in building fully automated scientific experiment loops. It provides a structured pipeline — **Benchmark → Research Environment → AutoRun** — to turn repetitive lab experiments into 24/7 autonomous AI workflows.

## Quick Start

```bash
git clone https://github.com/Osgood001/scigym-skill \
    ~/.claude/skills/scigym

# Invoke in Claude Code:
# /scigym
```

## Sub-skills

| Skill | Trigger | What it does |
|-------|---------|--------------|
| `/scigym` | `SciGym / auto research` | Full pipeline overview & entry point |
| `/scigym-benchmark` | `build benchmark` | Extract paper data, build validation scripts |
| `/scigym-gym` | `wrap as CLI / build env` | CLI + logging first; Gymnasium / FastAPI optional |
| `/scigym-interface` | `connect hardware / cluster` | Mock / Hardware / HPC backend switching |
| `/scigym-autorun` | `connect Cryochamber` | Cryochamber setup, CLAUDE.md task description, Zulip integration |
| `/scigym-submit` | `publish gym` | `scigym.json` manifest + GitHub Topics + registry submission |

## Core Design

**Research Environment — three layers (different use cases, not maturity levels)**

- **CLI + logging** — fastest to ship; Agent calls via subprocess; most flexible
- **FastAPI server** — stateful sessions; multi-step experiments share context
- **Gymnasium env** — maintains a running environment state; ground truth hidden from Agent (prevents reward hacking); ideal for constrained experiments, RL training, or theory-experiment hybrid research

Every step logs `--motivation` ("why are we running this experiment") + obs + result — a fully auditable research trail.

**Benchmark Principles**
1. **Independent** — never use the same model for both training and validation
2. **Auditable** — data sources must be traceable (paper DOI / experiment records)
3. **Quantified error** — provide confidence bounds, not just "looks close enough"

**AutoRun Safety Rules** (written in CLAUDE.md)
- Parameters must have boundary checks
- N consecutive runs with no improvement → hibernate

## Real-world Examples

| Project | Domain | Status |
|---------|--------|--------|
| **fib-gym** | FIB milling parameter optimization | Benchmark verified (<5% error) |
| **quantum-cal-gym** | Superconducting qubit calibration | Real hardware connected, Cryochamber running |
| **AutoResearch** | Crystal structure RL search + DFT verification | 24/7 on Aliyun |

## Pipeline

```
Phase 1: Benchmark
  └─ Extract metrics from papers / hardware measurements
  └─ Write validation script → quantified error bounds

Phase 2: Research Environment
  └─ CLI: run_experiment.py --motivation "..." → log.jsonl
  └─ Optional: FastAPI server for stateful sessions
  └─ Optional: Gymnasium env for RL / constrained experiments

Phase 3: Interface
  └─ Mock → Hardware → HPC (unified config switch)

Phase 4: AutoRun
  └─ gh repo view GiggleLiu/cryochamber --readme
  └─ cryo init → cryo.toml + CLAUDE.md
  └─ cryo daemon + cron keepalive + Zulip notifications

Phase 5: Submit
  └─ scigym.json + GitHub topic scigym-research-env
  └─ Auto-discovered by scigym-registry
```

## Requirements

- [Claude Code](https://github.com/anthropics/claude-code) (claude-sonnet-4-6 or later)
- Python ≥ 3.10, `numpy`; `gymnasium` optional
- For AutoRun: [GiggleLiu/cryochamber](https://github.com/GiggleLiu/cryochamber)

## Links

- [scigym-registry](https://github.com/Osgood001/scigym-registry) — auto-discovery registry
- [fib-gym](https://github.com/Osgood001/fib-gym) — FIB milling gym
- [quantum-cal-gym](https://github.com/Osgood001/quantum-cal-gym) — qubit calibration gym
- [GiggleLiu/cryochamber](https://github.com/GiggleLiu/cryochamber) — AI agent persistence framework

---

<div align="center">
  <sub>Built with Claude Code · Powered by Cryochamber · Indexed by scigym-registry</sub>
</div>
