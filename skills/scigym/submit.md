# Submit — 发布 SciGym Gym 到社区

## 发布流程概览

```
1. 在 repo 根目录添加 scigym.json（元数据）
2. 在 GitHub repo 添加 Topics
3. （可选）向 scigym-registry repo 提交索引
```

---

## Step 1: scigym.json 元数据文件

在 repo 根目录创建 `scigym.json`：

```json
{
  "arm_version": "1.0",
  "name": "your-gym",
  "description": "短描述，一句话说清楚这个 Gym 做什么",
  "domain": "nanofabrication",
  "gym_id": "YourEnv-v0",
  "paper": "https://doi.org/10.xxxx/xxxxx",
  "benchmark": {
    "source": "paper",
    "metrics": ["metric_name_1", "metric_name_2"],
    "validated": true,
    "script": "benchmark/validation/test_gym.py"
  },
  "simulator": {
    "backend": "your-physics-lib",
    "fidelity_levels": ["low", "medium", "high"],
    "interfaces": ["gymnasium", "http-api"]
  },
  "hardware_interface": {
    "available": false,
    "sdk": "your-sdk-name",
    "setup_doc": "docs/hardware-setup.md"
  },
  "autorun": {
    "ready": true,
    "cryochamber": true,
    "claude_md": "CLAUDE.md"
  },
  "authors": ["your-github-username"],
  "license": "MIT"
}
```

---

## Step 2: 添加 GitHub Topics

### 方法 A：网页操作
1. 打开你的 GitHub repo 页面
2. 点击右上角 ⚙️（About 旁边的齿轮）
3. 在 Topics 输入框里添加：
   - `scigym-research-gym` ← **必须加这个**，用于中心网站发现
   - `gymnasium` ← Gymnasium 生态
   - 你的领域（如 `nanofabrication` / `quantum-computing` / `materials-science`）

### 方法 B：命令行（gh CLI）
```bash
gh api repos/YOUR_USERNAME/YOUR_REPO/topics \
  -X PUT \
  -H "Accept: application/vnd.github.mercy-preview+json" \
  -f 'names[]=scigym-research-gym' \
  -f 'names[]=gymnasium' \
  -f 'names[]=your-domain'
```

验证是否成功：
```bash
gh api repos/YOUR_USERNAME/YOUR_REPO/topics
```

### 中心网站发现原理

中心网站通过 GitHub API 自动搜索所有带 `scigym-research-gym` topic 的 repo：

```
GET https://api.github.com/search/repositories?q=topic:scigym-research-gym&sort=updated
```

只要你加了这个 topic，下次中心网站更新时就会自动出现。

---

## Step 3: scigym-registry（中心索引 Repo）

中心索引：https://github.com/Osgood001/scigym-registry
（如果还没创建，见下方）

SciGym Registry 是一个自动维护的 index，每天通过 GitHub Actions cron job 拉取最新的 `topic:scigym-research-gym` repos，更新 README。

---

## scigym-registry README 模板格式

每个 Gym 的条目格式：
```markdown
| Repo | Domain | Benchmark | Sim Fidelity | Hardware | AutoRun | Updated |
|------|--------|-----------|-------------|----------|---------|---------|
| [fib-gym](https://github.com/Osgood001/fib-gym) | nanofabrication | ✅ paper | low/med/high | ⚠️ FIB | ✅ | 2026-04 |
| [quantum-cal-gym](https://github.com/Osgood001/quantum-cal-gym) | quantum-computing | ✅ hardware | low/high | ⚠️ quark | ✅ | 2026-04 |
```

---

## 发布检查清单

```
☐ scigym.json 存在于 repo 根目录
☐ benchmark/ 目录包含论文数据或真机数据
☐ benchmark/validation/ 有自动化验证脚本
☐ examples/random_agent.py 可以运行
☐ README.md 说明了 Interface 类型（mock/hardware/hpc）
☐ GitHub topic 已添加 scigym-research-gym
☐ （可选）CLAUDE.md 写好 AutoRun 任务说明
```
