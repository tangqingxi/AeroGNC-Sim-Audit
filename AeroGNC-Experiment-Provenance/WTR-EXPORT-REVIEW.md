# WTR Export Review

## 1. 接口

```matlab
out = wtr_export(result, model, options)
```

允许的 options：

```text
output_root
run_id
save_simulation
policy
```

**禁止**的旧身份字段：

```text
scenario, signature, gains, method, seed
```

传入任一字段直接 `wtr:export:identity`，不静默忽略。`run_id` 仅影响输出目录，
不影响输入签名。

## 2. 数据来源

| 数据 | 来源 |
|---|---|
| scenario/method/seed | `result.config.experiment.definition` + `result.config` |
| 控制器参数 | `definition.controllers` |
| 参考条件 | `definition.guidance` |
| 分配参数 | `definition.allocation` |
| 运行配置 | `result.config` |
| 机型和物理快照 | `result.model_metadata` |
| 状态布局 | `result.model_metadata.state_layout`（model 布局仅用于一致性校验） |
| 软件来源 | `result.environment`（单独记录，不进入输入签名） |
| Topic 历史 | `result.communication.log` |

不再读取 `options.gains` 或 `model.parameters` 生成参数/几何。

## 3. 导出前校验

1. `result.config.experiment` schema。
2. `definition_sha256` 重算。
3. `identity_sha256` 重算（定义 + runtime + model metadata）。
4. scenario/method/seed 从 definition/result 派生。
5. `active_modules` 与组件表实际存在性。
6. 控制器/参考/分配快照。
7. 组件 `kind/period/priority/offset/delay/latches/topic`。
8. tspan/x0/initial_command/solver/environment/time_tolerance/seed。
9. model name/version 与 `result.model_metadata`。
10. `result.model_metadata.state_layout` 的 nx/position/velocity/quaternion/
    omega/servo/motor/units/description；传入 model 的布局必须与之一致。
11. aero source/trust/file SHA/content SHA/有效域。
12. 发布日志完整（`dropped==0`）。

所有校验在创建结果目录或写文件之前完成。测试证明失败时 `dir_exists=0`。

## 4. 参数快照

`parameters.json` 继续 `windtrace.parameters.v1`，从定义/结果/元数据生成。新增
参考字段：

| 字段 | 单位 |
|---|---|
| `experiment.reference.position` | m |
| `velocity` | m/s |
| `acceleration` | m/s^2 |
| `yaw` | rad |
| `yaw_rate` | rad/s |
| `axis` | 1 |
| `amplitude` | rad/s |
| `start_s` / `end_s` | s |
| `alpha_ff` | rad/s² |

同时输出 `experiment.definition_sha256`、`experiment.identity_sha256`、
`allocation.algorithm/B_tau/delta_limit`、`control.*` 实际参数。

软件来源参数：

```text
software.matlab_version / matlab_release / computer
software.git_commit / git_dirty
software.reproducibility_note（仅 dirty 时）
```

`manifest.source.software` 保存同样的结构化记录；不包含 `project_root`，也不进入
输入签名。若 `git_dirty=true`，明确声明 commit SHA 不能单独保证精确复现。

## 5. MAT/WTR 关联

- `simulation.mat` 是权威结果；`flight.wtr` 从它派生。
- `manifest.source.log_sha256` 为实际 MAT SHA256。
- `manifest.source.input_signature` 为 `identity_sha256`。
- 导出失败不删除 MAT；不覆盖已有文件。
- `save_simulation=false` 时 MAT 路径为空，WTR 不声称原始文件关联。

## 6. 验证

- `test_wtr_export`：成功导出、WTR validate、15 topics、参数路径、manifest
  scenario/method/seed、旧身份字段拒绝、控制器/参考/分配/tspan/x0/model 篡改拒绝、
  无残留目录、容量/覆盖/无日志保护。
- `test_experiment_provenance`：MAT save/load 后 WTR 导出签名一致。
- WindTrace `read_flight` 与 `--smoke` 成功；15 topics，86/106 parameters。
- 状态布局篡改（model 与 result）在创建目录前拒绝；软件来源字段存在且
  `project_root` 不泄露。
