# AeroGNC-Sim — Audit

审计与交接仓库，对应开发仓库 [tangqingxi/AeroGNC-Sim](https://github.com/tangqingxi/AeroGNC-Sim)。

**当前状态入口：[STATUS.md](STATUS.md)**（阶段级详细结论见下）

- 当前阶段：`Attitude-Reference-Scenario`（**in-progress / ready-for-review**，DEV 分支 `feat/attitude-reference-scenario` @ `abcc57b`）
  → [Attitude-Reference-Scenario/REPORT.md](Attitude-Reference-Scenario/REPORT.md)
- 场景设计/控制链：[SCENARIO-DESIGN.md](Attitude-Reference-Scenario/SCENARIO-DESIGN.md)、
  [CONTROL-CHAIN.md](Attitude-Reference-Scenario/CONTROL-CHAIN.md)
- 交接单：[Attitude-Reference-Scenario/HANDOFF.md](Attitude-Reference-Scenario/HANDOFF.md)
- 证据清单：[Attitude-Reference-Scenario/evidence-manifest.json](Attitude-Reference-Scenario/evidence-manifest.json)
- 上一阶段：`Velocity-Reference-Scenario` → [Velocity-Reference-Scenario/REPORT.md](Velocity-Reference-Scenario/REPORT.md)
- 上一阶段：`AeroGNC-Experiment-Provenance` → [AeroGNC-Experiment-Provenance/REPORT.md](AeroGNC-Experiment-Provenance/REPORT.md)
- 上一阶段：`PX4-Control-Baseline-Fidelity`（in-progress/ready-for-review）→ [PX4-Control-Baseline-Fidelity/REPORT.md](PX4-Control-Baseline-Fidelity/REPORT.md)
- 上一阶段：`GNC-Control-Modularization`（ready-for-review）→ [GNC-Control-Modularization/REPORT.md](GNC-Control-Modularization/REPORT.md)
- 归一化研究（含偏航权限修正）→ [Normalized-Control-Interface-Study/REPORT.md](Normalized-Control-Interface-Study/REPORT.md)
- 已结案：`V1.0-Foundation-Cleanup`（ACCEPTED）→ [V1.0-Foundation-Cleanup/REPORT.md](V1.0-Foundation-Cleanup/REPORT.md)
- 已结案：`Communication-Baseline`（ACCEPTED）、`SHC09-Aircraft`（ACCEPT 限制性）、
  `Simulation-Kernel`、`Architecture-Baseline`

DEV main 仍为 `57bbe68`；GNC 分支未合并。42 项测试通过；真实气动与物理闭环验收 **BLOCKED**，
fixture 结果只证明软件链路。`v1.0.0` 未发布。

本仓库不存放可执行源码、原始数据集或凭据；一切结论以 DEV 的完整 commit SHA 为准。
