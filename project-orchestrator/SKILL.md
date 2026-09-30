---
name: project-orchestrator
description: 软件项目开发流程的"总控"，负责管理完整开发流水线并落地项目状态文件。每当用户提出"开发一个项目 / 做一个系统 / 实现一个应用"等端到端开发需求时，务必先调用本技能调度后续 14 个阶段技能（requirements-analysis、product-design、architecture-design、database-design、api-design、ui-design、development、code-review、testing、debugging、security-review、performance-optimization、documentation、release），而非直接写代码。**强制**：定义并维护项目状态文件（当前阶段/各阶段状态/产出物路径/确认记录/回退历史）；按项目类型分流（全量/精简/增量/仅排查修复）；可跳过阶段规则；人机确认策略；禁止行为（未完成上游设计禁止进 development、带 Critical 禁止 release）。
---

# project-orchestrator

软件项目开发流程的总控。本技能**不直接负责具体开发**，而是负责调度、检查与推进整个流水线。

## 核心目标

> 让 AI 按软件工程流程开发，而不是拿到需求后直接写代码；全程可追踪、可回退、可确认。

## 一、项目状态文件（强制落地）

为每个项目创建并维护 `project-state.md`（或 JSON），贯穿始终。创建于项目启动，随阶段推进更新。**必须包含以下字段**：

```yaml
project: <名称>
type: <项目类型：full / slim / incremental / hotfix>   # 见「四、项目类型分流」
strategy: <人机确认策略：confirm / summary / auto>      # 见「七、人机确认策略」
created_at: <时间>
# ---------- 阶段状态 ----------
phases: &PHASES
  requirements-analysis: { status: done,      start: <t>, done_at: <t> }
  product-design:        { status: in-progress, start: <t>, done_at: null }
  architecture-design:   { status: skipped,   reason: "无复杂架构，精简路径跳过" }
  ...
# ---------- 产出物 ----------
artifacts:
  requirements-analysis: <路径>
  product-design:        <路径>
  ...
# ---------- 确认记录 ----------
confirmations:
  - { when: <t>, phase: product-design, summary: "确认采用任务列表+详情页面", result: accepted }
# ---------- 回退历史 ----------
rollbacks:
  - { when: <t>, from: testing, to: requirements-analysis,
      reason: "需求歧义导致验收冲突", affected_phases: [product-design, ui-design, development, testing] }
```

- `当前阶段`：由 `phases` 中第一个非 done 状态决定。
- 状态字段：`not-started / in-progress / done / blocked / skipped`。
- 状态文件是调度与恢复的依据；中断后据此继续，不重复劳动。

## 二、各阶段输入 / 输出契约（强制）

每个阶段有明确输入与输出契约。**输出字段时间**——阶段完成判定要检查其**必填字段**与**验收标准**。

### 输入契约（前置）
- 要求的输入产物**未完成**则该阶段 `blocked`，不得启动。
- 例如：development 需要 requirements / product / architecture / database / api / ui 全部 valid；缺一即 blocked。

### 输出契约（必填字段 + 验收标准）

| 阶段 | 输出 | 必填字段 | 验收标准（做到才算 done） |
| ---- | ---- | -------- | ------------------------- |
| requirements-analysis | 需求规格 | FR/非功能/范围/验收标准 | 待确认项已闭环；范围清晰 |
| product-design | 产品方案 | 功能模块/流程/页面/权限矩阵/五态 | 可被 architecture/ui 直接引用 |
| architecture-design | 架构设计 | 技术栈/模块边界/部署/选择理由 | 目录与方案可落地 |
| database-design | DB 设计 | 表结构/字段/索引/关系 | 有 DDL 或等价描述 |
| api-design | API 契约 | 方法/路径/Schema/错误码/权限 | 前后端字段无歧义 |
| ui-design | UI 设计 | 页面清单/状态/交互/数据绑定 | 可被 development 直接实现 |
| development | 代码 | 可编译/可运行 | 设计对齐 + 验证证据齐全 |
| code-review | 报告 | 问题分级/结论 | 无 Critical/Major 未处理 |
| testing | 报告 | 用例/执行结果/失败项 | 核心需求有覆盖；失败已登记 |
| debugging | 报告 | 现象/根因/最小修复/回归 | 根因明确、回归通过 |
| security-review | 报告 | 维度结论/高危/结论 | 高危清零或有豁免控制 |
| performance-optimization | 报告 | 测量/定位/前后对比 | 无基准不优化、前后对比有数据 |
| documentation | 文档 | README/快速开始/架构/API/部署/配置 | 新人可照文档跑通主流程 |
| release | 发布报告 | 版本号/门禁/冒烟/回滚 | 门禁全绿、回滚可用 |

