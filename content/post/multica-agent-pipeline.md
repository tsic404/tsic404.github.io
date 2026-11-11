---
title: "Multica 与我们的多 agent 开发流水线：从派发 issue 到合并 PR"
date: 2026-11-11T10:00:00+08:00
draft: true
comments: true
section: "post"
tags:
  - multica
  - ai-agent
  - workflow
  - gitea
  - self-hosted
---
# Multica 与我们的多 agent 开发流水线：从派发 issue 到合并 PR

这篇文章本身就是它要讲的那条流水线跑出来的。

TSI-3533 的题目是「介绍 Multica 及我们的开发流程」：讲清平台是什么、我们怎么用它搭出开发流水线、实际效果如何。之后的过程是：issue 挂在 writing squad 上，leader 是 Vesper，它把工作拆成阶段派给调研、结构、写作、审稿四个 agent；调研 agent 去官方文档和源码仓把事实查回来，每条附命令、位置和取值时间；结构 agent 据此拟大纲；确认之后我写稿；审稿 agent Vanta 按同一份事实清单复核。所有阶段都留在同一个 issue 的评论区。

写这一段的时候，我自己的这次运行也在证据链里。它跑在一个 Kubernetes Job 的 pod 里，下面是从 pod 内读到的几项（2026-11-11）：

```text
$ hostname
fm-tsi-3533-astra-a776dfc7f30b-qpmnr
$ tr '\0' ' ' < /proc/1/cmdline
/usr/bin/dumb-init -- /usr/local/bin/multica daemon start --foreground --profile foreman-job
$ tr '\0' '\n' < /proc/1/environ | grep -E 'MULTICA_DAEMON_ID|MULTICA_SERVER_URL'
MULTICA_SERVER_URL=http://foreman.foreman.svc.cluster.local:8080
MULTICA_DAEMON_ID=fm-01a1c66f-95e9-7470-8ca2-a776dfc7f30b
$ multica daemon status --output json | jq '{profile,daemon_id,server_url,running_task_count,device_name}'
{
  "profile": "foreman-job",
  "daemon_id": "fm-01a1c66f-95e9-7470-8ca2-a776dfc7f30b",
  "server_url": "http://foreman.foreman.svc.cluster.local:8080",
  "running_task_count": 1,
  "device_name": "company-01"
}
```

这里有两个名字，作用不同。`hostname` 是可读的显示名：`fm-` 加 issue 标识、agent 名、task id 的尾段，最后五位是 Kubernetes 给 pod 的随机后缀。身份名是 `MULTICA_DAEMON_ID`，格式为 `fm-<task_id>`，来自 pod 标签 `foreman.tsic.top/daemon-id` 的 fieldRef。模板的 `metadata.name` 是格式串，部署方可以写成可读格式；Job 名不参与定位，Foreman 按 `foreman.tsic.top/task-id` 标签找 Job，名字只影响 `kubectl get jobs` 的显示。daemon-id 标签、注解、凭据 Secret 名和 Job Token 的 `j` 字段都从 `DaemonIDFor` 派生。

Multica 是一个把 issue 派给 AI 编码 agent 的平台。issue 是工作容器，agent 接单、报告、交付，所有过程记录在 issue 里。平台以 source-available 方式发布，许可是 Multica License：Apache-2.0 的正文加 Part I 的附加条件；Part I 第 1(a) 条写明内部使用（含多个 workspace）不需要商业许可。支持自托管，模型和 CLI 都能换。GitHub 上 `multica-ai/multica` 在 2026-11-10 15:22:13 UTC 读到 55,544 stars、7,334 forks、6,122 commits；同一天 14:52 UTC 那次读数是 55,539、7,333，半小时涨了五个 star、一个 fork。这类指标是秒级计数器，写下来只能算某个时刻的快照。

版本节奏也能当参考。官方 release 从 v0.6.2（2026-10-13）到 v0.8.2（2026-11-10），28 天发了 10 个。本文引用的功能、行号和数字都取 2026-11-10 的快照，照着做之前建议回原文核对一遍。

下面先讲平台的核心对象和一次运行的链路，再讲我们的五类单据、双仓协作和真实部署拓扑，最后把规则和出处列成一张表。

## 核心对象和一次运行的完整链路

Multica 的模型是一条链：Issue → Agent → Run → Runtime → 回写 Issue。

把 issue 指派给 agent 或 squad 之后，一次触发产生一条新的 run 记录，更早的记录不会被覆盖（`tasks.mdx:22`）。run 在某个 runtime 上执行，过程和结果写回 issue。触发入口有五种，列在讲单据的那一章。

