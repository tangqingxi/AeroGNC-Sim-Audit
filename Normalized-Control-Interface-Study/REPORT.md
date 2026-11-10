# Normalized-Control-Interface-Study — 研究报告

| 项 | 值 |
|---|---|
| 阶段 | `Normalized-Control-Interface-Study` |
| 性质 | 架构研究（**只读**，无生产代码修改） |
| PX4 参考 | `mengchaoheng/DuctedFanUAV-Autopilot` `df-main` @ `2eb33667575619b02a2d9b52032df2fac7fb5291` |
| 旧 MATLAB 参考 | `tangqingxi/Ducted-Fan-SHC09-matlab` @ `43b6c7aef1912f8118e54d3decac1ac55465d5b1` |
| DEV 核对 | `feat/gnc-baseline` @ `59cf917c5f87bfbf767bc9e9de9ead559bd4929b`（只读） |
| DEV main | `57bbe6893d83808e3b46aa79864701b32b31c775` |
| AUDIT 基线 | `7c8e1bec`（main） |
| 结论 | 保持 SI 物理接口；归一化只作为 Allocation 的可选尺度层，暂不实施 |

## 1. 研究问题

1. PX4 RateControl 输出到底是什么，`[-1,1]` 是否被代码强制？
2. Control Allocation 的 `_control_allocation_scale`、`_mix`、`B` 如何联动？
3. 涵道式机型为何拆成电机/舵面两套矩阵，两个都 `normalizeRPY` 意味着什么？
4. SHC09 物理效能矩阵、配平、出口流速与归一化的关系？
5. 固定尺度与动态尺度对 PID、积分、稳定性的影响？
6. 归一化接口应放在 GNC、Allocation 还是 Aircraft？
7. 如何保留物理量与科研可解释性、如何保证与 PX4 基线一致？
8. 多机型扩展的最小通用接口是什么？

## 2. 主要结论

### 2.1 PX4 归一化的真实机制

`RateControl::update()`（`rate_control.cpp:71-86`）只做 PID 求和，**不裁剪输出**；
`[-1,1]` 是头文件约定（`rate_control.hpp:89-97`），不是函数保证。实际约束来自
`ControlAllocator` 对 `actuator_min/max` 的裁剪和 `unallocated_torque` 反馈。

`ControlAllocationPseudoInverse`：

```text
M = pinv(B_u)
S = diag(scale_roll, scale_pitch, scale_yaw, ...)
M_n = M·S⁻¹
u = u_trim + M_n·ν
```

其中 `B_u` 是归一化执行器效能矩阵（舵面 N·m/单位指令，电机 N/单位指令）；
`scale` 由 `M` 的列统计量计算（roll/pitch 的 RMS、yaw 的 max、thrust 的均值），
不是执行器物理上限。由 `B_u·M=I` 得：

```text
τ_rel = B_u·(u−u_trim) = S⁻¹·ν
ν = S·τ_rel
```

因此 PX4 的归一化控制量 = 物理力矩/力逐轴乘 `S`；`S` 的单位是 1/(N·m)（力矩轴），
不是无量纲百分比，也不是 `[-1,1]` 限幅。

`getAllocatedControl()`（`ControlAllocation.hpp:171-172`）正是
`(B_u·(u−trim)) ⊙ scale`，与 `ν` 一致。`unallocated_torque` 是 `ν` 的残差，
不是 N·m；要恢复到 N·m 必须乘 `S⁻¹`。

### 2.2 涵道式两套矩阵

`ActuatorEffectivenessDuctedFan`（`ActuatorEffectivenessDuctedFan.cpp:14-38`）：

- matrix 0：电机，只产生力；
- matrix 1：六/四舵面，只产生力矩；
- 两个矩阵都 `normalizeRPY=true`；
- `ControlAllocator` 为每个 matrix 建立独立 `ControlAllocation`、独立 `S` 与限幅。

这是“力/力矩物理解耦”的工程选择，不是把七维执行器硬拆。对 SHC09 的适配：
电机推力 `T` 与六舵力矩 `τ` 分开，禁止七维统一求解。

### 2.3 SHC09 数值

当前 DEV（只读核对）：

```text
B_tau = d_cs·Ve²·diag(l1,l1,l2)·B_M        [N·m/rad]
Ve_hover = 24.230354923 m/s
δ_trim = 0.127587337 rad (7.3102°), δ_max = 0.698131701 rad
rank(B_tau)=3, cond=2.8606
U = δ_max·I_6
S_PX4 = diag(1.165752149, 1.165752149, 1.925297848)   [1/(N·m)]
S⁻¹ = diag(0.857815, 0.857815, 0.519400)             [N·m]
```

对多个 τ（含越限）比较物理 `δ=δ_trim+pinv(B_tau)τ` 与归一化
`u=u_trim+pinv(B_u)·S·τ`：未饱和误差 `≤1.4e-16 rad`；饱和裁剪后误差
`1.6e-16 rad`。因此 SHC09 六舵同行程下，固定尺度归一化与物理 PINV 精确等价
（详见 `MATHEMATICAL-DERIVATION.md` §5–§6 与 `evidence/normalization-probe.txt`）。

### 2.4 固定 vs 动态尺度

- **固定 S**：`ν=S·τ`；`S⁻¹` 恒定，物理环路增益不变。即使 `B_tau(Ve)` 变化，
  只要分配器使用当前 `B(Ve)`，物理映射仍精确（数值验证 f=0.5/1.5）。
