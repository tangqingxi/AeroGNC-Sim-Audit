# GNC-Thrust-Setpoint-Unification

## 阶段名称

`GNC-Thrust-Setpoint-Unification`：实现完成，等待 NEXUS 验收；Physical Validation **BLOCKED**。
DEV `feat/thrust-setpoint-unification` @ `3acba790adcaad8215eccb69f495b5f89d8a435b`，已推送。
DEV 起点经用户修正为 `abcc57b20eac3194fea520e6db5b1199ee0cd183`，完整保留 Attitude Step 与四种 Reference；
AUDIT `study/thrust-setpoint-unification` 从 `228d95954f86267c3af6e86c3164cd15c5703ac4` 创建。
main、旧分支和 Tag 均未修改。

## 完成内容

1. 唯一 topic_specs 为两类 rate reference 增加 thrust_n，新增 control/thrust_setpoint（N，FRD -Z），加入实际日志。
2. Attitude P 将推力原样加入 rate setpoint；Rate PID 同次更新、同采样时刻发布 torque 与原样 thrust。
3. Allocation 只读取 torque/thrust 两个控制输出；去掉 attitude_topic 和 hover fallback，缺失或过期不产生新分配。
4. BodyRate 固定 thrust_reference_n 默认 trim.thrust_n；场景按电机实际推力范围验证，记录到唯一 Experiment Definition 与 WTR 参数。
5. WTR v1 新增 vehicle_thrust_setpoint.thrust_n，直接使用 Rate 发布日志，无插值；WindTrace 源码未改（bcdb6a8eca9334b395759c604d5c3978feb1bd8f）。
6. README/DESIGN 同步；全部修改均位于既有目录，生产路径登记不需要增加目录。

## 验证结果

- 历史 51 项保留，removed=[]；新增指定四项测试，全部注册，共 55 项。
- 本地全量：55/55，238.29 s，0 跳过；全新远端 LF clone：55/55，287.59 s，0 跳过。命令均为 `matlab -batch "run('tests/run_tests.m')"`，实际批次另启用 diary。
- 测试覆盖姿态/速率透传、双输出时间一致、PID 公式与积分不变、缺失/负值/过期拒绝、Allocation 双输入与 motor/Ve/物理限幅、四条链路、WTR 原生时间轴。
- 四场景修改前后逐点比较：完整状态轨迹最大差值均为 0，Rate 力矩最大差值均为 0，全部锁存执行器命令完全一致。
- 比较配置：HoverHold 为 [0,4] s、position_offset=[0.15,0,0.05] m；Velocity Step/Attitude Step/BodyRate Step 使用各自默认时间与阶跃参数。全用 synthetic fixture，MATLAB R2025b。
- 原有 Velocity 指标 max=0.7229 m/s、RMSE=0.4434 m/s、residual=0.0162 m/s、位移=1.4725 m；Attitude max roll=5.358°、末态整体误差=2.022°。数值一致限于本次配置，不代表物理有效或所有异步频率配置行为相同。
- 四场景新 WTR 全部独立校验、read_flight、CLI smoke 和真实 Viewer 曲线选择/绘图通过；均 16 topics。
- Position/Velocity 推力曲线变化；Attitude/BodyRate 固定推力及其参数快照一致。新曲线的原生时间和值与日志逐点一致。
- 源码片段比较：Attitude 控制数学、Rate PID/积分、PINV 与电机求解逐字一致；ARCHITECTURE、Simulation、Topic Bus、Aircraft、执行器、增益、Position/Velocity 控制器均未修改。
- 首轮新增链路测试固定推力 19 N，导致合成气动表迎角越界；测试改用合法的 22 N 后复测通过，未放宽气动域、测试阈值或控制约束。

证据 SHA256 校验：15/15 PASS；交接检查：READY，0 warning。日志副本仅统一 LF 并去除行末空格，保留所有输出与数值。

## 未解决问题

Physical Validation **BLOCKED**：synthetic fixture 与 IdealState，不是实机、真实气动或传感器误差导航验证。
未执行 PX4 SITL/编译对照；PX4 结论限于指定提交源码的数据流对照。
采样语义为 Attitude 更新时锁存推力、Rate 在自身更新时转发；非默认异步频率下的额外保持延迟是此接口的明确语义，不保证与旧 Allocation 直读上游的采样完全相同。
容量保护测试会主动产生既有 warning；0 跳过不等于日志中没有容量预警。

## 相关提交与文件

所有 DEV 路径基于完整 SHA `3acba790adcaad8215eccb69f495b5f89d8a435b`；重点文件为
src/GNC/Control/Internal/topic_specs.m、src/GNC/Control/Attitude/attitude_p.m、
src/GNC/Control/Rate/rate_pid.m、src/GNC/Control/Allocation/pinv_allocator.m、src/GNC/Guidance/body_rate_step.m、
src/Scenarios/rate_step_scenario.m、src/Scenarios/Internal/build_config.m 与 experiment_definition.m、
tools/wtr/wtr_build_topics.m / wtr_build_parameters.m、tests/GNC/test_*thrust*.m。
完整原始 MAT/WTR 只在 DEV 的 ignored results/ 下，不复制到 AUDIT；本目录仅含必要摘要与截图。
证据详见 evidence-manifest.json；架构接口与 PX4 来源见 ARCHITECTURE-REVIEW.md / CONTROL-CHAIN.md。

## 下一阶段建议

由 NEXUS 审查本提交的统一推力接口、55 项回归、四场景零差异与 WindTrace 证据，并记录接受或拒绝；不自行合并 main 或建立 Tag。
