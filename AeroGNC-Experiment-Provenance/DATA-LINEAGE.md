# Data Lineage

## 1. 数据流

```text
Scenario Options
   │
   ▼
Experiment Definition（cfg.experiment）
   │ 同一定义派生
   ▼
Controller callbacks / component schedule
   │
   ▼
aero_sim
   ├── result.config            （tspan/x0/command/seed/solver/environment/通信/调度）
   ├── result.config.experiment （定义 + definition_sha256 + identity_sha256）
   └── result.model_metadata    （机型/物理/执行器/trim/气动 trust+SHA）
   │
   ▼
aero_save_result → simulation.mat
   │
   ▼
wtr_export（只读 result）
   ├── wtr_experiment_identity 重新计算并校验定义/身份
   ├── wtr_build_topics
   ├── wtr_build_parameters（definition + runtime + model metadata）
   └── wtr_geometry（model metadata）
   │
   ▼
flight.wtr
   │
   ▼
WindTrace read_flight / viewer
```

## 2. 唯一来源

| 层 | 位置 | 不应从何处读取 |
|---|---|---|
| 实验定义 | `result.config.experiment.definition` | `s.gains`、`options.gains`、调用者 signature |
| 运行配置 | `result.config` | 调用者重新提供 tspan/seed/solver |
| 模型快照 | `result.model_metadata` | 可能被修改的 `model.parameters` |
| 组件调度 | `definition.components` + `result.config.components` | 调用者组件表 |
| 原始结果 | `simulation.mat` | WTR 重新生成的状态 |
| WTR | `flight.wtr` | 作为原始结果使用 |

## 3. 身份签名

```text
input_signature_sha256 =
  SHA256({
    schema: 'aero-experiment-identity/1',
    experiment: {schema, definition, definition_sha256},
    runtime: {
      tspan, x0, initial_command, seed, solver, environment,
      time_tolerance, components, communication
    },
    model: result.model_metadata
  })
```

排除：

- run_id、output_root、目录、wall_time；
- `step`/`sample` 函数句柄；
- 任何调用者事后填写的 gains/signature。

因此：

- 相同实验输入 → 同一签名；
- 控制参数、参考、模型、x0、seed、solver、环境、组件或通信配置变化 → 签名变化；
- 结果被篡改但未更新 `identity_sha256` → 导出拒绝。

## 4. MAT 与 WTR

```text
results/<scenario>/<run_id>/
├── simulation.mat   ← 权威原始结果（result.config.experiment 完整）
└── flight.wtr       ← 派生可视化（manifest.source.input_signature = identity_sha256）
```

- `manifest.source.log_sha256` 是实际 `simulation.mat` 的 SHA256；
- 若 `save_simulation=false`，WTR 明确不声称原始 MAT 关联；
- 导出失败不删除 MAT；
- 禁止覆盖已有 MAT/WTR；
- 所有身份校验在创建目录/写文件之前完成；
- WTR v1 协议未改，WindTrace 继续读取。

## 5. 数据完整性测试

| 场景 | 结果 |
|---|---|
| 定义结构/序列化/无句柄 | `experiment-definition-validation.txt` |
| 定义/身份 SHA | 同上 |
| 相同配置重复构造 | SHA 相同 |
| 修改有效参数 | SHA 改变 |
| 参考/控制器/分配篡改 | `wtr:identity:definition` |
| tspan/x0 篡改 | `wtr:identity:input` |
| 模型名称/元数据篡改 | `wtr:identity:model` |
| MAT save/load | `mat-roundtrip.txt` |
| WTR 读取/smoke | `wtr-validation.json`、`windtrace-smoke.txt` |
