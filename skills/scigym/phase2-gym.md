# Phase 2: Gym — 封装 Gymnasium 环境

## 目标

将模拟器封装成标准 **Gymnasium v1** 环境，支持两种 Agent 范式：
1. **RL Agent** — 连续/离散 action space，训练强化学习模型
2. **Coding Agent** — HTTP API，Claude Code/GPT-4 直接调用

## Gymnasium 协议核心

参考官方文档：https://gymnasium.farama.org/

### 必须实现的接口

```python
import gymnasium as gym
from gymnasium import spaces

class YourEnv(gym.Env):
    metadata = {"render_modes": []}

    def __init__(self, config: dict | None = None):
        super().__init__()
        # 定义 action space 和 observation space
        self.action_space = spaces.Box(low=0, high=1, shape=(5,), dtype=np.float32)
        self.observation_space = spaces.Dict({
            "signal": spaces.Box(-np.inf, np.inf, shape=(64,), dtype=np.float32),
            "state": spaces.Box(-np.inf, np.inf, shape=(9,), dtype=np.float32),
        })

    def reset(self, *, seed=None, options=None):
        super().reset(seed=seed)
        # 重置环境，返回初始 observation 和 info
        obs = self._get_obs()
        info = {}
        return obs, info

    def step(self, action):
        # 执行 action，返回 (obs, reward, terminated, truncated, info)
        obs = self._get_obs()
        reward = self._compute_reward()
        terminated = self._check_done()
        truncated = False
        info = {}
        return obs, reward, terminated, truncated, info

    def close(self):
        pass
```

### Action Space 设计原则

| 类型 | 适用场景 | 示例 |
|------|---------|------|
| `Box(low, high, shape)` | 连续控制 | 频率、幅度、时间 |
| `Discrete(n)` | 离散选择 | 实验类型（6种） |
| `MultiDiscrete([n1, n2, ...])` | 多个离散维度 | 扫描模式 + 材料选择 |
| `Dict({"a": Box(...), "b": Discrete(...)})` | 混合 | 实验类型 + 连续参数 |

**推荐**：用 `Box([0, 1])` 归一化所有连续参数，在 `step()` 内部映射到物理单位。

### Observation Space 设计

```python
spaces.Dict({
    "signal": spaces.Box(..., shape=(n_points,)),  # 测量信号
    "state": spaces.Box(..., shape=(n_params,)),   # 当前参数估计
    "metadata": spaces.Box(..., shape=(k,)),       # 步数、置信度等
})
```

**关键**：observation 必须是 **Agent 可见的信息**，不能包含 ground truth 参数（那是 info 里的）。

### Reward 设计

| 策略 | 适用场景 | 示例 |
|------|---------|------|
| Sparse | 只在达成目标时给奖励 | 校准完成 +10，其他 0 |
| Dense | 每步给进度奖励 | 参数误差减少 +1，增加 -1 |
| Shaped | 引导探索方向 | 接近共振峰 +0.5，远离 -0.2 |

**推荐**：Sparse + 小的 step penalty（如 -0.02）防止无限循环。

## 注册到 Gymnasium

```python
# your_gym/__init__.py
from gymnasium.envs.registration import register

register(
    id="YourEnv-v0",
    entry_point="your_gym.env:YourEnv",
    max_episode_steps=50,
)
```

使用：
```python
import gymnasium as gym
import your_gym

env = gym.make("YourEnv-v0", config={"param": 123})
```

## Coding Agent 接口（可选但推荐）

为 Claude Code / GPT-4 提供 **HTTP API**，无需训练即可调用。

### FastAPI Server 模板