| 对象 | 是什么 | 容易混的地方 |
|---|---|---|
| Issue | 一件工作的容器，描述、讨论、状态和执行记录都挂在它下面 | 派给 squad 不等于派给 squad 的每个成员 |
| Agent | 一份可复用的配置：instructions、模型、skills、访问权限、绑定的 runtime，外加并发上限、环境变量、CLI 参数、MCP 这些执行设置（`agents.mdx:17-23`） | 平时不占进程，只在被触发时执行；换模型或改 instructions 不会变成另一个 agent（`agents.mdx:25`） |
| Skill | 可复用的方法、参考资料与支撑文件（`agents.mdx:20`） | 它是 agent 配置的一部分 |
| Run | 一次触发产生的执行记录 | run 完成不等于 issue 完成 |
| Runtime | 一台连入 Multica 的机器，加上机器上已登录的一门 AI 编码 CLI（`daemon-runtimes.mdx:15`） | agent 是身份，runtime 是计算机 |
| Squad | 由一个 agent leader 带领的小组 | 派给 squad 时接手的是 leader，它再按成员名单派出去 |
| Autopilot | 按 schedule 或 webhook 自动起新工作的规则，也能用 Run now 手动跑一次（`autopilots.mdx:27,29`） | 要回到同一个 issue 的事归 issue wakeup |

平台的 agent 配置项都落在 `agents.mdx:15-23` 那张表里：名称、头像和描述只给人看，描述不进执行提示；instructions 每次运行都用；skills 提供可复用的方法和材料；runtime、模型、thinking level 决定在哪里跑、用哪个模型；access 决定谁能运行；执行设置里有并发、环境变量、CLI 参数、MCP 和外部集成。改这些不会丢历史 issue、评论和 run 记录。

代码和凭据的边界在 `daemon-runtimes.mdx:21,26`。AI 工具、工具自己的登录凭据、本机代码目录都留在连接的机器上；服务器不替本机工具执行命令，也不会自动上传整个工作目录。但「本地执行」不等于所有秘密只在本地：agent 的自定义环境变量和 MCP 配置存在服务器上，执行时下发到 runtime。

我们的 workspace 在 2026-11-10 的规模：3 个 squad（development 6 人、writing 5 人、architecture 6 人）、18 个 agent、8 个 autopilot（5 个 active，3 个 paused）、2 个 runtime。

| 小队 | 成员 |
|---|---|
| development | Lynx（leader）、Aureus（PM）、Vulcan（后端）、Vexel（前端）、Radian（审查）、Verity（QA） |
| architecture | Lynx（leader）、Orbis（Architect）、Vulcan、Vexel（契约面审阅）、Verity（验收面审阅）、Aureus（scope/adr 审阅） |
| writing | Vesper（leader）、Axiom（调研）、Caelum（结构）、Astra（写作）、Vanta（审稿） |

两个 runtime 都挂在同一台机器 `company-01` 上，daemon 身份是 `foreman-f311e8ff-e026-4c6e-b267-dcc004f08782`，一门是 omp 18.12.4，一门是 Claude Code 2.1.317，2026-11-10 15:14 UTC 都是 online。这些角色是我们自己配的，想照搬得自己建一份。

平台不内置模型。它驱动的是已经装好并登录的 AI 编码 CLI；官方文档的对照表列了 29 种，包括 Claude Code、Codex、Kimi CLI；同一页顶部的计数器显示 `Matching tools: 28 of 28`，差的那个是 ZCode，它只在文档表格里，页面的选择器组件没收录（`providers.mdx` 表格与 `provider-picker-demo.tsx` 的 `TOOLS` 数组）。

服务端本身是 Go 后端（`server/go.mod` 里 go 1.26.9）、Next.js 前端（`pnpm-workspace.yaml:41` 是 next ^16.4.5）、PostgreSQL 加 pgvector（`docker-compose.selfhost.yml:37` 用 `pgvector/pgvector:pg17`）。我们这台服务器自报 v0.8.2，commit `b7f8efc`，`/health` 返回 pid 1 和 `started_at`（12:37Z 前后）。这些公开接口看不出部署方式；部署见后面的拓扑章。

## issue 的状态由谁翻

平台的 issue 有 7 个内置状态，分属 4 个生命周期类别（`issues.mdx:60-70`）：

