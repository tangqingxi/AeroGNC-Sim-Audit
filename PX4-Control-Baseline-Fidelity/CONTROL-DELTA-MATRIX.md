# PX4 控制基线差异矩阵

图例：**PX4** = 指定 `df-main` 提交的纯 PID 路径；**旧** = DEV `59cf917`；
**新** = DEV `61507b4`。所有单位均为 SI，除非显式注明。

## 1. Position

| 项 | PX4 | 旧 AeroGNC | 新 AeroGNC | 原因/说明 |
|---|---|---|---|---|
| 位置误差 | `Kp_pos·(p_sp−p)` | 同 | 同 | 无变化 |
| 速度前馈 | 加到 `_vel_sp` | 同 | 同 | 无变化 |
| 水平限幅 | `constrainXY(v_pos, v_ff, max)`，位置方向优先 | 合成后统一缩放 | `constrain_xy.m` 同 PX4 | 避免前馈把位置误差方向拉偏 |
| 垂直限幅 | `−vel_up..vel_down` | `velocity.limit_*` | `position.velocity_limit_*` | 参数归 Position，四级独立 |
| 输出 | `local_position_setpoint` | 速度 Topic | 增加 `yaw/yaw_rate` | 航向参考向下游传递 |

## 2. Velocity

| 项 | PX4 | 旧 AeroGNC | 新 AeroGNC | 原因/说明 |
|---|---|---|---|---|
| PID | `Kp e + I − Kd a_dot` | 同，但 Kd 注释单位 `s` | 同，Kd 单位 `1` | 输出/输入均 m/s²，Kd 无量纲 |
| D 反馈 | 测量加速度 | `navigation/state.acceleration` | 同 | 不读真值 |
| 垂直积分 | PID 前钳位 ±g | 仅总积分限幅 | 前/后均钳位 ±g | PX4 垂直约束 |
| 积分抗饱和 | 垂直饱和条件 + 水平 Tracking AW | 加速度上限冻结 | 同 PX4 | 不再使用加速度限幅替代 |
| 水平 Tracking AW | `2/Kp_x·(a_des−a_produced)` | 无 | 有 | 推力矢量饱和时卸载 |
| 推力上限/下限 | `MPC_THR_MIN/MAX` 归一化 | 仅最大倾角 | 物理 `T_min/T_max` | 适配 SHC09 SI 推力 |
| 水平推力余量 | `MPC_THR_XY_MARG` 归一化 | 无 | `0.3·T_max` 物理余量 | 适配 SHC09；参数可配置 |
| 倾角 | `MPC_TILTMAX_AIR` | `max_tilt_rad` | 同 | 无变化 |
| 加速度上限 | 无 | `acceleration_limit` 冻结 | 移除 | PX4 用推力/倾角约束，不另设加速度限幅 |

## 3. Attitude

| 项 | PX4 | 旧 AeroGNC | 新 AeroGNC | 原因/说明 |
|---|---|---|---|---|
| 误差 | 默认 reduced attitude + yaw weight | 全四元数 `2·vec(q⁻¹qd)` | `quat_error_default.m` | 采用六舵 airframe 默认模式 0 |
| yaw weight | `MC_YAW_WEIGHT` 默认 0.4；yaw 增益 `/yaw_w` | 无 | 有 | 与 PX4 分解/补偿一致 |
| 参考模型 | 二阶临界阻尼 ZOH，`MC_REF_W_N=50`，`MC_REF_FF=0` | 无 | 有，状态 `control_state.attitude` | 属于默认路径 |
| yaw_rate | `yaw_sp_move_rate`，世界系 → 参考机体系 | 无/丢弃 | 经 Position/Velocity 传入，参考模型使用 | 动态偏航前馈 |
| 角速度限制 | `MC_*RATE_MAX` | `attitude.rate_limit` | 同 | 无变化 |
| 角加速度 | 不直接加 | `alpha=0` | `alpha=0` | 保持正确量纲 |

## 4. Rate

| 项 | PX4 | 旧 AeroGNC | 新 AeroGNC | 原因/说明 |
|---|---|---|---|---|
| PID | `Kp e + I + FF ω − Kd α` | 同 | 同 | 无变化 |
| 本地力矩裁剪 | **无** | `rate.torque_limit` | **移除** | 执行器约束归 PINV；不复制非 PX4 机制 |
| 积分 | 一阶欧拉；条件积分；`i_factor` | 条件积分；无 i_factor | 加 `i_factor` | PX4 400°/s 衰减 |
| 积分限幅 | `_lim_int` | 同 | 同 | SI N·m |
| 饱和反馈 | `unallocated_torque` 符号 | `allocation_status.saturated` | 同 | 唯一 Topic |

## 5. Allocation / PINV

| 项 | PX4 | 旧 AeroGNC | 新 AeroGNC | 原因/说明 |
|---|---|---|---|---|
| 接口 | 归一化 ν / B_u | SI τ / B_tau | SI τ / B_tau | 归一化研究决定暂缓 |
| 伪逆 | `geninv(B)` | `pinv(B_tau)` | 同 | Moore–Penrose |
| 配平 | actuator_trim + control_trim | `δ_trim` | 同 | 相对配平力矩 |
| 限幅 | 执行器 min/max | 生效行程 | 同 | 六舵 |
| 残差 | `unallocated = sp−allocated` | `τ_d−B_tau(δ−δ_trim)` | 同 | N·m |
| 电机 | matrix 0 独立 | `Ω=sqrt(T/k_T0)` | 同 | 不七维耦合 |

## 6. WTR / 实验身份

| 项 | 旧 | 新 |
|---|---|---|
| 签名 | 场景/增益/求解器等部分字段 | validated cfg：environment、time_tolerance、solver.InitialStep、组件 kind/period/priority/offset/delay/latches/topic、x0/initial_command |
| seed/method | manifest 独立 | 导出强制与签名一致 |
| 参数 | 缺 guidance_hz、ve_min、saturation_tolerance、InitialStep、部分涵道/阻尼 | 补齐实际影响计算的参数 |
| 电机时间轴 | 只比数量 | 逐项与 `latch_motor` 事件时间比，误差 <1e-9 s |

## 7. 保留的 SHC09 必要适配

1. `B_tau` 使用物理 `N·m/rad`，不使用 PX4 的 `CA_SV_CSx_TRQ_*` 归一化 +1 力矩。
2. 推力上下限/水平余量用 N，不用 PX4 的归一化 hover-thrust。
3. `T_min = 0.12·T_max`、`margin = 0.3·T_max` 是把 PX4 的归一化参数映射到
   SHC09 物理推力；不是 PX4 原始数值。
4. 六舵同行程 `±40°`；配平偏航力矩由 `δ_trim` 承担。
5. 参考模型参数取六舵涵道 airframe 默认（mode 0 / yaw 0.4 / 50 / 0），
   不采用 22018 四舵 SIH 的 mode 3 覆盖。

## 8. 未纳入

- INDI、PX4 归一化 ν/S、可选姿态模式 1–7、WLS/PCA/CA16、电池缩放、
  hover-thrust 估计。
- 这些不影响本轮纯 PID 基线；也不应为了“看起来像 PX4”而引入。
