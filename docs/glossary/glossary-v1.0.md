# 术语表 v1.0（Ubiquitous Language）

> 生效日期：2026-10-26　维护人：杨杰（PO）　复核人：曹文
> 用途：统一团队与代码、文档、看板、答辩中使用的中英文术语，避免同物异名。
> 规则：新增术语必须在本表登记；代码中的类名 / 事件名以本表"代码标识"为准，中文解释只作说明。

## 1. DDD 通用术语

| 中文 | 英文 | 解释 | 代码标识 |
| --- | --- | --- | --- |
| 限界上下文 | Bounded Context | 一个模型只在一个边界内保持唯一含义；本项目有 identity、qa、reputation、discovery 四个 | `co-identity`、`co-qa`、`co-reputation`、`co-discovery` |
| 共享内核 | Shared Kernel | 多个上下文共同依赖的最小通用类型集合，不含业务规则 | `co-shared-kernel` |
| 聚合 | Aggregate | 一致性边界，一次事务只修改一个聚合 | 见聚合清单 |
| 聚合根 | Aggregate Root | 聚合的唯一外部入口，负责守护不变量 | `User`、`Course`、`Question`、`Answer`、`Vote`、`Comment`、`ReputationAccount`、`Bounty`、`BadgeAward`、`Notification` |
| 值对象 | Value Object | 无标识、不可变、按值比较 | `Title`、`MarkdownBody`、`Tags`、`TagName`、`TargetType`、`BountyStatus`、`ReputationReason` |
| 领域事件 | Domain Event | 聚合内已发生事实的描述，命名用过去式 | `QuestionPostedEvent` 等 |
| 集成事件 | Integration Event | 跨上下文发布的事件契约，实现 `IntegrationEvent`，经发件箱投递 | `co-shared-kernel` |
| 事务性发件箱 | Transactional Outbox | 事件与状态变更同事务落库，提交后再投递，保证不丢事件 | ADR-002 |
| 命令 | Command | 修改状态的意图，动词开头 | `RegisterUserCommand`、`PostQuestionCommand` |
| 读模型 | Read Model | 为查询优化的投影，可重建 | `dis_search_doc`、`QuestionDetailView` |
| 投影 | Projection | 订阅事件并折叠成读模型 | `SearchIndexProjector`、`NotificationProjector` |
| 不变量 | Invariant | 聚合在任何时刻都必须成立的业务规则 | 见聚合清单第 2 节 |
| 策略 | Policy | "当某事件发生时，执行某动作"的反应规则 | 采纳 → 计分 → 徽章 → 悬赏结算 |
| 软删除 | Soft Delete | 只标记删除时间，不物理删除，查询侧过滤 | ADR-006 |
| 幂等 | Idempotency | 同一事件重复投递不产生重复副作用 | 账本唯一键、`(sourceEventId, recipientId)` |
| 反腐败层 | Anti-Corruption Layer | 隔离外部模型、避免其污染本上下文的翻译层 | 本项目用 `api` 包替代 |

## 2. 业务术语

