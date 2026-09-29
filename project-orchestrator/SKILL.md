---
name: project-orchestrator
description: 软件项目开发流程的"总控"，负责管理完整开发流水线。每当用户提出"开发一个项目 / 做一个系统 / 实现一个应用"等对软件进行端到端开发的需求时，务必先调用本技能来调度后续 14 个阶段技能（requirements-analysis、product-design、architecture-design、database-design、api-design、ui-design、development、code-review、testing、debugging、security-review、performance-optimization、documentation、release），而不是直接开始写代码。也用于判断项目当前处于哪个阶段、检查阶段是否完成、决定是否进入下一阶段、以及发现问题时回退到对应阶段。
---

# project-orchestrator

软件项目开发流程的总控。本技能**不直接负责具体开发**，而是负责调度、检查与推进整个流水线。

## 核心目标

> 让 AI 按照软件工程流程开发，而不是拿到需求后直接开始写代码。

## 职责

- 判断项目当前处于哪个阶段
- 调用对应阶段的 Skill
- 检查上一阶段是否完成（产出物齐全且合格）
- 判断是否可以进入下一阶段
- 管理阶段之间的输入和输出（前一阶段产出作为后一阶段输入）
- 发现问题时回退到对应阶段，而不是带病推进
- 维护项目开发状态（阶段进度、产出物清单）

## 完整流水线

顺序执行以下 14 个阶段：

```
requirements-analysis  搞清楚做什么
product-design         搞清楚用户怎么用
architecture-design    搞清楚系统怎么实现
database-design        搞清楚数据怎么存
api-design             搞清楚前后端怎么通信
ui-design              搞清楚界面怎么做
development            把设计写成代码
code-review            检查代码有没有问题
testing                验证功能是否正确
debugging              解决 Bug
security-review        检查安全问题
performance-optimization  解决性能问题
documentation          把项目说明白
release                把项目发布出去
```

## 阶段状态机

每个阶段处于以下状态之一：

- `not-started` 未开始
- `in-progress` 进行中
- `done` 完成（已产出并通过检查）
- `blocked` 阻塞（缺少前置输入）

推进规则：

1. 只有当前阶段为 `done` 才能进入下一阶段。
2. 阶段完成后必须检查其**关键产出物**是否存在且满足质量要求。
3. 若后续阶段发现上游有问题，触发**回退**：将上游阶段状态置回 `in-progress`，修复后重新标记 `done` 再继续。

## 触发与入口

当用户给出一个开发需求时：

1. **确认意图**：明确开发范围、目标、约束。若描述含糊，先在 requirements-analysis 阶段澄清。
2. **初始化状态文件**：记录当前阶段与产出物清单。
3. **从 requirements-analysis 开始**进入流水线。
4. 每个阶段结束向用户汇报该阶段结论，征得确认（或按用户偏好自动推进）后再进入下一阶段。

## 阶段关键产出物清单

检查某阶段是否完成时，至少验证其核心产出：

| 阶段 | 关键产出物 |
| ---- | ---------- |
| requirements-analysis | 需求规格说明（功能需求、非功能需求、业务规则、验收标准） |
| product-design | 产品方案（功能模块、用户流程、页面结构、状态与异常流程） |
| architecture-design | 架构设计（技术栈、模块划分、部署方式、扩展方案） |
| database-design | 数据库设计（表结构、字段、索引、关系、约束） |
| api-design | API 契约（URL、方法、参数、响应、错误码、权限） |
| ui-design | 界面设计（页面、组件、交互、各状态） |
| development | 可运行的代码，编译/运行通过 |
| code-review | 复查报告（问题分级清单，无 Critical/Major 未处理） |
| testing | 测试用例 + 测试执行结果，符合需求 |
| debugging | 根因分析 + 最小修复 + 回归通过 |
| security-review | 安全检查报告（高风险项清零） |
| performance-optimization | 性能分析 + 优化方案/结果 |
| documentation | 各文档（README 等） |
| release | 已构建、已部署、健康检查与冒烟通过 |

## 回退处理

发现以下情况时回退到对应阶段：

- 需求含糊或变更 → 回退 requirements-analysis
- 界面与产品方案不符 → 回退 product-design / ui-design
- 代码与设计不符 → 回退 development
- 测试暴露问题 → 进入 debugging，若根因在设计层则回退相应设计阶段
- 安全/性能问题需要改设计 → 回退对应阶段后重跑受影响阶段

回退遵循**最小回溯**原则：只回退到问题根因所在阶段，不无谓重跑已合格的阶段。

## 常识

- 先理解，再编码：不要在未经各设计阶段前直接写代码。
- 每个阶段都应向用户呈现可审阅的产出，便于确认与纠偏。
- 维护一份阶段进度记录，避免重复劳动。