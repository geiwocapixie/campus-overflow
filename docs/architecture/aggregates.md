# 聚合清单（Aggregates）

> 版本：v1.0　整理日期：2026-10-26　整理人：曹文　配套：[事件风暴清单](event-storming.md)、[上下文映射](context-map.md)
> 说明：聚合边界以源码包结构为准（`backend/<模块>-domain`），不变量描述来自各聚合根的类注释与领域测试；"事务边界"列说明一次命令只修改一个聚合根。

## 1. 汇总表

| 上下文 | 聚合根 | 内部实体 / 值对象 | 仓储接口 | 事务边界 |
| --- | --- | --- | --- | --- |
| identity | `User` | 值对象：`Role`、`UserStatus`、username / displayName / email | `UserRepository` | 单用户 |
| identity | `Course` | 实体：`CourseMember`；值对象：`CourseRole` | `CourseRepository` | 单课程（含成员关系） |
| qa | `Question` | 值对象：`Title`、`MarkdownBody`、`Tags`（`TagName`） | `QuestionRepository` | 单问题 |
| qa | `Answer` | 值对象：`MarkdownBody` | `AnswerRepository` | 单回答 |
| qa | `Vote` | 值对象：`TargetType` | `VoteRepository` | 单条投票 |
| qa | `Comment` | 值对象：`MarkdownBody` | `CommentRepository` | 单条评论 |
| reputation | `ReputationAccount` | 值对象：`ReputationReason`；关联 `LedgerEntry`（独立仓储） | `ReputationAccountRepository`、`LedgerRepository` | 单账户 + 一条流水 |
| reputation | `Bounty` | 值对象：`BountyStatus`、`AcademicTerm` | `BountyRepository` | 单悬赏 |
| reputation | `BadgeAward` | 值对象：`BadgeType` | `BadgeRepository` | 单徽章记录 |
| discovery | `Notification` | 值对象：`NotificationType` | `NotificationRepository` | 单条通知 |
| discovery | 无聚合根 | `SearchDocument`、`SearchHit` 为投影数据 | `SearchIndex`（端口） | 投影更新（按事件幂等） |

## 2. 逐聚合说明

### 2.1 `User`（identity）

- **不变式**
  - 学号/工号 `username` 为 4–20 位字母或数字，属于个人信息，不对外展示（`User` 类注释）；
  - 昵称 `displayName` 2–20 字符、不含 `@` 与空格，全局唯一，用于 @提及；
  - 自助注册只能是 `STUDENT` 且未认证；教职工账号只能由管理员创建且创建即认证（`ID-1005`）；
  - 管理员不能修改自己的角色（`ID-1006`）、不能禁用自己（`ID-1007`）；
  - 重复认证抛 `ID-2006`。
- **命令**：`register`、`createStaff`、`verify`、`changeRole`、`disable`
- **事件**：`UserRegisteredEvent`

### 2.2 `Course` / `CourseMember`（identity）

- **不变式**：课程成员关系是采纳、教师认证、发起悬赏的授权依据；课程教师角色不可被降级（`ID-1013`）。
- **命令**：建课、加入课程成员、`changeRole`
- **事件**：无（授权判定同步读取）

### 2.3 `Question`（qa）

- **不变式**（源码 `Question` 类注释）
  - 最多只有一个被采纳答案，且采纳后不可更改；
  - 只有提问者或课程管理者（教师/助教/管理员）可以采纳；
  - 已删除的问题不能再被回答、编辑或采纳；
  - 提问者只能删除尚无回答的问题，课程管理者可删除任意问题；
  - `score` / `answerCount` / `viewCount` 为反规范化计数器，由仓储用原子 SQL 维护，聚合内不修改。
- **命令**：`post`、`edit`、`accept`、`softDelete`、`ensureOpenForAnswers`
- **事件**：`QuestionPostedEvent`、`QuestionEditedEvent`、`AnswerAcceptedEvent`、`ContentDeletedEvent`

### 2.4 `Answer`（qa）

- **边界理由**：独立于 `Question`，只通过 `questionId` 引用，降低回答与问题的高频并发冲突。
- **不变式**
  - "是否被采纳"以 `Question.acceptedAnswerId` 为单一事实来源，Answer 不重复存储；
  - 教师/助教可 `endorse`（教师认证）；课程管理者与作者可编辑；
  - 被采纳的回答不能由回答者本人删除（防"先骗采纳再删除"），课程管理者可删除。
