# 归一化数学推导

## 0. 符号与单位

| 符号 | 含义 | 维度/单位 |
|---|---|---|
| `a` | 物理执行器（舵面 rad，电机 N 或 rad/s） | `n×1` |
| `B_p` | 物理效能矩阵（舵面 N·m/rad；电机 N/N） | `3×n` |
| `U` | 归一化执行器到物理量的尺度（`a = U·u`） | `n×n`，对角 |
| `u` | 归一化执行器指令（舵面 −1..1，电机 0..1） | `n×1` 无量纲 |
| `B_u` | 归一化效能矩阵 `B_u = B_p·U` | `3×n`，N·m/单位 或 N/单位 |
| `τ_rel` | 相对配平的物理力矩/力 | `3×1`，N·m 或 N |
| `ν` | 归一化虚拟控制 | `3×1` 无量纲 |
| `S` | 控制轴归一化尺度 | `3×3`，对角，1/(N·m) 或 1/N |
| `M` | `B_u` 的 Moore–Penrose 伪逆 | `n×3`，单位执行器/(N·m) |

本报告以 SHC09 六舵面力矩为例（`n=6`，`B_p=B_tau`，`U=δ_max·I`，`δ_max=40°`）。
电机是独立一维问题，见 §7。

## 1. 物理接口（方案 A，当前 DEV）

配平舵角 `δ_trim`，定义相对量 `Δδ = δ − δ_trim`：

```text
τ_rel = B_tau · Δδ                                  [N·m]
δ = δ_trim + pinv(B_tau) · τ_rel                    [rad]
```

`pinv` 为 Moore–Penrose 伪逆；当 `rank(B_tau)=3` 时
`B_tau·pinv(B_tau)=I_3`，故上式精确复现 `τ_rel`（无饱和时）。

## 2. 归一化执行器与效能矩阵

取 `U = δ_max·I_6`（六舵同一行程）：

```text
δ = U·u,  B_u = B_tau·U                             [N·m/单位 u]
M = pinv(B_u)                                       [单位 u/(N·m)]
```

PX4 的 `B` 即此 `B_u`：`CA_SV_CSx_TRQ_*` 是归一化 +1 时的 N·m，
`CA_ROTOR0_CT` 是归一化 +1 时的 N。

## 3. PX4 的控制轴尺度 `S`

`ControlAllocationPseudoInverse::updateControlAllocationMatrixScale()`：

```text
M = pinv(B_u)
s_roll  = sqrt( ‖M_col1‖² / (n_nonzero1/2) )
s_pitch = sqrt( ‖M_col2‖² / (n_nonzero2/2) )
s_rp    = max(s_roll, s_pitch)
s_yaw   = max(M_col3)                    （无非零列时为 1）
s_thrust= mean(|M_col_j|), j=3..5         （非零项平均）
S = diag([s_rp, s_rp, s_yaw, ...])
M_n = M · S⁻¹
```

这是**数值条件化**的尺度，不是执行器物理上限：

- roll/pitch 使 `M_n` 列的非零项 RMS 约为 `1/√2`；
- yaw 使 `M_n` 列最大元素为 1；
- thrust 使 `M_n` 列非零项平均绝对值为 1；
- `s_j` 越小，表示单位物理力矩需要越大的执行器指令。

## 4. 归一化分配与精确映射

分配律：

```text
u = u_trim + M_n · ν
```

其中 `ν` 是 Rate PID 输出（相对配平）。代入：

```text
B_u·(u − u_trim) = B_u·M·S⁻¹·ν = I_3·S⁻¹·ν = S⁻¹·ν
τ_rel = B_u·(u − u_trim) = S⁻¹·ν
ν = S·τ_rel
```

`getAllocatedControl()` 实现 `(B_u·(u−u_trim)) ⊙ s = ν`，与上式一致。

因此 **PX4 的归一化虚拟控制 = 物理力矩/力逐轴乘以 `S`**；
`S` 的量纲为 1/(N·m)（对力矩轴），`S⁻¹` 为 N·m/单位虚拟控制。

## 5. 与物理 PINV 的等价条件

把 `ν = S·τ_rel` 代入：

```text
u = u_trim + M·S⁻¹·S·τ_rel = u_trim + M·τ_rel
δ = U·u = δ_trim + U·pinv(B_u)·τ_rel
```

