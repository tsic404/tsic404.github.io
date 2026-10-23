---
title: "Multica 与我们的多 agent 开发流水线：从派发 issue 到合并 PR"
date: 2026-10-23T11:00:00+08:00
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

TSI-3533 建出来的时候只有一句话的需求：介绍 Multica，以及我们基于它搭的开发流程。后面的事情是这么发生的：issue 派给一个 squad，leader 是个 agent，它把工作拆成阶段逐个派出去；调研 agent 去官方文档和源码仓库把事实查回来，列成带来源和日期的清单；结构 agent 据此拟大纲；我拿到确认后写出你正在读的这篇稿子。每一步都留在同一个 issue 的评论区里，翻记录就知道谁做了什么。

Multica 就是干这个的工具。它把一个工作区里的 issue 派给 AI 编码 agent，像给同事派活：agent 自己认领任务，报告进展，遇到阻塞提出来，做完交回审查。平台是 source-available 的，许可为 Multica License，自托管免费，官方文档列出 25 种 agent CLI（截至 2026-10-18），换模型供应商只是换一个下拉选项。截至 2026-10-18，官方仓库有 53.1k stars、6,959 forks，累计 5,609 commits，按官方说法大部分工作日都在发版。

下面先讲平台的核心对象和一次运行的完整链路，再讲我们工作区的五类单据、双仓协作和自托管部署，最后是踩过的坑和写进 instructions 的规则。

## 核心对象和一次运行的完整链路

Multica 的模型可以压成一条链：

Issue → Agent → Run → Runtime → 回写 Issue

Issue 触发 Agent 有四种方式：把 issue 指派给某个 agent，在评论里 @ 它，在 Chat 里直接对话，或者由 Autopilot 按规则触发。触发之后产生一次 Run，Run 在某个 Runtime 上执行，过程和结果写回 issue。

整条链里最容易被忽略的一点是 Run 完成不等于 issue 完成。agent 跑完一轮，issue 的状态不会自己动，要有人显式去翻。

| 对象 | 是什么 | 容易混的地方 |
|---|---|---|
| Issue | 一件工作的容器，描述、讨论、状态和执行日志都挂在它下面 | 派给 squad 不等于派给 squad 的每个成员 |
| Agent | 一份可复用的配置：instructions、模型、skills、访问权限，外加绑定的 runtime | 只在被触发时执行，平时不占进程 |
| Skill | 「怎么做某事」的能力包，可以挂到多个 agent | 可以同时服务多个 agent，归属不在某一个身上 |
| Run | 一次触发产生的执行记录 | Run 完成不代表 issue 完成 |
| Runtime | 一台连入 Multica 的机器，加上机器上已登录的 AI 编码 CLI | agent 是身份，runtime 是计算机 |
| Squad | 由一个 agent leader 带领的「agent 加人」小组 | 派给 squad 等于派给 leader，不会广播给全体 |
| Autopilot | 按 cron 或 webhook 自动触发 run 的规则 | 也能手动触发，不是只有定时 |

Agent 是一份可复用的配置，里面有 instructions、模型、skills 和访问权限，并绑定到一个 runtime。没被触发的时候它什么都不做，所以一个工作区可以养很多 agent。

Skill 是「怎么做某事」的能力包，可以挂到多个 agent 上。比如「发评论必须走文件」这条规则，写进 skill 之后，所有加载它的 agent 都遵守，不用在每份 instructions 里各写一遍。

Runtime 是实际执行发生的地方：一台连入 Multica 的机器，机器上装着已经登录的 AI 编码 CLI。agent 是身份，runtime 是计算机，代码不出这台机器。Multica 服务器管调度和记录，仓库和构建过程都在你自己的机器上。

Squad 是由一个 agent leader 带领的「agent 加人」小组。把 issue 指派给 squad，等于指派给 leader，由它协调分派，不会自动广播给全体成员。我们工作区给这条流水线配了 leader、调研、结构、写作、审稿几个角色，都是自己建的，想照搬得自己配一份。

Autopilot 按规则自动触发 run，触发条件可以是 cron 计划，也可以是外部 webhook 事件，当然也能手动点一下。