| 状态 | 生命周期类别 | 含义 |
|---|---|---|
| backlog | unstarted | 候选池；指派了 agent 也要等离开 backlog 才起 run（`issues.mdx:54,79`） |
| todo | unstarted | 已排期，可以开始 |
| in_progress | started | 正在做 |
| in_review | started | 交回审查 |
| blocked | started | 卡住了，等外部输入 |
| done | done | 完成 |
| cancelled | closed | 取消，记录保留 |

类别是状态的分组，创建自定义状态时选定，之后不能改；具体行为挂在单个状态上。owner 或 admin 可以加自定义状态（`issues.mdx:105-115`）；内置状态不能改名、改色或归档，归档自定义状态前要先把用它的 issue 迁走。

状态默认由 agent 自己写。官方文档写得很直接：agent 开始动手时设为 `in_progress`，交付时设为 `in_review`（`issues.mdx:96`）。如果项目配了 workflow，就按 workflow 的步骤和交接走。

服务器只在两种情况下自动改状态（`issues.mdx:101-102`）：run 失败、该 issue 没有其他 run、也没有触发重试时，`in_progress` 回滚到 `todo`；关联到该 issue 的所有 PR 都合并后，issue 置成工作区选定的状态，默认 `done`。工作区也可以选择不动状态，或者让这个 issue 关掉自动流转。

status 表示大阶段（在做、待审、完成），property 记录流程走到哪一步。我们的阶段全部记在自定义 property 上：每个流程一张 multi_select property，选项按顺序排开，leader 找第一个没勾选的选项，就是当前阶段。status 只承担归口，谁接手谁把它设成 `in_progress`，交出去就设成对应的状态。

我们的做法是让状态跟着实际走：读完一段工作，就把 issue 推到下一个状态，等于把球传出去。

评论里的「调研中」「结构中」这类阶段标记是团队约定；平台原生的阶段表达是 status（7 个内置加自定义）、子 issue 的 stage（数字批次）和自定义 property。

子 issue 的 stage 有自己的唤醒规则：最早未完成 stage 的子 issue 全部到达 `done` 或 `cancelled`，父 issue 会收到「子 issue 已完成」通知；如果父 issue 的 assignee 是 agent，它会被唤醒，自己决定要不要放行下一阶段（`issues.mdx:145`）。没有 stage 的子 issue 算作一批，全部完成后只通知一次。

## 五类单据与阶段机

平台给的是通用对象，流程得自己搭。这一章讲我们工作区自己定的五类单据、阶段序列和门禁；平台负责的是 issue、状态、property 这些通用对象。

先说触发。让一个 issue 产生 run 的入口有五种（`tasks.mdx:26-32`）：把它指派给 agent 或 squad；在评论里 mention 一个 agent；在 Chat 里给 agent 发消息；Autopilot 按 schedule 或外部事件触发；issue wakeup 按事件、时间或条件触发。五种入口给的上下文不同，执行路径一样：平台创建 run，runtime 认领，再调起 agent 配置的 AI 编码工具（`tasks.mdx:34`）。指派和 mention 在官方文档里是张对照表：mention 不改 assignee，也不改状态，只针对触发它的那条评论（`mentioning-agents.mdx:10,16-20`）。

阶段的真相源是 issue 上的 multi_select property。每个流程一张，选项按顺序排开；第一个选项「起始」只标类型，不推进。

| 单据类型 | property 阶段序列 | 谁推进 | 门禁 |
|---|---|---|---|
| 需求单 | 起始 → 需求分析 → 任务拆分 → 验证 | Aureus（PM） | 对照设计仓做覆盖判定；开了设计单就等它合并后再拆 |
| 设计单 | 起始 → 设计 → 审阅 → 合并 | Orbis（设计）→ 按面审阅人 → Lynx 计票合并 | 审阅票齐；L3/L4 另需 `tsic404` 的 `APPROVED <pr-url>` 评论 |
| 开发单 | 起始 → 开发 → 审查 → 验证 | Vulcan / Vexel → Radian → Verity | `APPROVED` 加 `QA_PASSED`，加设计锚点与两张核对表，加 CI 全绿 |
| Bug 单 | 起始 → 开发 → 审查 → 验证 | 同上 | 同上 |
| 验收单 | 起始 → 验收 | Verity | 部署层、TC 层、UAT 层三层证据 |

截至 2026-11-10，带各 property 的 issue 数（`multica property list` 的 `usage_count`，并用 `issue list --property` 的 `total` 交叉核对）：开发单 688、Bug 单 1067、验收单 424、设计单 221、需求单 39；`前置依赖` 是文本 property，值是逗号分隔的 TSI 编号，有 215 个 issue 用它。这些数字是快照，随流水线继续跑会变。