对均匀 `U = δ_max·I`：

```text
U·pinv(B_u) = U·pinv(B_tau·U) = pinv(B_tau)
```

故 `δ = δ_trim + pinv(B_tau)·τ_rel`，与物理接口**逐元素相同**。

**等价条件**：

1. `rank(B_u)=3`（或至少 `B_u·M=I_3`）；
2. `S` 非奇异正对角，且 `ν = S·τ_rel` 精确成立；
3. `U` 对同一组执行器是均匀尺度（SHC09 六舵 ±40° 满足）；
4. 没有弱轴整行置零、没有阈值截断（PX4 的 `<1e-3` 置零会引入微小差异）；
5. 使用同一种伪逆（未加权的 Moore–Penrose）；
6. 配平基线一致（`u_trim` 与 `δ_trim` 对应）。

若 `U` 非均匀，则 `wrench` 仍精确（`B_u·Δu=S⁻¹ν`），但执行器解不同：
归一化分配最小化 `‖Δu‖`，物理分配最小化 `‖Δδ‖`；两者只在零空间分量上不同。
SHC09 六舵行程相同，不存在该差异。

## 6. 饱和与残差

归一化分配后按 `u ∈ [−1,1]` 裁剪（电机 `[0,1]`）：

```text
u_c = clip(u, u_min, u_max)
τ_c = B_u·(u_c − u_trim)                  [N·m]
ν_c = S·τ_c
ν_res = ν − ν_c                           [归一化单位]
τ_res = S⁻¹·ν_res                         [N·m]
```

PX4 的 `control_allocator_status.unallocated_torque` 是 `ν_res`，不是 N·m；
RateControl 只用其符号冻结积分。若要在 SI 界面反馈，必须乘 `S⁻¹`。

SHC09 数值验证（合成气动 fixture，悬停点）：

```text
Ve = 24.230354923 m/s, δ_trim = 0.127587337 rad, δ_max = 0.698131701 rad
B_tau 行满秩 3，cond(B_tau)=2.8606
S = diag(1.165752149, 1.165752149, 1.925297848)   [1/(N·m)]
S⁻¹ = diag(0.857815, 0.857815, 0.519400)         [N·m]
u_trim = 0.182755（六舵）；B_u·u_trim = [0,0,0.094923] N·m
（物理配平偏航力矩；PX4 的 control_trim 通常为 0，配平属于 actuator_trim 基线）
```

绝对/相对配平的权限必须分开：

```text
绝对 B_tau·δ 范围（绕零力矩）:
  roll ±1.400806, pitch ±1.213134, yaw ±0.519400 N·m
相对配平 B_tau·(δ−δ_trim) 范围:
  roll ±1.400806, pitch ±1.213134 N·m
  yaw +0.424477 / −0.614323 N·m
对应 ν = S·τ_rel:
  roll ±1.633, pitch ±1.414, yaw +0.817 / −1.183
```

PX4 的 `control_trim = B·linearization_point` 在默认零线性化点时为 0；
`actuator_trim = trim + linearization_point` 才是执行器基线。`getAllocatedControl`
计算的是 `B·(u−actuator_trim)`，因此 control setpoint 是**相对执行器配平**的
增量，不包含 `B·actuator_trim` 的绝对配平力矩。SHC09 的 `δ_trim` 属于同一
语义：零力矩指令回到 `δ_trim`，但配平偏航力矩 `0.094923 N·m` 仍由对象承担。


对 `τ = [0.05,−0.03,0.01]` N·m：物理与归一化分配舵角误差 `1.4e-17 rad`；
对 `τ=[0.4,0.2,0.05]`：`1.4e-16 rad`；对大扭矩（越限）两者裁剪后的舵角误差
`1.6e-16 rad`，饱和残差一致。详见 `evidence/normalization-probe.txt`。

## 7. 电机（独立一维）

SHC09 单旋翼推力：

```text
T = k_T0·Ω²,  Ω ∈ [Ω_min, Ω_max],  T_max = k_T0·Ω_max²
u_m = T / T_max ∈ [0,1]（可选归一化）
B_rotor = [0; 0; −T_max]（FRD，推力沿 −z）
```

归一化推力接口只需标量 `T` 与 `T_max`，与六舵面 3×6 矩阵物理分离；
不能把 `Ω`、`T` 与六舵角拼成 7 维统一求解器。PX4 的 matrix 0（电机）
与 matrix 1（舵面）独立分配，正是这一边界。

