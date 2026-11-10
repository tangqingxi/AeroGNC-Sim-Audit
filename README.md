# AeroGNC-Sim — Audit

审计与交接仓库，对应开发仓库 [tangqingxi/AeroGNC-Sim](https://github.com/tangqingxi/AeroGNC-Sim)。

**当前状态入口：[STATUS.md](STATUS.md)**（阶段级详细结论见下）

- 当前阶段：`PX4-Control-Baseline-Fidelity`（**in-progress / ready-for-review**，DEV 分支 `feat/px4-control-fidelity` @ `61507b4`）
  → [PX4-Control-Baseline-Fidelity/REPORT.md](PX4-Control-Baseline-Fidelity/REPORT.md)
- 源码地图/差异/决策：[PX4-SOURCE-MAP.md](PX4-Control-Baseline-Fidelity/PX4-SOURCE-MAP.md)、
  [CONTROL-DELTA-MATRIX.md](PX4-Control-Baseline-Fidelity/CONTROL-DELTA-MATRIX.md)、
  [IMPLEMENTATION-DECISIONS.md](PX4-Control-Baseline-Fidelity/IMPLEMENTATION-DECISIONS.md)
- README 审查：[README-REVIEW.md](PX4-Control-Baseline-Fidelity/README-REVIEW.md)
- 交接单：[PX4-Control-Baseline-Fidelity/HANDOFF.md](PX4-Control-Baseline-Fidelity/HANDOFF.md)
- 证据清单：[PX4-Control-Baseline-Fidelity/evidence-manifest.json](PX4-Control-Baseline-Fidelity/evidence-manifest.json)
- 上一阶段：`GNC-Control-Modularization`（ready-for-review）→ [GNC-Control-Modularization/REPORT.md](GNC-Control-Modularization/REPORT.md)
- 归一化研究（含偏航权限修正）→ [Normalized-Control-Interface-Study/REPORT.md](Normalized-Control-Interface-Study/REPORT.md)
- 已结案：`V1.0-Foundation-Cleanup`（ACCEPTED）→ [V1.0-Foundation-Cleanup/REPORT.md](V1.0-Foundation-Cleanup/REPORT.md)
- 已结案：`Communication-Baseline`（ACCEPTED）、`SHC09-Aircraft`（ACCEPT 限制性）、
  `Simulation-Kernel`、`Architecture-Baseline`

DEV main 仍为 `57bbe68`；GNC 分支未合并。38 项测试通过；真实气动与物理闭环验收 **BLOCKED**，
fixture 结果只证明软件链路。`v1.0.0` 未发布。

本仓库不存放可执行源码、原始数据集或凭据；一切结论以 DEV 的完整 commit SHA 为准。
