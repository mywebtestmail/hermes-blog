---
title: "把博客生产流水线搬进 Hermes Kanban：从微信选题到 GitHub Pages 自动发布"
date: 2026-09-10T09:00:00+08:00
tags: ["Hermes", "Kanban", "自动化"]
author: "mywebtestmail"
---

## 引言：一条选题从微信进来之后

某天你在地铁上，通过微信提了一条需求：用 Hermes Kanban 协作，把一篇 2000 字技术博客从选题一路推进到 GitHub Pages 上线（原文是口语化长句，此处为转述）。如果你把它丢给一个单轮长 prompt 的 agent，接下来大概率会撞上三件事：

一是上下文漂移。调研、写作、审核、发布全挤在一次生成里，模型写到后半段时，前半段定下的口径已经模糊，数字开始对不上。

二是无法审核。一次生成直接产出终稿，中间没有可停留的检查点，你只能要么全信、要么重写。

三是失败不可重试。如果发布那一步因为一个 frontmatter 字段写错而失败，你重跑一遍，前面的调研和写作也全部重来——没有哪一步的结果被单独存下来。

这三个不稳定点，本质是同一个问题：把一个多阶段、可分工、可独立失败的生产过程，塞进了一次不可分割的生成里。Hermes Kanban 要做的，就是把它拆开。

[为什么重要]：流水线之所以是流水线，是因为每一步都有明确的输入、输出和状态；单轮长 prompt 恰好抹掉了这三样。

## 为什么是 Kanban，而不是一条长 prompt

在 Hermes 里，能串起多步工作的不只 Kanban：`delegate_task` 能并行发多个子代理，`cron` 能定时重跑。它们的边界在哪里？

| 机制 | 适合场景 | 阶段依赖 | 失败恢复 | 审核点 |
| --- | --- | --- | --- | --- |
| 一条长 prompt | 单轮、短流程 | 隐式 | 整体重来 | 无 |
| `delegate_task` | 并行独立子任务 | 弱（结果汇总） | 单任务重跑 | 无固定 |
| `cron` | 定时重跑 | 无 | 下次执行 | 无 |
| Kanban | 有向无环的多阶段 | 显式（`parents`） | 单卡重试 + 熔断 | 有（`review` 列） |

Kanban 的四个属性正好对上博客生产的痛点：可审计——每张卡有自己的评论、事件流和 run 记录；可重试——连续 2 次非成功（spawn_failed / timed_out / crashed）即熔断进入 blocked（生效上限依次取任务级 `max_retries` → `kanban.failure_limit` → 内置 2，本机未设，实际就是 2），但单卡失败不会让整条链重跑；可并行——没有依赖关系的卡可以在同一批里同时跑；可分工——每张卡指定一个 `assignee`（一个 profile），不同角色用不同的 SOUL、技能和模型。

[为什么重要]：选错编排工具比不编排更糟。如果流程有「上一步输出决定下一步做什么」这种硬依赖，还要求失败能单点重试、中途有人审核，那 Kanban 是四个选项里唯一把依赖和审核显式化的。

## 拆图：五个 profile 与七个阶段

本机 `hermes profile list` 列出六个 profile：`default`、`orchestrator`、`publisher`、`researcher`、`reviewer`、`writer`。除 `default` 外，其余五个正好对应流水线的五个角色。模型也有区分：`writer` 用 deepseek-v4-pro，其余角色用 deepseek-v4-flash——写作这种「一次性产出 2000 字成品」的活，给了更强的模型。

七个阶段连成一条链：

微信选题接收（根卡）→ orchestrator 拆解并产出简报 → researcher 调研 → writer 初稿 → reviewer 审核 → writer 终稿 → publisher 发布

阶段顺序不靠自然语言留言，而是靠 `parents` 数组显式表达。上游卡 `kanban_complete` 之后，下游卡才会从 `todo` 提升为 `ready`。这条提升规则在源码里很硬（下文源码结论均基于本机 v0.21.0）：`recompute_ready()` 遍历所有 `todo`/`blocked` 卡，只有当全部直接父任务的状态属于 {done, archived} 时才提升。更关键的是，`claim_task()` 在事务内还会再查一次 `_parents_satisfied()`——父没全 done 就强行认领，会在写入层被拦下，把卡降回 `todo` 并写一条 `claim_rejected` 事件。也就是说，「父未完成就进入 running」这个 bug 在数据库层面就不可能发生。这条链还有一条回边：reviewer 审核不通过，卡回到 writer 返工重写。

[为什么重要]：依赖显式化之后，「顺序」从「某个 agent 恰好记得」变成「数据库里的一行约束」，这是整条流水线可被信任的前提。

## 简报：把一句话选题变成可执行 spec

「帮我写一篇博客」这句话本身不可执行——它没说写给谁、论点是什么、多少字、发到哪。orchestrator 这步的产出是一份 brief，把它落成七件套：

1. 选题来源假设（无微信原文时按兜底规则以根卡正文为选题来源）
2. 目标读者
3. 核心论点
4. 关键问题清单
5. 大纲
6. 字数
7. 发布目标

发布目标在本例是具体的：仓库 `mywebtestmail/hermes-blog`，main 分支，站点 `https://mywebtestmail.github.io/hermes-blog/`，slug 建议 `hermes-kanban-blog-pipeline`。

