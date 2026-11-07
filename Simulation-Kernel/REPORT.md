# Simulation-Kernel — 阶段报告

| 项 | 值 |
|---|---|
| 阶段名称 | `Simulation-Kernel` |
| 状态 | `ready-for-review`（等待 NEXUS 验收；不是 ACCEPT） |
| 报告时间 | 2026-11-07 20:40 +08:00 |
| DEV | <https://github.com/tangqingxi/AeroGNC-Sim> |
| DEV 分支 / HEAD | `main` / `2d9b9bffad6051fe6ceb66cf489ad85f175fe7fc`（已推送） |
| AUDIT | <https://github.com/tangqingxi/AeroGNC-Sim-Audit> |
| 起点 | DEV `29f9951c3c98b4dc21b9271e4bf63db3e02f18e8`（`Architecture-Baseline`，已由 NEXUS ACCEPT） |
| 环境 | MATLAB R2025b `25.2.0.2998904`，PCWIN64；Node/Python 未参与运行 |

## 1. 完成内容（Gate A–F）

### Gate A — 基线收尾与环境预检（PASS）

- 核对：DEV `main` = `29f9951…`（与已验收版本一致）；AUDIT `main` = `1f0718a…`；两仓工作区干净。
- `Architecture-Baseline` 状态由 `in-progress` 更新为 `accepted`，记录 NEXUS 于 2026-11-07 的判定（AUDIT 提交 `05fce56`）；DEV 技术基线未改动。
- 环境：`matlab -batch` 可用；`ode45(@(t,y)-2*y,[0 1],1)` 返回 41 个内部点、退出码 0（0.29 s）。
- 工具箱：本机安装了 111 个产品，但内核**只依赖 MATLAB 基础函数**（`ode45/odeset/save/dir`），未引入任何工具箱函数；零额外依赖成立。

### Gate B — 最小工程骨架（PASS）

新增 `src/`（内核）、`models/`（解析测试替身）、`tests/`、`examples/`、`README.md`、`.gitignore`。核心入口 `aero_sim(model, cfg)`：

- `model.dynamics = @(t,x,u,env)`，状态维度由 `x0` 决定，内核不出现 6DOF 或任何机型下标。
- `cfg` 覆盖时间区间、初态、组件、求解器容差、环境、种子、结果保存；`aero_validate_config` 对非法输入直接报 `aero:*` 错误，不静默修补（14 类非法输入均有回归测试）。
- `matlab -batch` 可直接运行示例与测试。

### Gate C — 分段 `ode45` 与输入锁存（PASS）

- 每个事件边界之间单独调用 `ode45`，上一段末状态作为下一段初值；段内 `held_command` 恒定（零阶保持）。
- 记录每段 `t0/t1/held/internal_points`；`ode45` 内部点与事件/采样/日志时间点分开保存。
- 解析验证：`dy/dt = u`（阶跃锁存）最大误差 `2.22e-16`；`dx/dt = -a x + b u` 最大误差 `1.38e-11`（阈值 `1e-9`）。
- 自适应证据：振荡段 0.5 s 内 197 个内部点（收紧容差后 813 个），步长非均匀，重复调用逐位一致。

### Gate D — 多速率与因果时序（PASS）

- **不同频率**：125/200/800 Hz 配置分别产生 125/200/800 次事件，`max|t_k − k·T| = 0`。
- **非整倍数**：73/127 Hz 在 5 s 内产生 365/635 次事件，漂移 `8.9e-16`，无固定网格取整。
- **同时到期**：同一时刻按 `truth sample → publish → nav → guidance → control → latch` 执行（优先级 10/11/20/30/40/50），同实例传播验证通过；两次运行事件表完全一致。
- **测量延迟**：传感器 20 ms 采样、50 ms 可用延迟，控制 100 Hz；控制器只看到 `available_time ≤ t` 的最新样本，8 条测量全部满足 `available = sample + 50 ms`。
- **锁存与物理响应**：控制 100 Hz、锁存 40 Hz，锁存值只在 40 Hz 时刻改变，且取该时刻最新计算值（0.075 s 处取 0.07 s 的命令）。

### Gate E — 标准结果与 Results 生命周期（PASS）

`SimulationResult` 区分：边界时间与状态（真值）、事件表（组件/动作/优先级）、测量（`sample_time`/`available_time`）、计算命令与实际锁存命令、分段积分信息、求解器设置、生效配置、模型名、种子、MATLAB 版本与 Git 提交。

**Results 容量治理**（`src/aero_results_policy.m` 等）：

- 上限 `10e9` 字节（10 GB，按十进制，较 10 GiB 保守），预警 `8e9`；默认只驻留内存，写盘需显式 `cfg.save.enabled` 且路径必须位于 results 根目录（默认 `<repo>/results`，已 `.gitignore`）。
- 先写临时文件，再按"现有用量 + 新文件"判定；超限即删除临时文件并报 `aero:results:limit`，**不删除、不覆盖已有证据**。
- 文件数预算 5,000（临时建议值）：≥4,000 提示复核，≥5,000 仍只告警，不阻断。
- 不向 AUDIT 复制原始数据；证据只含本报告、测试摘要与清单。

### Gate F — 自动化回归（PASS）

`matlab -batch "run('tests/run_tests.m')"` 从干净提交运行：**13/13 通过，1.67 s，退出码 0**（证据 `evidence/matlab-test-run.txt`）。

## 2. 验证记录

### 2.1 验收对照

