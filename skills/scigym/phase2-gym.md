# Phase 2: Research Environment — CLI 优先，Gymnasium 可选

## 核心观点

> Agent 天然适合 CLI，不适合 GUI。把科研环境包装成 CLI，就已经是一个很好的研究系统。

**Gymnasium 是一种抽象**，对成熟的环境很有价值（统一接口、支持 RL 训练），但它不是必须的。更务实的路径：

```
最简可用：  CLI 脚本 + logging
进一步：    CLI + FastAPI server（Agent 用 HTTP 调用）
完整形态：  Gymnasium env（包一层 gym.Env，底层还是那些 CLI）
```

选哪层取决于项目成熟度和实际需求，不要为了"标准"而过度设计。

---

## 第一步：包装成 CLI

把你的实验/模拟代码暴露为可以直接调用的命令行工具。

```bash
# 示例：量子校准环境
python run_experiment.py s21 --qubit Q0 --freq-center 5.2e9 --span 100e6 \
    --motivation "找腔频，第一步"

# 示例：材料模拟
python run_dft.py --structure Ti4Mn6Rh2.cif --property energy \
    --motivation "验证 ORB 弛豫后的 ehull"

# 查看历史记录
python show_log.py --last 10
python show_log.py --experiment-type s21 --date today
```

Agent 直接调用这些命令，读取 stdout/log，决定下一步。**不需要任何中间层。**

### CLI 设计原则

- **每个命令做一件事**，输出结构化结果（JSON 或固定格式文本）
- **必有 `--motivation` 参数**（见下方 Logging 章节）
- 参数用物理单位，不要归一化（CLI 是给人和 Agent 看的，不是给 RL 看的）
- 命令失败时明确报错（exit code 非 0，stderr 说明原因）

```python
# run_experiment.py 骨架
import argparse, json, sys
from your_module import run_s21
from your_logger import ResearchLogger

parser = argparse.ArgumentParser()
parser.add_argument('experiment')           # s21 / rabi / t1 / ramsey
parser.add_argument('--qubit', default='Q0')
parser.add_argument('--motivation', required=True, help='为什么要做这个实验')
parser.add_argument('--params', type=json.loads, default={})
args = parser.parse_args()

logger = ResearchLogger()

with logger.step(
    experiment_type=args.experiment,
    motivation=args.motivation,
    params=vars(args),
) as step:
    result = run_s21(qubit=args.qubit, **args.params)
    step.record(result)          # 自动上传数据和图
    print(json.dumps(result))    # stdout 给 Agent 读
```

---

## Logging 系统：每步都有记录

参考 `qubit-characterize` skill 的 `lab_logger_client` 模式。

### 核心设计

每次实验调用必须产生一条日志，包含：

| 字段 | 内容 |
|------|------|
| `motivation` | **为什么做这个实验**（必填，Agent 在 `--motivation` 里写） |
| `experiment_type` | 实验类型（s21 / rabi / t1 / dft / mlff_relax / …） |
| `params` | 本次用的参数 |
| `result` | 关键数值结果（不含原始数据） |
| `verdict` | pass / fail / retry（质量门控结论） |
| `figures` | 附件图片路径列表 |
| `timestamp` | ISO 格式 |

### 本地 JSONL 实现（最简）

```python
# your_project/logger.py
import json, time, uuid
from pathlib import Path
from contextlib import contextmanager

LOG_FILE = Path("runs/log.jsonl")
LOG_FILE.parent.mkdir(parents=True, exist_ok=True)

@contextmanager
def log_step(experiment_type: str, motivation: str, params: dict):
    entry = {
        "id": str(uuid.uuid4())[:8],
        "timestamp": time.strftime("%Y-%m-%dT%H:%M:%S"),
        "experiment_type": experiment_type,
        "motivation": motivation,
        "params": params,
        "result": None,
        "verdict": None,
        "figures": [],
    }
    try:
        yield entry
    finally:
        with open(LOG_FILE, "a") as f:
            f.write(json.dumps(entry, ensure_ascii=False) + "\n")

# 用法：
with log_step("s21", motivation="找腔频，第一步", params={"freq_center": 5.2e9}) as step:
    result = run_s21(...)
    step["result"] = {"f_cavity_GHz": result.freq / 1e9, "r2": result.r2}
    step["verdict"] = "pass" if result.r2 > 0.9 else "fail"
    step["figures"] = [save_figure(result)]
```

### 接 Lab Logger Server（推荐，有 Web UI）

如果项目里有 lab logger 服务（参考 `qubit-characterize` skill 的配置）：

