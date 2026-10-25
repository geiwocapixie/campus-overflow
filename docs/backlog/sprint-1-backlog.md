# Sprint 1 Backlog

> 版本：v1.0　制定日期：2026-10-26　PO：杨杰　SM：曹文
> 上游文档：[Product Backlog v1.0](product-backlog.md)　验收标准：`docs/backlog/stories/`

## 1. Sprint 目标

**两周内打通「注册 → 登录 → 浏览问题 → 查看详情 → 发布问题 → 回答问题」的最小闭环（Walking Skeleton），
并在 Sprint Review 上用真实账号演示：新同学注册后提一个问题，另一名同学回答，提问者能在详情页看到回答。**

时间盒：2026-10-26 ～ 2026-11-08（2 周）
容量估算：团队首次 Sprint，按每人 4 SP 保守估算，5 人 × 4 SP = **20 SP**

## 2. 承诺的故事（20 SP）

| ID | 故事 | SP | 责任人（主） | 验收标准 |
| --- | --- | --- | --- | --- |
| US-01 | 用学校邮箱注册账号 | 3 | 向宇航 / 林纹宇 | `stories/US-01.feature` |
| US-02 | 登录并保持会话 | 3 | 向宇航 / 林纹宇 | `stories/US-02.feature` |
| US-03 | 浏览问题列表并分页 | 2 | 向宇航 / 鲍欣宇 | `stories/US-03.feature` |
| US-05 | 查看问题详情 | 2 | 向宇航 / 林纹宇 | `stories/US-05.feature` |
| US-06 | 发布带标签的问题 | 5 | 向宇航 / 林纹宇 | `stories/US-06.feature` |
| US-07 | 回答问题 | 5 | 向宇航 / 林纹宇 | `stories/US-07.feature` |

合计 20 SP，与容量持平，不再追加故事（不接受范围蔓延）。

> 说明：`stories/` 下另外准备了 US-04（搜索）与 US-08（投票）两份已写好的 Gherkin，
> 它们属于 Sprint 2，提前写是为了让契约讨论一次做完，**不计入本 Sprint 承诺**。

## 3. 任务分解与责任分配

| 任务 ID | 所属故事 | 任务内容 | 责任人 | 预估 | 状态 |
| --- | --- | --- | --- | --- | --- |
| S1-T01 | 跨故事 | 分支保护 / CI 守护、PR 模板与评审清单落地 | 曹文 | 2h | To Do |
| S1-T02 | 跨故事 | 联调环境（MySQL + Redis compose）与演示账号准备 | 曹文 | 3h | To Do |
| S1-T03 | 跨故事 | Backlog 澄清会与验收标准评审，维护故事状态 | 杨杰 | 3h | To Do |
| S1-T04 | 跨故事 | 将 6 个 `.feature` 场景转成接口级自动化用例 | 鲍欣宇 | 4h | To Do |
| S1-T05 | US-01 | 注册接口联调：学校邮箱校验、密码强度、ID-2001 冲突 | 向宇航 | 4h | To Do |
| S1-T06 | US-01 | 注册页表单、密码强度提示与错误码文案 | 林纹宇 | 4h | To Do |
| S1-T07 | US-01 | 注册用例测试：8/64 位密码边界、重复邮箱 | 鲍欣宇 | 3h | To Do |
| S1-T08 | US-02 | 登录 / 会话保持接口验证与 `/auth/me` 联调 | 向宇航 | 3h | To Do |
| S1-T09 | US-02 | 登录页、登录态路由守卫与退出登录 | 林纹宇 | 4h | To Do |
| S1-T10 | US-02 | 登录失败限流（AUTH-429）与未登录 401 测试 | 鲍欣宇 | 2h | To Do |
| S1-T11 | US-03 | `GET /api/v1/questions` 列表接口联调与分页参数 | 向宇航 | 3h | To Do |
| S1-T12 | US-03 | 问题列表页：卡片、分页器、加载态、空态 | 鲍欣宇 | 6h | To Do |
| S1-T13 | US-03 | 列表接口契约测试：第 2 页无重复、空结果 200 | 鲍欣宇 | 3h | To Do |
| S1-T14 | US-05 | `GET /api/v1/questions/{id}` 详情接口联调（回答、myVote） | 向宇航 | 3h | To Do |
| S1-T15 | US-05 | 详情页布局：标题、标签、正文 Markdown、回答列表 | 林纹宇 | 6h | To Do |
| S1-T16 | US-05 | 详情页 404 与无回答场景测试 | 鲍欣宇 | 2h | To Do |
| S1-T17 | US-06 | `POST /api/v1/questions` 命令链路（含标签 1–5 校验） | 向宇航 | 5h | To Do |
| S1-T18 | US-06 | 提问页表单、Markdown 预览、代码块与标签选择 | 林纹宇 | 8h | To Do |
| S1-T19 | US-06 | 提问校验测试：标题 150/151、正文 30000/30001、标签 0/6 | 鲍欣宇 | 3h | To Do |
| S1-T20 | US-07 | `POST /api/v1/questions/{id}/answers` 命令链路与开放校验 | 向宇航 | 4h | To Do |
| S1-T21 | US-07 | 回答编辑器、提交后刷新详情与错误提示 | 林纹宇 | 4h | To Do |
| S1-T22 | US-07 | 回答接口集成测试：已采纳不可答、空正文拒绝 | 鲍欣宇 | 3h | To Do |

