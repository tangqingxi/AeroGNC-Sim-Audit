# Simulation-Kernel — 阶段报告（最终整改）

| 项 | 值 |
|---|---|
| 阶段名称 | `Simulation-Kernel` |
| 状态 | `accepted`（NEXUS 于 2026-11-07 判定 ACCEPT；AUDIT 状态由 NEXUS 决定更新，技术内容未改） |
| DEV | <https://github.com/tangqingxi/AeroGNC-Sim> |
| DEV 分支 / HEAD | `main` / `848c0710ccc553047ad20a2528a45a42e2a2f458`（已推送，全新 clone 复验通过） |
| 上轮 DEV | `2d9b9bffad6051fe6ceb66cf489ad85f175fe7fc` |
| AUDIT | <https://github.com/tangqingxi/AeroGNC-Sim-Audit> |
| 环境 | MATLAB R2025b `25.2.0.2998904`，PCWIN64；无额外工具箱 |

## 1. 本轮整改内容

### 1.1 Results 三个缺陷（NEXUS 提出）

| 缺陷 | 修复 |
|---|---|
| 临时文件被重复计数 | 统计活跃用量时排除本次运行的临时文件，只把临时文件自身大小计入投影（`aero_results_usage(root, excludePath)`）；新结果只计一次 |
| `movefile(...,'f')` 可能覆盖已有证据 | 目标已存在即报 `aero:results:exists` 并拒绝；写入前、改名前后都检查；原文件字节与 SHA256 不变 |
| 仅按字符串前缀判断路径归属 | 新增 `aero_results_path`：词法折叠 `.`/`..`、根目录边界判定（`root + filesep`，排除同前缀目录）、拒绝 UNC、逐级拒绝符号链接/Junction（Windows 用 .NET `ReparsePoint` 属性），且校验发生在任何 `mkdir` 之前 |

补充的持久化安全：

- 每次写入使用同目录唯一临时文件（UUID），不同运行互不冲突；失败只清理本次临时文件，不触碰其他任务文件。
- 写盘仍为"先临时文件、后改名"，超限或覆盖冲突时删除本次临时文件并报错。
- 崩溃残留的 `.aero-tmp-*` 不会被自动删除（避免误删他人文件），会计入活跃用量——这是已知取舍，见风险 5。

### 1.2 目录整理

| 之前 | 之后 |
|---|---|
| `src/` 下 14 个内核文件平铺 | `src/Simulation/`（10 个：`aero_sim`、`aero_schedule`、`aero_integrate`、配置/校验/测量/根目录/环境） |
| Results 代码混在 `src/` | `src/Simulation/Results/`（5 个：`aero_save_result`、`aero_results_path`、policy/usage/check） |
| `models/` 与正式代码同级 | `tests/fixtures/`（3 个解析测试替身） |
| 各入口各自 `addpath('src')` | 根目录 `aero_setup.m` 显式登记已实现模块目录（不递归 addpath） |
| 项目根按固定层级推算 | `aero_project_root` 向上查找 `ARCHITECTURE.md`，目录调整后仍可靠 |

未创建 `Aircraft/Components/GNC/Scenarios` 空目录；未使用 `.gitkeep`。文件移动主要用 `git mv`；因内容同时重写，部分文件在 Git 中显示为删除+新增（历史仍线性，见风险 4）。

### 1.3 代码可读性

- `aero_sim.m` 按"校验 → 调度 → 日志预分配 → 主循环（积分 / 同刻事件 / 记录）→ 结果 → 持久化"分段，事件动作改用 `'sample' | 'publish' | 'step'` 名称，去掉数字动作编码。
- `aero_schedule.m` 用命名常量与注释说明"时刻由 k 直接计算、`[t0,tf)` 半开区间、同刻事件排序规则"。
- 事件计数改为显式计数器（原实现存在 O(E²) 的重复统计写法）。
- 中文注释覆盖 `src/` 全部内核与 Results 文件、`aero_setup`、测试入口与新增测试；物理模型注明单位、状态顺序与解析解。未修改的旧测试文件保持原注释，避免无意义的历史差异。

### 1.4 验证过程中发现并修复的构建缺陷

目录整理后首次推送时，`.gitignore` 中的 `results/` 规则在大小写不敏感的 Windows 工作区里同时匹配了
`src/Simulation/Results/`，导致新文件 `aero_results_path.m` 未被 Git 跟踪；本地测试仍能通过，但全新
clone 会因缺少该函数而失败。修复：把忽略规则锚定为仓库根目录 `/results/`，并补交该文件（`848c071`）。
此后所有验证均改为**从 GitHub 全新 clone** 执行，以证明远端内容自洽。

## 2. 验证记录

### 2.1 回归测试

从 `https://github.com/tangqingxi/AeroGNC-Sim` 全新 clone（HEAD `848c071`）后执行
`matlab -batch "run('tests/run_tests.m')"`，退出码 0：

```text
14/14 tests passed in 5.51s
```

13 项原有测试全部保留且阈值未变，新增 `test_results_safety`（覆盖保护、哈希不变、路径穿越、绝对越界、同前缀、Junction、临时文件残留）。`run_tests` 支持 `SKIP` 计数：Junction 无法创建时该测试标记为跳过而非通过（本机实际执行并通过）。

### 2.2 数值一致性（阈值与上轮相同，未放宽）

| 模型 | 解析 | 最大绝对误差 | 阈值 |
|---|---|---|---|
| `dy/dt=u` 阶跃锁存 | y(1)=−0.5 | 2.22e-16 | 1e-9 |
| `dx/dt=−2x+3u` | 1.197155 | 1.38e-11 | 1e-9 |
| 三状态对角模型 | 解析式 | 1.50e-11 | 1e-9 |
| 自适应振荡段 | −7.931 | 7.35e-6 | 1e-5 |