```python
# your_gym/server.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import gymnasium as gym
import your_gym

app = FastAPI()
sessions = {}  # {session_id: env}

class RunRequest(BaseModel):
    session_id: str
    experiment: str  # "T1" / "Ramsey" / ...
    params: dict     # {"delay_us": 50, "amplitude": 0.5}

@app.post("/session/create")
def create_session(session_id: str):
    env = gym.make("YourEnv-v0")
    obs, info = env.reset()
    sessions[session_id] = env
    return {"session_id": session_id, "initial_obs": obs}

@app.post("/session/run")
def run_experiment(req: RunRequest):
    env = sessions.get(req.session_id)
    if not env:
        raise HTTPException(404, "Session not found")

    # 将 experiment + params 映射到 action
    action = env.action_space.sample()  # 实际需要根据 req 构造
    obs, reward, terminated, truncated, info = env.step(action)

    return {
        "observation": obs,
        "reward": reward,
        "terminated": terminated,
        "info": info,
    }

@app.post("/session/submit")
def submit_calibration(session_id: str, params: dict):
    # Agent 提交最终参数估计，计算得分
    env = sessions.get(session_id)
    true_params = env.unwrapped.true_params  # 从 env 获取 ground truth
    score = compute_score(params, true_params)
    return {"score": score, "true_params": true_params}
```

启动：
```bash
uvicorn your_gym.server:app --port 8765
```

### CLAUDE.md 任务描述模板

```markdown
# Qubit Calibration Task

You are a physicist. A superconducting qubit is connected to this API:

**POST http://localhost:8765/session/create?session_id=test**
**POST http://localhost:8765/session/run**
```json
{
  "session_id": "test",
  "experiment": "T1",  // T1 | Ramsey | PowerRabi | ...
  "params": {"delay_us": 50, "amplitude": 0.5}
}
```
Returns: `{"observation": {"signal": [...], "x_axis": [...]}, "info": {...}}`

**POST http://localhost:8765/session/submit**
```json
{"session_id": "test", "params": {"f_q_GHz": 5.2, "T1_us": 45, ...}}
```

**Budget: 30 experiments.** Calibrate all 6 parameters within 5% error.

Hidden parameters: f_q, f_r, T1, T2*, amp_pi, t_pi.

Use physics intuition. Fit curves. Adapt strategy.
```

## 项目结构

```
your-gym/
  your_gym/
    __init__.py       # Gymnasium registration
    env.py            # YourEnv class
    simulator.py      # 物理模拟器
    server.py         # FastAPI (optional)
    logger.py         # EpisodeLogger (见下)
  examples/
    random_agent.py   # RL smoke test
  benchmark/          # Phase 1 输出
  pyproject.toml
  README.md
```

## EpisodeLogger — 自动记录每步

```python
# your_gym/logger.py
import json
from pathlib import Path
import matplotlib.pyplot as plt

class EpisodeLogger:
    def __init__(self, session_id: str, output_dir: Path):
        self.session_id = session_id
        self.output_dir = output_dir / session_id
        self.output_dir.mkdir(parents=True, exist_ok=True)
        self.log_file = self.output_dir / "log.json"
        self.step_count = 0

    def log_step(self, action, obs, reward, info):
        entry = {
            "step": self.step_count,
            "action": action,
            "observation": obs,
            "reward": reward,
            "info": info,
        }
        with open(self.log_file, "a") as f:
            f.write(json.dumps(entry) + "\n")

        # 保存图片
        fig, ax = plt.subplots()
        ax.plot(obs["x_axis"], obs["signal"])
        ax.set_title(f"Step {self.step_count}")
        fig.savefig(self.output_dir / f"step_{self.step_count:03d}.png")
        plt.close(fig)

        self.step_count += 1
```

**输出**：`runs/server_sessions/{sid}/log.json` + `step_NNN.png`

## 关键链接

- Gymnasium 官方文档：https://gymnasium.farama.org/
- FastAPI 文档：https://fastapi.tiangolo.com/
- 参考实现：
  - fib-gym: https://github.com/Osgood001/fib-gym
  - quantum-cal-gym: https://github.com/Osgood001/quantum-cal-gym

## 输出

- `your_gym/` Python package，可 `pip install -e .`
- `examples/random_agent.py` 验证 Gym 接口正常
- （可选）`your_gym/server.py` 支持 Coding Agent