工时合计约 83h，分配：曹文 5h、杨杰 3h、向宇航 22h、林纹宇 26h、鲍欣宇 27h（含 QA 职责）。
林纹宇与鲍欣宇负载偏高，已把 S1-T12 列表页与 S1-T13 契约测试合并排期，允许延至 Sprint 第二周完成。

## 4. 看板卡片清单（Sprint 1）

看板列名固定为 **To Do / In Progress / In Review / Done**。Sprint 1 开始时在 To Do 列建以下 9 张卡片：

| 卡片标题 | 说明（Description 里写） | 负责人 | 列 |
| --- | --- | --- | --- |
| [US-01] 注册账号 | 3 SP；任务 S1-T05~T07；验收 `US-01.feature` | 向宇航 | To Do |
| [US-02] 登录并保持会话 | 3 SP；任务 S1-T08~T10；验收 `US-02.feature` | 向宇航 | To Do |
| [US-03] 浏览问题列表 | 2 SP；任务 S1-T11~T13；验收 `US-03.feature` | 鲍欣宇 | To Do |
| [US-05] 查看问题详情 | 2 SP；任务 S1-T14~T16；验收 `US-05.feature` | 林纹宇 | To Do |
| [US-06] 发布问题 | 5 SP；任务 S1-T17~T19；验收 `US-06.feature` | 林纹宇 | To Do |
| [US-07] 回答问题 | 5 SP；任务 S1-T20~T22；验收 `US-07.feature` | 向宇航 | To Do |
| [基础] 联调环境与演示账号 | S1-T02，Sprint Review 前必须可用 | 曹文 | To Do |
| [基础] 场景自动化用例 | S1-T04，覆盖 6 份 .feature | 鲍欣宇 | To Do |
| [PO] 验收与故事澄清 | S1-T03，每个故事进入 In Review 前澄清一次 | 杨杰 | To Do |

拖动规则：开工时卡片进 **In Progress**；PR 提交后进 **In Review**；满足 DoD 后由 SM 拖入 **Done**。

## 5. 依赖与风险

| 风险 | 影响 | 应对 |
| --- | --- | --- |
| 后端 Q&A 领域层仍是 `TODO(S2)`，接口返回 500 | 前端无法联调 | 前端先用接口契约与 Mock 数据开发，后端按 `docs/学生任务清单.md` 逐个实现，优先 Question 聚合 |
| 团队首次 Sprint，Velocity 无历史数据 | 估算偏差 | 按 4 SP/人 保守估算，第二周中检查剩余量，必要时砍 US-07 的编辑能力（不在本期验收内） |
| 中文检索依赖 MySQL ngram 配置 | 后续搜索不可用 | 本期不做搜索，Sprint 2 前确认 compose 中的 `ngram_token_size=2` |
| 跨成员评审响应慢 | 卡片卡在 In Review | 交叉评审固定配对，24 小时内必须给意见（章程 §3） |

## 6. Sprint Review 演示脚本（5 分钟）

1. 用全新浏览器注册账号 `demo01@campus.edu`（展示密码强度提示）；
2. 登录并刷新页面，证明会话保持；
3. 打开问题列表，翻到第 2 页；
4. 发布问题「MySQL 索引失效怎么排查？」，标签 `MySQL`、`索引`、`实验`；
5. 切换到另一账号 `demo02@campus.edu` 回答该问题；
6. 回到 `demo01` 打开详情页，展示回答与 `myVote` 字段；
7. PO 对照 `US-01/02/03/05/06/07.feature` 逐条判定通过与否。

## 7. Sprint 1 的完成定义（DoD）

- 6 个故事对应的 `.feature` 场景全部可执行并通过；
- 后端 `mvn -pl co-qa/co-qa-domain test` 无新增失败，ArchUnit `ArchitectureTest` 通过；
- 前端 `npm run typecheck` 与 `npm run build` 通过；
- 每个故事至少 1 个 PR，经交叉评审 Approve + `LGTM` 后由 SM 合并进 `dev`；
- 看板卡片全部进入 Done，Review 演示脚本可一次跑通。

## 8. 变更记录

| 版本 | 日期 | 变更内容 | 修改人 |
| --- | --- | --- | --- |
| v1.0 | 2026-10-26 | Sprint 1 计划形成，承诺 6 条故事 / 20 SP | 杨杰 |