```python
from lab_logger_client import LabLoggerClient

logger = LabLoggerClient("http://your-server:8000")
project = logger.find_or_create_project(name="your-project", machine_id=...)

with logger.experiment_session(
    project_id=project.id,
    machine_id=machine.id,
    title=f"{experiment_type} — {motivation}",
    metadata={
        "experiment_type": experiment_type,
        "motivation": motivation,   # ← 关键字段
        **params,
    },
) as exp:
    result = run_experiment(...)
    logger.update_experiment(exp.id, content=format_result(result))
    logger.upload_attachment(exp.id, file_path=fig_path)
```

### 查看日志的 CLI

```bash
# 查最近 10 条（给 Agent 用）
python show_log.py --last 10

# 查某类实验
python show_log.py --type s21

# 查失败的
python show_log.py --verdict fail

# 给 Agent 的标准输出格式（紧凑 JSON，便于 LLM 解析）
python show_log.py --last 5 --format compact
```

输出示例（Agent 可以直接读）：
```
[08:42] s21        pass  f_cav=5.201GHz  r2=0.97  "找腔频，第一步"
[08:55] spectrum   pass  f_q=4.883GHz    r2=0.94  "确认比特频率"
[09:12] power_rabi fail  pi_amp=?        r2=0.61  "Rabi振幅太小，需要调measure_amp"
[09:18] power_rabi pass  pi_amp=0.039    r2=0.95  "调大measure_amp后重做"
```

---

## 第二步：FastAPI Server（可选，Agent 用 HTTP 调用）

当项目需要**有状态的会话**（多步实验共享状态），或者想让 Agent 用 HTTP 而不是 subprocess 调用时，加一层 FastAPI：

```python
# server.py
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class ExperimentRequest(BaseModel):
    experiment: str        # "s21" / "rabi" / "t1" / "ramsey"
    params: dict = {}
    motivation: str        # Agent 必须说明为什么

@app.post("/run")
def run(req: ExperimentRequest):
    with log_step(req.experiment, req.motivation, req.params) as step:
        result = dispatch(req.experiment, req.params)
        step["result"] = result
        step["verdict"] = quality_gate(result)
    return {"result": result, "verdict": step["verdict"]}

@app.get("/log")
def get_log(last: int = 10):
    return read_last_n(LOG_FILE, last)
```

Agent 的 CLAUDE.md 里只需要：
```markdown
## 实验接口

POST http://localhost:8765/run
{"experiment": "s21", "params": {...}, "motivation": "你为什么做这个"}

GET http://localhost:8765/log?last=5
```

---

## 第三步：Gymnasium 封装（成熟项目）

如果项目已经稳定、想支持 RL 训练，在 CLI/Server 上面包一层 `gym.Env`：

```python
class YourEnv(gym.Env):
    def step(self, action):
        params = self._action_to_params(action)
        # 调用底层 CLI 或 Server，而不是重新实现逻辑
        result = subprocess.run(
            ["python", "run_experiment.py", self.exp_type, "--motivation", "RL step",
             "--params", json.dumps(params)],
            capture_output=True, text=True
        )
        obs = self._parse_result(json.loads(result.stdout))
        reward = self._compute_reward(obs)
        return obs, reward, False, False, {}
```

**Gymnasium 是外壳，CLI 是内核。** 这样 Gymnasium 环境自然就有了完整的 logging（底层 CLI 已经在记录了）。

---

## 项目结构（CLI-first）

```
your-project/
  run_experiment.py     # 主 CLI 入口
  show_log.py           # 日志查询 CLI
  your_module/
    experiments.py      # 实验函数
    simulator.py        # 物理模型
    logger.py           # log_step 上下文管理器
    quality.py          # R² / verdict 判断逻辑
  server.py             # FastAPI（可选）
  env.py                # Gymnasium 封装（可选）
  runs/
    log.jsonl           # 所有实验记录
    figures/            # 自动保存的图
  benchmark/            # Phase 1 输出
  CLAUDE.md             # Agent 任务说明
```

---

## 关键：CLAUDE.md 怎么告诉 Agent 用这套系统

```markdown
## 实验工具

运行实验：
  python run_experiment.py <type> --motivation "<你的理由>" [--params '{"key": val}']

查看历史：
  python show_log.py --last 10

实验类型：s21 / spectrum / power_rabi / t1 / ramsey

## 要求

- 每次调用 run_experiment.py 前，先查 show_log.py 了解当前状态
- --motivation 必须写清楚"为什么做这个实验"，不能写"继续实验"这种无意义的话
- verdict=fail 时，分析原因，调整参数，不要直接重试相同参数
```
