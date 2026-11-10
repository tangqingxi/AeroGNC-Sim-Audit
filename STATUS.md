# AeroGNC-Sim — 项目状态入口

> 唯一状态索引，不替代 ARCHITECTURE/DESIGN；仅在阶段验收、合并、HOLD 或方向变更时更新；不记录本文件所在 AUDIT 提交的 SHA（AUDIT 版本由 Git HEAD 确定）。

## 仓库与基线

| 项 | 值 |
|---|---|
| DEV | <https://github.com/tangqingxi/AeroGNC-Sim> |
| DEV main | `57bbe6893d83808e3b46aa79864701b32b31c775`（2026-11-09 核验；仅 `main` 分支） |
| AUDIT | <https://github.com/tangqingxi/AeroGNC-Sim-Audit> |
| 测试基线 | MATLAB R2025b；27 项回归，全新 clone 通过 |

## 已验收阶段

| 阶段 | 结论 | 报告 |
|---|---|---|
| Architecture-Baseline | ACCEPT | [报告](Architecture-Baseline/REPORT.md) |
| Simulation-Kernel | ACCEPT | [报告](Simulation-Kernel/REPORT.md) |
| SHC09-Aircraft | ACCEPT（限制性；物理有效性待验证） | [报告](SHC09-Aircraft/REPORT.md) |
| Communication-Baseline | ACCEPTED | [报告](Communication-Baseline/REPORT.md) |
| V1.0-Foundation-Cleanup | ACCEPTED（DEV main `57bbe68`） | [报告](V1.0-Foundation-Cleanup/REPORT.md) |

## 当前任务状态

- 当前阶段：`GNC-Control-Modularization` 实现完成（`feat/gnc-baseline` @ `59cf917`），待 NEXUS 复核；未合并 main。

## 已批准且仍有效的决策

- ARCHITECTURE §2 冻结：NED/FRD、分段 `ode45`、GNC 三域独立、控制分配可替换、Simulation 不承载气动。
- V1.0 Foundation 已完成；基础 GNC（悬停/角速度 Guidance、IdealState Navigation、PID、PINV 分配）
  与悬停保持、角速度阶跃场景仍在 V1.0 范围内（ARCHITECTURE §13.2）。
- 通信：唯一 Latest-Value Topic Bus，始终启用；`command` / `held_command` 独立锁存。
- SHC09 几何、效能矩阵、动力学、积分与执行器约束冻结。
- `v1.0.0` 未发布；需基础 GNC 与场景通过后建立 Tag。
- 分支政策：`main` 为主线；临时分支合并核验后删除（AGENTS §10）。

## 未解决问题与限制

- V1.0 尚未发布：基础 GNC、闭环场景与验收未完成。
- 真实气动数据许可与物理有效性未确认；`trust='verified'` 仅为调用方声明。
- 惯量与执行器动态参数为参考/名义值，未做实机辨识。
- Topic 日志有界、仅分析用途；`result.measurements` 随仿真时间增长。
- `truth/` 前缀是命名约定，不是访问控制。

## 下一步授权状态

- ARCHITECTURE §5/§8 已批准基础 GNC 的接口语义、职责边界与交付范围。
- 具体 Topic/载荷与四级控制实现已记录于 DESIGN §15；跨阶段仍不视为冻结。
- 未授权：合并 main、发布 `v1.0.0`、启动后续阶段。
- PROPOSED：无正式提交待批设计；讨论意见不得记为已批准决策。
- HOLD：无（V1.0 Foundation 的 HOLD 已由返工解除）。

## 权威文档

- [ARCHITECTURE.md](https://github.com/tangqingxi/AeroGNC-Sim/blob/main/ARCHITECTURE.md)（§2/§8/§9/§13）
- [DESIGN.md](https://github.com/tangqingxi/AeroGNC-Sim/blob/main/DESIGN.md)
- [AGENTS.md](https://github.com/tangqingxi/AeroGNC-Sim/blob/main/AGENTS.md)（§10 版本/分支政策）
- [README.md](https://github.com/tangqingxi/AeroGNC-Sim/blob/main/README.md)
- 最近交接：[GNC-Control-Modularization/HANDOFF.md](GNC-Control-Modularization/HANDOFF.md)
