<div align="center">
  <img src="logo.svg" width="160" alt="SciGym Logo"/>
  <h1>SciGym</h1>
  <p><strong>Automated Research Machine — Claude Code Skill</strong></p>
  <p>
    <a href="README_EN.md">English</a>
  </p>
</div>

---

## SciGym 是什么？

SciGym 是一个 **Claude Code Skill**，帮助科研人员将重复性实验自动化，构建 24/7 自主运行的 AI 实验循环。

核心框架：**Benchmark（独立验证）→ 研究环境（CLI / Gym）→ AutoRun（自主执行）**

没有 Benchmark，RL 会自欺欺人；没有好的实验环境，探索成本过高；没有 AutoRun，人就被绑在机器旁边。三者缺一不可。

## 快速上手

```bash
git clone https://github.com/Osgood001/scigym-skill \
    ~/.claude/skills/scigym

# 在 Claude Code 中调用：
# /scigym
```

## 子 Skill 说明

| Skill | 触发词 | 功能 |
|-------|--------|------|
| `/scigym` | 建 gym / SciGym / 自动化研究 | 完整 pipeline 入口 |
| `/scigym-benchmark` | 建 benchmark / 验证数据 | 从论文/硬件提取基准数据，编写验证脚本 |
| `/scigym-gym` | 建研究环境 / 包成 CLI | CLI + logging 优先，Gymnasium / FastAPI 可选 |
| `/scigym-interface` | 接硬件 / 接集群 | Mock / 硬件 / HPC 后端统一切换 |
| `/scigym-autorun` | 接 Cryochamber / 让它自己跑 | Cryochamber 配置、CLAUDE.md 任务描述、Zulip 推送 |
| `/scigym-submit` | 发布 gym / 提交 registry | scigym.json 清单 + GitHub Topics + 注册表收录 |

## 核心设计原则

**研究环境三层（按需选择，不是高低）**
- **CLI + logging** — 最快落地，Agent 直接 subprocess 调用，最灵活
- **FastAPI server** — 有状态会话，多步实验共享上下文
- **Gymnasium env** — 维护运行中环境状态，底层对 Agent 不透明（防 reward hack），适合受限实验 / RL 训练 / 理论-实验混合研究

每步记录 `--motivation`（"为什么做这个实验"）+ obs + result，形成完整可追溯的研究日志。

**Benchmark 三原则**
1. **独立性** — 不能用训练 RL 的同一个模型做验证
2. **可审计** — 数据来源必须可追溯（论文 DOI / 实验记录）
3. **量化误差** — 给出置信区间，不接受"看起来差不多"

**AutoRun 安全规则**（写在 CLAUDE.md 里）
- 参数必须有边界检查
- 连续 N 次无改进 → hibernate，不乱跑

## 真实案例

| 项目 | 领域 | 状态 |
|------|------|------|
| **fib-gym** | 聚焦离子束（FIB）加工参数优化 | Benchmark 已验证（误差 <5%） |
| **quantum-cal-gym** | 超导量子比特校准 | 真机接口已接通，Cryochamber 运行中 |
| **AutoResearch** | 晶体结构 RL 搜索 + DFT 验证 | 阿里云 24/7 自主运行 |

## 发布你的研究环境

```bash
# 1. 在 repo 根目录放 scigym.json
cp scigym.json.template scigym.json

# 2. 添加 GitHub topic
gh api repos/你的用户名/你的repo名/topics \
  -X PUT -f 'names[]=scigym-research-env'

# 3. 自动出现在 scigym-registry
```

## 相关链接

- [scigym-registry](https://github.com/Osgood001/scigym-registry) — 自动发现注册表
- [fib-gym](https://github.com/Osgood001/fib-gym) — FIB 加工 Gym
- [quantum-cal-gym](https://github.com/Osgood001/quantum-cal-gym) — 量子校准 Gym
- [GiggleLiu/cryochamber](https://github.com/GiggleLiu/cryochamber) — AI Agent 持久化框架

---

<div align="center">
  <sub>Built with Claude Code · Powered by Cryochamber · Indexed by scigym-registry</sub>
</div>