## 8. 动态效能（Ve 变化）

SHC09 `B_tau(Ve)=f²·B_tau0`，`f=Ve/V_e0`。

**固定 S**（按悬停标定）：

```text
ν = S·τ_rel（与 Ve 无关）
u = u_trim + pinv(B_u(Ve))·S⁻¹·S·τ_rel
  = u_trim + pinv(B_u(Ve))·τ_rel
δ = δ_trim + U·pinv(B_u(Ve))·τ_rel
```

物理映射仍精确；`S⁻¹` 恒定，Rate PID 的 N·m/ν 增益不变。

**动态 S**（按 PX4 公式在每个 Ve 重算，`S(Ve)=S0/f²`）：

```text
ν(Ve) = S0/f²·τ_rel = ν0/f²
τ_rel = S(Ve)⁻¹·ν = f²·S0⁻¹·ν
```

同一物理力矩需要的 `ν` 随 `f²` 变化；若 Rate PID 增益仍按 `ν` 固定，
则物理环路增益随 `Ve²` 变化，等价于未声明的增益调度，可能破坏稳定性。
要在动态 S 下保持物理行为，必须同步把 `Kp/Ki/Kd/积分限幅` 乘 `S(Ve)`，
或让控制器继续输出 N·m、仅在分配器内部做尺度换算。

此外，积分状态 `I_ν` 以 ν 单位存储；若 `S` 变化而 `I_ν` 不换算，物理积分力矩
`S⁻¹I_ν` 会在切换瞬间跳变。必须按 `S_new·S_old⁻¹` 重标定，或重置积分并做
无扰切换；否则动态尺度会引入积分 bump，而不仅是增益变化。

PX4 的 `B` 来自配置参数，不随 `Ve` 连续变化，因此其尺度只在配置/故障更新时
重算，不存在该调度问题；SHC09 的 `B_tau∝Ve²` 是额外自由度。

## 9. PID 基线一致性

物理 PID → 归一化 PID 的等价换算（固定 S）：

```text
ν = S·τ
Kp_ν = S·Kp_τ,  Ki_ν = S·Ki_τ,  Kd_ν = S·Kd_τ
I_lim,ν = S·I_lim,τ,  τ_limit,ν = S·τ_limit
```

误差、积分时间、微分（角加速度）都不变；抗饱和符号不变（S 正对角）。
若只改变接口而不换算增益，闭环增益会乘以 `S` 或 `S⁻¹`，属于算法变化，
必须显式记录为“尺度转换”，不能称为“仅接口重构”。

V1.0 建议保持 `Kp_τ` 等物理增益不变（S=I 语义）；若未来引入 ν 接口，
必须同时给出 `S` 与 `Kp_ν` 的换算表，并用旧/新闭环对比验证。

### 9.1 分层记录

| 项目 | PX4 原始实现 | AeroGNC-Sim 当前 | 候选归一化 | 必要适配 |
|---|---|---|---|---|
| Rate 输出 | `ν`（S·τ），无裁剪 | `τ` N·m | `ν=S·τ` | 仅接口/增益换算 |
| P/I/D 数学 | 并行/理想形式，`K` 缩放 | 同一 PID 形式 | 不变 | `K_ν=S·K_τ` |
| 积分状态 | ν 单位，`_lim_int` | N·m，积分限幅 | ν 单位 | `I_lim,ν=S·I_lim,τ` |
| D 反馈 | 角加速度 | 角加速度 | 不变 | 无 |
| 抗饱和 | `unallocated_torque` 符号 | `allocation_status.saturated` | ν 残差符号 | 仅单位换算 |
| Velocity PID | 推力归一化另路 | 加速度/推力物理量 | 不变 | 不耦合 |
| Attitude P | 速率 setpoint | 速率 setpoint | 不变 | 无 |
| Position P | 速度限幅 | 速度限幅 | 不变 | 无 |
| PINV | `M_n=M·S⁻¹` | `pinv(B_tau)` | `pinv(B_u)` | 仅矩阵单位 |

不应修改：PID 的误差定义、积分时间常数、D 的角加速度反馈、抗饱和方向逻辑、
限幅优先级。只允许尺度/单位转换，且必须记录为转换而非算法变化。