| 中文 | 英文 | 定义 | 所属上下文 | 代码标识 |
| --- | --- | --- | --- | --- |
| 学号 / 工号 | username | 4–20 位字母或数字，登录凭据，属个人信息，不对外展示 | identity | `User.username` |
| 昵称 | displayName | 2–20 字符、不含 `@` 与空格的公开名称，全局唯一，用于 @提及 | identity | `User.displayName` |
| 校园认证 | verified | 学院管理员确认该账号属于校内成员，展示认证徽章 | identity | `User.verified` |
| 课程成员 | CourseMember | 用户与课程的从属关系，是采纳、认证、悬赏的授权依据 | identity | `CourseMember` |
| 课程角色 | CourseRole | 课程内的身份：学生 / 教师（助教在实现中按教师管理） | identity | `CourseRole` |
| 问题 | Question | 一次提问，含标题、Markdown 正文、1–5 个标签，可关联课程 | qa | `Question` |
| 回答 | Answer | 对某个问题的回复，独立聚合，仅通过 `questionId` 关联问题 | qa | `Answer` |
| 采纳 | Accept | 提问者或课程管理者选中一个回答作为最佳答案，全问题唯一且不可更改 | qa | `Question.accept` / `acceptedAnswerId` |
| 教师认证 | Endorse | 课程教师/助教给优质回答打上认证标记 | qa | `Answer.endorse` / `endorsedBy` |
| 投票 | Vote | 对问题或回答的赞成 / 反对 / 撤销，同一用户对同一目标只有一条记录 | qa | `Vote.cast`、`Vote.change` |
| 投票目标 | TargetType | 投票对象类型：问题或回答 | qa | `TargetType` |
| 评论 | Comment | 问题下的短回复，可含 @提及 | qa | `Comment` |
| @提及 | Mention | 正文中 `@昵称` 的写法，单条最多提醒 5 人 | discovery | `MentionParser` |
| 标签 | Tag | 问题分类，1–5 个，用于检索与相关问题推荐 | qa | `Tags`、`TagName` |
| 正文渲染 | Markdown Body | 用户输入的 Markdown 文本，展示前渲染为 HTML | qa | `MarkdownBody`、`ContentRenderer` |
| 声誉 | Reputation | 贡献度积分，由回答被采纳与被投票产生，用于等级与榜单 | reputation | `ReputationAccount` |
| 声誉流水 | Ledger Entry | 每次声誉变动的不可变记录，用于对账 | reputation | `LedgerEntry` |
| 每日上限 | Daily Cap | 每人每自然日（UTC）因被投票获得的正向声誉最多 200 | reputation | `ReputationRulebook.DAILY_VOTE_CAP` |
| 悬赏积分 | Bounty Points | 与声誉分离的"货币"，开户时按角色发放，可用于悬赏 | reputation | `initialPoints` |
| 悬赏 | Bounty | 课程管理者为问题挂出的积分奖励，10–500 分、最长 14 天 | reputation | `Bounty` |
| 悬赏状态 | Bounty Status | `OPEN` → `AWARDED`（采纳）或 `EXPIRED`（到期退款），终态不可变 | reputation | `BountyStatus` |
| 徽章 | Badge | 达成条件后授予的荣誉标记，同一徽章只授予一次 | reputation | `BadgeAward`、`BadgeType` |
| 排行榜 | Leaderboard | 周榜 / 月榜 / 学期榜来自 Redis ZSET，总榜读账户表 | reputation | `LeaderboardService` |
| 搜索文档 | Search Document | 由问答事件投影出的可检索记录，表名 `dis_search_doc` | discovery | `SearchDocument` |
| 相关问题 | Related Question | 按标签集合 Jaccard 相似度排序的推荐列表，同分时得分高、已采纳优先 | discovery | `RelatedQuestionPolicy` |
| 站内通知 | Notification | 把上游事件转换为用户可见提醒，6 种类型 | discovery | `Notification`、`NotificationType` |

## 3. 易混淆对照

| 易混淆 | 正确用法 |
| --- | --- |
| "积分" | 区分"声誉 reputation"与"悬赏积分 bounty points"，不可混称 |
| "删除" | 一律指软删除（`deletedAt` 置位），物理删除只由运维脚本执行 |
| "采纳" vs "教师认证" | 采纳由提问者/课程管理者决定且唯一；教师认证可多个，仅表示教师背书 |
| "通知" vs "消息" | 本项目只有站内通知，不含私信功能 |
| "搜索索引" | 指 `dis_search_doc` 投影表，不是 Elasticsearch 集群 |

## 4. 变更记录

| 版本 | 日期 | 变更内容 | 修改人 |
| --- | --- | --- | --- |
| v1.0 | 2026-10-26 | 首次发布：DDD 通用术语 16 条、业务术语 26 条、易混淆对照 5 条 | 杨杰 |