[为什么重要]：spec 一旦写成文字，后续每一张卡都能据此自检，而不是各自发挥。

## 调研：结论先行 + 可溯源

researcher 的输出有一条纪律：每条都按「结论 / 证据 / 不确定项」三段式写。结论直接给答案；证据落到源码文件行号或实测命令；拿不准的必须显式标「不确定项」。

这套纪律解决了「AI 编造」这个老问题，因为证据是可定位的。比如这次调研就挖出一个文档与代码打架的坑：[官方微信文档](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/weixin/)写「消息上限 4000 字符」，但 v0.21.0 源码里 `WeixinAdapter.MAX_MESSAGE_LENGTH = 2000`（分片阈值 1800，注释说 iLink 在约 2048 字符处切分）。写文章时必须按 2000 引用，不能抄文档的 4000。

凡是没实测的，一律标注。例如微信 `errcode -2`（频率限制）的熔断行为——第一次收到 -2 就打开 30 秒熔断、丢弃同一消息的剩余分片——是源码级证据，本机没有真实触发复现，笔记里就明确写了「非实测」。

[为什么重要]：可溯源是写作者与读者之间的信任契约；标注不确定项让下游 reviewer 知道该盯哪里。

## 写作与审核：把 review 变成可执行清单

writer 产出初稿后，reviewer 不做「我觉得挺好」这种模糊评价，而是按三级给清单：阻断（不解决不能发布）、重要（强烈建议）、建议（可选）。每一条都带「位置 + 问题 + 改法」，例如「正文引用的微信字数上限与源码不符，位置：调研一节，改法：改为 2000 并注明文档与代码不一致」。

写作与审核分属两个 profile，且写作横跨初稿与终稿两张卡，这正是「可审核」的落点——写作的人不能自己给自己发通过。

[为什么重要]：可执行的 review 清单是返工成本最低的审核形式，因为它直接告诉 writer 改哪里、改成什么。

## 发布：Hugo 仓库里的一个 commit

publisher 的产出不是一句「发布完成了」，而是一个可验证的 commit。文章落到 `content/posts/hermes-kanban-blog-pipeline.md`（不带日期前缀，与 `welcome.md` 同形制），URL 为 `https://mywebtestmail.github.io/hermes-blog/posts/hermes-kanban-blog-pipeline/`；frontmatter 的 `title` / `date` / `tags` / `author` 四字段与仓库现有文章一致：

```markdown
---
title: "把博客生产流水线搬进 Hermes Kanban：从微信选题到 GitHub Pages 自动发布"
date: 2026-09-10T09:00:00+08:00
tags: ["Hermes", "Kanban", "自动化"]
author: "mywebtestmail"
---
```

本地构建与预览（Hugo v0.166.0 extended）：

```bash
cd C:/Users/yangt/projects/hermes-blog
hugo --minify --gc
hugo server --buildDrafts   # 本地预览
```

这里有一条硬约束来自仓库的 PR 模板：`date` 不能是未来时间。Hugo 默认不构建未来日期的文章，页面会静默消失，而且 CI 不报错——本地构建 `hugo --minify --gc` 通过不代表文章能上线。

发布链路分两步：workflow 的 `deploy` job 只在 push / workflow_dispatch 到 main 时执行，所以 PR 只做 build 验证，合并进 main 才会真正触发 GitHub Pages 部署，构建用的是 Hugo v0.166.0 extended。

[为什么重要]：可验证的交付物（一个 commit + 一个线上 URL）比任何口头「已发布」都可信，因为下游可以自己去看。

## 复盘：值得固化的三件事（以及什么时候别用 Kanban）

三件事值得固化：

1. 依赖显式。用 `parents` 表达阶段顺序，而不是靠谁在评论里「记得」。
2. handoff 结构化。阶段间传递靠 run.summary 和 run.metadata 这两个列级字段——父卡一 done 就自动进子卡上下文，不依赖人工转述。这是「结构化 handoff 比自然语言留言可靠」的机制证据：worker 上下文里的 `## Parent task results` 区块就是干这个的，还带相对时间戳和「快照过旧需回源复核」的提示。
3. 审核独立。写作与审核分属不同 profile，杜绝自审自过。

但 Kanban 不是银弹，两个边界要认清：调度有延迟——dispatcher 默认 60 秒一轮 tick，不是实时的；上下文重复传递——worker 上下文的正文与父任务交接字段都有配额上限（正文 8 KB、每个父任务字段 4 KB），长链会把中间产物反复塞进每个下游卡。如果你的流程只有两三个无强依赖的独立子任务，`delegate_task` 更轻；如果只是定时重跑同一件事，`cron` 更合适。

## 总结

一条微信选题，最终目标是 GitHub Pages 上一个可点开的 URL，中间经过五个 profile、七张卡。真正让这条流水线可靠的，不是某个 agent 有多聪明，而是几件看起来笨的事：依赖写进数据库、结论带证据、审核分等级、交付物可验证。如果你手头正有一个「多阶段、可分工、需要审核」的生产过程，不妨把它拆成卡试试——先从把一句话选题写进一份七件套 brief 开始。
