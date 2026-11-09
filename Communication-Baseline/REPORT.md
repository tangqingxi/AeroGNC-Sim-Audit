# Communication-Baseline — Communication-Core 阶段报告

| 项 | 值 |
|---|---|
| 阶段名称 | `Communication-Core`（Communication-Baseline 第一阶段） |
| 状态 | `ready-for-review`（仅核心与单元测试；未接入 `aero_sim`） |
| DEV 分支 | `feat/communication-baseline` @ `cce1b328e6a414500d8bf0fb9e70e66645650025` |
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
