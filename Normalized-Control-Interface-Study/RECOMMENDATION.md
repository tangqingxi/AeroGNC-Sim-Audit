# 推荐方案

## 1. 决策摘要

**V1.0 不引入归一化虚拟控制接口。保留当前 SI 物理接口：**

```text
Position P → Velocity PID → Attitude P → Rate PID
    → τ [N·m] → PINV(B_tau) → δ [rad] → 舵机
    → T [N]  → 电机
```

理由：

1. 四级控制器、Topic、WTR 与测试已经以 SI 为单位闭合；归一化不解决当前任何
   已确认的缺陷，反而会引入控制器增益与机型尺度耦合。
2. PX4 的“归一化”不是物理限幅，而是由 `B_u` 伪逆列统计量得到的控制轴条件化尺度
   （`S` 单位 1/N·m）；把它当成 `[-1,1]` 会误导调参与验收。
3. SHC09 六舵同一 ±40°，固定 `S` 与物理 PINV 可精确等价；但动态 `S(Ve)` 会改变
   物理环路增益，等价于未声明的增益调度，风险高于收益。
4. 电机与六舵面必须保持独立；PX4 的两矩阵设计支持这一点，七维统一求解器不可取。

## 2. 若未来引入归一化：推荐接口

### 2.1 定义

采用**物理参考力矩**定义尺度，而不是直接复制 PX4 的 RMS 启发式：

```text
τ_ref = [τ_ref,roll; τ_ref,pitch; τ_ref,yaw]     [N·m]，由机型给出
S = diag(1./τ_ref)                               [1/(N·m)]
ν = S·τ_rel = τ_rel./τ_ref                       [无量纲]
B_u = B_p·U                                      [N·m/单位 u]
u = u_trim + pinv(B_u)·(τ_ref.*ν)
δ = U·u
```

- `ν=1` 明确表示“达到声明参考力矩”，物理意义清楚；
- `S` 固定于一个飞行阶段（悬停/前飞），不随 `Ve` 连续变化；
- PX4 的 `S` 可等价写成 `τ_ref=1./S_PX4`；SHC09 悬停为
  `[0.8578,0.8578,0.5194] N·m`，可直接对照；
- 若必须复现 PX4 的 `geninv` 尺度，把它作为 `τ_ref` 的一种配置来源，而不是
  控制器内的隐式行为。

### 2.2 归属

| 内容 | 归属 |
|---|---|
| `τ_ref`（或 `S`） | Aircraft effectiveness 配置（按飞行阶段） |
| `B_p`、`U`、`trim` | Aircraft |
| `B_u=B_p·U`、`pinv`、限幅、残差 | Allocation（通用） |
| `ν` 的产生与增益换算 | Rate（可选适配器） |
| 物理 `τ` | Rate→Allocation 默认 Topic（N·m） |
| 电机 `T`/`Ω` | 独立于六舵 PINV |

**归一化不属于 GNC 控制律本身，也不属于 Aircraft 动力学；它是 Allocation 的
输入/输出尺度层，由机型 effectiveness 参数化。**

### 2.3 控制器与 PID 基线

- 默认：Rate 仍输出 `τ [N·m]`；Allocation 内部如需 ν，自行乘 `S`，控制器不变。
- 可选：Rate 输出 `ν` 时，必须同步换算
  `Kp_ν=S·Kp_τ`、`Ki_ν=S·Ki_τ`、`Kd_ν=S·Kd_τ`、
  `I_lim,ν=S·I_lim,τ`、`τ_limit,ν=S·τ_limit`；
  误差、积分时间、角加速度反馈、抗饱和符号不变。
- 不允许只改接口不改增益，然后把闭环差异归因于“归一化”。
- Position/Velocity/Attitude 不改；Velocity 加速度输出与推力归一化无直接关系，
  本轮不引入推力归一化到控制链。

### 2.4 物理量与归一化量并存

建议的 Topic 契约（**本轮不新增，仅作为未来实施规格**）：

| Topic/字段 | 单位 | 说明 |
|---|---|---|
| `control/torque_setpoint.torque` | N·m | 默认物理接口，保持不变 |
| `control/thrust_setpoint.thrust` | N | 电机推力，独立于六舵 |
| `control/normalized_torque_setpoint.nu` | 无量纲 | 可选；仅当启用 ν 路径 |
| `.tau_ref` | N·m | ν 的参考力矩，随消息发布 |
| `control/allocation_status.delta_rad` | rad | 实际舵角 |
| `.tau_achieved_nm` | N·m | 实际可达力矩 |
| `.residual_nm` | N·m | 物理残差 |
| `.residual_normalized` | 无量纲 | `S·residual_nm` |
| `.scale` | 1/(N·m) | `S` 对角 |
| `.saturated` | 布尔/枚举 | 每轴/每执行器 |

Allocation 状态至少同时给出：

```text
delta_rad              实际舵角 [rad]
tau_physical_nm        实际/可达力矩 [N·m]
tau_normalized         实际/可达力矩的 ν = S·τ [无量纲]
residual_nm            残差 [N·m]
residual_normalized    S·residual_nm
saturated              每轴/每执行器饱和
scale                   S 与 τ_ref
```

