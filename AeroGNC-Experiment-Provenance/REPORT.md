# Experiment-Provenance-Final-Fix — 最终整改报告

## 阶段名称与状态

`Experiment-Provenance-Final-Fix`；整改完成，等待 NEXUS 最终验收，尚未验收。
沿用审计目录 `AeroGNC-Experiment-Provenance/`，不新增阶段目录。
DEV `feat/px4-control-fidelity` @ `31159589f743b205f68b5777a70a24377dc57723`，已推送；AUDIT 沿用
`study/aerognc-experiment-provenance`。未合并 main、未创建 Tag。
开工远端 SHA 与指定基线完全一致：DEV `d2ddff4a944bd454f9cd1f4523965d0f989d44cf`，
AUDIT `a26f349fc903edb1ba20170f675ffb832b003efe`；两工作区当时均干净。

## 完成内容

| 整改 | 结果 | 实现与证据 |
|---|---|---|
| A 软件来源动态刷新 | PASS | 每次 aero_sim 入口重新查询 HEAD/dirty；只缓存 MATLAB/platform；入口快照写 result.environment |
| B WTR 气动追溯 | PASS | 从 result.model_metadata.aero 导出 content_sha256、duct_domain_rad、wing_domain_rad；synthetic/external 均覆盖 |
| C 完整 N 维尺寸 | PASS | size 的全部维度按顺序编码；2D 编码规则、字段排序、数据顺序和特殊值规则保留 |

输入签名的定义不变，软件来源继续排除；N 维修复使此前发生尺寸碰撞的输入可以区分。
未修改 GNC/PINV/动力学/执行器/Topic Bus/积分与调度/Experiment Definition 架构/WTR v1/WindTrace。
所有新增 MATLAB 注释为简体中文，新增测试均在现有测试目录注册。

## 验证结果

- 原有 42 项全部保留，removed=[]；新增 3 项，总计 45 项。
- 本地完整回归 45/45，55.40 s；远端全新 LF clone 完整回归 45/45，55.35 s；0 失败、0 跳过。
- 命令：`git -c core.autocrlf=false clone --branch feat/px4-control-fidelity --single-branch https://github.com/tangqingxi/AeroGNC-Sim <fresh>`；
  在 clone 执行 `matlab -batch "run('tests/run_tests.m')"`。实际证据批次还启用 diary 并运行附加双场景验证。
- `test_environment_refresh`：隔离 Git 仓库，同一 MATLAB 会话 clean→dirty→重复 dirty→提交后 clean/新 HEAD；
  运行回调改源码时仍保存入口 clean，下一次运行识别 dirty。
- `test_canonical_json_nd`：3D/4D、相同内容不同尺寸、字段排序、空数组尺寸、逻辑数组、列主序。
- `test_wtr_aero_provenance`：fixture 系数改变 WTR content SHA；双有效域改变 WTR 参数；external 换路径身份与参数不变；调用者参数修改无影响。
- 两场景（rate 1.5 s / hover 3 s）结果 save/load 后数值日志、环境、模型快照、通信快照及实验定义 `isequaln` 一致，运行身份重算通过，再从加载结果导出；MATLAB 独立 wtr_validate 均通过。
- WindTrace `bcdb6a8eca9334b395759c604d5c3978feb1bd8f` 未修改；read_flight 与 app.py --smoke 均通过，
  每份 15 topics，参数分别 89/109 项；软件来源为本 DEV SHA、dirty=false。
- 测试登记、目录治理、which -all 唯一性由 test_project_layout 通过；git diff --check 通过。
- 初次新增 WTR 测试因 0.02 s 未覆盖必要锁存事件失败；仅将测试时长改为 0.05 s 后完整重跑，未放宽任何阈值或校验。
- 首次 clone 因默认 CRLF 转换产生 dirty，已中止该批次；重新以 clone 时 core.autocrlf=false 检出，正式证据来自干净 LF clone。

附加 MAT 检查初次直接比较含回调句柄的整个结果不适用（load 后句柄实例变化）；改为逐项比较数值日志与追溯快照，并以生产 WTR 身份重算验证运行配置，复测通过。

证据 SHA256 校验：9/9 PASS；交接检查：READY，0 warning。

## 未解决问题

**物理验证 BLOCKED**：气动为 synthetic fixture；IdealState 不代表传感器误差导航验证。
未执行 PX4/SITL 对照或真实气动/实机验证。NEXUS 尚未给出最终验收结论。
容量测试主动触发的 warning 属既有保护行为；“0 warning”专指交接一致性检查。

## 相关提交、文件与证据

所有 DEV 路径均引用完整提交 `31159589f743b205f68b5777a70a24377dc57723`：
`src/Simulation/Internal/aero_env_info.m`、`src/Simulation/aero_sim.m`、
`src/Simulation/Internal/aero_canonical_json.m`、`tools/wtr/wtr_build_parameters.m`、
`tests/run_tests.m`、`tests/Simulation/Core/test_environment_refresh.m`、
`tests/Simulation/Core/test_canonical_json_nd.m`、`tests/Export/WTR/test_wtr_aero_provenance.m`。

本次证据为 `evidence/final-fix-*` 与 evidence-manifest.json 列明的文档。
目录内其它旧证据仅对应 d2ddff4 及更早历史，不作为本次新提交的执行证明。
原始 MAT/WTR 仅在 DEV clone 的 ignored `results/final-fix-*` 下，未复制到 AUDIT。

## 下一阶段建议

由 NEXUS 对本 DEV SHA 及本次 manifest 证据执行最终验收；本 Agent 停止开发等待结论。
