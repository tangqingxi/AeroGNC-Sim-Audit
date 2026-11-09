# SHC09-Aircraft — 阶段报告（Gate A：物理来源审计）

| 项 | 值 |
|---|---|
| 阶段名称 | `SHC09-Aircraft` |
| 状态 | `ready-for-review`（2026-11-09 解冻并完成实现；Gate A 曾因 D1–D3 blocked，历史记录见 §1–§10） |
| DEV | <https://github.com/tangqingxi/AeroGNC-Sim> |
| DEV HEAD | `848c0710ccc553047ad20a2528a45a42e2a2f458`（审计时的 main） |
| DEV 开发分支 | `feat/shc09-aircraft` @ `c53d5601c1a51319a8733526419fedb34ce2c045`（已推送，未合并 main） |
| AUDIT | <https://github.com/tangqingxi/AeroGNC-Sim-Audit> |
| 参考工程 | <https://github.com/tangqingxi/Ducted-Fan-SHC09-matlab> @ `43b6c7aef1912f8118e54d3decac1ac55465d5b1`（只读） |
| 审计日期 | 2026-11-07 |

## 1. 审计范围与方法

- 将参考工程浅克隆到临时目录并检出上述 commit，只读检查任务授权的文件；未修改参考仓库、未复制源码到 DEV/AUDIT。
- 逐行阅读方程实现，不依据函数名推断物理含义。
- 用 `physical_reference.json` 记录的 SHA256 复算 8 个参考文件：**8/8 全部 MATCH**（见 `evidence/reference-provenance.txt`）。
- 气动表实测形状与角度域：`duct_raw.mat` 20×4、0～130°；`wing_raw.mat` 22×4、−20～130°。
- 结论：Gate A 的方程与参数可追溯，但 **惯量、气动数据许可、舵面通道映射**三项无法在现有授权资料内闭合，按指令停止 Gate B–F。

## 2. 方程清单（对应实现位置）

参考实现为 12 状态欧拉角模型，状态 `x = [p_n(3); v_n(3); euler(3); ω_b(3)]`，机体系为旧原生系（前向 y、轴向 z、侧向 x）。

| 物理项 | 参考位置 | 实现要点 |
|---|---|---|
| NED 位置运动学 | `SHC09_Sys.m:140` | `ṙ_n = v_n` |
| NED 平动动力学 | `SHC09_Sys.m:142` | `v̇_n = R_nb' F_b/m + F_ext,n/m + [0;0;g]` |
| 姿态运动学（欧拉） | `SHC09_Sys.m:74-77, 143` | `Q(φ,θ)` 把 `ω_b` 映射为欧拉角速率 |
| 机体系角速度动力学 | `SHC09_Sys.m:144` | `ω̇_b = J⁻¹(M_b − ω_b×(Jω_b))` |
| 涵道风扇力/力矩 | `Ducted_Fan_FnM.m:44-51`；`SHC09_Sys.m:111,121-124,130-131` | 拉力 `y1`、侧向力 `y2`、俯仰力矩 `y3`、风扇扭矩 `y4` |
| 出口风速 | `SHC09_Sys.m:120` | `Ve = −w/2 + sqrt((w/2)² + T/(sd·ρ·S))` |
| 机翼气动 | `Wing_FnM.m:20-22`；`SHC09_Sys.m:115,122-123,130` | 升力、阻力、绕 x 轴力矩，均为 `V_vw²·ppval(spline, α)` |
| 六舵面力矩 | `SHC09_Sys.m:133`；`P_cs` 见 `Initialized_SHC09.m:39-41` | `M_cs = d_cs·Ve²·diag([l1,l1,l2])·P_cs·u_cs` |
| 风扇反扭矩 | `SHC09_Sys.m:127` | `M_fan = −Dir·k_Q0·Ω²` |
| 固定片反扭矩 | `SHC09_Sys.m:128` | `M_sta = Dir·k_sta·Ve²` |
| 风扇陀螺力矩 | `SHC09_Sys.m:129` | `M_gyro = −Dir·J_fan·Ω·[q;−p;0]` |
| 非轴流俯仰阻尼 | `SHC09_Sys.m:126,130` | `M_my = −p·k_my0 − p·k_myv·V_vw` |
| 风、外力、外力矩 | `SHC09_Sys.m:47-55,79,135,142` | 风速在 NED 减后转机体系；外力在 NED；外力矩在机体系 |

