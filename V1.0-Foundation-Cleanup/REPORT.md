# V1.0 Foundation Cleanup — 审查与收敛报告

| 项 | 值 |
|---|---|
| 阶段 | `V1.0-Foundation-Cleanup`（临时分支，不长期维护） |
| 状态 | `ready-for-review`（未合并 main，未删除任何分支） |
| DEV 分支 | `chore/v1-foundation-cleanup` @ `d23e2ff2e423f25aaeae7e5045700f598007e1aa` |
| DEV main | `c8aae919281741a0e0554e78f4a8f085cb0f09e7`（未变） |
| AUDIT | <https://github.com/tangqingxi/AeroGNC-Sim-Audit> |

## 1. REMOVE / REPLACE / KEEP / REVIEW

### REMOVE（已删除）

| 对象 | 原因 |
|---|---|
| `src/Simulation/Events/aero_measurement.m` | 旧测量读取函数；所有使用方已迁移到 Topic `tryRead` |
| `bus.measurements` 运行时共享数组 | 旧测量广播通道；测量统一由 Topic Bus 承载，避免第二套运行时状态 |
| `cfg.communication.enabled` 及所有双模式分支 | V1.0 始终使用唯一 Bus，不再保留关闭路径 |
| 快照中的 `enabled` 字段 | 恒为真的模式标志，单一模式下无意义 |
| Sensor 的 `topic` 可省略语义 | V1.0 要求每个 sensor 声明 Topic 契约 |
| SHC09 气动的“预构建 `aero` 结构”加载分支 | 只保留 `duct_raw`/`wing_raw` 原始表格式，减少未使用兼容入口 |
| `tests/fixtures/shc09_aero_fixture.m` 的 `is_physical` | 已废弃字段；可信等级统一用 `trust` + `physical_validation_allowed` |
| 测试中的 `bus.nav`、`bus.guidance` 共享链 | 改为 Topic 传递；`bus` 只保留锁存命令、真值与 `topics` |

### REPLACE（先迁移后删除）

- 测量延迟、同刻事件、可复现性测试：`aero_measurement(bus,...)` → `bus.topics.tryRead(...)`；
  sensor 在可用事件发布，测试链 `sensor → nav → guidance → control → latch` 全部经 Topic。
- 配置：`enabled` 开关 → 单一 `topics`/`command_topic`/`held_command_topic`/`log`。
- 文档：README/DESIGN/AGENTS/ARCHITECTURE 更新为 V1.0 单一 Bus 与版本/分支政策。

### KEEP（保留）

- `result.measurements`：测量的**唯一权威记录**（内核在发布事件追加），与 Topic 快照职责分离。
- `command` / `held_command` 的计算/锁存职责与独立 Topic 映射；锁存语义未改。
- 有界 Topic 日志（默认关闭、仅分析用途，不是第二份权威数据）。
- SHC09 几何、效能矩阵、动力学、积分、执行器约束与全部数值回归。
- `tests/fixtures/` 合成气动与解析模型（明确标注，不可用于物理验证）。
- `aero_empty_component`（组件 schema 文档）、`bus.truth` 隔离字段。

### REVIEW（已核实并保留）

| 对象 | 结论 |
|---|---|
| `result.schema = 'aero-sim-result/1'` | 结果字段没有删除，仅新增/统一 communication，保持 /1 |
| `aero_command_payload` | 仍被命令 Topic 映射使用，保留 |
| `aero_is_valid_command` | 仍被内核与测试使用，保留 |
| SHC09 外部气动 `trust` 字段 | 与 V1.0 可信等级契约一致，保留 |

未发现来源不明、无调用或需要猜测删除的对象。

## 2. 变更文件

- 删除：`src/Simulation/Events/aero_measurement.m`。
- 修改：`aero_sim.m`、`aero_communication_setup.m`、`aero_communication_snapshot.m`、
  `aero_default_config.m`、`aero_validate_config.m`、`shc09_aero_data.m`、
  `test_project_layout.m` 及 6 个受影响测试、fixture。
- 文档：`README.md`、`DESIGN.md`（§14）、`AGENTS.md`（§10）、`ARCHITECTURE.md`（§13）。

## 3. V1.0 行为变化（有意破坏兼容）

1. `cfg.communication.enabled` 会被明确拒绝（`aero:config:communication`）。
2. 每个 sensor 必须声明 `topic`，且 Topic 必须已注册。
3. 测量不再出现在 `bus.measurements`；消费方必须使用 `bus.topics.read/tryRead`。
4. `result.communication` 在每次仿真中都存在（普通值快照），不再有“无线通信字段”模式。

以上均属于“删除过渡路径”，不改变 SHC09 物理与锁存语义。

## 4. 验证结果

从 GitHub **全新 clone** 分支 @ `d23e2ff`：

```text
matlab -batch "run('tests/run_tests.m')"  →  27/27 通过，7.41 s，退出码 0
```

新增静态检查（`test_project_layout`）：

- `which('aero_measurement')` 必须为空；
- `src/` 中不得出现 `bus.measurements`、`aero_measurement`、`is_physical`；
- 默认配置不得含 `communication.enabled`。

覆盖能力保持：多速率/延迟/同刻顺序、命令与锁存独立、SHC09 动力学/几何/执行器、结果持久化、
Topic 日志与 MAT 保存、唯一 Bus 与冲突拒绝。

## 5. Git 分支清理清单（验收后执行）

| 分支 | 状态 | 处置 |
|---|---|---|
| `feat/communication-baseline` | 与 main 同 SHA，0 独有提交 | 验收合并后删除 |
| `feat/shc09-aircraft` | main 祖先，0 独有提交 | 验收合并后删除 |
| `chore/v1-foundation-cleanup` | 1 个独有提交（本清理） | 合并并核对后删除 |

本地仅有单一 worktree；无未推送提交；不删除 main、AUDIT 历史、Tag 或未合并工作。

## 6. 遗留限制（非阻断）

- Topic 日志是有界分析工具，不是历史队列；逐样本消费/回放未实现。
- `truth/` 命名仍不是权限系统；真实隔离依赖 Simulation 边界与模块纪律。
- 快照 `.mat` 持久化仅在 MATLAB R2025b 验证。
- 真实气动数据许可、物理有效性与实机参数辨识仍未完成。

## 7. 建议

**建议合并 main**（fast-forward），随后删除两个已合并 `feat/*` 分支与临时 `chore/*` 分支；
首次 V1.0 发布时再建立 `v1.0.0` Tag。未启动 Rate-Control-Baseline。
