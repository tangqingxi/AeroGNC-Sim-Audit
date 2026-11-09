# GNC-Baseline — 阶段报告

| 项 | 值 |
|---|---|
| 阶段 | `GNC-Baseline`（含 `GNC-Source-Review`、`WTR-Integration`、`Scenario-Validation`） |
| 状态 | `ready-for-review`（未合并 main，未发布 v1.0.0） |
| DEV 分支 | `feat/gnc-baseline` @ `ade4b6938522a760eef61aad4b8f68c7c19bc6f3` |
| DEV main | `57bbe6893d83808e3b46aa79864701b32b31c775`（未变） |
| 参考 | PX4 `df-main` @ `2eb3366757…`；旧 MATLAB @ `43b6c7ae…`；WindTrace @ `bcdb6a8e…` |
| 测试 | 31/31 通过（全新 clone，38.83 s） |

## 1. 完成内容

### 1.1 控制链（`src/GNC/`）

- Guidance：`gnc_hover_hold_step`、`gnc_body_rate_step`。
- Navigation：`gnc_ideal_state_step`（`ideal=1`，真值派生测试适配器，非 EKF）。
- Control：`gnc_position_velocity_step`（位置 P + 速度 PID）、
  `gnc_attitude_step`（四元数 P）、`gnc_rate_step`（SI 力矩 PID + 抗饱和）、
  `gnc_allocate_pinv_step`（六舵面 PINV + 独立电机推力）、两个独立锁存组件。
- 场景：`gnc_hover_scenario`、`gnc_rate_step_scenario`（50–400 Hz 多速率）。

### 1.2 关键设计

- 全部经唯一 `aero_topic_bus`；控制器只读 `navigation/state`，不读取真实状态或真值效能。
- PX4 归一化增益未直接搬用；控制量全部 SI（m、m/s、rad、rad/s、N、N·m）。
- 分配使用名义 `B_tau`（由指令推力按涵洞名义模型计算 `Ve`），并保留配平舵角与饱和残差；
  电机推力与舵面力矩分开求解。
- `command` 计算值与 `held_command` 锁存值独立；舵面 200 Hz、电机 50 Hz 锁存。
- 详细接口、公式、速率与限制写入 DEV `DESIGN.md` §15。

## 2. WTR 集成（提前完成）

- `tools/wtr/wtr_export`：从完整 Topic 日志（要求 `dropped==0`）与 `result.state` 生成
  `windtrace.flight.v1`，先写 `simulation.mat`、后写 `flight.wtr`，拒绝覆盖。
- `tools/wtr/wtr_validate`：MATLAB 独立校验容器、成员 SHA256、微秒时间戳、字段宽度、
  四元数模长与六舵面几何。
- 已验证：
  - Python WindTrace `read_flight` 读取成功（14 topics、18 parameters、duration 正确）；
  - `python app.py --smoke` 输出 `execution=completed`；
  - `--screenshot` 离屏实际渲染成功（rate 与 hover 两张，见 `evidence/`）；
  - WTR 与 `.mat` 的 SHA256 已记录，重复导出被拒绝且原文件不变。

## 3. 场景结果（fixture 合成气动）

| 场景 | 结果 |
|---|---|
| BodyRate Step（roll 0.5 rad/s，0.5–1.5 s） | 0.8–1.4 s 平均 0.507 rad/s、std 0.008、max\|yaw\| 0.021 rad/s |
| HoverHold（初始偏差 [0.2,0,0.1] m） | 3 s 末位置误差 0.0085 m；2 s 后最大 0.0476 m；最大倾角 0.78° |

**物理验证状态：BLOCKED。** 使用 `tests/fixtures/shc09_aero_fixture.m` 的全零合成气动表，
只证明软件链路、数值稳定性与方向正确；不得表述为真实气动或实机闭环验证。真实气动数据
许可未闭合，闭环物理验收需外部授权数据后单独执行。

## 4. 测试

```text
matlab -batch "run('tests/run_tests.m')"  →  31/31 通过，38.83 s，退出码 0
```

- 原有 27 项 Foundation 测试全部保留并通过；
- 新增：`test_gnc_math`、`test_gnc_rate_step`、`test_gnc_hover_hold`、`test_wtr_export`；
- 覆盖四元数/推力/倾角/PINV 解析检查、角速度跟踪与命令锁存独立、悬停收敛与多速率、
  WTR 契约/覆盖保护/原始结果安全。

## 5. 未解决问题与风险

1. 真实气动许可与物理有效性未确认；场景物理验收 BLOCKED（非软件失败）。
2. 控制增益为仿真名义值；未做实机辨识或鲁棒性/蒙特卡洛评估。
3. 未实现 PX4 INDI、CA16、参考模型、增益调度；纯 PID 基线固定。
4. 导航为 IdealState；无传感器噪声/偏置/延迟不确定性，不能作为导航性能结论。
5. WTR 依赖有界日志，长时实验需增大 `max_records` 并检查 `dropped`。
6. Viewer 离屏截图可用，但未做交互式人工回放与视觉验收。

## 6. 相关提交

| 提交 | 内容 |
|---|---|
| `27a160c` | GNC 控制链、场景装配、路径登记 |
| `8e0c2a9` | WTR 导出、独立校验、几何 |
| `ade4b69` | 31 项测试、DESIGN §15、README |

## 7. 下一步

NEXUS 复核该分支；通过后再决定 fast-forward 合并 main。未创建 Tag、未合并、未启动后续阶段。
