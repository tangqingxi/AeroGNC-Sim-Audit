# PX4 归一化控制源码地图

参考提交：`mengchaoheng/DuctedFanUAV-Autopilot`，分支 `df-main`，
SHA `2eb33667575619b02a2d9b52032df2fac7fb5291`（以下行号均对该提交）。

## 1. 调用链

```text
RateControl::update()                    src/lib/rate_control/rate_control.cpp
        │  输出 torque_setpoint（无量纲）
        ▼
MulticopterRateControl::update()         src/modules/mc_rate_control/MulticopterRateControl.cpp
        │  发布 vehicle_torque_setpoint.xyz
        ▼
ControlAllocator::Run()                  src/modules/control_allocator/ControlAllocator.cpp
        │  c[0..2] = _torque_sp；c[3..5] = _thrust_sp
        ▼
ControlAllocationPseudoInverse::allocate()
        │  u = trim + M_n · ν
        ▼
ControlAllocation::getAllocatedControl()
        │  w_norm = (B·(u−trim)) ⊙ scale
        ▼
control_allocator_status.unallocated_torque  →  RateControl 饱和反馈
```

## 2. RateControl

| 项目 | 源码位置 | 事实 |
|---|---|---|
| PID 求和 | `rate_control.cpp:71-86` | `torque = P⊙e + I − D⊙α + FF⊙ω_sp`；**无输出限幅** |
| 积分 | `rate_control.cpp:88-118` | 一阶积分、`i_factor`、`_lim_int` 限幅；按分配器饱和方向限制误差符号 |
| 输出语义 | `rate_control.hpp:89-97` | 注释称 `[-1,1] normalized torque`；仅约定，不是函数内约束 |
| 增益尺度 | `MulticopterRateControl.cpp:126-140` | `MC_*RATE_K` 同时乘 P/I/D，用于并行/理想 PID 形式换算 |
| 发布 | `MulticopterRateControl.cpp:416-447` | `torque_setpoint` 直接写入 `vehicle_torque_setpoint.xyz`，无 N·m 转换 |

结论：RateControl 输出是**分配器控制空间中的无量纲虚拟力矩**；`[-1,1]` 是调参约定，
函数本身不裁剪。上下界由控制分配的 `actuator_min/max` 裁剪和 `unallocated_torque`
反馈体现。

## 3. ControlAllocation 基类

| 项目 | 源码位置 | 事实 |
|---|---|---|
| 效能矩阵/配平 | `ControlAllocation.cpp:52-65` | `_actuator_trim = trim + linearization_point`；`_control_trim = B·linearization_point` |
| 已分配控制 | `ControlAllocation.hpp:171-172` | `getAllocatedControl = (B·(u−trim)) ⊙ scale` |
| 执行器限幅 | `ControlAllocation.cpp:78-92` | 按 `actuator_min/max` 逐元素裁剪 |
| 归一化执行器 | `ControlAllocation.cpp:94-110` | `(u−min)/(max−min)`；仅用于状态/日志，不参与分配 |

## 4. 伪逆与归一化尺度

| 项目 | 源码位置 | 事实 |
|---|---|---|
| 伪逆 | `ControlAllocationPseudoInverse.cpp:62-78` | `M = geninv(B)`；配置更新时重算 |
| 尺度计算 | `ControlAllocationPseudoInverse.cpp:80-148` | roll/pitch：`sqrt(‖M_col‖²/(n_nonzero/2))`；yaw：`max(M_col)`；thrust：非零项平均 |
| 矩阵归一化 | `ControlAllocationPseudoInverse.cpp:150-177` | `M_n = M·S⁻¹`；小于 1e-3 的元素置 0 |
| 分配 | `ControlAllocationPseudoInverse.cpp:179-189` | `u = trim + M_n·(ν − control_trim)` |
| 弱轴处理 | `ControlAllocator.cpp:868-893` | 行内最大元素 ≤0.05 的轴整行置 0；原始 `B·U` 另存供 INDI |

关键：`scale` 不是执行器物理上限，而是由 `B` 的伪逆列统计量得到的**控制轴尺度**；
`ν = scale ⊙ (B·(u−trim))`，即归一化虚拟控制 = 物理力矩逐轴乘 `scale`（`scale` 单位
1/N·m，若 `B` 为 N·m/单位执行器）。

实现细节（影响等价性证明）：

- yaw 尺度用 `_mix.col(2).max()`（有符号最大值），不是绝对值；代码假定偏航列
  元素为正。若某机型偏航列全负，`scale` 会为负，需视为边界缺陷，不能直接套用。