Issue 承载一件工作的全部上下文。描述和讨论、状态与执行日志都挂在同一个 issue 下面，优先级、标签、自定义 property、父子关系这些属性也一样，不用另找地方查。

平台不内置模型。它驱动的是你已经安装并登录的 agent CLI，官方文档列出 25 种（截至 2026-10-18），Claude Code、Codex、Cursor、Copilot、Kimi、OpenCode 都在其中。想换供应商，改一个配置项就行。

## issue 的状态由谁翻

平台的 issue 有 7 个内置状态，分属 4 个生命周期类别：

| 状态 | 生命周期类别 | 含义 |
|---|---|---|
| backlog | unstarted | 候选池 |
| todo | unstarted | 已排期，可以开始 |
| in_progress | started | 正在做 |
| in_review | started | 交回审查 |
| blocked | started | 卡住了，等外部输入 |
| done | done | 完成 |
| cancelled | closed | 取消 |

类别决定状态在板上的归类，unstarted 是还没开始，started 是进行中，done 和 closed 分别对应完成和取消。owner 和 admin 可以给工作区加自定义状态，类别在创建时定死，之后改不了，加之前想清楚它属于哪一类。

状态怎么变？由 agent 通过 CLI 显式写入，服务器不会在 run 开始或结束时自动翻状态。run 跑完了，issue 还停在原地，除非有 agent 主动把它推到 in_review 之类的位置。

只有两个系统例外。run 失败、该 issue 没有其他 run 且没有触发重试时，in_progress 会回滚到 todo；关联到该 issue 的 PR 全部合并时，issue 会置成工作区选定的状态，默认是 done。

status 回答「这件事现在归谁」，property 回答「走到哪一步」。我们的阶段全部记在自定义 property 上：每个流程一张 multi_select property，选项按顺序排开，leader 找第一个没勾选的选项，就是当前阶段。状态只承担归口，谁接手谁把它设成 in_progress，工作交出去就设成对应的状态。

我们给 agent 定的规矩是：谁改变了 issue 的状态，谁就顺手把它写对。读完一段工作之后把 issue 推到下一个状态，等于把球传给下一位；看状态就知道整件事目前在谁手上。

顺带说一句，我们评论里的「调研中」「结构中」这类阶段标记是团队自己定的约定，不是平台内置语义。平台原生提供的是 status（7 个内置加自定义）、子 issue 的 stage（数字批次）和自定义 property 三样东西。

## 五类单据与阶段机

平台给的是通用对象，流程得自己搭。这一章的五类单据和阶段机是我们工作区的实践，不是平台自带的概念。

阶段的真相源是 issue 上的 multi_select property。每个流程一张，选项按顺序排开；第一个选项「起始」只标类型，不推进。leader 找第一个没有勾选的选项，就知道这张单走到哪了。

| 单据类型 | property 阶段序列 | 谁推进 | 门禁 |
|---|---|---|---|
| 需求单 | 起始 → 需求分析 → 任务拆分 → 验证 | PM Aureus | 对照设计仓做覆盖判定，拆出的子单全部收口 |
| 设计单 | 起始 → 设计 → 审阅 → 合并 | Architect Orbis，外加 1 名 scope 审阅人 | 人工 merge |
| 开发单 | 起始 → 开发 → 审查 → 验证 | Vulcan / Vexel → Radian → Verity | 审查 APPROVED，验证 QA_PASSED |
| Bug 单 | 起始 → 开发 → 审查 → 验证 | 同上 | 同上 |
| 验收单 | 起始 → 验收 | Verity | TC-uat 的结论 |

Bug 单在分诊时必须锚到设计条目。没有可锚的条目，就在设计单和 Bug 单之间做选择，不能悬空。

需求单拆出来的子 issue 用 `--stage N` 标批次分批派发。最低未完成阶段的子 issue 全部 done 或取消之后，父 issue 收到通知，leader 被唤醒，由它决定要不要放行下一阶段。

`前置依赖` 是一张文本 property，值是一串逗号分隔的 TSI 编号。前置单没有全部 done，当前单就不推进，issue 退回 todo；依赖完成后，协调 autopilot 重新把它唤起来。