- **动态 S(Ve)**：`S∝1/Ve²`，同一物理 τ 需要的 ν 随 `Ve²` 变化；若 Rate PID 增益
  仍按 ν 固定，物理环路增益随 `Ve²` 变化，等价于未声明的增益调度。
- PX4 的 `B` 来自配置，不随 `Ve` 连续变化，所以 PX4 没有这个问题；SHC09 有。

结论：SHC09 若归一化，应使用**固定 S/τ_ref**，不要动态 S(Ve)。

### 2.5 架构归属

推荐边界：

```text
GNC（SI τ/T）
   ↓
Allocation（通用 PINV；可选 S/τ_ref 变换）
   ↓
Aircraft Effectiveness（B_p、U、trim、τ_ref）
   ↓
Components（舵机/电机）
```

- `S`/`τ_ref` 由 Aircraft effectiveness 配置提供，Allocation 使用；**不在 GNC 内**；
- 通用 PINV 不知道 SHC09 的 `d_cs`、`l1`、`l2`、`B_M`；这些在 Aircraft 生成 `B_p`；
- 电机与舵面独立；
- 更换机型只换 `B_p`、`U`、`trim`、`τ_ref`，控制器接口不变。

### 2.6 PID 基线一致性

若采用 ν 接口，必须做尺度换算：

```text
Kp_ν=S·Kp_τ, Ki_ν=S·Ki_τ, Kd_ν=S·Kd_τ
I_lim,ν=S·I_lim,τ, τ_limit,ν=S·τ_limit
```

误差、积分时间、角加速度反馈、抗饱和符号不变。只改接口不改增益属于算法变化，
必须显式记录，不能称为“纯重构”。

## 3. 方案比较摘要

| 方案 | 复杂度 | 物理可解释性 | 控制器独立性 | 结论 |
|---|---|---|---|---|
| A 物理接口（当前） | 最低 | 最高 | 高 | **推荐保留** |
| B PX4 式归一化 | 中 | 中 | 降低 | 仅在明确 PX4 对齐需求时 |
| C 物理输出+内部归一化 | 低中 | 高 | 高 | 可选，收益有限 |
| D 双接口适配器 | 中 | 高 | 高 | **未来实施的首选形态** |
| E Aircraft 后归一化 | — | — | — | 不可行 |

## 4. 推荐

1. **V1.0 不实施归一化接口**；保留 `τ [N·m]`、`T [N]`、`δ [rad]`、`Ω [rad/s]`。
2. 若未来实施，归一化放在 **Allocation/Aircraft effectiveness 边界**，而不是
   Rate PID 或 Aircraft 动力学。
3. 用显式参考力矩定义 `S=diag(1./τ_ref)`；`τ_ref` 固定于飞行阶段；
   PX4 的 `S` 可作为 `τ_ref=1./S_PX4` 的对照配置。
4. Rate PID 默认不改；可选 ν 输出必须同时换算增益与限幅。
5. 电机独立；可做标量 `T/T_max`，禁止七维求解。
6. 物理量为主日志，ν/S 为派生字段；WTR 不以 ν 替代 N·m。
7. 多机型统一到 6 维物理 wrench；`B_p`、`U`、`trim`、`τ_ref` 归机型；
   不建复杂抽象框架。

## 5. 验证与证据

| 内容 | 方式 | 结果 |
|---|---|---|
| PX4 源码机制 | 指定提交逐函数阅读、调用链追踪 | 见 `PX4-SOURCE-MAP.md` |
| SHC09 B_tau/S/等价性 | 独立临时 MATLAB 脚本（未入 DEV） | 误差 ≤1.4e-16 rad；饱和映射精确 |
| 动态 Ve | 同一脚本 f=0.5/1.5 | 固定 S 精确；动态 S 改变 ν/物理增益 |
| 旧 MATLAB 对照 | `ca_model.m`、`ca_shc09_aerodynamics.m` | 物理 B_tau、配平、Ve² 一致 |
| DEV 核对 | 只读阅读 `shc09_*`、`pinv_allocator`、`rate_pid` | 当前为物理接口 |

未执行：PX4 编译/飞行、WindTrace 渲染、真实气动验证。本研究的数值验证仅覆盖
分配数学，不代表物理闭环。

## 6. 限制

- PX4 代码为指定 fork，不等于 PX4 官方上游；结论只对该提交负责。
- `S` 的公式含 `geninv` 数值实现；不同矩阵库可能有微小差异。
- SHC09 气动为合成 fixture；`Ve`、配平、B_tau 数值只用于接口数学，不是物理验证。
- 未做真实飞行/台架；未验证动态 S 的稳定性（仅做增益关系分析）。
- 未对 PX4 的全部分配方法（WLS、PCA、INDI）做等价性证明；本研究以
  PseudoInverse 路径为主。

## 7. 交付物

```text
Normalized-Control-Interface-Study/
├── REPORT.md
├── PX4-SOURCE-MAP.md
├── MATHEMATICAL-DERIVATION.md
├── SHC09-NORMALIZATION.md
├── ARCHITECTURE-OPTIONS.md
├── RECOMMENDATION.md
├── HANDOFF.md
└── evidence/
    ├── normalization-probe.txt
    └── source-versions.txt
```

## 8. 最终结论

PX4 的成熟经验值得学习的是**力/力矩解耦、独立分配矩阵、显式饱和反馈与执行器
归一化边界**，而不是把控制器输出改成无量纲数。AeroGNC-Sim 当前 SI 接口在数学上
与固定尺度归一化等价，且更符合四级独立与科研可解释性。建议保留现状；把归一化
作为未来 Allocation 的可选适配器，用固定参考力矩实现，并以等价性测试作为准入门槛。