WTR 记录物理量为权威，归一化量作为派生字段；不得用 ν 替代 N·m。

### 2.5 电机

- 保留 `T [N]` 与 `Ω [rad/s]`；
- 如需归一化推力，定义 `u_m=T/T_max`、`T_max` 由设备/机型给出，单独 Topic/字段；
- 电机矩阵 `B_rotor` 与舵面矩阵 `B_surface` 分开，不构造 7 维求解器；
- 电机饱和用 `T` 或 `u_m` 单独反馈，不混入三轴力矩残差。

## 3. 架构边界

```text
GNC Control Law（SI）
    ↓ τ [N·m], T [N]
Control Allocation（通用 PINV）
    ↓ 可选 ν = S·τ（Allocation/Aircraft 配置）
Aircraft Effectiveness（B_p, U, trim, τ_ref）
    ↓ δ [rad], Ω
Components（舵机/电机动力学）
```

- 通用 PINV 不知道 SHC09 的 `B_M`、`l1`、`l2`、`d_cs`；这些由 Aircraft 生成
  `B_p` 后传入；
- SHC09 专属的 `B_tau(Ve)` 与配平留在 Aircraft；
- 归一化尺度 `S`/`τ_ref` 由 Aircraft/Allocation 配置提供，GNC 不 import 机型参数；
- 更换机型只换 `B_p`、`U`、`trim`、`τ_ref`，控制器接口不变。

## 4. 多机型最小通用接口

建议只统一到 **6 维物理虚拟控制**：

```text
wrench = [τ_x; τ_y; τ_z; F_x; F_y; F_z]   [N·m; N]
B_p ∈ R^{6×n},  U ∈ R^{n×n},  trim ∈ R^n
```

- 多旋翼/固定翼/涵道尾座/过驱动都用同一 `B_p·U` 与 PINV；
- 控制轴定义、执行器数量、范围、配平、动态效能全部在 Aircraft；
- `S`/`τ_ref` 可选，且是机型/飞行阶段属性，不是通用常数；
- 不建立机型注册中心、抽象基类或通用归一化框架。

## 5. 对现有 GNC 的迁移范围（若未来实施）

| 模块 | 改动 |
|---|---|
| Position/Velocity/Attitude | 无 |
| Rate | 默认无；仅可选 ν 输出 + 增益换算 |
| Allocation | 增加可选 `S`/`τ_ref`、`B_u`、`u_trim`、双单位状态 |
| Topic | 默认 `torque_setpoint` N·m 不变；可选 normalized 字段/Topic |
| Aircraft | 增加 `τ_ref`/`U` 输出；`B_p` 已有 |
| Simulation | 无 |
| WTR | 记录 `S`、`τ_ref`、`ν`、物理 τ 与残差；不改 v1 协议 |
| 测试 | 等价性、饱和、增益换算、跨机型接口、物理日志 |

## 6. 实施门槛（何时才做）

满足任一条件再进入实现：

1. 需要直接复用 PX4 Rate/INDI 控制器或与 PX4 参数逐项对照；
2. 出现第二个机型，且希望共享同一套 ν 调参；
3. 需要 INDI 的归一化力矩反馈/优先级分配；
4. NEXUS 明确要求统一虚拟控制接口。

否则保持 SI，不为“架构漂亮”增加复杂度。

## 7. 最终回答

- **PX4 归一化机制**：`ν = S·B_u·(u−u_trim)`，`S` 由 `B_u` 伪逆列统计得到；
  Rate 输出 ν，Allocation 用 `M_n=M·S⁻¹` 求 u；不是 N·m→[-1,1] 的限幅。
- **RateControl 输出**：无量纲虚拟力矩，`[-1,1]` 只是约定；实际由分配器限幅与
  残差反馈约束。
- **是否对应执行器物理范围**：不直接对应；`S` 是条件化尺度，执行器范围在
  `u_min/u_max` 与 `U` 中。
- **SHC09 最适合**：保持物理 τ；若归一化，固定 `S`/`τ_ref`（悬停标定），
  不用动态 `S(Ve)`。
- **固定 vs 动态**：固定尺度保持物理环路增益；动态尺度改变增益，需调度。
- **接口归属**：Allocation/Aircraft effectiveness 边界，不在 GNC 控制律。
- **四级控制器是否改**：默认不改；可选 Rate ν 输出需增益换算。
- **PINV 是否改**：算法不改；只增加 `B_u`、`S`、`u_trim` 输入。
- **电机是否归一化**：可独立做标量 `T/T_max`，不与六舵合并。
- **SI 与可解释性**：物理 τ/δ/T/Ω 为主，ν/S 为派生并记录。
- **PX4 对齐**：保留 PID 数学与抗饱和；仅做尺度换算并显式记录。
- **是否进入下一阶段实施**：**不建议立即实施**；先由 NEXUS 评审本研究。
  若评审通过，下一阶段最多做一个“Allocation 可选归一化适配器 + 等价性测试”
  的最小实现，不改变默认 SI 路径。