| 验收项 | 结果 | 依据 |
|---|---|---|
| 通用内核真实可运行 | PASS | `examples/demo_first_order.m`，`tests/run_tests.m` |
| 不需要 SHC09 | PASS | 5 个解析模型/场景，无任何机型代码 |
| 分段 `ode45` + 零阶保持数值验证 | PASS | 2.2 表第 1–3 行 |
| 多速率时序验证 | PASS | 2.3 第 1–2 行 |
| 同时事件与可用时刻因果 | PASS | 2.3 第 3–5 行 |
| 日志区分状态/事件/命令/锁存 | PASS | `time,state,events,measurements,computed,held` |
| Results 10 GB 规则 | PASS（见 2.4 范围说明） | 常量断言 + 缩放阈值行为测试 |
| 必要测试全部通过 | PASS | 13/13 |
| `ARCHITECTURE.md` 未变 | PASS | SHA256 `7A17A41B…5949`（提交前后一致） |
| DEV/AUDIT 提交、报告、Handoff 一致 | PASS | `check_handoff.py`（结果见交付消息） |

### 2.2 数值一致性（默认 `RelTol=1e-9`，`AbsTol=1e-11`，阈值在实现前设定）

| 测试 | 解析结果 | 数值结果 | 最大绝对误差 | 阈值 | 结果 |
|---|---|---|---|---|---|
| 单状态 `dy/dt=u`，0.5 s 处 u: 1→−2 | y(1) = −0.5 | −0.5 | 2.22e-16 | 1e-9 | PASS |
| 一阶 `dx/dt=−2x+3u`，0.2 s 阶跃 | x(1) = 1.197155 | 1.197155 | 1.38e-11 | 1e-9 | PASS |
| 三状态对角模型，u=1，0.5 s | x_i=(1−e^{−λ_i t})/λ_i | 同左 | 1.50e-11 | 1e-9 | PASS |
| 自适应振荡段（`RelTol=1e-6`） | 8(cos25−1) = −7.931 | −7.931 | 7.35e-6 | 1e-5 | PASS |

容差未因失败而放宽；`test_solver_tolerance` 证明 `RelTol=1e-6` 与 `1e-10` 下事件时刻逐位相同（状态差 1.20e-11）。

### 2.3 时序与因果

| 检查 | 结果 |
|---|---|
| 125/200/800 Hz 事件计数与漂移 | 125/200/800，漂移 0 |
| 73/127 Hz 5 s 计数与漂移 | 365/635，漂移 8.9e-16 |
| 同时事件顺序 | `sample→publish→nav→guidance→control→latch`，两次运行一致 |
| 测量延迟泄漏 | 8 条测量，控制从未使用 `available_time > t` 的样本 |
| 指令锁存 | 10 次控制、4 次锁存、3 次锁存值变化，值等于锁存时刻最新命令 |
| 分段边界 | 153 段，全部事件落在边界，段链连续 |

### 2.4 Results 策略验证范围（诚实说明）

- 默认常量在测试中断言：`10e9 / 8e9 / 5000 / 4000`。
- 行为用**缩放阈值**验证（1000/800 字节、10/8 个文件）：预警、超限拒绝、临时文件清理、根目录外拒绝、内存运行不落盘、文件数 80% 提示，全部为真实代码路径。
- **未**实际写满 8 GB/10 GB 数据（会污染工作区且无必要）；因此"10 GB 上限"的结论是"常量 + 同一代码路径的行为已验证"，不是大规模实测。未实现的保护没有被报告为 PASS。

### 2.5 复现

```bat
git clone https://github.com/tangqingxi/AeroGNC-Sim
cd AeroGNC-Sim
matlab -batch "run('tests/run_tests.m')"
matlab -batch "run('examples/demo_first_order.m')"
```

## 3. 未解决问题与风险

1. **同时事件分组容差**：相差小于 `1e-12·max(1,tf)` 的事件会被并入同一时刻执行；这是浮点合并策略，不改变声明的组件频率，但把"更近"的两事件视为同一物理时刻。
2. **半开区间**：组件更新发生在 `[t0, tf)`，`tf` 只作为积分终点；若未来的验收要求 `tf` 时刻也触发组件，需要显式修改并记录。
3. **measurement.available_time** 取发布执行时刻；分组可能使其与 `sample_time + delay` 相差不超过分组容差（测试中为 0）。
4. **配置含函数句柄**：`result.config` 保存组件句柄，跨 MATLAB 版本加载 `.mat` 时句柄可能不可用；结果数值本身不受影响。
5. **未做大规模容量实测**（见 2.4）。
6. **环境**：仅在 R2025b/Windows 验证；未声明其他版本兼容性。

## 4. 相关提交与文件

| 仓库 | 内容 |
|---|---|
| DEV | `2d9b9bffad6051fe6ceb66cf489ad85f175fe7fc`（37 个文件：`src/`、`models/`、`tests/`、`examples/`、`README.md`、`.gitignore`） |
| DEV 历史 | `29f9951` → `2d9b9bf`；`ARCHITECTURE.md` 与 `AGENTS.md` 未改 |
| AUDIT | `05fce56`（Gate A 验收记录）、本提交（报告 + 证据 + 交接） |

## 5. 下一阶段建议

1. NEXUS 审核本报告与 `evidence/matlab-test-run.txt`，决定 `Simulation-Kernel` 是否 ACCEPT。
2. 若 ACCEPT，再按 `ARCHITECTURE.md` §9 进入 `SHC09-Aircraft`，首个最小切片建议只做 **Aircraft 接口与 NED/FRD 边界适配契约**（不含气动实现），先以解析机型验证接口可替换性。
3. 若要求修改本阶段行为（例如 `tf` 时刻触发组件、或不同的分组规则），应作为本阶段的复审项，而不是直接进入下一阶段。