子 issue 可以带 `--stage N` 标批次；最低未完成 stage 的子 issue 全部 done 或取消之后，父 issue 收到通知、leader 被唤醒，由它决定要不要放行下一阶段。

`前置依赖` 卡的是单子推进：前置单没有全部 done，当前单不推进，issue 退回 todo；依赖完成后，协调 autopilot 重新把它唤起来。需求分析阶段还要把设计锚点冻进 issue 的 `metadata.design_anchor`，之后每个阶段的评论都带一行 `设计依据: tsic/<project>-design@<sha7>`，项目没有设计仓就写 `DESIGN_REPO_ABSENT`。锚点之后别人合并进来的设计改动不算 finding；只有本 issue 自己的条目被动了，才回头读一遍，按 D0 处理。

门禁不落在状态上。Radian 给出 `APPROVED`，Verity 给出 `QA_PASSED`，再加上设计锚点、实现与契约两张核对表、全绿 CI，leader 才合并实现 PR；本条 issue 声明的设计 PR 全部 merged 之后，再显式把 issue 写成 done。设计的门禁在下一章。

PR 和 issue 的关联规则要记牢，写错了不报错，只是不关联。PR 标题或分支名里带 issue 标识会自动关联；标识只出现在正文里时，必须紧跟在 Closes、Fixes、Resolves 这类关闭关键词后面；正文里裸提一句不算，commit message 和 PR 评论也不触发关联（`github-integration.mdx:69-77`、`vcs-integration.mdx:107`）。关联到该 issue 的 PR 全部合并之后，issue 按工作区设置自动流转。

测试用例分两个并列的族，按用户影响分级和按 P0/P1/P2 分级是两把不同的尺子（`acceptance-design/SKILL.md:44-53`）。TC-tech 是可以由 `tests/acceptance/<TC-id>` 自动执行的契约与行为用例，按 P0/P1/P2 分级，P2 可以不跑但要逐条标注。TC-uat 是用户视角的旅程与探索用例，按用户影响分级：核心旅程不可用、崩溃、数据丢失、安全问题算 P0，主要能力不可用算 P1，体验细节算 P2。TC-uat 不要求有 `tests/acceptance` 载体，但必须写清前置、操作步骤、可观察结果和证据形式。把 UAT 一律标成 P2，或者用「P2 未跑不算失败」把它从门禁里去掉，都不行。

验收本身是三层证据（`acceptance-design/SKILL.md:5-24`）：D 部署层，从受测提交构建产物，按项目文档在干净环境落地；T 用例层，跑设计仓 `docs/03-acceptance.md` 里的 TC；U UAT 层，以真实用户身份从真实入口用部署后的版本。`QA_PASSED` 要求 D 通过、T 的 P0/P1 全过、U 的 P0/P1 全过，且 UAT 入口清单没有未说明的缺口。部署或真实入口不可用时是 `QA_BLOCKED`，不能拿测试夹具替代部署判通过。

## 双仓协作架构

这套流水线做产品开发时用双仓：设计仓放意图、边界、契约、验收和 ADR，实现仓放代码。博客仓不属于双仓。

设计仓只在 Gitea。2026-11-10 的仓库列表里，`tsic` 下有 6 个设计仓：`agent-shell-design`、`eap-design`、`foreman-design`、`kos-design`、`rffmpeg-design`、`torrentfs-design`，外加一个 `design-repo-template`。同名的 `-design` 仓在 GitHub 上全部 404，没有镜像。

实现仓在 GitHub 的 `tsic404` 名下，2026-11-10 有 31 个公开仓，其中 3 个是 fork（JellyPlex-Watched、dde-control-center、seaweedfs）。Gitea 里另有 4 个代码仓是 GitHub 的 pull mirror，`original_url` 指向 `github.com/tsic404/*`，分别是 `agent-shell`、`eap`、`rffmpeg`、`torrentfs`。

| 维度 | 设计仓 | 实现仓 |
|---|---|---|
| 位置 | 自建 Gitea | GitHub，部分在 Gitea 有 pull mirror |
| 内容 | 目标与边界、对外契约、验收判据、ADR | 代码、测试、CI |
| 审查 | Gitea review 加 issue 评论，按面抽审阅人 | Radian 审代码，Verity 验部署与 UAT |
| 合并 | Lynx 计票后合并；L3/L4 需人签 | leader 在 GitHub squash 合并 |

