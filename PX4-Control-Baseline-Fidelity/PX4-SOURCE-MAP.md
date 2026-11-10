# PX4-Control-Baseline-Fidelity — 源码地图

## 1. 固定参考

| 项 | 值 |
|---|---|
| 主参考仓库 | <https://github.com/mengchaoheng/DuctedFanUAV-Autopilot> |
| 分支 | `df-main` |
| SHA | `2eb33667575619b02a2d9b52032df2fac7fb5291` |
| 旧 MATLAB | `tangqingxi/Ducted-Fan-SHC09-matlab` @ `43b6c7aef1912f8118e54d3decac1ac55465d5b1` |
| 当前 DEV | `feat/px4-control-fidelity` @ `61507b4edd68975c962949a2cd750bb38569abb4` |

以下行号均指 PX4 固定提交。只研究该 fork，不替换为官方上游。

## 2. 调用链

```text
PositionControl::update()
  ├── _positionControl()         位置 P + constrainXY + 垂直限幅
  ├── _velocityControl()         Velocity PID + 推力/倾角限幅 + 抗饱和
  ├── _accelerationControl()     加速度 → 推力方向/总推力（含 INDI 分支）
  └── getAttitudeSetpoint()      ControlMath::thrustToAttitude + yaw_rate

AttitudeControl::setAttitudeSetpoint()
  ├── propagateReferenceModel()  PX4 默认二阶临界阻尼参考模型
  └── calculateAttitudeErrorDefault()
AttitudeControl::update()        Kp·e + yaw_rate + ref-model FF → rate limit

RateControl::update()            P反馈 + 积分 + Feedforward − D·α
RateControl::updateIntegral()    条件积分 + i_factor + 限幅

ControlAllocator::Run()
  ├── c[0..2]=torque_sp, c[3..5]=thrust_sp
  ├── ControlAllocationPseudoInverse::allocate()
  └── control_allocator_status（unallocated / saturation）
```

## 3. PositionControl

| 函数/位置 | 事实 | AeroGNC 对应 |
|---|---|---|
| `PositionControl.cpp:129-143` `_positionControl` | `vel_sp_position=(p_sp−p).*Kp`；前馈加到速度；`constrainXY(v0=position, v1=feedforward)`；z 限幅 `-vel_up..vel_down` | `position_p.m` |
| `ControlMath.cpp:116-178` `constrainXY` | 位置反馈方向优先；若 v0 已超限只保留 v0 方向；否则解二次方程给前馈剩余能力 | `constrain_xy.m` |
| `PositionControl.cpp:145-222` `_velocityControl` | 垂直积分先钳位 ±g；PID `Kp e + I − Kd a_dot`；垂直/水平推力饱和；垂直积分抗饱和；水平 Tracking AW | `velocity_pid.m` |
| `PositionControl.cpp:280-311` `_accelerationControl` | `body_z=normalize([−a_x,−a_y,g−a_z])`；`limitTilt`；`thrust_ned_z=m(a_z−g)`；`collective=min(thrust_z/cos,−thr_min)` | `velocity_pid.m` + `attitude_from_thrust_direction.m` |
| `PositionControl.cpp:355-359` `getAttitudeSetpoint` | `thrustToAttitude(thr_sp, yaw_sp)`；`yaw_sp_move_rate=yawspeed_sp` | `control/attitude_setpoint.yaw_rate` |

INDI 路径（`_allocated_thrust_acceleration` 有效时）不移植；只保留 PID 分支。

## 4. AttitudeControl

| 函数/位置 | 事实 | AeroGNC 对应 |
|---|---|---|
| `AttitudeControl.cpp:260-283` `setProportionalGain` | `yaw_w=clamp(MC_YAW_WEIGHT)`；decomposed 模式 yaw 增益 `/yaw_w` | `attitude_p.m` |
| `AttitudeControl.cpp:291-306` `setAttitudeSetpoint` | 首拍/无效 dt 时 snap `q_ref=qd`；否则 `propagateReferenceModel` | `attitude_p.m` 状态 |
| `AttitudeControl.cpp:308-355` `propagateReferenceModel` | 临界阻尼 ZOH；`MC_REF_W_N` 默认 50；yaw rate 投影到参考机体系；`_omega_correction` / `_omega_command` | `attitude_p.m` |
| `AttitudeControl.cpp:373-413` `calculateAttitudeErrorDefault` | reduced tilt + yaw weight + yaw 分量重组；`eq=2·canonical(q⁻¹qd).imag` | `quat_error_default.m` |
| `AttitudeControl.cpp:667-700` `update` | `Kp·eq`（yaw 已补偿）+ yaw_rate + ref FF；逐轴 rate limit | `attitude_p.m` |
| `mc_att_control_main.cpp:91-107` | 参数装载：P、YawWeight、ErrorMode、RateLimit、RefWn/FF/FFMax | `control_defaults.m` |

