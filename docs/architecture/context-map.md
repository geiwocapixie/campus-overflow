# 上下文映射（Context Map）

> 版本：v1.0　整理日期：2026-10-26　整理人：曹文
> 配套：[事件风暴清单](event-storming.md)、[聚合清单](aggregates.md)、[ADR-001 模块化单体](../adr/001-模块化单体.md)
> 图示：[context-map.png](context-map.png)（可编辑源：`context-map.svg`）

## 1. 上下文清单

| 上下文 | 代码模块 | 职责 | 是否核心域 |
| --- | --- | --- | --- |
| 身份 identity | `co-identity` | 账号、登录会话、昵称、校园认证、课程与课程成员授权 | 支撑域 |
| 问答 qa | `co-qa` | 问题、回答、投票、评论、采纳、软删除，及全部内容事件发布 | **核心域** |
| 声誉 reputation | `co-reputation` | 声誉账户与账本、徽章、悬赏、排行榜、对账 | 支撑域 |
| 内容发现 discovery | `co-discovery` | 搜索索引投影、相关问题推荐、站内通知 | 通用域 |
| 共享内核 shared-kernel | `co-shared-kernel` | `IntegrationEvent`、发件箱、`Guard`、`Actor/Role`、分页、领域异常 | 不是上下文 |
| 前端 frontend | `frontend` | 浏览器端页面，只通过 HTTP 契约访问后端 | 外部系统 |

## 2. 关系一览

| # | 上游 → 下游 | 集成方式 | 模式 | 一致性 | 失败处理 |
| --- | --- | --- | --- | --- | --- |
| R1 | identity → qa | 同步调用 `IdentityApi`（`UserSummary`、`CourseSummary`） | 开放主机服务 OHS + 发布语言 | 强一致（同一事务内只读） | 查询失败则该命令快速失败，不落库 |
| R2 | identity → reputation | 同步调用 `IdentityApi` 取昵称、角色 | OHS + 发布语言 | 强一致（只读） | 排行榜展示降级为"匿名用户" |
| R3 | identity → discovery | 同步调用 `IdentityApi` 取昵称、解析 @昵称 → 用户 | OHS + 发布语言 | 强一致（只读） | 无法解析的 @ 直接忽略，不阻断评论提交 |
| R4 | qa → reputation | 异步集成事件（`AnswerSubmitted`、`AnswerAccepted`、`AnswerEndorsed`、`VoteCast`） | Customer/Supplier（qa 为上游）+ 发布语言 | 最终一致 | 事务性发件箱至少一次投递；消费侧幂等（账本唯一键、徽章存在性检查） |
| R5 | qa → discovery | 异步集成事件（`QuestionPosted/Edited`、`AnswerSubmitted`、`AnswerAccepted`、`VoteCast`、`ContentDeleted`） | Customer/Supplier + 发布语言 | 最终一致 | `ProcessedEventStore` 先占位后更新投影，重复消费被丢弃 |
| R6 | reputation → discovery | 异步集成事件（`ReputationChanged`、`BadgeAwarded`、`BountyOpened/Awarded/Closed`） | Customer/Supplier + 发布语言 | 最终一致 | 同上，且通知以 `(sourceEventId, recipientId)` 幂等 |
| R7 | discovery → qa | 同步只读 `QaQueryApi`（取问题标题与链接用于通知/搜索展示） | OHS，只读依赖 | 强一致（只读） | 查不到则通知中退化为问题 ID |
| R8 | 所有上下文 → shared-kernel | 编译期依赖 | 共享内核 Shared Kernel | 不涉及 | 内核只允许放稳定通用类型；业务规则一律留在各上下文 |

> R4–R6 三类事件全部实现 `IntegrationEvent`，由发件箱在本地事务提交后投递（ADR-002），
> 因此问答上下文的"采纳"命令不会等待声誉计分与通知落库；页面刷新才能看到声誉变化属预期行为。

## 3. 边界判定理由

1. **qa 是核心域，事件是唯一出口**：投票、采纳、删除等状态变化只在 qa 内完成，其他上下文通过事件得知结果，避免下游逻辑反向污染核心域。
2. **reputation 独立**：声誉规则（+15 / +10 / −2 / 每日 200 上限）与账本、悬赏、徽章耦合紧密，且需要独立对账，故不与 qa 合并；问答侧完全不感知"分数怎么算"。
3. **discovery 只做读模型**：搜索与通知均为可重建的投影，索引损坏可整体重建，不影响问答主流程，因此列为通用域并允许替换实现（`SearchPort` 有 MySQL 全文与 LIKE 两套适配器）。
4. **identity 不反向依赖业务上下文**：用户与课程是授权事实来源，qa / reputation / discovery 只能读取，不得回写身份数据。
5. **shared-kernel 最小化**：只放跨模块稳定的类型（事件契约、发件箱、异常、安全主体）。ADR-001 用 ArchUnit 适应度函数守护"跨模块只经 `api` 包交互"。

## 4. 禁止事项（评审检查点）

- 禁止跨上下文直接引用对方的 `domain` 或 `infrastructure` 包（只能经 `api` 包或事件）。
- 禁止在 discovery / reputation 中复制问答业务规则（如"采纳唯一"），需要判定时通过 `QaQueryApi` 读取。
- 禁止用同步调用串联写操作（例如采纳时同步调用声誉计分），跨上下文写一律走事件。
- 禁止让 `co-shared-kernel` 引入任何领域的业务概念（如 `Question`、`Bounty`）。

## 5. 变更记录

| 版本 | 日期 | 变更内容 | 修改人 |
| --- | --- | --- | --- |
| v1.0 | 2026-10-26 | 首次发布：4 个上下文 + 共享内核，8 条关系（R1–R8） | 曹文 |