- **命令**：`submit`、`edit`、`endorse`、`softDelete`
- **事件**：`AnswerSubmittedEvent`、`AnswerEndorsedEvent`、`ContentDeletedEvent`

### 2.5 `Vote`（qa）

- **不变式**
  - 以 `targetType`（QUESTION / ANSWER）+ `targetId` + `voterId` 唯一确定一条投票，改投与撤销通过 `change` 完成，不新增记录；
  - 同一用户对同一目标只能存在一条有效投票；投票值只接受 `-1 / 0 / 1`（`checkValue`）；
  - `score` 变化由仓储原子维护，事件携带 `oldValue / newValue` 供下游计算增量。
- **命令**：`cast`、`change`
- **事件**：`VoteCastEvent`

### 2.6 `Comment`（qa）

- **不变式**：评论正文必填；正文中的 `@昵称` 由发现上下文解析，单条评论最多提醒 5 人（`MentionParser.MAX_MENTIONS`）。
- **命令**：`post`
- **事件**：`CommentPostedEvent`

### 2.7 `ReputationAccount`（reputation）

- **不变式**
  - 声誉与悬赏积分是两种"货币"，同一账户内分别维护总分与可用/冻结积分；
  - 冻结积分只能用于悬赏，`spendFrozen` / `refundFrozen` 必须成对出现，余额不允许为负（`requirePositive`）；
  - 每次变动写入一条 `LedgerEntry`（原因取自 `ReputationReason`），账本是对账依据；
  - 计分规则集中在 `ReputationRulebook`：采纳 +15（自问自答 +0）、被赞 +10、被踩 −2、正向每日上限 200、改投按增量。
- **命令**：`open`、`applyReputation`、`freezePoints`、`spendFrozen`、`refundFrozen`
- **事件**：`ReputationChangedEvent`

### 2.8 `Bounty`（reputation）

- **不变式**
  - 状态机：`OPEN → AWARDED`（问题采纳）或 `OPEN → EXPIRED`（到期），终态不可再变；
  - 悬赏积分 10–500，期限最长 14 天（`MIN_POINTS` / `MAX_POINTS` / `MAX_DAYS`）；
  - 发起人须为课程管理者；问题不能已删除或已采纳；
  - 若被采纳的恰好是发起人自己的回答，不发放奖金而退款（`close`）。
- **命令**：`open`、`award`、`close`、`isDue`
- **事件**：`BountyOpenedEvent`、`BountyAwardedEvent`、`BountyClosedEvent`

### 2.9 `BadgeAward`（reputation）

- **不变式**：同一用户同一徽章只授予一次；触发条件由 `BadgePolicy.forAcceptedCount`（累计采纳数）与 `BadgePolicy.forReputation`（声誉阈值）判定。
- **徽章**：`FIRST_ANSWER` 初来乍到、`FIRST_ACCEPTED` 初露锋芒、`HELPER_10` 解惑达人、`TEACHER_ENDORSED` 名师认可、`REPUTATION_100` 小有名气、`REPUTATION_1000` 德高望重。
- **命令**：`grant`
- **事件**：`BadgeAwardedEvent`

### 2.10 `Notification`（discovery）

- **不变式**：幂等键为 `(sourceEventId, recipientId)`，同一事件对同一接收人只产生一条通知；类型限定为 6 种：`ANSWER_RECEIVED`、`ANSWER_ACCEPTED`、`ANSWER_ENDORSED`、`MENTIONED`、`BOUNTY_AWARDED`、`BADGE_AWARDED`。
- **命令**：`create`、`markRead`
- **事件**：无（终端消费者）

## 3. 聚合间引用规则

| 引用方 → 被引用方 | 方式 | 说明 |
| --- | --- | --- |
| `Answer` → `Question` | 仅存 `questionId` | 不持有对象引用，跨聚合一致性由应用层保证 |
| `Vote` → `Question` / `Answer` | 仅存 `targetType + targetId` | 计分由仓储原子更新 |
| `Bounty` → `Question` / `Answer` | 仅存 `questionId` / `answerId` | 结算依赖问答上下文事件 |
| `ReputationAccount` → `User` | 仅存 `userId` | 昵称等展示信息由 `IdentityApi` 查询 |
| `Notification` → 各上下文 | 仅存业务键与链接 | 不跨上下文取对象 |

## 4. 变更记录

| 版本 | 日期 | 变更内容 | 修改人 |
| --- | --- | --- | --- |
| v1.0 | 2026-10-26 | 首次发布，覆盖 identity / qa / reputation / discovery 共 10 个聚合根 | 曹文 |
