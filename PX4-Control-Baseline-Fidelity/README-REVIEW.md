# README Project Map 审查

## 1. 定位检查

| 要求 | 结果 | 说明 |
|---|---|---|
| 项目是什么 | PASS | 第 1 节：MATLAB 科研仿真平台；SHC09 是首个完整机型，不是唯一机型 |
| 总体架构图 | PASS | Guidance→Navigation→Control→Aircraft/Components→Simulation→Results→WTR |
| 实际源码目录地图 | PASS | 按当前 Git Tree 重写；无规划中的空目录 |
| 功能—源码索引 | PASS | 第 3 节 28 条主路径 |
| 模块职责/边界 | PASS | 第 4 节，含 GNC 不读真值、Topic Bus 唯一 |
| 运行方法 | PASS | 初始化、fixture、外部气动、HoverHold、BodyRate、WTR、测试 |
| 结果与可视化 | PASS | `results/<scenario>/<run_id>/simulation.mat` 与 `flight.wtr` |
| 当前能力/限制 | PASS | 已实现、未实现、物理 BLOCKED 分表 |
| 文档入口 | PASS | ARCHITECTURE/DESIGN/AGENTS/AUDIT |

## 2. 不应出现的内容

| 禁止项 | 结果 |
|---|---|
| 开发阶段报告 | 未出现 |
| Commit SHA / 提交记录 | 未出现 |
| Agent 工作日志 | 未出现 |
| 验收结论/轮次编号 | 未出现 |
| 把未来规划写成已实现 | 未出现（INDI/MPC/EKF 等明确列未实现） |
| 把 fixture 写成真实物理验证 | 未出现；明确 BLOCKED |

## 3. 源码路径核对

`evidence/readme-tree-check.txt` 使用 README 中的行内源码路径逐项 `Test-Path`：

```text
checked=28
missing=0
```

覆盖目录/文件：

```text
src/Simulation/{aero_sim,aero_integrate,Events,Communication,Configuration,Results}
src/Aircraft/SHC09/{shc09_model,shc09_parameters,shc09_dynamics,Aerodynamics}
src/Components/Actuators/{Servo,Motor}
src/GNC/{Guidance,Navigation,Control/{Position,Velocity,Attitude,Rate,Allocation,Internal}}
src/Scenarios
tools/wtr/{wtr_export,wtr_validate}
tests/run_tests.m
```

README 的目录树也逐项与 `Get-ChildItem src -Recurse` 对照；未加入不存在目录。

## 4. 使用命令检查

| 命令/选项 | 与实际接口一致 |
|---|---|
| `aero_setup()` | 是 |
| `hover_scenario(struct('aero_source',...,'tspan',...,'position_offset',...))` | 是 |
| `rate_step_scenario(... 'axis','amplitude','start_s','end_s','tspan')` | 是 |
| `aero_sim(s.model,s.cfg)` | 是 |
| `wtr_export(result,s.model,struct('scenario',s.name,'signature',s.signature,'gains',s.gains,'seed',s.cfg.seed))` | 是 |
| `matlab -batch "run('tests/run_tests.m')"` | 是 |

未写不存在的气动文件或虚构 API；外部数据明确需要 `duct_raw`/`wing_raw`。

## 5. 可维护性

- README 不复制 DESIGN 的全部数学公式，只保留一级控制链和单位要点。
- 功能表按“用户要找的功能”组织，不按内部辅助函数堆叠。
- 当前限制单独成表，几个月后仍能据此判断项目能力边界。
- 源码地图以目录职责为主，新增文件按 DESIGN §13.1 放置后可继续维护。

## 6. 结论

README 已从旧版“项目介绍 + 阶段摘要”改写为项目地图；未包含开发流水账，
路径检查 28/28 通过。可提交 NEXUS 复核。
