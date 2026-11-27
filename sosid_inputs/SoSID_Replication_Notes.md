# SoSID 野火航空灭火体系复现校核笔记

本文档梳理上游仓库 `ibrahimoguzc/wildfire-rl-new`（基于 DLR / AIAA SoSID X-Challenge 工具包扩展的强化学习与机队选型版本）的数据口径、物理/智能体模型结构、RL 环境接口及使命效能指标（MoE），并与 `sosid_inputs/` 下的结构化表格一一对应。

---

## 1. 结构化输入文件及源码来源

| 文件 | 一行的粒度 | 上游源码定位 |
|---|---|---|
| [scenarios.csv](sosid_inputs/scenarios.csv) | 一个野火扑救任务场景，共 3 行（`salamis`、`pyrenees`、`palisades`） | `examples/wildfire/data/scenarios/inputs/*.json` 及 `examples/wildfire/MoE.py` 第 40–42 行基准常数 |
| [aircraft_catalog.csv](sosid_inputs/aircraft_catalog.csv) | 一种航空器构型（MoP 参数），共 22 行（4 型参考机型 + 18 型 `Config_P-*_R-*_S-*` 逆向设计候选构型） | `examples/wildfire/data/aircraft/*.json` 与根目录 `planes/*.json` |
| [tactics.csv](sosid_inputs/tactics.csv) | 一种离散灭火战术组合，共 24 行（主键 `tactic_id` 为 `0..23`） | `examples/wildfire/ppo_runner.py` 第 83–106 行 `TACTIC_COMBINATIONS` |
| [config.json](sosid_inputs/config.json) | 全局统一配置（场景列表、RL 状态/动作维度、两套 MoE 评价公式参数） | `examples/wildfire/ppo_runner.py` 第 56–81 行、`examples/wildfire/MoE.py` 第 28–75 行 |

---

## 2. 三个标准野火场景参数对比

全部场景栅格分辨率均为 `cell_size = 5 m`，运行模式默认无头模式（`"run_headless": true`）。

| 场景 | 地形命名空间 | 起始时刻 | 最大仿真时长 | 响应延迟 | 主风向 / 风程 | 水源数 / 机场数 |
|---|---|---|---:|---:|---|---:|
| **Salamis** | `Salamis_2004x1996_5m` | `2023-07-17T15:39:00` | 43,200 s（12 h） | 7,200 s | 230° / 305 km | 17 / 2 |
| **Pyrenees** | `Pyrenees_2000x1996_5m` | `2023-08-17T06:15:00` | 43,200 s（12 h） | 1,800 s | 300° / 350 km | 0 / 4 |
| **Palisades** | `Palisades_2004x1996_5m` | `2025-01-07T10:30:00` | 86,400 s（24 h） | 3,600 s | 15° / 320 km | 1 / 2 |

---

## 3. 候选机型目录（体系级选型 / 逆向设计空间）

18 个正交设计构型 `Config_P-{Low,Med,High}_R-{Low,Med,High}_S-{Low,Med,High}` 对应飞机级性能指标（MoP）的三维组合：
- **载水量 `P`（Payload）**：`Low = 2,000 kg`、`Med = 4,000 kg`、`High = 6,200 kg`（投放流量统一为 `510.25 kg/s`，均可水面汲水 `can_scoop = true`）。
- **航程/燃油量 `R`（Range / Propellant）**：随载水与航程等级由约 `965 kg` 至 `8,991 kg` 变化。
- **巡航速度 `S`（Speed）**：`Low = 66.88 m/s`、`Med = 74.59 m/s`、`High = 90.03 m/s`（巡航高度均为 `3,000 m`）。
- 另有 4 型异构参考平台：`dhc_515`（常规起降汲水机，载水 `6,200 kg`）、`example_aircraft_1`（纯电垂直起降 `vertipad`，载水 `360 kg`，能量以 `kJ` 计）、`example_seaplane_1`（水上起降，载水 `1,300 kg`）、`example_seaplane_hybrid_1`（混动水上飞机，载水 `2,205.4 kg`）。