- `normalizeControlAllocationMatrix()` 把 `|M_n(i,j)|<1e-3` 置 0；这会改变零空间
  分配，严格等价只在未被截断的矩阵上成立。
- 尺度只在 `_normalization_needs_update` 且无执行器故障时重算；故障后冻结，
  避免故障瞬间改变控制尺度。


## 5. 涵道式机型的两套矩阵

`ActuatorEffectivenessDuctedFan.cpp:14-38`：

- `numMatrices() = 2`；
- matrix 0：`_motors`（`ActuatorEffectivenessRotors`），只产生力；
- matrix 1：`_torque`（`ActuatorEffectivenessControlSurfaces`），只产生力矩；
- `getNormalizeRPY` 对两个矩阵都返回 true；
- 注释说明电机与舵面物理解耦；若未来某几何同时产生力与力矩，才应合并矩阵。

`ControlAllocator.cpp:663-690` 将 `_torque_sp` 放入 `c[0..2]`、`_thrust_sp` 放入
`c[3..5]`；多矩阵时 instance 1 单独接收力矩/推力 setpoint。`ControlAllocator.cpp:
1309-1318` 把两个矩阵各自的 `_control_allocation_scale` 发布给 INDI 使用。

两个矩阵都 `normalizeRPY=true` 的原因：

- matrix 1（舵面）三个力矩轴都有效，需要分别归一化，避免滚转/俯仰/偏航的
  `M` 列量级差异导致伪逆数值条件差；
- matrix 0（电机）通常只有 thrust 行非零，RPY 列为零；`normalizeRPY` 对零列
  不产生实际除法（yaw scale=0 时跳过，roll/pitch 用 1），因此对纯推力矩阵无害；
  若电机有偏置/倾斜产生力矩，同一开关也会把其 RPY 列纳入归一化；
- 统一使用同一 flag，避免按机型写特例；真正的物理边界由 `numMatrices()` 和
  两个独立 `ControlAllocation` 实例保证。

## 6. 效能参数的物理单位

`ROMFS/px4fmu_common/init.d/airframes/22002_ductedfan4`（及 SIH 22018、六舵 22010）：

```text
CA_ROTOR0_CT 30          % 最大推力 [N]（归一化 +1 时的推力）
CA_SV_CSx_TRQ_R/P/Y      % 归一化舵面 +1 时的力矩 [N·m]
v = (B*U)*u_norm
```

因此该工程的 `B` 是**物理量/单位归一化执行器**：
`B_surface` 列 = N·m per 单位舵面指令，`B_rotor` 列 = N per 单位电机指令。
`U` 是执行器物理范围换算（舵面 ±max、电机 0..T_max），在参数中已折算到列。

旧 MATLAB `models/shc09/ca_model.m:19-33` 使用
`B = d_cs·Ve²·diag(l1,l1,l2)·P_cs_b`（N·m/rad），配平 `pinv(B)·[0;0;−residual]`；
`ca_shc09_aerodynamics.m:45` 在每个采样用真实 `Ve` 重建 `C`。这证明旧实现是
**物理弧度接口**，不是 PX4 的归一化执行器接口。

DEV 当前实现（只读核对）：

- `shc09_control_surfaces.m`：`B_tau = d_cs·Ve²·diag(l1,l1,l2)·B_M`，单位 N·m/rad；
- `shc09_hover_equilibrium.m`：共同配平舵角，`M_fan+M_sta+B_tau·δ_trim=0`；
- `pinv_allocator.m`：`δ = δ_trim + pinv(B_tau)·τ_rel`，τ_rel 为 N·m；
- `rate_pid.m`：输出 N·m。

## 7. 归一化层次区分

| 层次 | 定义 | 单位 | 源码 |
|---|---|---|---|
| 控制轴归一化 | `ν = S·τ_rel` | 无量纲（若 τ N·m，S 为 1/N·m） | `getAllocatedControl`、`scale` |
| 效能矩阵归一化 | `M_n = M·S⁻¹` | 执行器/单位虚拟控制 | `normalizeControlAllocationMatrix` |
| 执行器指令归一化 | `u ∈ [-1,1]` 舵面、`[0,1]` 电机 | 无量纲 | `ControlAllocator` min/max |
| 物理量转换 | `τ = S⁻¹ν`、`δ = U·u` | N·m、rad | 参数 `U` 与 `B` |

四者不能混为“N·m → [-1,1]”。
