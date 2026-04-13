<div align="center">
  <img src="logo.svg" width="160" alt="SciGym Logo"/>
  <h1>SciGym</h1>
  <p><strong>Automated Research Machine — Claude Code Skill</strong></p>
  <p>
    <a href="#english">English</a> · <a href="#中文">中文</a>
  </p>
</div>

---

## English

### What is SciGym?

SciGym is a **Claude Code skill** that guides researchers in building fully automated scientific experiment loops. It provides a structured pipeline — **Benchmark → Simulator → AutoRun** — to turn repetitive lab experiments into 24/7 autonomous AI workflows.

```
              📐 Benchmark
             /             \
       验证边界             反馈调参
           /                 \
  🔬 Simulator  ——快速试错——  🤖 AutoRun
```

| Component | Role |
|-----------|------|
| **Benchmark** | Independent ground truth: paper data / real hardware / DFT |
| **Simulator** | Gymnasium-compatible environment (Low / Medium / High fidelity) |
| **AutoRun** | Cryochamber-based AI agent, runs 24/7, reports via Zulip |

### Quick Start

Install the skill into Claude Code:

```bash
# Clone into Claude Code skills directory
git clone https://github.com/Osgood001/scigym-skill \
    ~/.claude/skills/scigym

# Done — invoke in Claude Code:
# /scigym
```

### Sub-skills

| Skill | Trigger | What it does |
|-------|---------|--------------|
| `/scigym` | `建 gym / SciGym / 自动化研究` | Full pipeline overview & entry point |
| `/scigym-benchmark` | `建 benchmark / 验证数据` | Extract paper data, build validation scripts |
| `/scigym-gym` | `建 Gymnasium 环境` | Implement `gym.Env` subclass + FastAPI server |
| `/scigym-interface` | `接硬件 / 接集群` | Mock / Hardware / HPC backend switching |
| `/scigym-autorun` | `接 Cryochamber / 让它自己跑` | Cryochamber setup, CLAUDE.md task description, Zulip integration |
| `/scigym-submit` | `发布 gym / 提交 registry` | `scigym.json` manifest + GitHub Topics + registry submission |

### Pipeline

```
Phase 1: Benchmark
  └─ Extract metrics from papers (WebPlotDigitizer)
  └─ Or collect hardware measurements (JSONL)
  └─ Write validation script → quantified error bounds

Phase 2: Gym
  └─ Subclass gym.Env (reset / step / close)
  └─ Optional: FastAPI coding-agent interface
  └─ EpisodeLogger: per-step obs/action/reward → log.json

Phase 3: Interface
  └─ config={"backend": "mock|hardware|hpc"}
  └─ MockInterface  → dev & testing
  └─ HardwareInterface → real instruments via SDK
  └─ HPCInterface → SLURM/PBS submission via SSH

Phase 4: AutoRun
  └─ gh repo view GiggleLiu/cryochamber --readme  # 读官方 README 安装
  └─ cryo init → configure cryo.toml + CLAUDE.md
  └─ cryo daemon (keepalive cron every 30 min)
  └─ Zulip notifications (see cryochamber README)

Phase 5: Submit
  └─ Add scigym.json to repo root
  └─ gh api repos/OWNER/REPO/topics -X PUT -f 'names[]=scigym-research-env'
  └─ Auto-discovered by scigym-registry
```

### Registry