两种迎角定义不同，必须分开保留：`SHC09_Sys.m:96-100` 用 `acos(−w/V_vw)`（涵道/机体迎角，`V_vw=|(v,w)|`），`Ducted_Fan_FnM.m:33-38` 用 `acos(−w/|V_a|)`（涵道风扇迎角，用总空速）。

## 3. 参数来源审计

| 参数 | 数值 | 单位 | 来源 | 可信等级 |
|---|---:|---|---|---|
| 质量 m | 2.1 | kg | `Initialized_SHC09.m:27` | 旧模型记录值，非实测 |
| 转动惯量 J | 见 §4 | kg·m² | 两处冲突记录 | **未解决** |
| 风扇惯量 J_fan | 3.7e-5 | kg·m² | `Initialized_SHC09.m:29` | 旧模型记录值 |
| 桨盘面积 S | π·0.114² = 0.040828 | m² | `Initialized_SHC09.m:26` | 旧模型记录值 |
| 扩压比 sd | 0.7 | — | `Initialized_SHC09.m:31` | 旧模型记录值 |
| 风扇方向 FanDir | +1（CW） | — | `Initialized_SHC09.m:32` | 旧模型记录值 |
| 舵面系数 d_cs | 0.0032 | — | `Initialized_SHC09.m:33` | 旧模型记录值 |
| 固定片系数 k_sta | 5.3688e-4 | — | `Initialized_SHC09.m:34` | 旧模型记录值 |
| 力臂 l1 / l2 | 0.267 / 0.066 | m | `Initialized_SHC09.m:35-36` | 旧模型记录值 |
| 涵道系数 k_T0,k_Th,k_Ts,k_Ns,k_Q0,l_cpz,k_cpx | 2.8964e-5, −3.7614e-4, −2.6733e-4, 5.0699e-4, 5.7792e-7, −0.15, 2.6619 | SI | `Initialized_SHC09.m:13-21` | 旧模型记录值 |
| 非轴流阻尼 k_my0,k_myv | 0.0198, 0.0015 | — | `Initialized_SHC09.m:47-48` | 旧模型记录值 |
| 重力 g / 密度 ρ | 9.788 / 1.225 | m/s² / kg·m³ | `ca_shc09_parameters.m:46` | 参考环境值 |
| 舵面机械限幅 | ±40 | ° | 本轮任务给出的硬件配置 | 硬件确认值 |
| PWM 输出更新 | 200 | Hz | 本轮任务给出的硬件配置 | 硬件确认值 |
| 舵机（KST X08H, 8 V） | — | — | 任务给出型号 | 硬件确认型号；动态参数未辨识 |
| 电机（MAD M6C06 320KV V2） | — | — | 任务给出型号 | 硬件确认型号；`Ω(ω)` 未辨识 |
| 舵机时间常数 | 0.03 | s | `configs/params/ca_config.m:29-30` | **仿真假设**（文件自注“需要通过实测替换”） |
| 舵机速率限制 | 600 | °/s | `ca_config.m:32` | **仿真假设** |
| 旧 PWM 频率 | 333 | Hz | `ca_config.m:26` | 旧仿真设置，低于硬件 200 Hz，不可作为装配常量 |
| 旧舵面饱和 | 28 | ° | `ca_config.m:11` | 控制器饱和设置，不是机械行程 |
| 电机时间常数 | 0.05 | s | `ca_config.m:16` | **仿真假设** |
| 电机转速范围 | 200～1200 | rad/s | `Initialized_SHC09.m:44-45` | 旧模型假设 |
| 悬停转速 / 出口速度 | ≈842.42 / 24.2304 | rad/s / m/s | `docs/reference/physical_model_audit.md:22` | 由旧参数推导，非实测 |