机队预算枚举逻辑见 `upstream/examples/wildfire/fleet_selection/fleet_selection.ipynb`：给定单机采购单价与总采购预算上限（如 `100,000,000 USD`），枚举满足预算约束的候选机队组合。

---

## 4. 强化学习战术调度接口（`WildfireHourlyEnv`）

位于 `upstream/examples/wildfire/ppo_runner.py`（稀疏奖励变体见 `ppo_runner_sparse_rewards.py`）：

1. **决策步长**：默认 `DEFAULT_DECISION_INTERVAL_MINUTES = 10`（每步推进 600 秒仿真时间；24 小时 `Palisades` 场景最多 144 步，火场完全扑灭则提前终止）。
2. **观测空间（`Box(30,)`）**：
   - **12 维全局环境与火场态势**：气温 `temperature_c`、相对湿度 `humidity_pct`、风速 `wind_speed_ms`、风向 `wind_direction_deg`、距日落分钟数 `time_to_sunset_min`、火线距隔离带距离 `distance_to_fire_line_m`、火场质心距最近水源距离 `distance_to_water_m`、活跃火线栅格数 `active_firefront_count`、过火面积占比 `burnt_area_fraction`、火场质心坐标 `(fire_center_x, fire_center_y)`、最快蔓延方向角 `spread_angle_deg`。
   - **3 架受控航空器 × 6 维个体状态（共 18 维）**：归一化平面坐标 `(norm_x, norm_y)`、高度 `altitude_m`、是否满载灭火剂 `has_payload`、剩余推进剂比例 `propellant_fraction`、返航最近机场剩余油量安全裕度 `return_margin`（截断至 `[-1, 1]`）。
3. **动作空间（`MultiDiscrete([24, 24, 24])`）**：
   每架受控飞机在决策点独立选择一个离散战术编号 `0..23`，由三段子战术笛卡尔积构成：
   - 目标点选择 `SelectPOIType`（4 种）：`water`、`vegetation`、`topography`、`indirect`
   - 航迹跟踪 `TrackPOIType`（3 种）：`direct`、`indirect`、`follow_firefront`
   - 投水压制 `SuppressType`（2 种）：`direct`、`indirect`

---

## 5. 使命效能指标（MoE）两套口径与量纲差异核对

上游代码中存在**两套不同用途的 MoE 公式**，后续复现实验时需明确区分：

1. **任务级五项综合 MoE（`MoE.py`，用于机队逆向设计与全周期评估）**：
   $$\mathrm{MoE}_{\text{mission}} = 1 - \left(0.2\frac{A_{\text{burnt}}}{A_{\text{ref}}} + 0.2\frac{C_{\text{fire}}}{C_{\text{ref}}} + 0.2\frac{E_{\text{fire}}}{E_{\text{ref}}} + 0.3\frac{C_{\text{acq}}}{C_{\text{acq,ref}}} + 0.1\frac{C_{\text{ops}}}{C_{\text{ops,ref}}}\right)$$
   其中包含机队采购成本 $C_{\text{acq}}$（权重 `0.3`）与运行燃油/工时成本 $C_{\text{ops}}$（权重 `0.1`）。
2. **RL 步进三项奖励（`ppo_runner.py`，固定机队下的在线战术优化）**：
   $$r_t = 1 - 0.33\left(\frac{\Delta A_{\text{burnt}}}{90{,}870{,}000} + \frac{\Delta C_{\text{fire}}}{191{,}106{,}000{,}000} + \frac{\Delta E_{\text{fire}}}{131{,}224}\right)$$
   - **量纲换算差异提示**：在 `Palisades` 场景下，`MoE.py` 的基准常数为 `(9087, 1911058, 13122)`（分别以公顷/千欧元级简化单位计），而 `ppo_runner.py` 直接读取底层仿真的国际单位增量（平方米 $\text{m}^2$、欧元 EUR、吨 tonnes），因此归一化分母分别放大了 $10^4$、$10^5$、$10^1$ 倍。