设计仓的文档集合是固定的（`design-doc-authoring/SKILL.md:16-41`）：

```text
README.md                一句话定位，无索引、无文档清单、无状态列
docs/01-scope.md         目标 / 非目标 / In-scope / Out-of-scope / Forbidden + AI 自查规则
docs/02-contracts.md     C-x 外部可观察承诺
docs/03-acceptance.md    AC-x 与覆盖它的 TC-x
docs/adr/ADR-xxx.md      context / drivers / options / outcome / consequences
```

`docs/04-architecture.md`、`docs/05-modules/`、`test-cases/`、`proposals/` 永久删除，不再出现；索引、覆盖矩阵、状态列、实现对应关系表也不进设计仓。硬禁清单里还有行号锚、逐字表达式、常量与枚举取值、env 键名、退出码数字、符号名与目录布局、UUID 与 sha 和 TSI 号这类运行时事实，以及「与实现一致」这种理由（`SKILL.md:31-41`）。进设计仓的文字要过两条判据：删掉实现，这段文字还能驱动一次重写吗；实现重构之后，这段文字必须跟着改吗。第二条命中，就不属于这里（`SKILL.md:11-13`）。

谁产生变更谁写。D1 决策落 `docs/adr/`，D2 边界落 `01-scope`，D3 对外契约落 `02-contracts`，D4 判据落 `03-acceptance`（`SKILL.md:49-52`）。D0 是待决事项：越出 In-scope、与既有条目冲突、没有人工裁决就无法确定正确行为，写成同一 PR 里 `status: proposed` 的 ADR，停下等 `tsic404` 的 `APPROVED`（L4）。

一条 issue 默认一条设计 PR，分支 `design/TSI-<n>`；唯一允许的第二条是 D0 落地，分支 `design/TSI-<n>-ratify`，正文记 `人签:`，按落地内容定级，不重签（`SKILL.md:63`）。设计 PR 未合入不挡开发、不挡代码 PR 合入，但 issue 的 done 必须等它合并；代码合入时设计 PR 还开着，issue 留在 `in_review`，记一行 `设计同步: <PR url>`（`SKILL.md:65`）。

审阅按面抽人，一个面一名审阅人，多面最多两名，按风险序 `adr > scope > contracts > acceptance` 取前两名（`SKILL.md:99-108`）。作者不审自己的 PR；首选审阅人是作者时，这个面整面改由候选接替，其他 login 在同一面上的票不算。

| 面 | 首选 | 首选是作者时的候选 |
|---|---|---|
| `01-scope` / `adr/` | Aureus | Orbis |
| `02-contracts` | Orbis | Vulcan / Vexel |
| `03-acceptance` | Verity | Aureus |
| 机械（L0） | 审阅池任一非作者 | 同 |

计票只算当前 head 上的票：`official=true`、`dismissed=false`、`stale=false`、`commit_id` 等于 head；当前 head 上有任何 `REQUEST_CHANGES` 直接阻塞，新 commit 之后旧票作废；作者与合并者 Lynx 不计票，同一个 login 不能同时满足两个面；每条计票的 review 要写 `分级复核: L<n> 成立` 和 `weakening: none|<条目>`，缺这句的 approval 不计票（`SKILL.md:110-114`）。人签是 `tsic404` 在来源 issue 的 `APPROVED <pr-url>` 评论，它不替代 L0–L2 的 agent 票。

合并分五级（`SKILL.md:137-143`）：

| 级别 | 触发 | 需要什么 |
|---|---|---|
| L0 | 纯机械修订 | 一名非作者审阅人；机械不等于免审 |
| L1 | clarify；或只动 03 的 tighten / additive | 对应面的审阅人一名 |
| L2 | 02 上新增契约或纯插入子项 | 契约面加验收面，两个不同 login |
| L3 | modify / weakening、D1、D2 | 所有触及面（最多两个）加 `tsic404` 的 `APPROVED` |
| L4 | 合入后仍有未批准的 proposed ADR | 只有 `tsic404`，agent 票不计 |

L0–L2 在票齐后由 Lynx 直接合并；L3/L4 的**决定**由人签作出，合并仍由 Lynx 执行：读 open、mergeable、head sha，计票，`POST /pulls/<n>/merge`，`do=squash`，`head_commit_id` 钉住 head，删分支，回读 `merged=true`（`SKILL.md:187-199`）。人也可以直接在 Gitea 合并；之后要补一条 `MERGED <pr-url>` 评论，因为没有 webhook 替人补这一步（`SKILL.md:196`）。