硬件确认值、旧模型记录值、仿真假设已按上表分列；新模型实现时不得把它们混为同一可信等级。

## 4. 坐标关系与惯量冲突（A3，**未解决**）

### 4.1 坐标关系

- 旧原生系：x 侧向（左）、y 前向、z 轴向（下）；不是 FRD。
- 中间 `b` 系：`b1 = FRD y`（右）、`b2 = −FRD x`、`b3 = FRD z`，`R_old_to_b = diag([-1,-1,1])`（`ca_shc09_parameters.m:18`）。
- 任务给定 `C_old_to_frd = [0 1 0; −1 0 0; 0 0 1]`，`v_frd = C·v_old`。
- 惯量变换 `J_frd = C·J_old·Cᵀ`；姿态 `R_ned_frd = R_ned_old·Cᵀ`。

### 4.2 两个候选

| 候选 | 记录 | 所处坐标系 | 换算到 FRD |
|---|---|---|---|
| A | `diag(0.0438, 0.0436, 0.005006)`（`Initialized_SHC09.m:28`） | 旧原生系（x 侧向） | `diag(0.0436, 0.0438, 0.005006)` |
| B | `J_b = diag(0.0436, 0.0438, 0.005006)`（`ca_shc09_parameters.m:20-24`） | 中间 `b` 系 | `diag(0.0438, 0.0436, 0.005006)` |

两者仅前两轴互换，差 0.0002 kg·m²（约 0.46%），但决定滚转/俯仰通道。

### 4.3 证据评估

- 旧仓库参数文件注释称“PX4/SDF 的惯量按 FRD 给出：Jx=0.0438, Jy=0.0436”，据此把 B 作为修正值；但**参考仓库内没有任何 SDF/CAD 文件**（`**/*.sdf` 命中 0），`apps/windtrace/assets/geometry_source.json` 还明确写着“repository has no airframe geometry”，且该文件只列出 `[0.0438, 0.0436, 0.005006]` 而**未标注坐标系**。
- 旧仓库物理审计文档 `physical_model_audit.md:14` 仍把惯量记为 `diag(0.0438,0.0436,0.005006)` 且称新旧对象“相同”，与参数文件的修正说法**互相矛盾**。
- `test_b_frame_dynamics_equivalence.m:19-29` 把 A 称为“旧代码实际使用的轴序”，把 B 称为“正确”，但这是代码内部断言，不是独立的实机或 CAD 证据。
- 因此：**没有带坐标系标注的一手来源可以判定 A 或 B**。按指令不得猜测，Gate A 在此停止。

## 5. 气动数据与再分发许可（A4，**未解决**）

- `duct_raw.mat` / `wing_raw.mat` 与 5 个参考 `.m` 均来自 `ICGNC_matlab2.0`，SHA256 与 `docs/physical_reference.json` 全部一致。
- 参考仓库 `LICENSE` 为 BSD-3-Clause，但 `THIRD_PARTY_NOTICES.md` 明确：
  - “`models/shc09/reference`及气动表来自 ICGNC_matlab2.0 参考工程”；
  - “来源尚未明确公开许可的参考文件，不因新增本许可证而视为已获得 BSD-3-Clause 授权”。
- 参考仓库内没有 ICGNC 的许可文件；外部检索也未找到 ICGNC_matlab2.0 的公开许可声明。
- 新 DEV 仓库为 **Public**，把这两张表纳入即构成再分发，权利基础不成立。

**结论**：气动表再分发权限不明确，停止数据入库；不用合成或虚构表替代（会破坏物理可追溯性）。

## 6. 六舵面通道映射（相关 Stop Condition）

- `P_cs` 是 3×6 几何/符号矩阵（`Initialized_SHC09.m:39-41`），列顺序只是参考定义。
- 旧仓库 `ca_shc09_parameters.m:30` 声称“列仍为真实舵面 1-6”，但同一文件把铰链轴向坐标标为 `NaN`，状态为 `"unknown_no_sdf_cad_evidence"`（第 33-39 行），且仓库无 CAD/SDF。
- 因此“`P_cs` 第 i 列 = 实物第 i 号舵机通道”**没有可核对证据**。若后续继续实现，必须把映射做成显式配置并标注“未确认”；当前不自行猜测。

