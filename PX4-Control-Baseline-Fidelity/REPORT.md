# PX4-Control-Baseline-Fidelity — 阶段报告

| 项 | 值 |
|---|---|
| 阶段 | `PX4-Control-Baseline-Fidelity` + `README-Project-Map` |
| 状态 | `in-progress (ready-for-review)`（未合并 main，未创建 Tag） |
| DEV 分支 | `feat/px4-control-fidelity` @ `1765ac1cf57a842e75eaaf2d81cbd8252d1687d9` |
| 分支基点 | `feat/gnc-baseline` @ `59cf917c5f87bfbf767bc9e9de9ead559bd4929b` |
| DEV main | `57bbe6893d83808e3b46aa79864701b32b31c775` |
| PX4 参考 | `df-main` @ `2eb33667575619b02a2d9b52032df2fac7fb5291` |
| AUDIT 分支 | `study/px4-control-baseline-fidelity` |
| 测试 | 39/39 通过（全新 clone，42.64 s） |
| 物理验证 | **BLOCKED**（合成气动 fixture） |

## 1. 本阶段完成内容

### 1.1 Position

- 速度限幅参数从 Velocity 移到 Position：
  `position.velocity_limit_xy/up/down`。
- 水平限幅改为 PX4 `ControlMath::constrainXY` 语义：位置反馈方向优先，
  前馈只使用剩余速度能力；垂直保持 `−up..down`。
- Guidance 的 `yaw/yaw_rate` 写入 `control/velocity_setpoint`。

### 1.2 Velocity

- `Kd` 单位修正为 `1`（无量纲），因为输出与测量加速度均为 m/s²。
- 默认 `MPC_ACC_DECOUPLE=1`：`z_b=[−a_x,−a_y,g]/‖·‖`，垂直加速度不改变倾角。
- PX4 hover-thrust 的 SI 映射：`T=T_hover·(1−a_z/g)/cos`；
  `T_hover=trim.thrust_n`，不直接复制 PX4 `MPC_THR_HOVER`。
- 垂直积分 PID 前钳位 `±g`；按垂直推力上下限做方向抗饱和。
- 水平余量 `min(实际水平需求, T_margin)`，垂直优先，再限制水平。
- 最终受限力矢量同时决定 q 与 `thrust_n`；Tracking AW 用最终受限推力。
- 移除旧的 `acceleration_limit` 冻结，避免非 PX4 的双重限幅。

### 1.3 Attitude

- 默认 PX4 reduced-attitude 误差 + yaw weight（默认 0.4）。
- yaw 增益按 `1/yaw_weight` 补偿，和 PX4 `setProportionalGain` 一致。
- 实现 PX4 默认二阶临界阻尼参考模型（`MC_REF_W_N=50`、`MC_REF_FF=0`、
  `MC_REF_FF_MAX=100°/s`），状态归 Attitude。
- `yaw_rate` 经参考机体系投影后进入角速度参考，不再静默丢弃。
- 角速度逐轴限幅；`alpha=0`，不把角加速度混入角速度。

### 1.4 Rate

- 删除 `rate.torque_limit`；Rate PID 不再做本地力矩裁剪。
- 增加 PX4 `i_factor=max(0,1−(e/(400°/s))²)`，在饱和方向钳位之后使用。
- 保留一阶积分、积分限幅、条件积分和 `control/allocation_status.saturated`
  反馈；不新增其它积分算法。

### 1.5 Allocation

- 保持 SI `B_tau [N·m/rad]`、`δ_trim`、PINV、六舵限幅、相对配平残差、
  每轴饱和。
- 电机 `Ω=sqrt(T/k_T0)` 与六舵面独立；无七维耦合。
- 未引入归一化 `nu/S/tau_ref` 接口（沿用研究结论）。

### 1.6 WTR

- 场景先 `aero_validate_config`，再用实际 cfg 生成签名；组件包含
  `kind/period/priority/offset/delay/latches/topic`，签名含
  `environment/time_tolerance/solver.InitialStep/x0/initial_command`。
- `wtr_export` 强制 manifest 的 seed/method/scenario 与签名一致。
- 参数增加 `rates.guidance_hz`、`allocation.ve_min_m_s`、
  `control.rate.saturation_tolerance`、`solver.InitialStep`、`duct.k_cpx`、
  固定片/阻尼/扩散器、装配频率和四级实际限幅。
- 电机锁存时间轴逐项比较，误差 `<1e-9 s`（与舵面一致）。

### 1.7 README

- 重写为项目地图：架构图、实际源码树、功能—源码索引、模块职责、运行方法、
  结果/WTR、能力/限制、文档入口。
