# SciGym — Automated Research Machine

## 适用场景

触发条件（满足任一）：
- 用户有一套模拟器/实验装置，想让 AI Agent 自主运行实验
- 用户想把重复性科研流程自动化，构建 24/7 运行的实验循环
- 用户想把研究环境发布给社区复用

**子文档**：
- [phase1-benchmark.md](phase1-benchmark.md) — 建立独立验证基准
- [phase2-gym.md](phase2-gym.md) — 包装研究环境（CLI / Gymnasium / FastAPI）
- [phase3-interface.md](phase3-interface.md) — 连接真实硬件/计算集群
- [phase4-autorun.md](phase4-autorun.md) — 部署 Cryochamber 实现 24/7 自主实验
- [submit.md](submit.md) — 发布到社区

---

## 什么是 SciGym？

SciGym = **Benchmark + Research Environment + Interface + AutoRun**

```
                    ┌─────────────────────────────────┐
                    │           SciGym Package        │
                    │                                 │
  Paper / Hardware ─► Benchmark  ◄─ 独立验证，防 hack  │
       data         │     ▼                           │
  Physics model  ──►  CLI / API  ◄─ Agent 直接调用    │
  (sim / MLFF)   │   (Gym 可选)                       │
  Real hardware  ──►  Interface  ◄─ API / CLI / SDK   │
                 │     ▼                              │
  Cluster/Cloud  ──►  AutoRun   ◄─ Cryochamber 24/7  │
                    └─────────────────────────────────┘
                              ▼
                    scigym.json + topic:scigym-research-env
```

**核心洞察**：

> Agent 天然适合 CLI，不适合 GUI。把科研流程包装成 CLI + 好的 logging，就已经是一个优秀的研究系统。Gymnasium 是一种锦上添花的抽象，不是前提条件。

1. **Benchmark** = 独立验证，reward 无法被 hack
2. **CLI-first** = Agent 直接 subprocess 调用，灵活、可审计
3. **Logging with motivation** = 每步记录"为什么做这个实验"，Agent 行为完全可追溯
4. **AutoRun** = Agent 在服务器 24/7 自主探索，你只审阅 Zulip 推送

---

## Pipeline 总览

```
Phase 1: Benchmark  →  Phase 2: CLI Environment  →  Phase 3: Interface  →  Phase 4: AutoRun  →  Submit
  从论文提取指标        包装 CLI + logging              连接真实硬件/计算       Cryochamber 部署      发布
```

**Phase 2 成熟度阶梯（不是高低，是不同适用场景）：**
```
CLI + logging      — 最快落地，最灵活，Agent 直接 subprocess 调用
FastAPI server     — 有状态会话，多步共享上下文
Gymnasium env      — 维护运行中环境状态，适合受限实验 / RL / 理论-实验混合
```

Gymnasium 的核心不是"标准"，而是：**底层对 Agent 不透明**（它看不到真值，只能通过 step() 探索），防止 reward hack，实现 authentic 探索。CLI 也可以做到这点，但 Gymnasium 的 step/reset/obs 结构在需要跨步保持状态时更自然。

详见子技能文档：
- [phase1-benchmark.md](phase1-benchmark.md)
- [phase2-gym.md](phase2-gym.md)
- [phase3-interface.md](phase3-interface.md)
- [phase4-autorun.md](phase4-autorun.md)
- [submit.md](submit.md)

---

## 快速启动（已有环境的情况）

```bash
# 最简：直接用 CLI
python run_experiment.py s21 --motivation "找腔频" --qubit Q0

# 查日志
python show_log.py --last 10

# 有 server 时
uvicorn server:app --port 8765

# 启动 AutoRun
cryo daemon   # 见 phase4-autorun.md
```