## 7. 未完成的工作

Gate B–F 全部未开始：未创建 `src/Aircraft/SHC09`、`src/Components/Actuators`、`tests/aircraft`、`examples/shc09`，未写任何 MATLAB 代码，DEV 无新提交（HEAD 仍为 `848c071`）。

原因：三个阻断项都直接影响“物理正确性 > 坐标正确性 > 数值正确性”的最高优先级；先写代码再返工违背本阶段原则。

## 8. 需要 NEXUS 的决策

| 编号 | 事项 | 可接受的处理方式 | 未决影响 |
|---|---|---|---|
| D1 | 惯量 A/B 二选一 | (a) 提供带坐标系标注的 PX4/SDF 或 CAD 原始文件 → 采用 B；(b) 确认参考初始化值即旧体系表达 → 采用 A；(c) 提供实测/称重惯量 | 无法写转动动力学与配平 |
| D2 | 气动表再分发 | (a) 确认你方拥有 ICGNC_matlab2.0 数据版权并书面授权以 BSD-3-Clause 再发布；(b) 提供第三方许可；(c) 改为私有仓库（需另行授权）；(d) 数据不入库、运行时外置路径（会削弱可复现与自动测试）；(e) 改用有明确许可的替代数据（不得合成） | 无法实现涵道/机翼气动与验证 |
| D3 | 舵面通道映射 | (a) 提供装配图/CAD/实测通道对应；(b) 授权以“显式未确认映射”实现 | 单舵面方向与分配矩阵无法定版 |

## 9. 下一阶段建议

1. 先由 NEXUS 裁决 D1–D3；在 D1、D2 闭合前不进入 Gate B。
2. D1 若选 (a)，请同时提供 SDF 中的质量、惯量与安装几何；若选 (c)，请给出测量方法与不确定度。
3. D2 若选 (a)/(b)，请把授权文本写入项目 `THIRD_PARTY_NOTICES.md` 并在 AUDIT 留档；若选 (d)，需明确“干净 clone 下测试不可运行”的验收口径。
4. 三项闭合后，Gate B 的最小切片建议：先做坐标与状态接口 + 空动力学占位测试，再逐项接入气动与执行器。

## 10. 相关提交与文件

| 仓库 | 内容 |
|---|---|
| DEV | 无新提交，HEAD `848c0710ccc553047ad20a2528a45a42e2a2f458` |
| AUDIT | 本报告 + `HANDOFF.md` + `evidence/reference-provenance.txt` + 证据清单 |
| 参考（只读） | `43b6c7aef1912f8118e54d3decac1ac55465d5b1`，审计后临时克隆已删除 |

## 11. 2026-11-09 更新：决策落地与 SHC09 实现

### 11.1 三项阻断的处置

| 编号 | 处置 |
|---|---|
| D1 惯量 | NEXUS 冻结 `J_frd = diag([0.0438, 0.0436, 0.005006]) kg·m²`，标记为 Gazebo/PX4 FRD 参考值，未实机辨识；不再并行比较 Gate A 的两个候选 |
| D3 舵面通道 | 冻结 CS1–CS6 方位角 0–300°、`r_i`、`a_i` 与 `B_M`；实机接线编号与之一致 |
| D2 气动数据 | 许可仍未闭合：真实表不入库，改为外部数据接口 `shc09_aero_data`；缺失时显式失败；物理气动验证列为外部依赖条件 |

### 11.2 实现内容

- 内核（最小通用扩展）：`initial_command` 支持“字段均为有限实数的结构体”，内核只做通用校验、锁存与日志，
  不识别 `servos` / `motor` 字段名；新增可选 `model.project(x)`，用于四元数归一化与执行器限幅。
- Components：`aero_servo_dynamics` / `aero_motor_dynamics` 通用一阶动力学（限幅、速率/行程、转速范围），
  设备型号 `ServoDevices.KST_X08H`、`MotorDevices.MAD_M6C06_320KV_V2` 仅含设备参数与可信等级。