- 不含开发日志、提交记录、Agent 过程或阶段验收结论。
- README 路径检查 28/28 通过（`evidence/readme-tree-check.txt`）。

## 2. 关键数学/单位映射

| 内容 | PX4 | AeroGNC-Sim |
|---|---|---|
| 姿态误差 | `2·canonical(q⁻¹qd).imag` (default reduced) | 同名 MATLAB 实现，标量在前 Hamilton |
| 角速度参考 | `Kp·e + yaw_rate + ref_ff` | 同，rad/s；`alpha=0` |
| Rate PID | `Kp e + I + FF ω − Kd α` | 同，N·m |
| Rate 积分 | `i_factor·Ki e dt` | 同；400°硬编码 |
| 垂直积分 | `±g` | 同 |
| 水平 ARW | `2/Kp_x`，produced=F_final,xy/m | 同；最终受限力 |
| 推力 | PX4 normalized hover thrust | `T_hover(1−a_z/g)/cos`，N |
| 姿态方向 | decouple=1：`z=[−a_x,−a_y,g]` | 同，不含 a_z |
| 分配 | `u=trim+M_n(sp−control_trim)` | `δ=δ_trim+pinv(B_tau)τ_rel`，rad |
| 电机 | 独立 matrix | `Ω=sqrt(T/k_T0)`，独立 |

## 3. 关闭的历史审查问题

| 问题 | 结果 |
|---|---|
| Velocity Kd 单位 | 关闭：代码/默认/WTR/DESIGN/测试统一为 `1` |
| Position 参数归属 | 关闭：`position.velocity_limit_*`；Velocity 不再持有 |
| Guidance yaw/yaw_rate 传递 | 关闭：Position→Velocity→Attitude；动态测试 |
| Rate 本地力矩裁剪 | 关闭：删除 `torque_limit`；测试断言输出可超过旧限幅 |
| Rate i_factor | 关闭：实现 + 单元测试 |
| WTR 签名 | 关闭：validated cfg + 全组件字段 + 初态/命令 |
| WTR 参数 | 关闭：补齐 guidance/ve_min/saturation/InitialStep/涵道/阻尼/扩散器 |
| 电机锁存时间轴 | 关闭：逐项 <1e-9 s |
| Velocity 解耦/hover thrust | 关闭：`z_b=[−ax,−ay,g]`；`T=T_hover(1−az/g)/cos`；独立测试 |
| 水平余量语义 | 关闭：`min(实际需求,margin)`，垂直优先；独立测试 |
| 力/姿态一致性 | 关闭：最终受限矢量同时决定 q/T；重建误差 `4.4e-16 N` |
| README 架构图 | 关闭：Guidance/Navigation 并列，Simulation 为运行环境；检查通过 |
| SHC09 参数来源 | 关闭：22005 覆盖/全局默认/最终生效表；Tmax 差异显式记录 |
| WTR 输入一致 | 关闭：gains/cfg/model/aero 比较，失败前不建目录；测试通过 |

## 4. 测试

```text
全新 clone feat/px4-control-fidelity @ 1765ac1
matlab -batch "run('tests/run_tests.m')"
→ 39/39 通过，42.64 s，退出码 0
```

- 原有 38 项能力全部保留：内核、SHC09、通信、WTR、场景、四级控制器。
- 新增 `test_velocity_px4`（本次）及上一轮的 `test_guidance_yaw`。
- 测试清单与磁盘一致（`test_project_layout` 检查 39 个文件）。
- 无跳过、无失败、未降低历史阈值。

## 5. 闭环场景对比

同一 fixture、默认增益、tspan、初始偏差：

| 指标 | 旧 `59cf917` | `61507b4` | 新 `1765ac1` |
|---|---|---|---|
| 悬停末误差 | 0.047605 m | 0.043276 m | 0.042548 m |
| 悬停最大误差 | 0.223607 m | 0.223607 m | 0.223607 m |
| 悬停最大倾角 | 0.0181 rad | 0.0185 rad | 0.0188 rad |
| Rate 均值（0.8–1.4 s） | 0.505326 rad/s | 0.505292 rad/s | 0.505292 rad/s |
| Rate max\|yaw\| | 0.070264 rad/s | 0.070263 rad/s | 0.070263 rad/s |

差异来源：PX4 位置限幅优先级、Velocity 推力/抗饱和、Attitude 默认误差与
参考模型；Rate i_factor 在 0.5 rad/s 时约 0.995，故速率阶跃变化很小。
未做隐藏调参，未降低测试阈值。证据：`evidence/scenario-comparison.txt`。

## 6. WTR / WindTrace

- 导出 1.5 s rate_step 与 3 s hover_hold：MATLAB `wtr_validate` 通过，
  15 topics、93/94 parameters。
