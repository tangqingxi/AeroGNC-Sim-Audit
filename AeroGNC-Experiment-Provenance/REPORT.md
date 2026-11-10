# AeroGNC-Experiment-Provenance — 阶段报告

| 项 | 值 |
|---|---|
| 阶段 | `AeroGNC-Experiment-Provenance` |
| 状态 | `in-progress (ready-for-review)`（未合并 main，未创建 Tag） |
| DEV 分支 | `feat/px4-control-fidelity` @ `4f6f7546e597dce7f22edb213e48b42dd102561f` |
| DEV 基点 | `1765ac1cf57a842e75eaaf2d81cbd8252d1687d9` |
| DEV main | `57bbe6893d83808e3b46aa79864701b32b31c775`（未变） |
| AUDIT 分支 | `study/aerognc-experiment-provenance`（自 `77283910a89c234c32631ae3739bb97d8531c3ba`） |
| 测试 | 40/40 通过（全新 clone，40.99 s） |
| 物理验证 | **BLOCKED**（合成气动 fixture） |

## 1. 本阶段目标与结果

本阶段消除 Scenario 构造、运行回调、签名与 WTR 导出的多源参数管理，建立单一实验定义：

```text
Scenario Options
      ↓
Experiment Definition（cfg.experiment）
      ↓
Controller Parameters（同一 definition）
      ↓
Component Callbacks
      ↓
Simulation
      ↓
result.config.experiment + result.config + result.model_metadata
      ↓
simulation.mat
      ↓
WTR（只读结果）
      ↓
flight.wtr
```

完成内容：

1. `cfg.experiment` 由 `experiment_definition.m` 构造，包含 schema、scenario、
   method、active_modules、guidance、navigation、controllers、allocation、
   rates、最终 components。
2. Position/Velocity/Attitude/Rate/Allocation 的运行时参数全部从同一
   `definition` 派生；不再存在 `s.gains`/`s.signature`。
3. `definition_sha256` 保护定义；`identity_sha256` 保护定义+运行配置+模型快照。
4. `result.config.experiment` 原样保存；`result.model_metadata` 增加 trim 快照。
5. `wtr_export` 不再接受 scenario/signature/gains/method/seed；所有身份与参数从
   `result` 派生，且在写文件前校验。
6. `parameters.json` 从 definition、`result.config`、`result.model_metadata` 生成。
7. 新增 `test_experiment_provenance`；原有 39 项测试全部保留，共 40 项。

## 2. 唯一来源

| 数据层 | 权威位置 | 内容 |
|---|---|---|
| 实验定义 | `result.config.experiment.definition` | 控制器、参考、分配、模块、组件调度 |
| 定义哈希 | `result.config.experiment.definition_sha256` | schema+definition |
| 完整身份 | `result.config.experiment.identity_sha256` | 定义+运行配置+模型快照 |
| 运行配置 | `result.config` | tspan、x0、command、seed、solver、environment、通信、调度 |
| 模型快照 | `result.model_metadata` | 机型、质量/惯量、trim、执行器、气动来源/trust/SHA |

没有新增 `result.experiment` 副本；没有两套可独立编辑的 gains/signature。

## 3. Active modules

| 场景 | active_modules | method |
|---|---|---|
| HoverHold | guidance, position, velocity, attitude, rate, allocation | `P_PID_P_PID_PINV` |
| BodyRate Step | guidance, rate, allocation | `RATE_PID_PINV` |

控制器参数结构中即使包含全部增益，也不会把未启用级加入 `active_modules`。

## 4. 实验结果

```text
全新 clone feat/px4-control-fidelity @ 4f6f754
matlab -batch "run('tests/run_tests.m')"
→ 40/40 通过，40.99 s，退出码 0
```

- 原 39 项测试全部保留（逐项比对 removed=空）；
- 新增 `test_experiment_provenance`；
- `test_wtr_export` 改为从结果导出，覆盖身份字段拒绝、定义/配置/模型篡改、
  参考/分配篡改、无残留文件；
- `test_experiment_provenance` 覆盖定义/身份 SHA、相同配置稳定性、参数变化、
  active modules、运行配置变化、模型变化、MAT save/load 和 WTR 导出。

## 5. WTR

- WTR v1 schema 未改变；WindTrace 读取和 smoke 通过。
- `wtr_export` 选项仅保留 output_root/run_id/save_simulation/policy；
  传入 scenario/signature/gains/method/seed 直接报错。
- manifest 的 scenario/method/seed 从 definition/result 派生；
  `source.input_signature` 使用 `identity_sha256`；
  `source.log_sha256` 对应 MAT 文件。
- 导出前校验 definition schema/hash、identity hash、active modules、组件、
  seed/tspan/solver/environment/x0/command、模型身份/元数据、气动 trust/SHA、
  日志完整性。
- 所有校验在创建目录/写文件之前完成。

## 6. 证据

| 文件 | 内容 |
|---|---|
| `evidence/experiment-definition-validation.txt` | schema、模块、定义/身份 SHA、重复构造稳定性、句柄检查 |
| `evidence/scenario-a-b-mismatch.txt` | 定义/参考/控制器/运行配置篡改拒绝 |
| `evidence/model-metadata-validation.txt` | 模型快照/模型替换/元数据篡改 |
| `evidence/runtime-config-validation.txt` | seed/tspan/x0/solver/environment/component 变化改变签名 |
| `evidence/mat-roundtrip.txt` | `aero_save_result` → load → WTR 导出 |
| `evidence/wtr-validation.json` | WTR SHA、read_flight、smoke、参数数量 |
| `evidence/windtrace-smoke.txt` | WindTrace smoke 报告 |
| `evidence/regression-test.txt` | 全新 clone 40/40 |

## 7. 未解决/限制

1. **物理验证 BLOCKED**：合成气动 fixture，不是真实气动或实机验证。
2. 未做 PX4 编译/SITL 并排对照。
3. `IdealState` 无传感器噪声/偏置；不能描述为 EKF。
4. 未实现 INDI/MPC/EKF/归一化虚拟控制接口；本轮不改变控制算法。
5. WTR 仍依赖完整发布日志（`dropped==0`）。

## 8. 提交

| 提交 | 内容 |
|---|---|
| `0e448cd` | 唯一实验定义、身份 SHA、结果派生 WTR、参数快照、测试 |
| `2a11dec` | README/DESIGN 的实验追溯说明 |
| `4f6f754` | 增加 Scenario 身份篡改拒绝测试 |

完整分支审查应覆盖 `main...feat/px4-control-fidelity`，本阶段只描述相对
`1765ac1` 的增量。

## 9. 结论

任何一份仿真结果现在都能从自身恢复实验定义、控制器参数、参考条件、运行配置、
求解器、模型身份和物理快照；WTR 不再信任调用者事后提供的身份。可以提交
NEXUS 复核。
