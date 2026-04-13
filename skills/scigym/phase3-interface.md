# Phase 3: Interface — 连接真实硬件/计算资源

## 目标

建立一层 **Interface**，让 Gym 的动作空间能映射到：
- 真实实验仪器（量子计算机、电子显微镜、合成机器人）
- 高性能计算集群（DFT、分子动力学）
- 外部 API 服务

## 架构模式

```
Gym action_space
      ▼
  Interface Layer       ← 唯一需要修改的地方
      ▼
  Real System
（hardware / HPC / API）
```

Interface 层的职责：
1. 将归一化 action `[0, 1]` 翻译成物理指令
2. 执行指令（调用 SDK / SSH / HTTP）
3. 读取结果，翻译回 observation 格式
4. 处理超时、错误、重试

## 三种 Interface 类型

### 类型 A：模拟 Mock（开发/测试用）

```python
# your_gym/interfaces/mock.py
class MockInterface:
    """用内置模拟器响应，无需真实硬件。开发阶段默认使用。"""

    def run_experiment(self, exp_type: str, params: dict) -> dict:
        # 调用 qubit_sim.py / taichi_sim 等
        result = self.simulator.run(exp_type, params)
        return {"signal": result.signal, "metadata": result.info}
```

### 类型 B：真实硬件 SDK

```python
# your_gym/interfaces/hardware.py
class HardwareInterface:
    """连接真实仪器。需要 SDK 访问权限。"""

    def __init__(self, host: str, port: int):
        # 根据具体 SDK 初始化连接
        # 例如 quark SDK、FIB 控制软件、NMR 仪器驱动等
        self.client = YourSDK(host=host, port=port)

    def run_experiment(self, exp_type: str, params: dict) -> dict:
        job = self.client.submit(exp_type=exp_type, **params)
        result = job.wait(timeout=300)  # 等待结果，超时 5 分钟
        return {"signal": result.data, "metadata": result.meta}
```

### 类型 C：HPC 计算接口

```python
# your_gym/interfaces/hpc.py
class HPCInterface:
    """通过 SSH 提交 SLURM/PBS 作业，读取计算结果。"""

    def __init__(self, cluster_host: str, user: str, work_dir: str):
        self.cluster_host = cluster_host
        self.user = user
        self.work_dir = work_dir

    def submit(self, input_files: dict, script: str) -> str:
        # 上传输入文件，提交作业，返回 job_id
        import subprocess
        job_id = subprocess.check_output([
            "ssh", f"{self.user}@{self.cluster_host}",
            f"cd {self.work_dir} && sbatch {script}"
        ]).decode().strip().split()[-1]
        return job_id

    def wait_and_read(self, job_id: str, timeout: int = 3600) -> dict:
        # 轮询作业状态，读取输出
        ...
```

## Interface 切换（config 控制）

```python
# your_gym/env.py
class YourEnv(gym.Env):
    def __init__(self, config=None):
        cfg = config or {}
        backend = cfg.get("backend", "mock")

        if backend == "mock":
            from .interfaces.mock import MockInterface
            self.interface = MockInterface(cfg)
        elif backend == "hardware":
            from .interfaces.hardware import HardwareInterface
            self.interface = HardwareInterface(cfg["host"], cfg["port"])
        elif backend == "hpc":
            from .interfaces.hpc import HPCInterface
            self.interface = HPCInterface(**cfg["hpc"])
```

调用方式：
```python
# 纯模拟（快速迭代）
env = gym.make("YourEnv-v0", config={"backend": "mock"})

# 真实硬件（生产验证）
env = gym.make("YourEnv-v0", config={"backend": "hardware", "host": "192.168.1.10", "port": 8000})

# HPC 计算（高精度验证）
env = gym.make("YourEnv-v0", config={"backend": "hpc", "hpc": {"cluster_host": "...", "user": "..."}})
```

## Fidelity Flags（多精度模拟器）

通过 flag 控制模拟精度，在速度和准确性之间取舍：

```python
config = {
    "fidelity": "low"   # fast: simplified equations, ~0.01s/step
                        # medium: default physics model, ~0.1s/step
                        # high: full simulation, ~10s/step
}
```

实现方式：
```python
def _step_physics(self, action):
    if self.fidelity == "low":
        return self.fast_approx(action)  # 解析近似
    elif self.fidelity == "medium":
        return self.physics_model(action)  # 标准物理模型
    else:
        return self.full_simulation(action)  # 完整数值模拟
```

## Interface 文档要求

在 `README.md` 的 Interface 章节注明：

```markdown
## Hardware Interface

| Backend | Status | Requirements | Notes |
|---------|--------|-------------|-------|
| mock | ✅ Always available | None | Uses built-in simulator |
| hardware | ⚠️ Lab access required | quark SDK, VPN | See docs/hardware-setup.md |
| hpc | ⚠️ Cluster access required | SSH key, SLURM | See docs/hpc-setup.md |
```

## 安全注意事项

- **硬件保护**：设置参数边界，防止 Agent 发送越界指令损坏设备
- **超时控制**：所有硬件调用必须有 timeout，避免实验卡死
- **日志记录**：所有真实操作记录到 `hardware_log.jsonl`，含时间戳和操作者
- **Mock 优先**：开发和测试阶段始终用 mock，只在验证阶段切换到真实硬件
