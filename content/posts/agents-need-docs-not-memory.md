---
title: "Agent 要的是文档，不是记忆"
date: 2026-10-05T22:30:00+08:00
tags: ["Agent", "上下文工程"]
author: "mywebtestmail"
---

2026 年 10 月 3 日，Kevin Liao 发了篇短文[《Agents Don't Need Memory. They Need Documentation.》](https://liao.gg/blog/agents-dont-need-memory)。两天之内，它在 Hacker News 上攒到 368 分、274 条评论（截至 2026-10-05，数据取自 [HN 官方 API](https://hacker-news.firebaseio.com/v0/item/49945933.json)，讨论树见 [Algolia 快照](https://hn.algolia.com/api/v1/items/49945933)）。讨论热，不是因为观点新，而是因为它把一件大家默认对的事反过来问了一遍：agent 记不住东西，给它加记忆不就完了吗？

原文的答案是不，那句被反复引用的话是：Because agents don't need memory. They need documentation. 口号看着简单，文章里倒给了一条挺完整的推理链。我想顺着这条链捋一遍，顺带说说哪些地方站得住，哪些地方还只是个人主张。

## 记忆插件的回路

先看现在主流的「给 agent 加记忆」是怎么做的。[原文](https://liao.gg/blog/agents-dont-need-memory)把这类插件的架构拆成固定五步：

1. Go through session transcripts
2. Generate snippets of 'memories'
3. Insert into a RAG database
4. On every prompt, retrieve the top 5 and inject
5. Need more? Give the agents a tool to search through the RAG database

这套流程不新鲜。[Mem0 官方文档](https://docs.mem0.ai/overview)把自己的工作方式写成三步：Add → Extract and store → Recall，还明确讲 "Mem0 runs the vector store and rerankers"。[LongMemEval](https://arxiv.org/abs/2410.10813) 那篇论文（ICLR 2025）把长期记忆统一拆成 indexing / retrieval / reading 三个阶段。也就是说，「抽取、存储、按相关性检索、注入」不是某个人的归纳，而是这一层产品的共同骨架。

原文要质疑的，是骨架背后的那个假设：

> Agents forget: that's the problem. So the fix is to remember. To remember better, we should capture more, index better, retrieve smarter.

问题在于，「记得更多」和「用得上」是两件事。五步流程优化的是存储与召回，它默认被召回的东西就是 agent 此刻需要的东西。这个默认值值得单拎出来看。召回率衡量的是「该找的有没有找全」，可它默认了「找出来的都应该用」。这两件事之间隔着一层判断：模型看到注入的片段之后，得自己决定要不要采信、要不要据此行动，而这层判断的输入从一开始就被筛选过。流程把筛选放在模型之前，也就把判断的前提先定死了。

原文还提了一处更细的变化。它说 agentic loop 正在从 `prompt → build → forget` 变成 `prompt → consult → build → update`。差别在开头和结尾：以前是干完就忘，下次从头来；现在动手之前先查一遍已有文档，干完把学到的写回去。注意它把这一步叫 update，不是 append。文档是被改写的，旧结论该推翻就推翻，而不是攒成一堆等着被检索。

五步里的第二步是 Generate snippets of 'memories'，这一步本身就有问题。一段对话被压成一句记忆，压掉的主要是上下文：当时为什么这么决定、前提是什么、有没有被推翻过。原文说这些记忆是 stored without context，等它们被检索出来注入 prompt，agent 看到的是一句没有因果的断言。它只能选择信或不信，没法追问。而追问恰恰是人在处理不确定信息时最本能的动作。

还有一处是第 4 步：On every prompt, retrieve the top 5 and inject。这个 5 是固定值，由检索器在注入前拍板，而不是模型读到内容之后再决定。这就把「该给多少上下文」这个判断，从使用知识的模型手里，挪到了一个只会算相似度的检索器手里。原文说这套流程的优化目标是 capture more、index better、retrieve smarter，可它从头到尾没人问一句：这 5 条对当前任务是不是够，或者是不是太多了。

[为什么重要]：如果回路的目标只是「召回更多」，那它就没有任何机制去回答「这条信息现在还成立吗、和当前任务有关吗」。召回率上去了，理解不一定跟上。

## 相似度排序的两处盲区

原文列了五个小标题，我挑两个最要命的。

第一个是 Memories are surfaced by similarity。向量检索按语义相似度排序，相似不等于相关，更不等于正确。[Chroma 在 2025 年 7 月做过一组实验](https://research.trychroma.com/context-rot)，叫 Context Rot：把任务复杂度固定，只拉长输入，在 18 个模型上测，性能仍然随输入变长而退化，而且是 "often in surprising and non-uniform ways"。他们顺手点了一个更隐蔽的问题：常见的 NIAH 测试只考词面检索，会低估真实难度。[Anthropic 官方博客](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)也引用了这组结论，[复现代码](https://github.com/chroma-core/context-rot)公开在 GitHub。

第二个盲区更根本：Agents can't search for what they don't know。检索的前提是你知道要找什么。agent 不知道明天会遇到什么任务，也就没法为未来去检索。[LongMemEval 那篇](https://arxiv.org/abs/2410.10813)给了一组数：商用助手和长上下文模型在跨会话记忆上出现 30% 的准确率下降，并且把 knowledge updates（知识更新）和 abstention（该拒答时拒答）列为长期记忆的核心能力。这两项恰好对应原文说的另外两个毛病：The past is treated as truth，以及 The store is unauditable。旧记忆被注入时，没人问它有没有过期；agent 也说不清自己是「不知道」，还是「记得的是过时的」。

更麻烦的是，这三处毛病不是换个更好的 embedding 模型能修的。相似度只对「像不像」负责，对「还对不对」没有发言权；检索工具只对「找没找到」负责，对「该不该找」没有发言权。原文那句 Agents can't search for what they don't know，说的其实是检索这件事的天花板：它需要一个已知的问题，而 agent 真正需要知识的时候，往往还没意识到自己缺的那块是什么。这是一个先有鸡还是先有蛋的困境，向量库没法自己爬出来。

[为什么重要]：相似度排序解决的是「找得出来」，解决不了「找出来的是对的」。加厚向量库是在放大召回，不是在修补这两处盲区。

## 文档即状态

如果记忆的毛病不是存得不够，那该换成什么？原文的答案是文档。Memory turns from a RAG database you bolt onto the agent into a workspace you can read, update, and even share.

这句话到 2026 年已经不是个人主张了，一线厂商就是这么实现的。

[Anthropic 2025 年 9 月 29 日的工程博客](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)讲 Claude Code 的上下文策略：CLAUDE.md 被 "naively dropped into context up front"，需要的细节再靠 glob 和 grep 做 just-in-time 检索，"effectively bypassing the issues of stale indexing and complex syntax trees"。同一篇里，随 Sonnet 4.5 发布的 memory tool 是 file-based，官方描述是 "store and consult information outside the context window through a file-based system"。

[Claude Code 的官方文档](https://code.claude.com/docs/en/memory)把跨会话知识收窄成两个机制：CLAUDE.md 和 auto memory，都是纯 Markdown，"Everything is plain markdown you can read, edit, or delete"。代价也写得很直白：auto memory 每个会话只加载 first 200 lines or 25KB，文件超过 4 MiB 直接跳过，"Longer files consume more context and reduce adherence"。官方给的出路，是把多步流程下放到按需加载的 skills 和 path-scoped rules。

原文点名的那个插件更极端。GitHub 上的 [aerovato/operator-memory](https://github.com/aerovato/operator-memory)（BSD-3-Clause，TypeScript，2026-08-16 建仓，最近一次 push 在 2026-10-05，截至 2026-10-05 有 318 stars），README 三条卖点分别是 Zero Infrastructure（No background agents, no embeddings, no vector database, no model configuration）、Transparent Memory（Memory is stored as Markdown documents you can read, update, and delete）、Sharable Knowledge（Track documents with Git and share knowledge with your team）。它把记忆分成 `.operator/`、`.operator-shared/`、`~/.operator/user/` 三个区，本地、随仓库、跨项目各一份，循环就三步：Consult、Build、Update。它的 README 里还有一句立场很鲜明的话：Snippet capture, RAG retrieval, and context compression all fail the same way. 这是作者自家产品的宣传语，不是第三方评测，读的时候得打个折扣。

生态这一侧也在往同一个方向收。[AGENTS.md 官网](https://agents.md/)说这个格式由 OpenAI Codex、Amp、Google Jules、Cursor、Factory 等一起推动，现在归 Linux Foundation 下面的 [Agentic AI Foundation](https://aaif.io/) 托管，支持的工具有 Codex、Jules、Cursor、GitHub Copilot、VS Code、JetBrains Junie、Devin/Windsurf、Gemini CLI、goose、Zed、Amp、opencode、Aider、RooCode、Warp、Augment 等。官网自称 GitHub 上有 60k+ examples，OpenAI 主仓库自己就有 88 个 AGENTS.md（这两个数都来自官网自述，没有独立统计）。规则上支持嵌套，离被编辑文件最近的 AGENTS.md 生效，FAQ 里那句写得很清楚：Treat AGENTS.md as living documentation.

不过这里有一处分歧值得点明。原文反对「给 agent 一个工具去翻一千万 token 的历史」，Anthropic 却主张让 agent 用文件系统、glob、grep 做 just-in-time 检索。两边都反对「预先把一切都塞进向量库」，但在「要不要给检索工具」上站到了对面。原文说的是别拿检索工具去翻旧账，Anthropic 说的是拿检索工具去翻当前工作区的文件。差的不是工具，是工具指向的东西：一个是历史切片堆，一个是活的文档。

[为什么重要]：文档路线之所以比向量库路线结实，不是因为它更「先进」，而是因为厂商已经在实现它。一边是产品里的默认行为，一边是还在补评测的新项目，成熟度不在一个量级。

## 可审计与提交

文档路线真正多出来的东西是两样：可读，和可提交。

Markdown 落在仓库里，就能进 git。谁改的、什么时候改的、改之前是什么，diff 说得清。这一点向量库给不了：数据库里的记忆块没有版本，改了就是覆盖，错了也追不回来。原文把这条概括成 The store is unauditable。

可提交还意味着可以被评审。AGENTS.md 的 FAQ 说它是 living documentation，隐含的意思就是它会变，变的时候要走和代码一样的流程。一条记忆该不该留下，原则上可以像一行代码那样被人 review，而不是写进某个黑箱里等它下次被检索出来。

这里得正面回应一个最强的反驳。HN 上有人（jdw64，评论 49948288）搬出 Peter Naur 的《Programming as Theory Building》：一个程序真正的价值，是建造它的团队脑子里那套「理论」，也就是为什么这样设计、哪些取舍被考虑过。按这个说法，文档记的是结论，记不住理论；换个人看文档，也未必能重建同一个心智模型。这个反驳有力，但我觉得它更支持文档路线，而不是反对它。向量库里那一句句记忆片段同样承载不了理论，甚至更差，因为它连结论的来龙去脉都压掉了。文档不能自动保存理论，可它是唯一能被反复阅读、被人补充、被 review 的载体，理论有机会附着在上面。数据库给不了这个位置。

但这条路线眼下有个躲不掉的软肋：没有第三方评测。2026-10-04，用户 mpicard 在 operator-memory 的 [issue #48](https://github.com/aerovato/operator-memory/issues/48) 里直接问 "Benchmarks? Evals?"。作者 kevinMEH 当天回复并关闭了 issue，说得很实在：现有的记忆 benchmark（LoCoMo、LongMemEval、MemBench、MemoryAgentBench）"are designed to test RAG database wrappers, not actual agent memory systems"，他自己也没有资源去跑长周期的 SWE benchmark。另一头，原文说自己用这套系统 "over a year"，可公开产物只有约 50 天：仓库和 npm 包都是 2026-08-16 建的（npm 包叫 [@aerovato/operator-helper](https://registry.npmjs.org/@aerovato%2Foperator-helper)，最新 1.8.0，发布者 kmeh@liao.gg）。自述时间和公开时间线对不上。

[HN 评论区](https://hn.algolia.com/api/v1/items/49945933)也不是一边倒。有人（docheinestages，评论 49948639）觉得一条 AGENTS.md 指令就够了，何必专门装工具；还有人（stbenjam，评论 49950536）担心 agent 生成的 Markdown 会涨到 megabytes，却没有有效性证据。这些是观点，不是事实，但值得摆出来，因为文档路线要成立，得先回答它们。这里只说代表性意见，不排高低。

[为什么重要]：把话说满就成了另一种营销。文档式记忆现在是一个有厂商实现、有生态标准、但缺第三方评测的方向。它比向量库更可读、更可审计，这不等于它已经被证明更有效。

## 先定义问题

绕一圈回来，这篇文章真正立得住的地方，是它换了问题的问法。

向量库路线的问法是：怎么记得更多、检索更准。文档路线的问法是：agent 此刻该知道什么，这些知识谁能读、谁能改写、谁能为它负责。前一个问题的答案是一堆参数，后一个问题的答案是一张工作区。

Anthropic 那篇博客里有个说法可以接着往下想：上下文是一种有限资源，agent 要的是 "smallest possible set of high-signal tokens"。这句话的方向和「记得更多」正好相反。文档路线顺着它，是因为一份写清楚的文档可以只放结论和约束，需要细节的时候再去读；向量库里那些碎片做不到这一点，颗粒度太细，你不知道该信哪几条，只好整个塞进去。结果就是花掉预算，换来一堆需要自己再判断的东西。

原文那句判断我认同：召回不等于理解，知识要可读、可提交。但它是方向性的判断，不是实验结论。谁要拿它当定论，得先补上它缺的那块：长周期任务上的对照评测。在那之前，更稳的姿势是把它当成一个提醒：在选检索方案之前，先把问题定义对。是「记不住」，还是「记的东西没有出处、没有版本、也没人知道它过期了」？

这两个问题长得像，解法差很远。前者是模型能力问题，方向是更大的窗口、更好的召回，那是模型厂商该去卷的；后者是工程和协作问题，方向是版本控制、评审、责任归属，得由用 agent 的团队自己补。把两件事混在一起，很容易得出「再买一个记忆插件」的结论。而那恰恰是原文最想否掉的那个默认动作。

## 参考

- [Agents Don't Need Memory. They Need Documentation. — Kevin Liao](https://liao.gg/blog/agents-dont-need-memory)
- Hacker News 讨论：[官方 API 条目](https://hacker-news.firebaseio.com/v0/item/49945933.json) · [完整讨论树（Algolia 快照）](https://hn.algolia.com/api/v1/items/49945933)
- [Mem0 官方文档 · Overview](https://docs.mem0.ai/overview)
- [LongMemEval: Benchmarking Chat Assistants on Long-Term Interactive Memory（ICLR 2025）](https://arxiv.org/abs/2410.10813)
- [Chroma · Context Rot](https://research.trychroma.com/context-rot) · [复现代码](https://github.com/chroma-core/context-rot)
- [Anthropic · Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- [Claude Code 文档 · Memory](https://code.claude.com/docs/en/memory)
- [aerovato/operator-memory](https://github.com/aerovato/operator-memory) · [issue #48 «Benchmarks? Evals?»](https://github.com/aerovato/operator-memory/issues/48) · [npm @aerovato/operator-helper](https://registry.npmjs.org/@aerovato%2Foperator-helper)
- [AGENTS.md](https://agents.md/) · [Agentic AI Foundation](https://aaif.io/)

*上述链接于 2026-10-05 采集、2026-10-15 复核可访问；HN 数据取自官方 API（news.ycombinator.com 本机不可直连）。*
