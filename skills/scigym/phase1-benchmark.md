# Phase 1: Benchmark — 从论文/真机数据建立独立验证指标

## 目标

建立一套**不能被 hack 的指标**，用于验证 Gym 模拟器的准确性。

## 为什么需要 Benchmark？

| 没有 Benchmark | 有 Benchmark |
|---------------|-------------|
| Agent 学会 exploit reward bug | Reward 由独立物理模型验证 |
| 模拟器漂移无人知晓 | 定期对齐论文数据，量化 sim-to-real gap |
| 结果无法复现 | 每个 episode 记录 obs/action/reward，可审计 |

## 数据来源（三选一或组合）

### 1. 论文数据
从已发表论文中提取关键曲线/数值：
- **实验曲线**：Rabi 振荡、T1 衰减、溅射产额 vs 角度
- **理论预测**：Yamamura 公式、Lindblad 方程解析解
- **对比实验**：不同参数下的结果对比

**工具**：
- WebPlotDigitizer: https://automeris.io/WebPlotDigitizer/
- 手动标注 → CSV → `benchmark/paper_data/`

### 2. 真机数据
从真实硬件采集的校准数据：
- **实验日志**：Jupyter notebook、实验记录
- **仪器输出**：示波器波形、光谱仪数据
- **多次运行**：统计误差、重复性验证

**格式**：JSON Lines，每行一条实验记录
```json
{"exp_type": "T1", "params": {"delay_us": 50}, "signal": [0.98, 0.95, ...], "timestamp": "2026-04-10T14:23:01Z"}
```

### 3. 高精度计算
用更精确（但更慢）的方法验证快速模拟器：
- **DFT/DFPT** 验证 MLFF 能量预测
- **QuTiP 全密度矩阵** 验证简化 Lindblad 模型
- **Monte Carlo** 验证 BCA 近似

## Benchmark 文件结构

```
benchmark/
  paper_data/
    yamamura_1982_fig3.csv       # 论文曲线数字化
    rabi_oscillation_theory.py   # 解析解
  hardware_data/
    qubit_T1_2026-03-15.jsonl    # 真机校准数据
    fib_sputter_yield.jsonl
  validation/
    test_gym_vs_paper.py         # 自动化对比脚本
    test_gym_vs_hardware.py
  README.md                       # 数据来源、采集方法、预期误差
```

## 验证脚本示例

```python
# benchmark/validation/test_gym_vs_paper.py
import gymnasium as gym
import pandas as pd
import numpy as np

# 加载论文数据
paper = pd.read_csv("../paper_data/yamamura_1982_fig3.csv")

# 运行 Gym
env = gym.make("YourGym-v0")
obs, _ = env.reset()
results = []
for angle in paper["angle_deg"]:
    action = env.action_space.sample()  # 根据 angle 构造 action
    obs, reward, _, _, info = env.step(action)
    results.append(info["sputter_yield"])

# 对比
mse = np.mean((paper["yield"] - results) ** 2)
print(f"MSE vs paper: {mse:.4f}")
assert mse < 0.05, "Gym 偏离论文数据过大"
```

## 关键原则

1. **独立性**：Benchmark 数据不能来自 Gym 本身
2. **可审计**：数据来源、处理方法写入 `benchmark/README.md`
3. **定期验证**：每次修改 Gym 后重跑 `validation/` 脚本
4. **量化误差**：记录 sim-to-real gap，不追求完美对齐

## 输出

- `benchmark/` 目录，包含数据 + 验证脚本
- `benchmark/README.md` 说明数据来源和预期误差范围
- CI 集成：每次 commit 自动跑验证脚本
