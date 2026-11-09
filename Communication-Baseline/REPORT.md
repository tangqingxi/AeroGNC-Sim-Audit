# Communication-Baseline — 阶段报告（Core + Integration）

| 项 | 值 |
|---|---|
| 阶段名称 | `Communication-Baseline`（`Communication-Core` 已完成；`Communication-Integration` 已追加，见 §7） |
| 状态 | `ready-for-review`（Core 已限制性验收；Integration 待 NEXUS 复核；Validation 未开始） |
| DEV 分支 | `feat/communication-baseline` @ `381e58b78ed02e20ba02511910c453b3f35a41e6` |
| DEV main | `c860972cdc223d2e0cf14bb6799bf290561bd50e`（未合并） |
| AUDIT | <https://github.com/tangqingxi/AeroGNC-Sim-Audit> |
| 前身参考 | `E:\Ducted-Fan-SHC09-matlab` @ `43b6c7aef1912f8118e54d3decac1ac55465d5b1`（只读） |

## 1. 完成内容

在 `src/Simulation/Communication/` 实现最小通用 Topic Bus：

| 文件 | 职责 |
|---|---|
| `aero_topic_bus.m` | 注册、发布、读取、latest、tryRead、契约校验、序号与时间规则 |
| `aero_topic_spec.m` | 声明式契约：字段、尺寸、单位、真值许可、最大消息年龄 |
| `tests/Simulation/Communication/test_topic_bus.m` | 注册、发布、读取、时间、契约与真值策略的单元测试 |

接口：

```matlab
bus = aero_topic_bus();
spec = aero_topic_spec({'position','velocity'}, {[3 1],[3 1]}, {'m','m/s'});
bus.register('vehicle_local_position', spec);
bus.publish('vehicle_local_position', data, sample_t, available_t, 'navigation');
msg = bus.read('vehicle_local_position', now);       % 严格：可用/过期/缺失
[msg, status] = bus.tryRead('vehicle_local_position', now);
msg = bus.latest('vehicle_local_position');          % 最近发布，不做时间门限
```

消息字段：`name`、`data`、`sample_time_s`、`available_time_s`、`sequence`（全局）、
`topic_sequence`（每 Topic 独立）、`source`。

规则：

- Latest-Value；同 Topic `sample_time_s` 不得倒退，相等允许（后发布覆盖并递增序号）。
- `read` 检查 `available_time_s` 与契约 `max_age_s`，分别返回 `not_available` / `stale`；
  `tryRead` 以状态字符串返回，不抛出。
- 契约校验字段集合、数值类型、尺寸与有限性；单位只声明不换算。
- 真值隔离：普通 Topic 不得承载真值；仅 `truth/` 前缀且 `allow_ground_truth=true` 的 Topic 可发布，
  普通 Topic 声明真值或发布真值均报错。
- 无时钟、调度、Timer、线程、网络、文件 I/O；时间由调用方传入。

## 2. 与前身的关系

只读参考 `Shc09Bus.m` / `shc09_topics.m` / `Shc09TopicLogger.m` / `test_bus.m` / `modules/README.md`
（固定 commit `43b6c7ae`）。前身 Bus 仅保存 payload/time/sequence；本实现独立重写，增加：

- 显式 `available_time_s`（防未来数据）；
- 声明式字段/尺寸/单位契约；
- 过期（`max_age_s`）与 `tryRead` 状态；
- 真值与命名策略；
- 全局 `sequence` 与每 Topic `topic_sequence` 分离。

未复制前身源码、Scheduler、日志缓冲或历史实验代码；前身仅作设计参考，无许可文本再分发。

## 3. 验证结果

从 GitHub **全新 clone** `feat/communication-baseline` @ `cce1b32`：

```text
matlab -batch "run('tests/run_tests.m')"  →  24/24 通过，7.19 s，退出码 0
```

覆盖：注册/重复注册/非法名称、发布与读取、多 Topic 独立版本、全局顺序、同一时刻连续发布、
时间倒退、NaN 时间、可用时间早于采样、缺失、未知 Topic、过期、字段缺失/多余/维度/类型/非有限值、
真值 Topic 命名策略、以及一次 `aero_sim` 短仿真确认既有接口未受影响。
原有 23 项测试全部保留（阈值未变）。

## 4. 未解决问题与限制

1. **尚未接入 `aero_sim`**：本轮不修改内核；Integration 阶段必须由 Simulation 持有唯一 Bus 实例，
   禁止第二套并行通信状态，且不得改变 `command` / `held_command` 独立锁存语义。
2. **Latest-Value 限制**：无历史队列；未来 ECL 等逐样本消费的高频数据需另行设计。
3. **契约不做物理解释**：不校验量程/单位换算，由模块负责。
4. **无过期清理**：过期消息保留在存储中，读取时判定，不主动删除。

## 5. 相关提交与文件

