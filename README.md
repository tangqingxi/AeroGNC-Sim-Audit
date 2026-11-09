# AeroGNC-Sim — Audit

审计与交接仓库，对应开发仓库 [tangqingxi/AeroGNC-Sim](https://github.com/tangqingxi/AeroGNC-Sim)。

**当前状态入口：[STATUS.md](STATUS.md)**（阶段级详细结论见下）

- 当前阶段：`GNC-Baseline`（**ready-for-review**，DEV 分支 `feat/gnc-baseline` @ `ade4b69`）
  → [GNC-Baseline/REPORT.md](GNC-Baseline/REPORT.md)
- 源码研究与复用分类：[GNC-Baseline/SOURCE-REVIEW.md](GNC-Baseline/SOURCE-REVIEW.md)
- 交接单：[GNC-Baseline/HANDOFF.md](GNC-Baseline/HANDOFF.md)
- 证据清单：[GNC-Baseline/evidence-manifest.json](GNC-Baseline/evidence-manifest.json)
- 已结案：`V1.0-Foundation-Cleanup`（ACCEPTED，DEV main `57bbe68`）→ [V1.0-Foundation-Cleanup/REPORT.md](V1.0-Foundation-Cleanup/REPORT.md)
- 已结案：`Communication-Baseline`（ACCEPTED）→ [Communication-Baseline/REPORT.md](Communication-Baseline/REPORT.md)
- 已结案：`SHC09-Aircraft`（ACCEPT 限制性）→ [SHC09-Aircraft/REPORT.md](SHC09-Aircraft/REPORT.md)
- 已验收：`Simulation-Kernel`、`Architecture-Baseline`

DEV main 仍为 `57bbe68`；GNC 分支未合并。31 项测试通过；真实气动与物理闭环验收 **BLOCKED**，
fixture 结果只证明软件链路。`v1.0.0` 未发布。

本仓库不存放可执行源码、原始数据集或凭据；一切结论以 DEV 的完整 commit SHA 为准。
