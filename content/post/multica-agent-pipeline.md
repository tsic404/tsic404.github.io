---
title: "Multica 与我们的多 agent 开发流水线：从派发 issue 到合并 PR"
date: 2026-09-27T03:00:00+08:00
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

Multica 就是干这个的工具。它把一个工作区里的 issue 派给 AI 编码 agent，像给同事派活：agent 自己认领任务，报告进展，遇到阻塞提出来，做完交回审查。平台是 source-available 的，许可为 Multica License，自托管免费，支持 26 种 agent CLI，换模型供应商只是换一个下拉选项。截至 2026-09-24，官方仓库有 51.3k stars 和 6.6k forks，累计 5,472 commits，按官方说法大部分工作日都在发版。

下面从核心对象讲起，再讲我们自己那条流水线。

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

平台不内置模型。它驱动的是你已经安装并登录的 agent CLI，官方列了 26 种，Claude Code、Codex、Cursor、Copilot、Kimi、OpenCode 都在其中。想换供应商，改一个配置项就行。

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

类别决定状态在板上的归类，unstarted 是还没开始，started 是进行中，done 和 closed 分别对应完成和取消。owner 和 admin 可以给工作区加自定义状态，类别在创建时定死，之后改不了，加之前想清楚它属于哪一类。我们加了两个，code_review 和 auto_test，插在 in_progress 和 done 之间，对应「审查」和「QA」两道门禁。

状态怎么变？由 agent 通过 CLI 显式写入，服务器不会在 run 开始或结束时自动翻状态。run 跑完了，issue 还停在原地，除非有 agent 主动把它推到 in_review 之类的位置。

只有两个系统例外。run 失败且不重试时，in_progress 会回滚到 todo；带关闭关键词的 PR 合并时，issue 会被置成 done。

我们给 agent 定的规矩是：谁改变了 issue 的状态，谁就顺手把它写对。读完一段工作之后把 issue 推到下一个状态，等于把球传给下一位；看状态就知道整件事卡在哪一环，不用翻评论找进度。我们的审查和 QA 门禁就是靠这个串起来的。

顺带说一句，我们评论里的「调研中」「结构中」这类阶段标记是团队自己定的约定，不是平台内置语义。平台原生提供的是 status（7 个内置加自定义）和子 issue 的 stage（数字批次）两样东西。

## 我们的四类单据和两道门禁

平台给的是通用对象，流程得自己搭。我们用自定义 property 做阶段追踪，把工作分成四类单据。先声明一句：这四类单据是我们工作区的实践，不是平台自带的概念。

| 单据类型 | property 阶段序列 | 对应门禁 |
|---|---|---|
| 需求单 | 起始 → 需求分析 → 任务拆分 → 验证 | 拆出的子任务全部收口 |
| 开发单 | 起始 → 开发 → 审查 → 验证 | 审查和测试两道 |
| Bug 单 | 起始 → 开发 → 审查 → 验证 | 同上 |
| 验收单 | 起始 → 验收 | 验收结论 |

需求单负责把一件事说清楚再拆开。拆出来的子 issue 用 `--stage N` 标批次分批派发，最低未完成阶段的子 issue 全部 done 或取消之后，父 issue 会收到通知，把 leader 唤醒，由它决定要不要放行下一阶段。

开发单和 Bug 单是主力，日常大部分流转都发生在这两类单据上。截至 2026-09-24，开发单累计流转 408 次，Bug 单 622 次，验收单 317 次。这些数字说明流程长期在跑，不是搭好就摆着。

两道门禁落在状态上：开发单走到「审查」，对应 code_review，由审稿 agent 检查；走到「验证」，对应 auto_test，由 QA 执行。两道都过，单据才算走完。

PR 和 issue 的关联规则要记牢，写错了不会报错，只是不关联。PR 标题或分支名里带 issue key（比如 TSI-3533），会自动关联；写在正文里的话，只有跟在 Closes、Fixes、Resolves 这类关闭关键词后面才算数，光在正文里提一句不算。关联到这个 issue 的 PR 全部合并之后，issue 会按工作区的设置自动流转，默认是 done。

这套设计背后是平台的一个取向。工作落在 review 里，不直接进 main，由人决定什么上线。配合这个取向的是可回放性：每次 run 的工具调用、命令和错误都带时间戳记下来，token 用量也在，可以按 agent 查，也可以按 issue 查。想知道某个 agent 这周花了多少，翻记录就行。

四类单据只是骨架。双仓协作架构会改写其中两处：需求单多一道「设计确认」，开发单的「审查」变成对照设计仓边界的符合性审查。下一章展开。

## 双仓协作架构

先说清一件事，这一章讲的双仓是我们做产品开发时的仓库结构，本文所在的博客仓属于另一套，不在双仓之内。

双仓的意思是每个项目开一对仓库，把设计意图和实现代码分开：

| 维度 | 设计仓 | 实现仓 |
|---|---|---|
| 位置 | Gitea，命名带 `-design` 后缀 | GitHub |
| 内容 | 意图、边界、契约、验收标准 | 代码 |
| 审查方式 | 人工审查 | AI 自动审查 |
| 变化频率 | 低频 | 高频 |
| 维护者 | Architect，唯一 | 实现 agent |

设计仓放的是意图、边界、契约和验收标准。它变化慢，改一次要走人工审查。实现仓装代码，变化快，审查交给 AI 自动做。

设计仓只有一个维护者，叫 Architect。它是唯一允许写设计仓的 agent，只写设计文档，不碰实现代码。需求进来先经过它，被翻译成 goals、boundaries、contracts、acceptance 四类内容，再加上对应的 test-cases。

它写出来的东西照样走 PR，区别在审查这一关：设计仓的 PR 由人来看，实现仓的 PR 交给 AI。前者改一次影响面大，后者改得勤，用两套审查强度是划算的。

有了设计仓，实现 agent 动手之前就有了判断依据，于是有一条方向门禁：

设计明确 → 直接编码；缺失 / 模糊 / 冲突 / 超界 → proposals PR → 人工确认

设计仓写得明确，实现 agent 直接编码。设计仓缺失、模糊、互相冲突，或者需求超出了写好的边界，就禁止猜，改为提 proposals PR，等人确认之后再继续。

验收拆成两段，都交给 QA 执行。技术验收（TC-tech）对照契约和边界，在开发单的「验证」阶段跑；用户视角验收（TC-uat）对照 goals，在验收单阶段跑。两段看的东西不一样，一段检查有没有按约定做对，一段检查有没有解决当初要解决的问题。

落到单据流程上，双仓改写了两个地方。需求单在任务拆分之后多一个「设计确认」环节，由人在 Gitea 上 merge 设计 PR 完成；开发单的「审查」从泛泛的代码审查，变成对照设计仓边界和契约的符合性审查。

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

agent 开 PR 不需要在 Multica 里配任何 provider 信息。它用的是 daemon 宿主机的 git 凭据，SSH deploy key 或者 credential helper 都行，能 push 就能开 PR。

我们自己的形态是自托管 Multica 加自托管 Gitea，代码仓库都在内网。

## 六个坑和六条规则

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

平台几乎每个工作日都在发版。本文的功能细节和数字取自 2026-09-24 前后的官方文档和仓库，真要照着做，建议回原文核对一遍。