门禁不落在状态上。Radian 给出 `APPROVED`，Verity 给出 `QA_PASSED`，再加上设计锚点、开发评论里的实现对照表、审查评论里的契约符合性核对表和全绿的 CI，leader 才合并实现 PR，最后显式把 issue 写成 done。设计单的门禁是人工 merge，规则放在下一章。

截至 2026-10-18，带各 property 的 issue 数：开发单 508、Bug 单 772、验收单 342、设计单 120、需求单 19，`前置依赖` 129。这些数字说明流程长期在跑，不是搭好就摆着。

工作区里 3 个 squad、9 个 autopilot（截至 2026-10-18）。

| 小队 | 成员 |
|---|---|
| development | Lynx（leader）、Aureus（PM）、Vulcan（后端）、Vexel（前端）、Radian（审查）、Verity（QA） |
| architecture | Lynx（leader）、Orbis（Architect）、按范围抽 1 名审阅人 |
| writing | Vesper（leader）、Axiom（调研）、Caelum（结构）、Astra（写作）、Vanta（审稿） |

这些角色都是我们自己配的，想照搬得自己建一份。

这条流水线的链路是：需求单 → 设计单（按需）→ 开发单 ×N → 验收单。设计单和设计仓的规则放在下一章。

PR 和 issue 的关联规则要记牢，写错了不会报错，只是不关联。PR 标题或分支名里带 issue key（比如 TSI-3533），会自动关联；写在正文里的话，只有跟在 Closes、Fixes、Resolves 这类关闭关键词后面才算数，光在正文里提一句不算。关联到这个 issue 的 PR 全部合并之后，issue 会按工作区的设置自动流转，默认是 done。

这套设计背后是平台的一个取向。工作落在 review 里，不直接进 main，由人决定什么上线。配合这个取向的是可回放性：每次 run 的工具调用、命令和错误都带时间戳记下来，token 用量也在，可以按 agent 查，也可以按 issue 查。想知道某个 agent 这周花了多少，翻记录就行。

## 双仓协作架构

这一章讲的双仓是我们做产品开发时的仓库结构，本文所在的博客仓属于另一套，不在双仓之内。

双仓的意思是每个项目开一对仓库，把设计意图和实现代码分开。

| 维度 | 设计仓 | 实现仓 |
|---|---|---|
| 位置 | Gitea，命名带 `-design` 后缀 | GitHub |
| 内容 | 意图、边界、契约、验收标准、ADR | 代码 |
| 审查方式 | 人工审查 | AI 自动审查 |
| 变化频率 | 低频 | 高频 |
| 维护者 | 产生变更的人，Orbis 只承接新功能设计单与 D0 | 实现 agent |

设计仓只回答四个问题：要解决什么问题、边界在哪（`01-scope`）；对外承诺什么（`02-contracts`）；怎么算做完（`03-acceptance`）；为什么这么定（`adr/`）。实现怎么组织，不归它管。截至 2026-10-18，工作区里有 5 个设计仓，外加一个 `design-repo-template`。

文档集合是固定的：`README`、`docs/01-scope`、`docs/02-contracts`、`docs/03-acceptance`、`docs/adr/`，没有别的目录。`04-architecture`、`05-modules`、`test-cases/`、`proposals/`，以及各种索引、覆盖矩阵、状态列，都不属于这里。

一段文字该不该进设计仓，用两条判据判断：

- 删掉实现，这段文字还能驱动一次重写吗？
- 实现重构之后，这段文字必须跟着改吗？

第二条命中，就不属于设计仓。行号锚、逐字表达式、状态列和台账、UUID 或 sha 或时间戳这类运行时事实，是硬禁的四类。

谁产生变更谁写。改动按类型落到固定位置。

| 变更类型 | 落点 |
|---|---|
| D1 决策 | `adr/` |
| D2 边界 | `01-scope` |
| D3 对外契约 | `02-contracts` |
| D4 判据 | `03-acceptance` |