### 2.3 时序与因果（未退化）

- 125/200/800 Hz → 事件数 125/200/800，漂移 0；73/127 Hz 5 s → 365/635，漂移 8.9e-16。
- 同刻事件顺序仍为 `sample → publish → nav → guidance → control → latch`。
- 测量延迟 50 ms / 控制 100 Hz：8 条测量，无未来数据泄漏。
- 指令锁存：控制 100 Hz / 锁存 40 Hz，3 次锁存值变化。

### 2.4 Results 边界（新增，真实触发代码路径）

| 场景 | 结果 |
|---|---|
| 合法保存（内存 → 磁盘） | 成功，用量等于文件大小 |
| 单次计数 | 现有 900 B + 新结果 214 B，上限设为 900+214+50 时保存通过，用量等于 900+214 |
| 缩放 8 GB 预警 | 触发 `aero:results:capacity`，仍完成保存 |
| 缩放 10 GB 限制 | 报 `aero:results:limit`，无结果文件、无残留 |
| 已存在文件拒绝覆盖 | 报 `aero:results:exists`，原文件 SHA256 不变 |
| `../` 穿越与嵌套穿越 | 报 `aero:results:outside`，根目录外无文件、无新目录 |
| 绝对路径越界 | 报 `aero:results:outside` |
| 同前缀路径（`results-evil`） | 报 `aero:results:outside` |
| Junction 指向根目录外 | 报 `aero:results:link`，外部目录无写入 |
| 失败残留 | 本次临时文件被清理，其他任务的临时文件保留 |
| 内存 / 持久化模式 | 内存模式零文件；持久化模式生成 `saved_path` 且 `results_check.status = ok` |

### 2.5 示例

在全新 clone 中执行 `matlab -batch "run('examples/demo_first_order.m')"`：x(2)=1.425319，解析误差 1.36e-10（证据 `evidence/demo-run.txt`）。

## 3. 验收对照

| 验收项 | 结果 | 依据 |
|---|---|---|
| Results 容量不重复计数、超限拒绝 | PASS | §2.4 |
| 已有结果默认不可覆盖 | PASS | §2.4，SHA256 不变 |
| 拒绝越界与链接路径 | PASS | §2.4（Junction 实测） |
| Results 边界回归 | PASS | `test_results_policy`、`test_results_safety` |
| 目录职责清楚、无空目录 | PASS | §1.2，`git ls-tree` |
| 移动后入口/测试/示例正常 | PASS | §2.1、§2.5，全新 clone 14/14 |
| 主要流程可读、中文注释 | PASS | §1.3；`src/` 无英文说明注释（仅 API 名称保留原文） |
| 冻结架构未改 | PASS | `ARCHITECTURE.md` SHA256 `7A17A41B…5949` 不变 |
| 数值与时序未退化 | PASS | §2.2、§2.3 |
| DEV/AUDIT 一致 | PASS | `check_handoff.py`（结果见交付消息） |

## 4. 未解决风险

1. **容量仍为缩放验证**：10 GB/8 GB 阈值由常量断言 + 缩放阈值行为测试覆盖，未实际写满 8 GB；结论是"同一代码路径已验证"，不是大规模实测。
2. **链接检查平台相关**：Windows 用 .NET `ReparsePoint`（已实测 Junction）；非 Windows 走 Java `getCanonicalPath` 回退，未在本机验证。
3. **TOCTOU 窗口**：校验与 `mkdir`/改名之间存在极短时间窗；假定单用户工作站、无恶意并发。
4. **Git 重命名可追溯性**：内容重写的文件显示为删除+新增，`git log --follow` 可能断链；提交历史本身线性、可审计。
5. **崩溃残留临时文件**：不自动清理（避免误删其他任务），会计入活跃用量，需要人工复核。
6. **分组容差与半开区间**：行为未变——相差 `<1e-12·max(1,tf)` 的事件合并；`tf` 不触发组件更新。
7. **函数句柄入结果**：`result.config` 含句柄，跨 MATLAB 版本加载 `.mat` 可能失效（数值不受影响）。

## 5. 提交与文件

| 仓库 | 提交 |
|---|---|
| DEV | `346b86e` Results 修复 + 目录整理 + 中文注释；`9d5f8ed` README/AGENTS；`d19381d` 持久化端到端测试；`8e195b8` 解析模型中文头；`848c071` 补交 `aero_results_path.m` 并锚定 `results` 忽略规则（HEAD） |
| DEV 文件 | `src/Simulation/*`、`src/Simulation/Results/*`、`tests/*`、`tests/fixtures/*`、`aero_setup.m`、`README.md`、`AGENTS.md` |
| AUDIT | 本报告 + `HANDOFF.md` + 测试/示例日志 + 证据清单 |

## 6. 是否达到 NEXUS 验收条件

按本轮验收表逐项核对（§3），**达到 `ready-for-review` 条件**；但以下两点需 NEXUS 明确判断：

1. §4 风险 1（未写满 8 GB）与风险 2（非 Windows 链接回退未验证）是否接受为"缩放验证足够"；
2. 目录重组造成部分文件以删除+新增记录，是否满足可追溯性要求。

NEXUS 若判定 `ACCEPT`，建议下一阶段按 `ARCHITECTURE.md` §9 进入 `SHC09-Aircraft`；若 `REJECT`，请指明需返工的条目与复现命令。