## 三、阶段完成判定规则（强制）

什么叫 done（**缺什么不能推进**）：

- 阶段状态置 `done` **仅当**：
  1. 输出契约的**必填字段齐全**；
  2. 该阶段的**质量门槛/验收标准**通过（见各阶段 SKILL 的质量门槛）；
  3. 有**确认记录**（按人机确认策略，见七节）到位；
  4. 阶段 `summary` 已写入状态文件，供下阶段使用。
- **缺项则不能推进**：
  - requirements 未闭环待确认项 → 不进入 product-design
  - 缺 target 验收标准 → 该阶段不得 done
  - 上游设计未 valid → 下游 blocked
- `done` 判定不是"我写完了"，而是"契约字段全 + 质量门槛过 + 已确认"。

## 四、项目类型分流（强制）

启动时先判定项目类型，选择性执行流水线（在状态文件 `type` 标注）：

- **full（全量）**：完整 14 阶段，端到端新系统。
- **slim（精简路径）**：中小项目，只跑必要阶段（需求→产品→架构→DB→API→开发→测试→发布为主），可跳过可选质量阶段（如 performance-optimization）。
- **incremental（增量迭代）**：在既有系统上加功能，从受影响阶段进入（常从 requirements / api / ui 开始），复用已有状态与产出。
- **hotfix（仅排查修复）**：只跑 debugging（+需要的 testing/code-review/security 复核），不重跑整条流水线。

实施：先写 `type`，再据此规划阶段清单；非全量路径在状态文件标注"本类型跳过阶段"。

## 五、可跳过阶段规则（强制）

允许在特定条件下跳过某些阶段，但**必须记录原因**（状态文件 `status: skipped, reason`），且不能因跳过低级缺陷：

- **无 UI**（纯 API/CLI/后台服务）→ 跳过 `ui-design`（及 product 中界面部分）。
- **无 DB / 无持久化** → 跳过 `database-design`。
- **无对外 API**（内部/单进程）→ 可跳过 `api-design`。
- **无性能诉求/小规模** → 可跳过 `performance-optimization`（需说明）。
- **无发布需求**（仅内部库/脚本）→ 可跳过 `release`。
- 规则：跳过**不豁免质量**，被跳阶段所需信息由替代物补齐（如无 UI 则跳过页面的数据绑定部分，但需求/流程仍要明确）；`skipped` 也可因后续暴露需要而重新打开。

## 六、最小回退规则（强制）

回退遵循**最小回溯**：

1. **确定根因阶段**：问题真正出在哪一层（需求/产品/架构/DB/API/UI/实现）。
2. **标记受影响下游**：根因阶段及其之后所有受影响阶段置回 `in-progress`（或 blocked），在状态文件记录 `affected_phases`。
3. **重跑范围**：只重跑"根因阶段 + 受影响下游"，**不重跑**根因之前的已合格阶段。
4. 记录回退 **原因、时间、范围** 到 `rollbacks`。

```
例：testing 发现需求歧义 → 根因 requirements-analysis
   → affected: [product-design, ui-design, development, testing]
   → 重跑这些阶段；requirements 之前阶段不动。
```

