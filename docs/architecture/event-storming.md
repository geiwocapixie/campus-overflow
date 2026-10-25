# 事件风暴清单（Event Storming）

> 版本：v1.0　整理日期：2026-10-26　整理人：曹文（SM）　参与：全体 5 人
> 上游依据：[ADR-001 模块化单体](file:///C:/Users/曹文/Desktop/软件工程/co-new/CampusOverflow-学生初始框架/campus-overflow-student/docs/adr/001-模块化单体.md)、[ADR-002 事务性发件箱](file:///C:/Users/曹文/Desktop/软件工程/co-new/CampusOverflow-学生初始框架/campus-overflow-student/docs/adr/002-事务性发件箱.md)、需求文档 `docs/backlog/product-backlog.md`
> 说明：本清单按"命令 → 聚合 → 领域事件 → 策略/订阅方 → 读模型"的顺序梳理，事件名取自源码 `api/event` 包，未命名的事件保持中文描述并标注来源。

## 1. 工作坊记录

| 项目 | 内容 |
| --- | --- |
| 形式 | 线上白板 + 源码走查（90 分钟） |
| 输入材料 | 3 份用户访谈（U1–U3）、US-01~US-16 故事、`docs/学生任务清单.md` |
| 产出 | 本清单、[聚合清单](aggregates.md)、[上下文映射](context-map.md)、[术语表 v1.0](../glossary/glossary-v1.0.md) |
| 争议点 | 采纳是否跨聚合、投票是否独立聚合、悬赏积分与声誉是否同账本（见第 5 节） |

## 2. 主线流程（按时间轴）

### 2.1 身份上下文（identity）

| 命令 | 发起者 | 聚合 | 领域事件 | 后续策略/订阅方 |
| --- | --- | --- | --- | --- |
| 注册账号 | 学生本人 | User | `UserRegisteredEvent` | 声誉上下文开户（初始悬赏积分）；通知中心无动作 |
| 管理员创建教职工 | 管理员 | User | `UserRegisteredEvent` | 同上（角色为 TEACHER / TA / ADMIN） |
| 校园身份认证 `User.verify` | 管理员 | User | 无事件（同步更新） | 前端展示"已认证"徽章 |
| 调整角色 `User.changeRole` | 管理员 | User | 无事件 | 授权判定立即生效 |
| 禁用账号 `User.disable` | 管理员 | User | 无事件 | 登录被拒（`canLogin`） |
| 建课 / 加课程成员 | 管理员、教师 | Course、CourseMember | 无事件 | 课程内采纳、教师认证、悬赏的授权依据 |

### 2.2 问答上下文（qa）——核心流程

| # | 命令 | 发起者 | 聚合 | 领域事件 | 后续策略/订阅方 |
| --- | --- | --- | --- | --- | --- |
| 1 | 发布问题 `Question.post` | 学生 | Question | `QuestionPostedEvent` | Discovery 建搜索文档 |
| 2 | 编辑问题 `Question.edit` | 作者 / 课程管理者 | Question | `QuestionEditedEvent` | Discovery 更新搜索文档 |
| 3 | 提交回答 `Answer.submit` | 已登录学生 | Answer | `AnswerSubmittedEvent` | Reputation 计分；Discovery 通知提问者 + 重建索引 |
| 4 | 采纳回答 `Question.accept` | 提问者 / 课程管理者 | Question（校验）+ Answer | `AnswerAcceptedEvent` | Reputation 给回答者 +15（自问自答 +0）、评估徽章、结算悬赏；Discovery 通知被采纳者、索引置顶 |
| 5 | 教师认证 `Answer.endorse` | 课程教师 / 助教 | Answer | `AnswerEndorsedEvent` | Reputation 评估"名师认可"徽章；Discovery 通知回答者 |
| 6 | 发表评论（含 @提及） | 已登录用户 | Comment | `CommentPostedEvent` | Discovery 解析 @昵称（≤5 人）生成通知 |
| 7 | 投票 / 改投 / 撤销 `Vote.cast` `Vote.change` | 已登录用户 | Vote | `VoteCastEvent`（携带 oldValue/newValue） | Reputation 按"新效果 − 旧效果"计分并受每日上限约束；Discovery 更新热度 |
| 8 | 软删除内容 `Question.softDelete` `Answer.softDelete` | 作者 / 课程管理者 | Question、Answer | `ContentDeletedEvent` | Discovery 从索引移除；历史通知保留 |

### 2.3 声誉上下文（reputation）

| 命令 | 发起者 | 聚合 | 领域事件 | 后续策略/订阅方 |
| --- | --- | --- | --- | --- |
| 开户 `ReputationAccount.open` | 系统（订阅 `UserRegisteredEvent`） | ReputationAccount | 无 | 按角色发放初始悬赏积分 |
| 记声誉 `applyReputation` | 系统（订阅问答事件） | ReputationAccount + LedgerEntry | `ReputationChangedEvent` | 排行榜更新；徽章阈值检查 |
| 授予徽章 `BadgePolicy.forAcceptedCount / forReputation` | 系统 | BadgeAward | `BadgeAwardedEvent` | Discovery 通知本人 |
| 发起悬赏 `Bounty.open` | 课程管理者 | Bounty | `BountyOpenedEvent` | Discovery 索引标记"悬赏中" |
| 结算悬赏 `Bounty.award` | 系统（订阅 `AnswerAcceptedEvent`） | Bounty | `BountyAwardedEvent` | Discovery 通知获奖者；Reputation 记"赢得悬赏" |
| 到期退款 `Bounty.close` | 定时任务 | Bounty | `BountyClosedEvent` | 解冻发起人积分；Discovery 移除悬赏标记 |
| 对账 `ReconciliationService.reconcile` | 定时任务 / 管理员 | ReputationAccount + Ledger | 无（修正偏差并留痕） | 运维告警 |

### 2.4 内容发现上下文（discovery）

| 触发事件 | 投影/策略 | 产出读模型 |
| --- | --- | --- |
| `QuestionPosted/Edited`、`AnswerSubmitted`、`AnswerAccepted`、`VoteCast`、`ContentDeleted`、`BountyOpened/Awarded/Closed` | `SearchIndexProjector`（先查 `ProcessedEventStore` 去重） | 表 `dis_search_doc` |
| `AnswerSubmitted`、`AnswerAccepted`、`AnswerEndorsed`、`CommentPosted`、`BountyAwarded`、`BadgeAwarded` | `NotificationProjector`（幂等键 `sourceEventId + recipientId`） | 站内通知 |
| `ReputationChangedEvent` | `LeaderboardService.on` | Redis ZSET：周榜 / 月榜 / 学期榜 |
| 用户查询（无事件） | `RelatedQuestionPolicy.jaccard / rank` | 相关问题列表（标签 Jaccard 相似度） |

## 3. 事件契约一览

| 事件 | 所属上下文 | 关键载荷 | 消费方 |
| --- | --- | --- | --- |
| `UserRegisteredEvent` | identity | userId、role、displayName | reputation |
| `QuestionPostedEvent` | qa | questionId、authorId、courseId、tags、title | discovery |
| `QuestionEditedEvent` | qa | questionId、title、tags | discovery |
| `AnswerSubmittedEvent` | qa | answerId、questionId、authorId | reputation、discovery |
| `AnswerAcceptedEvent` | qa | questionId、answerId、answerAuthorId、questionAuthorId | reputation、discovery |
| `AnswerEndorsedEvent` | qa | answerId、authorId、endorsedBy | reputation、discovery |
| `CommentPostedEvent` | qa | commentId、questionId、authorId、正文（用于解析 @） | discovery |
| `VoteCastEvent` | qa | targetType、targetId、targetAuthorId、voterId、oldValue、newValue | reputation、discovery |
| `ContentDeletedEvent` | qa | targetType、targetId、authorId | discovery |
| `ReputationChangedEvent` | reputation | userId、delta、reason、总分 | discovery（排行榜） |
| `BadgeAwardedEvent` | reputation | userId、badgeType | discovery |
| `BountyOpenedEvent` / `BountyAwardedEvent` / `BountyClosedEvent` | reputation | bountyId、questionId、points、winnerId | discovery |

> 全部集成事件实现 `IntegrationEvent`，经事务性发件箱在本地事务提交后投递，至少一次语义；消费方必须幂等（ADR-002）。

## 4. 策略（Policy）与热点

- **采纳 → 计分 → 徽章 → 悬赏结算** 是一条跨上下文的反应链：问答上下文只发布 `AnswerAcceptedEvent`，后续全部由声誉上下文异步完成，问答上下文的采纳事务不等待下游。
- **投票每日上限**：正向增量受"每人每自然日（UTC）200 分"约束，负向不受限；改投按增量计算，避免重复计分。
- **自问自答被采纳不加声誉**（`acceptedDelta` 返回 0），但悬赏奖金按规则退款给发起人。

## 5. 未决问题（Hotspot）

| 编号 | 问题 | 当前决策 |
| --- | --- | --- |
| H-1 | 采纳到底放在 Question 还是 Answer 聚合？ | 放 Question：以 `acceptedAnswerId` 作为单一事实来源，Answer 不重复存"是否被采纳" |
| H-2 | 投票是否并入 Question / Answer？ | 独立 Vote 聚合，按 `targetType + targetId + voterId` 定位，降低并发冲突 |
| H-3 | 悬赏积分与声誉是否同一账本？ | 否：声誉是贡献度，悬赏积分是"货币"，`ReputationAccount` 内以冻结/解冻区分 |
| H-4 | 软删除后历史通知是否撤回？ | 不撤回，仅在搜索索引与列表过滤；保留可追溯性 |
| H-5 | 通知是否做站内实时推送？ | 当前只落库轮询，实时推送留到 Sprint 3 评估 |

## 6. 变更记录

| 版本 | 日期 | 变更内容 | 修改人 |
| --- | --- | --- | --- |
| v1.0 | 2026-10-26 | 首次发布，覆盖 4 个上下文的命令、事件、策略与读模型 | 曹文 |