设计仓不接 webhook，也没有 commit status：设计 PR 的状态只存在于 Gitea review 与 issue 评论里（`SKILL.md:189`；`references/review-and-merge.md` §3）。2026-11-10 抽查 18 条最近 PR 的 head，commit status 全空。仓库里还留着一条 `.*-design$` 的 webhook 路由（`multica-agent` 仓的 `scripts/hooks.json:71-94`），它指向的 `design-review` autopilot 已于 2026-10-23 删除（commit `e19e9e4`），当前 autopilot 列表里没有任何设计审查条目；线上 webhook 注册读不到，仓库和组织的 hooks 接口都返回 403。设计流程本身不依赖 webhook。

设计仓的 `main` 已保护。具体字段值要仓库 admin 权限，读不到，这里不写。

门禁是否真的生效，可以看已合并 PR 的历史。统计口径：6 个设计仓，逐个翻 `state=closed` 的 PR，取 `merged_at` 非空的；每个 PR 取 reviews 第一页（默认 30 条），计其中 `state=APPROVED && official=true` 的票。截至 2026-11-10 共 324 条已合并 PR，其中 59 条（18.2%）没有任何 official APPROVED；这 59 条全部合并于 2026-10-23 及以前，其中 9 条就在 10-23 当天合并，合并人都是 `tsic404`。2026-10-24 起合并的 133 条，每条至少有一票。

## 自托管与 Gitea 集成

官方文档给了一条 Docker Compose 快速上手路径：两行命令装 CLI 和服务端（`install.sh:8,519-526`、`self-host-quickstart.mdx:20,269,288`），另有 Helm chart（`oci://ghcr.io/multica-ai/charts/multica`）；我们实际用的是 Kubernetes。

集群是 k8s。从 pod 内看到的网段是 pod `10.42.0.0/16`、service `10.43.0.0/16`、集群 DNS `10.43.0.10`，都是 k3s 的默认值；发行版名字从 pod 内确认不了，节点 `192.168.18.9:6443` 上的 k8s API 返回 401。工作负载分在三个 namespace 里：

| namespace | 里面是什么 | 证据 |
|---|---|---|
| `multica` | Multica Server：frontend、backend、postgres | 三个名字都能在集群 DNS 里解析到；frontend 把 `/health` 重写到 `backend.multica.svc.cluster.local:8080` |
| `foreman` | 调度器 Deployment（1 副本）、`foreman` Service、`foreman-ops` Service、Job 模板 ConfigMap | `deploy/25-job-template.yaml`、`deploy/30-foreman.yaml` |
| `multica-agents` | 每个任务一个 Job pod，加一个清扫用的 gc DaemonSet | `deploy/00-namespaces.yaml`、`40-foreman-gc.yaml` |

服务器自报 v0.8.2，commit `b7f8efc`，`/health` 返回 pid 1；`/api/config` 里 `vcs_integration_available` 是 true。frontend 是个 Next.js 应用，把 `/health` 重写进 `backend.multica.svc.cluster.local:8080`，响应头里有 `x-middleware-rewrite` 可以确认这条链路。

调度器是 `tsic404/foreman`。它的 README 开头写的是 `Kubernetes-native elastic scheduler for Multica tasks`：夹在 Multica Server 和 Kubernetes 之间，从 server 认领任务，把任务变成 K8s Job，再把结果报回去；对 server 来说它像一个高并发的 daemon，底下每个任务跑在自己的临时 pod 里（`README.md:3,5`）。

部署面就是 `deploy/`，全是 YAML，`kubectl apply -f deploy/` 一次装完，再 `kubectl -n foreman rollout status deploy/foreman`（`deploy/README.md:3-24`）。清单覆盖 namespaces、secrets、RBAC、Job 模板 ConfigMap、可选的 Job 策略 ConfigMap、foreman Deployment 与两个 Service、共享状态用的 Redis baseline、gc DaemonSet，`optional/` 下还有 ServiceMonitor 与 ops 端口的 NetworkPolicy。

Job 的形状来自必选 ConfigMap `foreman-job-template`（`deploy/25-job-template.yaml`）。Foreman 没有内置模板：文件缺失、读不到或解析失败会拒绝启动（ADR-030）。Deployment 把它以非可选的只读卷挂到 `/etc/foreman/job-template/`，每个任务按它渲染。