| 仓库 | 内容 |
|---|---|
| DEV | `cce1b32`：`src/Simulation/Communication/`、`tests/Simulation/Communication/`、`aero_setup`、`run_tests`、`DESIGN.md` §14、`AGENTS.md` 目录规则 |
| AUDIT | 本报告 + `HANDOFF.md` + `evidence/topic-bus-test.txt` + 证据清单 |

## 6. 下一阶段建议

`Communication-Integration`：

1. 由 `aero_sim` 创建并持有唯一 `aero_topic_bus` 实例（可通过 `cfg.communication` 注入，保持默认关闭）。
2. 将测量发布、command/held_command 锁存与事件调度接入总线，不改变时序语义。
3. 增加集成测试：测量在可用时刻之后才可读、锁存通道互不覆盖、无第二套 Bus。

## 7. Communication-Integration（2026-11-09 追加）

在 `cce1b32`（Core）基础上完成 `aero_sim` 集成，提交 `381e58b78ed02e20ba02511910c453b3f35a41e6`。

### 7.1 实际接口与改动

| 内容 | 实现 |
|---|---|
| 唯一总线 | `aero_sim` 在 `cfg.communication.enabled=true` 时创建并持有唯一 `aero_topic_bus`，作为 `bus.topics` 交给组件；默认关闭，不创建 Bus |
| 配置 | `cfg.communication = struct('enabled',false,'topics',{{}},'command_topic','','held_command_topic','')`；Topic 由 `topics`（name+spec）显式注册 |
| Sensor → Topic | 组件新增可选 `topic` 字段；内核在**发布事件**（真正可用时）发布 `struct('value', v(:), 'sample_time_s', sampleTime)`，`available_time` 为当前事件时刻 |
| 命令 → Topic | `bus.command` / `bus.held_command` 变化时分别发布到 `command_topic` / `held_command_topic`；数值命令打包为 `struct('value', cmd(:))`，结构体命令原样发布 |
| 通用性 | 内核不识别 SHC09/PID 字段名；`aero_communication_setup` / `aero_command_payload` 只做注册与形状整理 |
| 结果 | 启用时 `result.communication` 含 `enabled/topic_names/publish_count/bus`；关闭时结果结构不变 |

### 7.2 三项 Core 边界问题

1. **register 契约校验**：手工 spec 也强制经 `aero_topic_spec` 规范化校验；未知字段、重复字段、
   非法尺寸/单位/年龄均报 `aero:bus:invalidSpec`。
2. **latest 仅诊断**：必须显式传 `'diagnostic'`，否则报 `aero:bus:diagnosticRequired`；
   正式消费者使用 `read`/`tryRead`。
3. **真值访问**：文档与注释明确 `truth/` 前缀只是命名约定，不是访问控制；真实状态仍只由
   Sensor 回调与授权测试适配器读取；普通 Topic 发布真值标志会报 `aero:bus:groundTruthForbidden`。
4. **Latest-Value 安全**：`publish` 必须携带 `now` 且 `now >= available_time_s`，提前发布未来
   消息会报 `aero:bus:notYetAvailable`，避免覆盖仍有效的旧样本。

### 7.3 集成验证（全新 clone @ `381e58b`）

`matlab -batch "run('tests/run_tests.m')"` → **25/25 通过，6.73 s，退出码 0**。

新增 `test_communication_integration` 覆盖：

- Sensor 200 Hz / Navigation 100 Hz / Control 125 Hz / Latch 50 Hz 的事件计数（40/20/25/10）；
- t=0.04 同刻顺序 `baro sample → baro publish → nav → control → latch → guard`；
- 测量在可用事件才发布（`available = sample + delay`），可用前读取报错，过期读取报错；
- Navigation 只在 `tryRead` 成功时更新，从不读取未可用数据；
- 计算命令与锁存命令发布到不同 Topic，序号 18 vs 10，且结果中存在 computed ≠ held；
- 未注册的测量/命令 Topic 与非法契约在启动时失败；
- 默认关闭通信时数值结果与旧版完全一致（`isequaln`），空通信配置不产生发布。

### 7.4 兼容性

- 未启用通信时：`result` 无 `communication` 字段、无 Bus 创建、数值行为与 `cce1b32` 前一致。
- 未修改事件时序、`ode45` 分段、执行器锁存与 Aircraft 动力学；原有 24 项测试全部保留并通过。

### 7.5 未解决风险

1. **结果内含 Bus 句柄**：启用通信时 `result.communication.bus` 是 handle 对象，持久化 `.mat`
   的跨版本兼容性未验证（默认关闭，不影响旧流程）。
2. **Latest-Value 限制**：只保留最后一条；历史回放/逐样本消费不在本轮。
3. **契约只校验结构**：不校验物理量程与单位换算。
4. **命名非权限**：`truth/` 前缀不能阻止恶意模块注册；真实隔离依靠模块纪律与后续权限设计。

### 7.6 下一阶段

`Communication-Validation`（未开始）：跨更多模块的全量回归、错误路径与性能/容量验证；
通过后再由 NEXUS 决定合并 main。