## 七、人机确认策略（强制）

为项目设定确认策略（写于状态文件 `strategy`）：

- **confirm（强制确认）**：每个阶段产出需用户明确确认后才推进。适合需求易变/高风险项目。
- **summary（摘要确认）**：每阶段结束时向用户展示一页**摘要结论**，用户无异议即默认通过。适合常规项目（默认）。
- **auto（静默推进）**：低风险/熟悉领域，阶段完成自动推进，仅在用户要求时汇报。需明确的收敛条件与控制点。
- **默认超时策略**：summary 模式如发送摘要后超过**默认时间（如 30 分钟）**无反馈，按"无异议"推进并记录；confirm 模式超时不自动推进，提示用户确认。
- 任意时刻用户可提"暂停/回退/重新确认"覆盖默认策略。

## 八、禁止行为（强制）

- **未完成上游设计 → 禁止进入 development**：requirements/product/architecture/database/api/ui 未全部 valid，不得写生产代码。
- **带 Critical 问题 → 禁止 release**：code-review 未清 Critical/Major、security 高危未清零、testing 失败未处理，一律不得发布。
- 禁止跳过状态检查直接推进；禁止未确认就宣称阶段 done。

## 九、建议增强

### 变更管理（需求/设计变更分级与回灌范围）
- 变更分级：`S（小）/ M（中）/ L（大）`。
  - **S**：细节澄清，只影响当前阶段产出，不需回灌。
  - **M**：字段/接口/流程调整，影响相邻设计与实现 → 回灌到相关阶段（如改 API 字段 → 回灌 ui/db 对应处）。
  - **L**：需求范围/架构调整 → 触发走第六节最小回退流程。
- 每次变更在状态文件记录：变更内容、级别、回灌阶段、影响范围。

### 多仓库 / 前后端分仓协作调度
- 涉及多仓库（如后端 / 前端 / 基础设施分仓）时：
  - 一个**状态文件**统一调度，各阶段标注所属仓库/模块。
  - 依赖链：先完成后端 API/DB 契约，再并行前端按契约实现；公共契约有版本。
  - 明确仓库边界与交付顺序：`契约先行 → 后端实现 → 前端按契约对接 → 集成验证`。
  - 记录每个仓库的产出物路径与当前阶段，避免跨仓不同步。

### 进度汇报模板（每阶段结束固定结构）
每阶段结束向用户展示：
```
## 阶段小结：<阶段名>（<状态>）
- 做了什么 / 关键产出：<一句话 + 产出物路径>
- 结论：<通过 / 需调整 / 已跳过，原因>
- 未决事项：<待确认/待跟进>
- 下一步：<将进入的阶段>
（按确认策略等待：强制确认 / 摘要确认 / 静默推进）
```

## 十、输出（调度产物）

ALWAYS 维护/产出：

- **project-state.md**：见第一节结构，随进度持续更新。
- **阶段小结**：每阶段结束按第九节模板汇报。
- **回退记录 / 变更记录**：追加进状态文件。

## 十一、完整流水线与状态机

顺序执行（项目类型允许时）：

```
requirements-analysis → product-design → architecture-design
 → database-design → api-design → ui-design
 → development → code-review → testing → debugging
 → security-review → performance-optimization
 → documentation → release
```

阶段状态：`not-started / in-progress / done / blocked / skipped`。

推进规则：
- 仅当前阶段 `done` 才能进入下一阶段。
- 下游检查到上游产出合格才接收（输入契约）。
- 后续阶段发现上游问题 → 第六节最小回退。

## 十二、常识

- 先理解，再编码：不要在未经各设计阶段前写代码（硬规则）。
- 状态文件是流水线的"账本"：推进、回退、确认、恢复都依托它。
- 判断一个阶段是否 done，向他一眼是**契约字段全 + 质量门槛过 + 已确认**。