# 实现决策记录

## 1. 总体原则

- 保留 PX4 成熟 PID 数学、积分、条件积分、i_factor、Tracking Anti-Windup、
  参考模型和限幅顺序。
- 保留 AeroGNC-Sim 四级独立目录、独立参数/状态/频率和唯一 Topic Bus。
- 所有物理量使用 SI；不引入 PX4 归一化虚拟控制接口。
- 不为“形式一致”增加 PX4 可选算法；不把 INDI 带进 V1.0。

## 2. Position

1. **参数归 Position**：`velocity_limit_xy/up/down` 从 `velocity` 结构搬到
   `position`。Velocity PID 只保留自身增益、积分和输出约束。
2. **PX4 水平优先**：`constrain_xy.m` 直接实现 `ControlMath::constrainXY`：
   - 位置反馈方向优先；
   - 位置分量已超限时只保留该方向；
   - 否则用二次方程求前馈可用的剩余速度；
   - 垂直仍分别用 `−limit_up..limit_down`。
3. **yaw/yaw_rate 透传**：Position 把 Guidance 的 yaw/yaw_rate 写入
   `control/velocity_setpoint`；Velocity 继续写入
   `control/attitude_setpoint`；Attitude 使用 yaw_rate。不再用静态
   `params.yaw` 代替制导航向。
4. **无状态**：Position 仍不保存积分/滤波状态。

## 3. Velocity

1. **Kd 单位**：`a_cmd` 与 `a_est` 都是 m/s²，故
   `Kd = ∂a_cmd/∂a_est` 无量纲；文档、WTR、测试统一为 `1`。
2. **PID 顺序**：使用上一拍积分计算 `a_cmd`，随后更新积分；与 PX4
   `RateControl`/`PositionControl` 的顺序一致。不能把“本拍新积分”提前加进
   本拍输出，否则会改变相位。
3. **垂直积分**：`I_z` 在 PID 前钳位到 `±g`；积分更新后再钳位一次。
4. **默认解耦姿态**：`MPC_ACC_DECOUPLE` 在 `22005_SHC09` 未覆盖，全局默认 1。
   因此 `z_b=[−a_x,−a_y,g]/‖·‖`，**不含 a_z**；垂直加速度只改变推力大小。
5. **Hover thrust 的 SI 映射**：
   - PX4 归一化：`T_z,norm=a_z·(hover/g)−hover`，`T_norm=T_z,norm/cos`；
   - SI：`T = T_hover·(1−a_z/g)/cos`；
   - `T_hover=model.trim.thrust_n=20.554800 N`（不是直接假定 mg 后另取）；
   - `T_max=model 41.708160 N`，`hover_norm=0.492824`；
   - PX4 `22005` 的 `MPC_THR_HOVER=0.4865`、`CA_ROTOR0_CT=42.3016 N`
     作为来源对照，不直接复制到 AeroGNC 物理接口。
6. **物理推力约束与顺序**：
   1. 初始 `z_b`、最大倾角；
   2. `T_z`/`T` 垂直推力；
   3. 垂直积分抗饱和（用垂直推力上下限与误差方向）；
   4. `reserve=min(‖T_xy‖, margin)`，`T_z,max=sqrt(T_max²−reserve²)`；
   5. 水平剩余 `sqrt(T_max²−T_z²)`；
   6. 最终受限推力矢量；
   7. 从最终矢量导出 `z_b`、q 与 `T`；
   8. 最终倾角复核；若需重限，力和 q 同步重算；
   9. Tracking AW 使用最终受限推力。
7. **水平 Tracking AW**：`arwGain=2/Kp_x`；`a_produced=F_final,xy/m`。
   与 PX4 的 `T_xy·g/T_hover` 在 `T_hover=mg` 时等价；不使用限幅前推力。
8. **移除加速度限幅**：旧 `acceleration_limit` 不是 PX4 机制，且与推力/倾角
   约束重复；删除。
9. **参数映射**：
   - `T_max = k_T0·Ω_max²`（SHC09 物理模型）；
   - `T_min = 0.12·T_max`（`MPC_THR_MIN` 的物理映射）；
   - `T_margin = 0.3·T_max`（`MPC_THR_XY_MARG` 的物理映射）；
   - 仅在实际水平需求小于 margin 时才预留对应量，不无条件预留完整 margin。

## 4. Attitude

1. **默认路径**：六舵涵道 airframe 使用 `MC_ATT_ERR_MODE=0` 默认 reduced
   attitude + yaw weight；不实现模式 1–7。
