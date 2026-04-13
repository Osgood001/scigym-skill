# SciGym — Automated Research Machine

## When to Use This Skill

触发条件（满足任一）：
- 用户说「建 gym」「SciGym」「自动化研究环境」「auto research」
- 用户有一套模拟器/实验装置，想让 AI Agent 自主运行实验
- 用户想把研究环境发布给社区复用

**子技能**：调用对应子技能处理各阶段
- `/scigym:benchmark` — 从论文/真机数据建立 benchmark
- `/scigym:gym` — 封装 Gymnasium 环境（RL + Coding Agent 两套接口）
- `/scigym:interface` — 连接真实硬件/计算集群
- `/scigym:autorun` — 部署 Cryochamber 实现 24/7 自主实验
- `/scigym:submit` — 发布到社区，附加 scigym.json + GitHub topic

---

## 什么是 SciGym？

SciGym = **Benchmark + Gym + Interface + AutoResearch**

```
                    ┌─────────────────────────────────┐
                    │           SciGym Package            │
                    │                                  │
  Paper / Hardware ─► Benchmark  ◄─ 独立验证，防 hack   │
       data         │     ▼                            │
  Physics model  ──►  Gym Env   ◄─ Gymnasium 协议      │
  (sim / MLFF)   │     ▼                               │
  Real hardware  ──►  Interface ◄─ API / CLI / SDK     │
                 │     ▼                               │
  Cluster/Cloud  ──►  AutoRun  ◄─ Cryochamber 24/7    │
                    └─────────────────────────────────┘
                              ▼
                    scigym.json + topic:scigym-research-gym
                    → 中心网站自动发现
```

**核心洞察**：给 Agent 一个 Gym（而不是 prompt）之后：
1. Benchmark = 环境本身，无法被 reward hack
2. 通过 sim-to-real gap 量化你对结果的信心
3. Agent 在集群上 24/7 并行探索，你只审阅报告
4. 每次实验自动记录 obs/action/reward，完全可复现

---

## Pipeline 总览

```
Phase 1: Benchmark  →  Phase 2: Gym  →  Phase 3: Interface  →  Phase 4: AutoRun  →  Submit
  从论文提取指标        封装 Gym env       连接真实硬件/计算        Cryochamber 部署       发布
```

详见子技能文档：
- [phase1-benchmark.md](phase1-benchmark.md)
- [phase2-gym.md](phase2-gym.md)
- [phase3-interface.md](phase3-interface.md)
- [phase4-autorun.md](phase4-autorun.md)
- [submit.md](submit.md)

---

## 快速启动（已有 Gym 的情况）

```bash
# 1. 确认 scigym.json 存在
cat scigym.json

# 2. 启动 Gym server（如果支持 coding-agent 模式）
uvicorn your_gym.server:app --port 8765

# 3. 启动 AutoRun
cryo start   # 见 phase4-autorun.md
```