从模板和这次运行自己的 pod 能读出 Job 的形状：namespace `multica-agents`；容器以 `dumb-init` 启动 `multica daemon start --foreground --profile foreman-job`；`MULTICA_SERVER_URL` 写死为 `http://foreman.foreman.svc.cluster.local:8080`，也就是 Job 直接连 Foreman，不直连 server；`MULTICA_DAEMON_ID` 取自 pod 标签 `foreman.tsic.top/daemon-id`；`MULTICA_AGENT_RUNTIME_NAME` 是 `foreman-job`；`MULTICA_DAEMON_MAX_CONCURRENT_TASKS` 是 1，即一个 pod 同时只跑一个任务。pod 里没有挂 service account token，`/var/run/secrets/kubernetes.io/serviceaccount` 不存在，agent 拿不到集群 API 凭证。

Job 名和身份名分开：这次运行所在的 Job 用了可读格式 `fm-tsi-3533-astra-a776dfc7f30b`，pod 名再加 Kubernetes 的 5 位后缀，就是前面看到的 `fm-tsi-3533-astra-a776dfc7f30b-qpmnr`。身份名始终是 `fm-<task_id>`，由标签带进容器。

server 那边看到的 daemon 身份是 `foreman-f311e8ff-e026-4c6e-b267-dcc004f08782`，也就是 `foreman-<workspace_id>`；仓库清单里 `FOREMAN_DAEMON_ID` 的默认值是 `foreman`，而在发现形态下，每个 workspace 的身份由 base 加 `-` 加 workspace id 派生（`internal/workspace/workspace.go:327-343`）。线上 Deployment 实际用的是哪种形态、env 怎么组合，读不到，所以这里只写两边各自能观测到的值。

线上清单读不到。`~/.kube/config` 里只有几个 k3d 测试集群；另一份 kubeconfig 指向 `0.0.0.0:40151`，连接被拒绝。生产集群的清单没有可用凭证。从 Job pod 内能观测到的只有行为：`foreman.foreman.svc.cluster.local:8080` 的 `/foreman/healthz` 返回 200；`/foreman/tasks`、`/foreman/version`、`/metrics` 在 8080 上都是 404；仓库清单把这几条挂在另一个只发布 8081 的 `foreman-ops` Service 上，而 `foreman-ops` 这个名字在 Job pod 内解析不到。所以本文写的是仓库里的部署面，加上 pod 内能观测到的行为，不是线上清单的逐字复刻。

代码托管接进来走 VCS 集成。它只对自托管版提供，Cloud 版没有（`vcs-integration.mdx:11`）。服务端要开两个设置：`MULTICA_VCS_INTEGRATION_ENABLED=true`（`vcs-integration.mdx:27`），再给 `MULTICA_VCS_SECRET_KEY` 一个 base64 编码的 32 字节密钥（`:39,42`，生成方式见 `environment-variables.mdx:196-197` 的 `openssl rand -base64 32`）。两个都要有：开关决定功能开不开，密钥负责加密存储的 token 和 webhook secret；少一个，Settings → Code 里的 Connect 表单就是禁用的（`:11,32,42`）。再设 `MULTICA_PUBLIC_URL`，界面才会给出可直接粘贴的 webhook URL；不设就只给路径，origin 要自己拼（`:44`）。

连接按工作区配，入口在 Settings → Code，点 Self-hosted Git 那一行的 Connect（`:56`），填实例 URL 和一个 access token；Multica 会先拿 token 去实例上验一遍再存，token 只要有读权限（`:50`）。这里没有 GitHub App 那种应用模型，靠的是在仓库或组织上注册 webhook（`:19`）。连上之后界面给一个 webhook URL 和一个 webhook secret，secret 只显示一次（`:62-67`）。

Gitea 侧注册 webhook 的路径是 Settings → Webhooks → Add Webhook → Forgejo/Gitea，事件选 Pull Request 和 Commit Status，secret 用来校验 `X-Gitea-Signature` 的 HMAC（`:83-88`）；服务端逐条校验签名，secret 对不上就拒绝（`:102`）。CI 的 passed / failed / pending 会汇总成 head commit 上的 checks bar（`:108`）。

agent 开 PR 不需要在 Multica 里配 provider：它在 runtime 上 checkout 仓库，用宿主机自己的 git 凭据推分支、开 PR，SSH deploy key 或者 credential helper 都行（`:110-112`）。

设计仓这一侧不走这套：它是 Gitea 里的普通仓库，没有 hook 到设计审查服务，也没有 commit status，状态在 review 和 issue 评论里，见上一章。

## 规则与出处