默认路径选择：

- 六舵涵道 airframe（22010/22003）未覆盖 `MC_ATT_ERR_MODE`，使用默认 `0`
  （reduced attitude + yaw weighting）；`MC_YAW_WEIGHT` 默认 `0.4`；
  `MC_REF_W_N=50`、`MC_REF_FF=0`、`MC_REF_FF_MAX=100°/s`。
- 22018（四舵 SIH）显式使用 `MC_ATT_ERR_MODE=3`（DCM log）、`YAW_WEIGHT=1`；
  这是该 airframe 的覆盖，不作为 SHC09 六舵基线。
- 只实现模式 0；模式 1–7 不进入 V1.0。

## 5. RateControl

| 函数/位置 | 事实 | AeroGNC 对应 |
|---|---|---|
| `rate_control.cpp:71-86` `update` | `torque=P e + I − D α + FF ω_sp`；**无本地 output clamp** | `rate_pid.m` 无 torque_limit |
| `rate_control.cpp:88-118` `updateIntegral` | 先按 allocator 饱和方向限制误差符号；`i_factor=max(0,1−(e/400°)²)`；一阶积分；积分限幅 | `rate_pid.m` |
| `MulticopterRateControl.cpp:394-413` | 从 `control_allocator_status.unallocated_torque` 生成正/负饱和标志 | `control/allocation_status.saturated` |
| `rate_control.hpp:89-97` | 注释 `[-1,1] normalized torque`；不是函数内限幅 | 文档已注明约定 |

## 6. Control Allocation

| 函数/位置 | 事实 | AeroGNC 对应 |
|---|---|---|
| `ControlAllocation.cpp:52-65` | `actuator_trim=trim+linearization_point`；`control_trim=B·linearization_point` | `pinv_allocator.m` 的 `δ_trim` |
| `ControlAllocation.hpp:171-172` | `getAllocatedControl=(B(u−trim))⊙scale` | SI：`B_tau(δ−δ_trim)` |
| `ControlAllocationPseudoInverse.cpp:62-78` | `geninv(B)` | `pinv(B_tau)` |
| `ControlAllocationPseudoInverse.cpp:80-148` | 控制轴 scale（roll/pitch RMS、yaw max、thrust mean） | 未实施归一化；只参考 |
| `ControlAllocationPseudoInverse.cpp:179-189` | `u=trim+mix(sp−control_trim)` | `δ=δ_trim+pinv(B_tau)τ_rel` |
| `ControlAllocator.cpp:872-893` | 保留原始 `B·U` 供反馈；弱轴整行置零 | 无弱轴置零；SHC09 行满秩 |
| `ControlAllocator.cpp:983-1017` | `unallocated` 与每执行器饱和 | `allocator_status.residual/saturated` |
| `ActuatorEffectivenessDuctedFan.cpp:14-38` | matrix0 电机力；matrix1 舵面力矩；二者 `normalizeRPY` | `pinv_allocator.m` 六舵 + 独立电机 |

归一化研究已决定 V1.0 保持 SI；本阶段不引入 `nu/S/tau_ref` 生产接口。

## 7. 旧 DEV 与 PX4 的差异（整改前）

| 项 | 59cf917 旧实现 | 本阶段 |
|---|---|---|
| Velocity Kd | 注释单位 `s` | `1`（无量纲） |
| Position 限幅 | 复用 `velocity.limit_*` | `position.velocity_limit_*` |
| Position 水平限幅 | 合成后统一缩放 | `constrainXY` 位置反馈优先 |
| Velocity 抗饱和 | 仅加速度限幅冻结 | 垂直 ±g、垂直饱和、水平 Tracking AW |
| Velocity 推力 | 只做最大倾角 | 总推力/最小推力/水平余量/倾角 |
| Attitude 误差 | 全四元数误差 | PX4 默认 reduced + yaw weight |
| Attitude 参考模型 | 无 | PX4 ZOH 参考模型 |
| yaw_rate | 静默丢弃 | 经 Position/Velocity/Attitude 传递 |
| Rate 力矩限幅 | 控制器本地 `torque_limit` | 移除，交给 PINV/执行器 |
| Rate i_factor | 无 | PX4 400°/s 二次衰减 |
| WTR | 签名/参数不完整 | 完整 cfg/组件/参数/电机时间轴 |

## 8. 不纳入的 PX4 机制

- INDI/加速度 INDI 与 bumpless transfer（本阶段只纯 PID）。
- PX4 归一化 `nu/S` 虚拟控制接口（研究阶段已决定暂缓）。
- 可选姿态误差模式 1–7。
- 控制分配 WLS/PCA/CA16/序列去饱和。
- PX4 的 hover-thrust 估计与电池缩放（SHC09 用物理质量/推力）。
