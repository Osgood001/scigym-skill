# Phase 4: AutoRun — 24/7 自主实验 via Cryochamber

## 目标

部署 **Cryochamber**（AI agent 持久化运行框架），让 Claude Code 在服务器上昼夜不停地运行实验、分析结果、通知你。

## 安装 Cryochamber

**正确 repo**：`GiggleLiu/cryochamber`

1. 用 `gh` 获取官方 README，里面包含了完整安装方式：
   ```bash
   gh repo view GiggleLiu/cryochamber --readme
   ```

2. 阅读 README，明确以下问题后再动手：
   - 目标机器是什么（本地 / 阿里云 / 集群）？
   - 安装到哪个路径（用户 bin / 项目目录 / 系统级）？
   - 有没有依赖（Python 版本 / Claude Code 路径）？

3. 与用户确认安装位置，然后按 README 的步骤执行。

**不要依赖本文档里的安装命令**——README 是权威，本文档只是引导流程。

## 安装完成后

1. 在研究项目目录下初始化：
   ```bash
   cd your-research-project
   cryo init
   ```

2. 编写 `CLAUDE.md`（Agent 每次醒来读取的任务说明书）：

   ```markdown
   # AutoRun: [你的项目名]

   ## Goal
   [一句话说明实验目标]

   ## Every Session Flow
   1. Orient：读 plan.md，了解上次状态
   2. Check：查实验队列——在跑/结束/失败分别怎么做
   3. Analyze：解析新数据，与 benchmark 对比
   4. Act：提交任务 / 调整参数 / 发推送
   5. Record：cryo-agent note "做了什么，下一步是什么"
   6. Hibernate：选合适的 wake 间隔

   ## Wake Schedule
   | 情况 | 间隔 |
   |------|------|
   | 刚提交实验 | 30min |
   | 正常运行中 | 2h |
   | 等待人工决策 | 6h |
   | 出错 | 15min |

   ## 安全规则
   - 不删实验数据
   - 不确定时发消息问，不乱猜
   ```

3. 参考 README 配置 `cryo.toml`（通知后端、API provider 等）。

4. 启动：
   ```bash
   cryo daemon   # 或 nohup cryo daemon ... &（无 systemd 时）
   cryo status
   ```

## 已知注意事项

- 无 systemd 的机器（如阿里云普通用户）：用 `nohup cryo daemon` 手动启动，配合 cron 每 30min keepalive
- Zulip 双向消息：需额外配置 `cryo-zulip sync`（见 README）
- API provider 备用配置：在 `cryo.toml` 里指定 fallback（如 gpugeek），需设置对应环境变量