- SHC09：20 状态（位置、速度、四元数、角速度、六舵偏角、旋翼转速）；涵道/机翼/六舵面/陀螺/反扭矩/风扰；
  悬停配平；标准输出；FRD/NED 边界；气动表越界默认报错、可选显式钳位。
- 指令：`command.servos`（6×1）与 `command.motor`（标量）两条独立信号，可不同频率锁存；更新一条不覆盖另一条。
- 文档：根目录 `DESIGN.md`（长期设计）；README 精简为使用者文档；AGENTS 增加 DESIGN 阅读要求与目录职责。

### 11.3 验证结果

- **回归**：从 GitHub 全新 clone `feat/shc09-aircraft` 后 `matlab -batch "run('tests/run_tests.m')"`
  → **19/19 通过，5.05 s，退出码 0**（14 项内核测试全部保留）。
- **悬停配平**：Ω = 842.42 rad/s，Ve = 24.230 m/s，δ = 7.310°（与参考 7.3102° 一致）；
  配平平动/角加速度 < 1e-9。
- **与旧方程数值对照**（一次性、参考工程只读、30 组随机状态，修正惯量、零舵偏）：
  平动加速度最大差 `3.553e-15 m/s²`、角加速度 `2.109e-15 rad/s²`、出口流速 `3.553e-15 m/s`。
- **几何**：`B_M` 由 `r_i × f_i` 导出并与冻结矩阵一致；`rank(B_M)=3`；`B_M B_Mᵀ` 非对角为 0；
  单舵正偏角方向正确；力矩随 `Ve²` 缩放。
- **执行器**：一阶解析误差 4.4e-12（舵机）/ 3.2e-9（电机）；速率、行程、转速限幅有效；换参数不改通用动力学。
- **指令独立性**：0.2 s 只更新 a 路、b 路保持不变；0.5 s 只更新 b 路、a 路保持；数值命令接口回归通过。
- **集成**：SHC09 由现有 `aero_sim` 积分 200 段；配平段位置漂移 < 1e-6 m；四元数范数保持；
  周期组件始终读不到真值（`bus.truth` 为空）。

### 11.4 仍未解决 / 外部依赖

1. **气动原始表许可（高）**：真实 `duct_raw` / `wing_raw` 仍不能入库；因此机翼升阻力、带真实表的配平与气动性能
   **尚未做物理验证**，不能报告为 PASS。结构测试使用明确标注的全零合成系数，只验证通路与符号。
2. **实机参数辨识（中）**：惯量为参考值；舵机 τ/速率、电机 τ/转速范围为名义值。
3. **舵面侧向力未计入总平动力（低）**：与参考模型一致，本阶段未新增该物理项。
4. **分支未合并**：成果位于 `feat/shc09-aircraft`，需 NEXUS 审核后再合并 main。

### 11.5 提交

| 仓库 | 内容 |
|---|---|
| DEV | 分支 `feat/shc09-aircraft`：`77c5a94` 内核命令容器与投影；`7e9c832` Components；`e6eae94` SHC09 模型；`4c43399` 测试与示例；`c53d560` 文档（HEAD） |
| AUDIT | 本节 + `HANDOFF.md` + `evidence/shc09-test-run.txt` + `evidence/physics-crosscheck.txt` + 证据清单 |

目录审查：无重复职责目录、无平行 Simulation、无空目录、无 `.mat` 或缓存入库、未递归 `addpath`；
SHC09 专属参数只出现在 `src/Aircraft/SHC09/`，通用 Components 中无品牌判断。

### 11.6 下一步建议

1. NEXUS 审核 `feat/shc09-aircraft`（19 项测试 + 物理对照），决定 ACCEPT / REJECT。
2. 若 ACCEPT：合并 main，并单独决定 D2——取得气动表再分发授权（或保持外部数据），再补“带真实表的气动/配平验证”。
3. 在此之前不进入 `GNC-Baseline`。
