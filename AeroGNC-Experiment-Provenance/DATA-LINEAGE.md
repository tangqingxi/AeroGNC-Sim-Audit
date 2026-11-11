# 最终整改：运行入口来源与完整维度

当前语义阶段：Experiment-Provenance-Final-Fix；DEV `31159589f743b205f68b5777a70a24377dc57723`，已推送。

每次仿真入口采集最新 HEAD/dirty，MATLAB/version/release/platform 可缓存；result.environment 固定入口快照。软件来源仍不进入输入签名。数值及逻辑数组现在编码完整 size，包含空数组；MATLAB 本身不可区分的尾随单例维度遵循 size 语义。气动 content SHA 与双有效域从结果模型快照直达 WTR，不读取调用者重新提供的参数。

证据：evidence/final-fix-*；完整结果见 REPORT.md。物理验证 BLOCKED。

---

以下为前一基线的设计说明与历史验证，旧执行数值不作为当前 SHA 的证据。

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
    model: result.model_metadata（哈希前移除本地 aero.source_file）
  })
```

规范化规则：struct 字段按字母排序，struct 数组/cell 顺序保持，数值维度显式编码，
NaN/Inf 显式编码，函数/对象句柄拒绝。原始 `result.model_metadata` 仍保留
`aero.source_file`；身份只保留 source type/trust/file SHA/content SHA/有效域。

排除：

- run_id、output_root、目录、wall_time；
- `step`/`sample` 函数句柄；
- 本地气动文件绝对路径；
- 任何调用者事后填写的 gains/signature。
- 软件版本/`result.environment`（单独记录，不作为实验输入签名）。

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
- WTR `manifest.source.software` 与 `software.*` 参数记录 MATLAB version/release/
  computer/git commit/git dirty；`project_root` 不写入，软件来源不进入输入签名。
- 若 `git_dirty=true`，manifest 明确带 `reproducibility_note`：仅凭 commit SHA
  不能保证精确复现。

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
| 跨路径气动身份 | `cross-path-identity.txt` |
| fixture 内容身份 | `fixture-content-identity.txt` |
| 完整状态布局/篡改 | `state-layout-validation.txt` |
| 规范化序列化 | `canonical-json.txt` |
| 软件来源 | `software-provenance.txt` |
| MAT save/load | `mat-roundtrip.txt` |
| WTR 读取/smoke | `wtr-validation.json`、`windtrace-smoke.txt` |