2. **误差**：`quat_error_default.m` 实现 PX4 `calculateAttitudeErrorDefault()`：
   - 由当前/期望 z 轴构造 tilt quaternion；
   - 提取并压缩 yaw 差；
   - 重组期望姿态；
   - `eq=2·canonical(q⁻¹qd).imag`。
3. **Yaw weight**：`yaw_w` 缩放 yaw 误差，同时把 yaw 比例增益除以 `yaw_w`；
   与 PX4 `setProportionalGain` 一致。默认 0.4。
4. **参考模型**：PX4 默认 `MC_REF_W_N=50 rad/s`、`MC_REF_FF=0`、
   `MC_REF_FF_MAX=100°/s`。实现临界阻尼 ZOH（含 `omega_correction`/
   `omega_command`），首拍 snap；状态归 Attitude 所有。
5. **yaw_rate**：世界系 yaw_rate 经 `q_ref⁻¹·[0;0;1]` 投影到参考机体系，
   作为 `omega_command`；不把世界系速率直接加到机体 z 轴。
6. **输出**：`ω_ref = Kp·eq + yaw_rate + ref_ff`，逐轴 `rate_limit`；
   `alpha=0`（PX4 该路径不输出角加速度）。

## 5. Rate

1. **移除本地力矩限幅**：PX4 RateControl 没有 `torque_limit`；执行器约束由
   PINV/执行器 min/max 完成。删除 `rate.torque_limit` 及其测试/文档/WTR 项。
2. **i_factor**：在饱和方向钳位误差后计算
   `i_factor=max(0,1−(e/(400°))²)`；`400°` 是 PX4 硬编码常量，不新增参数。
3. **一阶积分**：`I += Ki·i_factor·e·dt`；`I` 限幅；饱和方向逻辑保留。
4. **输出**：PID 求和后直接发布 N·m，不裁剪；由 PINV 产生饱和/残差反馈。

## 6. Allocation

- 保持 SI `B_tau`、`δ_trim`、`pinv`、六舵限幅、相对配平残差、每轴饱和。
- 电机仍 `Ω=sqrt(T/k_T0)`，与六舵面分离。
- 不引入 `nu/S/tau_ref`；归一化研究结论是暂缓。
- 不增加弱轴整行置零；SHC09 六舵 `B_tau` 行满秩，当前不需要 PX4 的
  “弱轴抑制”特例。

## 7. WTR

1. **签名权威**：scenario 先 `aero_validate_config`，再用实际 cfg 生成签名；
   组件含 `kind/period/priority/offset/delay/latches/topic`，含
   `environment/time_tolerance/solver.InitialStep/x0/initial_command`。
2. **实验输入一致**：`wtr_export` 在创建目录/写文件之前检查
   `options.gains` 与签名、`result.config`（seed/tspan/solver/environment/
   time_tolerance/components/initial_state/initial_command）与签名、
   model/aero 身份；不一致直接拒绝，不留下结果目录或半成品。
3. **参数追溯**：增加 `rates.guidance_hz`、`allocation.ve_min_m_s`、
   `control.rate.saturation_tolerance`、`solver.InitialStep`、涵道 `k_cpx`、
   `control.velocity.hover_thrust_n`、固定片/阻尼/扩散器参数、执行器装配频率、
   实验参考（HoverHold 的 position/yaw/yaw_rate；BodyRate 的 axis/amplitude/
   start/end）等字段。
4. **电机锁存时间轴**：测试逐项比较 `latch_motor` 事件时间，阈值 `1e-9 s`，
   与舵面测试一致。
5. **不改 v1 协议**；新增字段只来自现有 Topic/参数，WindTrace 读取通过。

## 8. 明确不实施

- PX4 归一化 ν/S 生产接口（研究结论：保持 SI，未来可选适配器）。
- INDI、MPC、DCA、CA16、WLS、EKF2/ECL、传感器噪声模型。
- 可选姿态误差模式 1–7。
- 电池缩放/hover-thrust 估计。
- 真实气动数据或物理验证。

## 9. 风险与限制

- 使用合成气动 fixture；物理验证 **BLOCKED**。
- 参考模型与 PX4 的 ZOH 形式在 200 Hz 离散实现；未做 PX4 SITL 并排数值验证，
  只做了独立推导和单元测试。
- `T_min/T_margin` 是从 PX4 归一化参数映射到 SHC09 物理推力的工程选择，
  不是 PX4 原始数值。
- 未声称 PX4 实机/固件验证；只声称源码级数学对齐与 MATLAB 回归。
