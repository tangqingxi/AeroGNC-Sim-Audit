# GNC-Control-Modularization — 阶段报告

| 项 | 值 |
|---|---|
| 阶段 | `GNC-Control-Modularization` |
| 状态 | `ready-for-review`（未合并 main，未发布 v1.0.0） |
| DEV 分支 | `feat/gnc-baseline` @ `59cf917c5f87bfbf767bc9e9de9ead559bd4929b` |
| DEV main | `57bbe6893d83808e3b46aa79864701b32b31c775`（未变） |
| 上一阶段 | GNC-Baseline @ `ade4b69`（历史保留） |
| 测试 | 37/37 通过（全新 clone，39.24 s） |

## 1. 架构整改结果

- 四级控制器成为四个独立组件：`Position/position_p`、`Velocity/velocity_pid`、
  `Attitude/attitude_p`、`Rate/rate_pid`；各自目录、参数、状态、频率和单元测试。
- `Allocation/pinv_allocator` 独立可替换；Guidance/Navigation 与 Control 独立。
- 删除合并的 `gnc_position_velocity_step` 与全部 `gnc_` 前缀；旧函数无兼容包装。
- 每级只读写 `bus.control_state.<level>`；跨级数据只经 Topic。
- Position 与 Velocity 频率独立（默认均 100 Hz，可分别配置）。

### 1.1 各级接口/参数/状态/频率

| 级 | 输入 Topic | 输出 Topic | 主要参数 | 状态 | 默认频率 |
|---|---|---|---|---|---|
| Position | `navigation/state`、`guidance/reference` | `control/velocity_setpoint` | Kp_pos、速度限幅 | 无 | 100 Hz |
| Velocity | `control/velocity_setpoint`、`navigation/state` | `control/attitude_setpoint` | Kp/Ki/Kd、积分/加速度/倾角限幅 | 积分、时间 | 100 Hz |
| Attitude | `navigation/state`、`control/attitude_setpoint` | `control/rate_setpoint` | Kp_att、ω_ff、速率限幅 | 无 | 200 Hz |
| Rate | `control/rate_setpoint`、`navigation/state`、`control/allocation_status` | `control/torque_setpoint` | Kp/Ki/Kd/Kff、积分/力矩限幅 | 积分、时间 | 400 Hz |
| Allocation | `control/torque_setpoint`、`control/attitude_setpoint` | `control/allocation_status`、`bus.command` | B_M、trim、行程、电机模型 | 无 | 400 Hz |

## 2. 修复项

1. **速度 D 项**：`a_cmd = a_ff + Kp·e + I − Kd·a_est`，`a_est` 来自
   `navigation/state.acceleration`（IdealState 对测量速度做有限差分）；`test_velocity_pid`
   用非零 Kd 证明 D 项有效（倾角 6.99°→1.17°）。
2. **抗饱和**：`rate_pid` 只从 `control/allocation_status.saturated` 读取反馈，
   删除 `bus.gnc.allocation` 旁路；测试覆盖正饱和冻结与非饱和积分。
3. **姿态前馈量纲**：`omega_ff`（rad/s）与 `alpha_ff`（rad/s²）明确分离；
   当前 Attitude P 不使用角加速度前馈，输出 `alpha=0`，不再把 α 加到 ω。
4. **PINV 数学**：明确 τ_d 为相对配平增量，`δ=δ_trim+pinv(B_tau)τ_d`，
   `residual=τ_d−B_tau(δ−δ_trim)`，`saturated=sign(residual)`；零力矩回到配平，
   解析测试误差 <1e-12。
5. **默认等价**：在 Kd=0、相同增益/初值/气动 fixture 下，新旧悬停与速率指标
   打印到 6 位一致（见 `evidence/controller-comparison.txt`）。

## 3. WTR 修复

- `actuator_servos_pwm` 使用真实 `latch_servos` 事件时刻（200 Hz），新增
  `actuator_motors_pwm` 使用 `latch_motor` 事件时刻（50 Hz）；不使用状态时间轴、不插值。
- 场景签名覆盖场景/参考/方法/种子/机型版本/完整增益/求解器/tspan/气动来源与
  trust/SHA256/覆盖值/组件频率。
- `parameters.json` 记录机型、环境、气动来源与可信等级、执行器型号/动态、四级增益与
  限幅、PINV 配平/矩阵、频率、求解器与场景；未知值不填默认。
- `.wtr` 复用 `aero_results_path` / `aero_results_check`；临时 ZIP 在目标 Results
  目录同文件系统，容量检查后原子改名；拒绝覆盖，失败不删除权威 `.mat`。
- 成功读取：WindTrace `read_flight` 与 `--smoke` 均通过；离屏截图见 `evidence/`。

## 4. 测试

```text
全新 clone feat/gnc-baseline @ 59cf917
matlab -batch "run('tests/run_tests.m')"  →  37/37 通过，39.24 s，退出码 0
```

- 原 27 项 Foundation 测试全部保留；
- 新增：`test_position_p`、`test_velocity_pid`、`test_attitude_p`、`test_rate_pid`、
  `test_pinv_allocator`、`test_control_chain`、`test_control_multirate`；
- 场景测试更新为 `test_rate_step`、`test_hover_hold`；`test_wtr_export` 增加
  锁存时间轴、参数追溯、容量路径与覆盖保护断言；
- 无跳过、无失败、未降低阈值。

## 5. 闭环结果

| 场景 | 结果 |
|---|---|
| BodyRate Step（roll 0.5 rad/s） | 0.8–1.4 s 平均 0.507 rad/s、max\|yaw\| 0.021 rad/s |
| HoverHold（[0.2,0,0.1] m 初始偏差） | 3 s 末误差 8.5 mm；2 s 后最大 47.6 mm；最大倾角 0.78° |

物理验证仍 **BLOCKED**：仅使用全零合成气动 fixture，不得写成真实气动/实机闭环 PASS。

## 6. 未解决问题与风险

1. 真实气动许可/物理有效性未闭合（BLOCKED）。
2. 增益为名义值；无实机辨识、鲁棒性/蒙特卡洛评估。
3. Navigation 为 IdealState；加速度由测量差分，未知传感器噪声/偏置。
4. 未实现 INDI/MPC/CA16/参考模型/增益调度；WTR 固定 v1。
5. WTR 依赖有界日志（`dropped==0`）；长时实验需设置足够 `max_records`。

## 7. 提交

| 提交 | 内容 |
|---|---|
| `e0d3835` | 四级拆分、命名迁移、D 项/抗饱和/量纲修复、WTR 修复、37 项测试 |
| `59cf917` | DESIGN/README/AGENTS 文档 |

## 8. 下一步

NEXUS 复核该分支；通过后再决定合并。未合并 main、未创建 Tag、未启动后续阶段。