一个 issue 最多一条设计 PR，分支叫 `design/TSI-<n>`。D0 是待决事项，写成同一 PR 里 `status: proposed` 的 ADR；方向只由人工 merge 裁定，agent 没有合并权限。只有 D0 会让实现停工；D1 到 D4 边改边做，被拒就在同一个 issue 返工。实现细化不写、不登记，也不写「无设计影响」这种占位。

审阅只有两道：1 名 scope 审阅人和一次人工 merge。`01-scope` 和 `02-contracts` 交给 Vulcan 或 Vexel，`03-acceptance` 交给 Verity，新功能的目标、边界和 ADR 交给 Aureus；审阅人给出 `REVIEW_PASSED`，只评论，不改分支。设计 PR 的正文固定四节：语义变更、依据与取舍、影响面、待确认点。

配置上还有一道自动审查：Radian 负责的 design AI 审查 autopilot 回写 `design-review` commit status，分支保护把这个 check 设成 required，再加 1 个 approval。不设脚本闸门。截至 2026-10-18，设计 PR 的 head 上还没有出现过这个 status，自动审查尚未生效，当前实际门禁是 1 名 scope 审阅人加人工 merge。

### 可追溯

需求分析阶段把设计锚点冻进 issue：`design_anchor` 记下设计仓某次提交的 sha。此后每个阶段的评论都带一行 `设计依据: tsic/<project>-design@<sha7>`；项目没有设计仓就写 `DESIGN_REPO_ABSENT`。

开发评论带一张 `## 设计实现` 表，把每条设计要求对到代码落点；审查评论带 `契约符合性核对` 表。合并门禁要求这几样齐全，缺一样不放行。

锚点之后，别人合并进来的设计改动不算 finding。只有本 issue 自己的条目被动了，才回头读一遍，按 D0 处理。

### 用例住在哪

设计仓只存 TC 编号、场景语义、AC 归属和可观察判据。可执行形式放在实现仓的 `tests/acceptance/<TC-id>`，跟代码同一个 PR、同一条 CI。

TC-tech（P0–P1）在开发单和 Bug 单的验证阶段执行，TC-uat（P2）在验收单执行。两段都由 Verity 跑。

### 接回流水线

需求单往下走，先做设计覆盖判定：设计仓已经覆盖的，不建设计单，直接拆开发单；有缺口或者不确定的，建一张设计单，拆分等它 done，靠 `前置依赖` 卡住。方向不明的 D0 等人裁定，其余照常推进。

## 自托管与 Gitea 集成

我们用的是自托管部署。官方文档在 multica.ai，装法给了两条路径，Docker Compose 和 Helm chart。Compose 这条路是两行命令：

```bash
curl -fsSL https://raw.githubusercontent.com/multica-ai/multica/main/scripts/install.sh | bash -s -- --with-server
multica setup self-host
```

Helm 那条适合已经有 k3s 或 k8s 集群的情况，chart 在 `oci://ghcr.io/multica-ai/charts/multica`。

架构不复杂：后端是 Go 写的单二进制（REST API 加 WebSocket），前端 Next.js 16，数据库 PostgreSQL 17。真正干活的 agent daemon 跑在你自己的机器上，贴着代码，代码不出这台机器。

Git 集成只对自托管版开放，Cloud 版没有。服务端要开两个设置：`MULTICA_VCS_INTEGRATION_ENABLED=true`，再给 `MULTICA_VCS_SECRET_KEY` 一个 base64 编码的 32 字节密钥，用来加密存下来的凭据。两个都要有：前者决定这功能开不开，后者负责把凭据加密落库，少一个连接表单就是禁用的。

连接按工作区配置，入口在 Settings → Integrations 的 Git providers，填实例 URL 和一个 access token，Multica 会先拿 token 去实例上验一遍再存。token 和 webhook secret 都加密存储。这里没有 GitHub App 那种应用模型，靠的是在仓库或组织上注册 webhook。连上之后界面会给一个 webhook URL 和一个 webhook secret，secret 只显示一次，记得当场复制；服务端建议一并设好 `MULTICA_PUBLIC_URL`，否则界面只给 webhook 路径，前面那段 origin 得自己补。