Published gyms are auto-discovered at [scigym-registry](https://github.com/Osgood001/scigym-registry).

To publish your gym:

1. Add `scigym.json` to repo root (use `scigym.json.template`)
2. Add GitHub topic `scigym-research-env`
3. Your gym appears in the registry within 24 hours

### Existing Gyms

| Repo | Domain | Hardware | AutoRun |
|------|--------|----------|---------|
| [fib-gym](https://github.com/Osgood001/fib-gym) | Nanofabrication (FIB milling) | — | — |
| [quantum-cal-gym](https://github.com/Osgood001/quantum-cal-gym) | Quantum computing (qubit calibration) | ✅ quark SDK | ✅ Cryochamber |

### Requirements

- [Claude Code](https://github.com/anthropics/claude-code) (claude-sonnet-4-6 or later)
- Python ≥ 3.10, `gymnasium`, `numpy`
- For AutoRun: [cryochamber](https://github.com/GiggleLiu/cryochamber)

---

## 中文

### SciGym 是什么？

SciGym 是一个 **Claude Code Skill**，帮助科研人员将重复性实验自动化，构建 24/7 自主运行的 AI 实验循环。

核心框架：**Benchmark（独立验证）→ Simulator（仿真环境）→ AutoRun（自主执行）**

没有 Benchmark，RL 会自欺欺人；没有 Simulator，探索成本过高；没有 AutoRun，人就被绑在机器旁边。三者缺一不可。

### 快速上手

```bash
# 安装到 Claude Code skills 目录
git clone https://github.com/Osgood001/scigym-skill \
    ~/.claude/skills/scigym

# 在 Claude Code 中调用：
# /scigym
```

### 子 Skill 说明

| Skill | 触发词 | 功能 |
|-------|--------|------|
| `/scigym` | 建 gym / SciGym / 自动化研究 | 完整 pipeline 入口 |
| `/scigym-benchmark` | 建 benchmark / 验证数据 | 从论文/硬件提取基准数据，编写验证脚本 |
| `/scigym-gym` | 建 Gymnasium 环境 | 实现 gym.Env 子类 + 可选 FastAPI 接口 |
| `/scigym-interface` | 接硬件 / 接集群 | Mock / 硬件 / HPC 后端统一切换 |
| `/scigym-autorun` | 接 Cryochamber / 让它自己跑 | Cryochamber 配置、CLAUDE.md 任务描述、Zulip 推送 |
| `/scigym-submit` | 发布 gym / 提交 registry | scigym.json 清单 + GitHub Topics + 注册表收录 |

### 核心设计原则

**Benchmark 三原则**
1. **独立性** — 不能用训练 RL 的同一个模型做验证
2. **可审计** — 数据来源必须可追溯（论文 DOI / 实验记录）
3. **量化误差** — 给出置信区间，不接受"看起来差不多"

**Simulator 三层 Fidelity**
- **Low**：秒级，快速试错，牺牲物理精度
- **Medium**：分钟级，RL 训练用
- **High**：接近真机，最终验证用

**AutoRun 安全规则**（写在 CLAUDE.md 里）
- 参数必须有边界检查
- 超时自动终止
- 连续 N 次无改进 → hibernate，不乱跑

### 真实案例

| 项目 | 领域 | 状态 |
|------|------|------|
| **fib-gym** | 聚焦离子束（FIB）加工参数优化 | Benchmark 已验证（误差 <5%） |
| **quantum-cal-gym** | 超导量子比特校准 | 真机接口已接通，Cryochamber 运行中 |
| **AutoResearch** | 晶体结构 RL 搜索 + DFT 验证 | 阿里云 24/7 自主运行 |

### 发布你的 Gym

```bash
# 1. 在 repo 根目录放 scigym.json（复制模板）
cp scigym.json.template scigym.json  # 编辑填写

# 2. 添加 GitHub topic
gh api repos/你的用户名/你的repo名/topics \
  -X PUT -f 'names[]=scigym-research-env'

# 3. 等 24 小时，自动出现在 scigym-registry
```

### 相关链接

- [scigym-registry](https://github.com/Osgood001/scigym-registry) — 自动发现注册表
- [fib-gym](https://github.com/Osgood001/fib-gym) — FIB 加工 Gym
- [quantum-cal-gym](https://github.com/Osgood001/quantum-cal-gym) — 量子校准 Gym
- [Cryochamber](https://github.com/GiggleLiu/cryochamber) — AI Agent 持久化框架

---

<div align="center">
  <sub>Built with Claude Code · Powered by Cryochamber · Indexed by scigym-registry</sub>
</div>