下面每条规则都能指到官方文档、工作区规范或 runtime brief 的具体位置。官方文档行号取 `multica-ai/multica` main 分支的 mdx 源文（2026-11-10 快照）；runtime brief 的行号取本工作区注入的那一份 `AGENTS.md`。

| 规则 | 出处 |
|---|---|
| 评论正文先写文件，再用 `--content-file` 传；不用内联内容，不把 heredoc 和其他参数混用；临时文件放工作目录，不放 `/tmp` | runtime brief「Comment Formatting」（`AGENTS.md:115-117`）；`cli.mdx:124-125,140-141`；上游 `multica-ai/multica#4182`。brief 里引的 `MUL-2904`、`MUL-4252` 是 Multica 自家 tracker 的编号，不是 GitHub issue |
| turn 一结束，还在后台的工作变成孤儿，结果丢失，最终评论也发不出去 | runtime brief「Background Task Safety」（`AGENTS.md:8-14`） |
| mention 一个 agent 等于给它排一次 run；只是通知人就别用 mention | `mentioning-agents.mdx:10`；runtime brief「Mentions」（`AGENTS.md:190-198`） |
| PR 关联：标题或分支名带 issue 标识，或者正文在关闭关键词后紧跟标识；正文裸提及不关联，commit message 和 PR 评论也不触发 | `vcs-integration.mdx:107`；`github-integration.mdx:69-77` |
| leader 派发完不等于父 issue 完成；派发轮结束时父 issue 是 `in_progress`，目标达成才推 `in_review` | `squads.mdx:51`；`issues.mdx:96,101-102` |
| 子 issue 按 stage 分批，最早未完成 stage 全部 done 或 cancelled 才通知并唤醒父 issue | `issues.mdx:145` |
| status 由 agent 显式写；系统只做两处自动流转：run 失败回滚 `todo`，关联 PR 全合并后置工作区选定的状态 | `issues.mdx:96,101-102` |
| 触发一次 run 的入口有五种：指派、评论里 mention、Chat、Autopilot、issue wakeup | `tasks.mdx:26-32` |
| 设计 PR 审阅按面抽一名，多面最多两名，作者回避；L0–L2 由 Lynx 计票后合并，L3/L4 需 `tsic404` 的 `APPROVED <pr-url>` 评论、合并仍由 Lynx 执行 | `design-doc-authoring/SKILL.md:99-108,137-143,187-199`；`squads/architecture/squad.md:47-48` |
| 设计仓固定 4 份文档加 ADR，`04-architecture`、`05-modules`、`test-cases`、`proposals` 永久删除 | `design-doc-authoring/SKILL.md:16-41` |
| TC-tech 按 P0/P1/P2 分级，TC-uat 按用户影响分级，两者不是同一个优先级；UAT 不适用「P2 未跑不算失败」 | `acceptance-design/SKILL.md:44-53`；门禁见 `:21-24` |
| 本机代码目录和工具凭据留在连接的机器上；服务器不代执行命令、不上传整个工作目录；agent 的自定义环境变量与 MCP 配置存服务器，执行时下发 | `daemon-runtimes.mdx:21,26` |

唯一一件能指到 commit 的亲身经历，是设计仓收窄的那次。2026-10-17 同一天，5 个设计仓各有一个提交，把文档集合收敛到固定的 4 份加 ADR：新增 `docs/01-scope.md`、`docs/02-contracts.md`、`docs/03-acceptance.md` 三份文档（有的仓同时补了 ADR），删掉 `docs/04-architecture.md`、`docs/05-modules/*`、`test-cases/*`、`proposals/*`，以及那个编号体系里的 `01-goals`、`02-boundaries`、`03-contracts`、`06-acceptance`。

| 仓 | commit | 文件数 | 增 / 删 |
|---|---|---|---|
| foreman-design | `448ec69` | 85 | +1244 / -9499 |
| agent-shell-design | `0beae41` | 69 | +1269 / -8026 |
| eap-design | `3990477` | 46 | +698 / -8357 |
| torrentfs-design | `4bc1833` | 61 | +984 / -11946 |
| rffmpeg-design | `57e2027` | 30 | +651 / -3846 |

五个仓一共删掉 4 万多行，加回来不到 5 千行。现在的设计仓规范把判据固定下来：文档集合只有 4 份加 ADR，被删的路径永久禁止回流；判断一段文字该不该进设计仓，只问删掉实现之后它还能不能驱动一次重写、实现重构之后它是否必须跟着改。

平台的功能细节和数字都取自 2026-11-10 的快照，release 还在继续发，真要照着做，建议回原文核对一遍。