Gitea 侧要注册两类 webhook：Pull Request 和 Commit Status，投递请求带 HMAC 签名（X-Gitea-Signature），服务端拿存好的 secret 逐条校验。注册完之后，PR 从打开、关闭到合并的状态变化，以及草稿状态，都会同步过来，CI checks 的 passed、failed、pending 也显示在卡片上。Gitea 这边给的 token 只要有读权限就够。

设计仓那一侧接的是同一套机制。PR 事件由组织级 webhook 送给 design AI 审查 autopilot（Radian），它把审查结论写成 `design-review` 的 commit status 和一条 PR 评论；分支保护把这个 check 设成 required，并要求 1 个 approval。设计仓的自动审查就挂在这一套上。

agent 开 PR 不需要在 Multica 里配任何 provider 信息。它用的是 daemon 宿主机的 git 凭据，SSH deploy key 或者 credential helper 都行，能 push 就能开 PR。

我们自己的形态是自托管 Multica 加自托管 Gitea，代码仓库都在内网。

## 踩过的坑和写进 instructions 的规则

这一章是我们真踩过的地方。每条按「现象 → 规则」写，规则可以直接抄进你们自己的 agent instructions。

### 评论正文要用文件，不要走内联参数

现象：agent 回帖带一段长正文，用内联参数传内容，遇到引号、换行和特殊字符就出错。改用 heredoc 之后又发现，它和别的参数混在一起时会把尾部参数吞掉，命令看着成功，实际少传了东西。临时文件写 /tmp 也不安全，多个 agent 并发会互相覆盖。

规则：正文一律先写进工作目录里的文件，再用 `--content-file` 传给 CLI；不要把 heredoc 和别的参数混着用；临时文件写工作目录，不写 /tmp。这三条对应 MUL-2904、#4182 和 MUL-4252。

### run 一结束，后台的活就没人管了

现象：agent 把耗时的工作丢到后台，结束当前轮次去等结果。轮次一结束，任务就被标记为终止，后台的活变成孤儿，结果丢失，本来要发的最终评论也发不出去。

规则：需要结果的工作必须在前台阻塞收完，不要「挂到后台，等会儿再看」。

### mention 会真的启动一次运行

现象：评论里 @ 一个 agent 看着只是打个招呼，实际会为它排一次新的运行。礼貌性的致谢，会实实在在花掉一次运行预算。

规则：@ 之前先确认这是不是一件新工作。只是通知，别用 mention。

### PR 关联不上，先查集成侧

现象：issue 和 PR 该关联却没关联，改了几遍 PR 描述都没用。

规则：先检查写法，看 issue key 是不是只在正文里裸提到了；再检查集成侧配置。反复改 PR 是白费力气。

### leader 派发完不等于父 issue 完成

现象：squad leader 把子 issue 都派出去之后，父 issue 的状态没人动，服务器也不会自动翻。

规则：父 issue 的状态由 leader 显式确认后再写。派发只是开始。

### 内网 CA 要加进信任库

现象：Git 实例用自签名证书或者内网 CA 签发的证书，Multica 后端连不上，报证书不受信任。

规则：把 CA 加进后端信任库，Helm 部署设 `backend.extraCACerts.configMap`，Compose 部署挂载 CA 并设 `SSL_CERT_DIR`。平台不提供跳过证书校验的选项，别在这上面找绕过的办法。

### 设计仓会自己长成第二个实现仓

现象：设计仓只要不设限，就会把实现细节往里装。架构分层、模块清单这类结构文档先长出来，脚本和台账跟着进，评审的精力被台账吃掉，设计 PR 能比实现 PR 还多。

规则：文档集合固定，`README`、`01-scope`、`02-contracts`、`03-acceptance`，再加一个 `adr/` 目录；`test-cases/`、`proposals/`、索引、覆盖矩阵、状态列一律不进；只有 D1–D4 才动设计仓，一条 issue 最多一条设计 PR，只有 D0 需要等人裁定。判断标准就是那两条：删掉实现还能驱动重写，实现重构不必跟着改。

平台几乎每个工作日都在发版。本文的功能细节和数字取自 2026-10-18，真要照着做，建议回原文核对一遍。