- WindTrace `read_flight` 成功；`--smoke` execution=`completed`；
  离屏截图成功。证据：`evidence/wtr-validation.json`、
  `evidence/wtr-windtrace-screenshot.png`。
- 未做人工交互式回放验收；WTR 仍为派生数据，`simulation.mat` 为权威。

### 6.1 本轮新增证据

| 文件 | 内容 |
|---|---|
| `evidence/velocity-px4-reference.txt` | PX4 解耦/hover-thrust/余量/力一致性公式与数值 |
| `evidence/velocity-fidelity-tests.txt` | `test_velocity_pid`、`test_velocity_px4` 输出 |
| `evidence/velocity-force-consistency.txt` | q/T 重建力与独立期望误差 `4.4e-16 N` |
| `evidence/airframe-parameter-provenance.txt` | 22005 覆盖/全局默认/生效值及 Tmax 差异 |
| `evidence/wtr-mismatch-rejection.txt` | gains/tspan/x0 拒绝与无残留目录 |
| `evidence/readme-architecture-check.txt` | 并列架构与 28/28 路径检查 |
| `evidence/regression-test.txt` | 全新 clone 39/39 |

## 7. 未实施/不适用

- INDI、MPC、DCA、CA16、WLS、EKF2/ECL、传感器噪声模型。
- PX4 归一化 `nu/S` 接口、可选姿态误差模式 1–7、电池缩放、hover-thrust 估计。
- 真实气动/实机验证；物理验证 **BLOCKED**。

## 8. 与 PX4 原始实现的差异

1. **物理量接口**：Rate 输出 N·m，Allocation 用 `B_tau`，不是 PX4 的 ν/S。
   - 原因：归一化研究结论 + 保持 SI 可解释性；不改变控制数学。
2. **推力限幅**：`T_min=0.12T_max`、`T_margin=0.3T_max` 是 PX4 归一化参数的
   物理映射，不是 PX4 原始数值；水平余量按 `min(实际需求,margin)` 使用。
3. **Hover thrust**：控制模型使用 `T_hover=trim.thrust_n` 与
   `T_max=model 41.70816 N`，PX4 22005 的 `MPC_THR_HOVER=0.4865`、
   `CA_ROTOR0_CT=42.3016 N` 只作为来源对照（差异见 provenance 证据）。
4. **姿态方向**：默认 `MPC_ACC_DECOUPLE=1`，`z_b=[−a_x,−a_y,g]`；
   垂直加速度不改变倾角。
5. **参考模型**：采用 SHC09 默认 `mode 0/YAW_WEIGHT 0.4/50/0`；
   不采用 22018 四舵 SIH 的 `mode 3` 覆盖。
6. **无 INDI 切换**：只保留纯 PID 路径。
7. **无弱轴整行置零**：SHC09 六舵 B_tau 行满秩，不需要 PX4 的弱轴特例。

## 9. 提交与完整分支审查

本阶段提交（NEXUS 拒绝后的整改续提交）：

| 提交 | 内容 |
|---|---|
| `57bb4e2` | Velocity PX4 SHC09 解耦/hover thrust/余量/力一致性 + 新测试 |
| `f654370` | WTR 实验输入一致性拒绝 + 参考参数快照 |
| `1765ac1` | README Guidance/Navigation 并列架构图修正 |

前序（上一轮）提交：`935f60d` 四级控制器/WTR/测试；`61507b4` README/DESIGN。

**NEXUS 必须审查 `main...feat/px4-control-fidelity` 的完整变更**，不能只看这两个
提交：该分支继承 GNC-Baseline 与 GNC-Control-Modularization 的未合并成果。
本报告的“本阶段”只描述相对 `59cf917` 的增量。

## 10. 限制

- 合成气动 fixture；不是真实气动/实机闭环。
- 未运行 PX4 编译/SITL 并排数值对照；PX4 侧结论来自源码阅读和独立数学推导。
- `IdealState` 无噪声/偏置；不能作为传感器/EKF 验证。
- 参考模型在 200 Hz 下的闭环行为只由 SHC09 场景回归验证，未做频域鲁棒性分析。
- 参数为名义值，未实机辨识。

## 11. 结论

本轮完成了指定 PX4 `df-main` 纯 PID 基线的源码级对齐：Position 水平优先级、
Velocity 积分/抗饱和/推力约束、Attitude 默认误差/参考模型、Rate i_factor/无本地
力矩裁剪、PINV SI 配平与 WTR 完整性。四级架构、唯一 Topic Bus、SI 接口、
电机/舵面独立均保留。README 已可作为项目地图使用。

可以提交 NEXUS 复核；不建议在复核前合并 main 或发布 v1.0.0。
