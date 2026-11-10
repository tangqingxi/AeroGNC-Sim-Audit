# AeroGNC-Sim — Audit

审计与交接仓库，对应开发仓库 [tangqingxi/AeroGNC-Sim](https://github.com/tangqingxi/AeroGNC-Sim)。

**当前状态入口：[STATUS.md](STATUS.md)**（阶段级详细结论见下）

- 当前阶段：`GNC-Control-Modularization`（**ready-for-review**，DEV 分支 `feat/gnc-baseline` @ `59cf917`）
  → [GNC-Control-Modularization/REPORT.md](GNC-Control-Modularization/REPORT.md)
- 源码审查与文件映射：[GNC-Control-Modularization/SOURCE-REVIEW.md](GNC-Control-Modularization/SOURCE-REVIEW.md)
- 交接单：[GNC-Control-Modularization/HANDOFF.md](GNC-Control-Modularization/HANDOFF.md)
- 证据清单：[GNC-Control-Modularization/evidence-manifest.json](GNC-Control-Modularization/evidence-manifest.json)
- 上一阶段：`GNC-Baseline`（ready-for-review，已被本轮重构取代）→ [GNC-Baseline/REPORT.md](GNC-Baseline/REPORT.md)
- 已结案：`V1.0-Foundation-Cleanup`（ACCEPTED）→ [V1.0-Foundation-Cleanup/REPORT.md](V1.0-Foundation-Cleanup/REPORT.md)
- 已结案：`Communication-Baseline`（ACCEPTED）、`SHC09-Aircraft`（ACCEPT 限制性）、
  `Simulation-Kernel`、`Architecture-Baseline`

DEV main 仍为 `57bbe68`；GNC 分支未合并。37 项测试通过；真实气动与物理闭环验收 **BLOCKED**，
fixture 结果只证明软件链路。`v1.0.0` 未发布。

本仓库不存放可执行源码、原始数据集或凭据；一切结论以 DEV 的完整 commit SHA 为准。
