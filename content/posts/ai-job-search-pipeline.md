---
title: "把简历当编译产物：ai-job-search 如何把求职做成一条可验证的流水线"
date: 2026-10-19T22:15:00+08:00
tags: ["AI", "Agent", "求职"]
author: "mywebtestmail"
---

2026 年 3 月 18 日建仓，到 10 月 19 日，[MadsLorentzen/ai-job-search](https://github.com/MadsLorentzen/ai-job-search) 已经有 55,747 个 star 和 17,841 个 fork。这个 fork/star 比是 32%，而同期几个大仓库是：`anthropics/claude-code` 17%、`openai/codex` 15%、`NationalSecurityAgency/ghidra` 11%、`heygen-com/hyperframes` 9%。

一个没有服务端、没有云端的仓库，超过三成的 star 变成了 fork，原因写在 README 第一行：*Fork it and own it*。它的用法就是把这个仓库复制成你自己的求职工作区。这个产品形态决定了后面所有架构——包括它的安全模型和它最容易被忽略的那个坑。

## 它不是一个 SaaS，是一份能执行的规程

克隆下来看目录结构，会发现代码很少，说明书很多：`.claude/commands/` 下 12 个 slash command，其中 `apply.md` 40 KB、`rank.md` 21.7 KB；`.claude/skills/job-application-assistant/` 下 7 份方法论文档；`.agents/skills/` 下 6 个招聘网站 CLI；产物是 LaTeX 编出来的 PDF。

换句话说：**方法论被写成了说明书，而说明书就是程序。** 40 KB 的 `apply.md` 不是给人读的文档，是给模型的执行规程——每一步做什么、验收什么、什么情况下停下来问人，全在里面。

[为什么重要]：在这类 Agent 项目里，规格文件的字节数是它的复杂度预算。一个 40 KB 的规程，意味着作者把大量"模型很容易做错的地方"提前写成了硬性步骤，而不是靠临场发挥。

## 两道理性闸门，先于打分

大多数"AI 改简历"工具从一句话开始：把 JD 和简历喂进去，出来一段更漂亮的文字。这个框架的第一步是**否决**。

**Eligibility Gate**：把招聘启事里关于公民身份、永居、安全审查的措辞逐字读出来，分成 FAIL / PASS / 未验证三类。两条容易出错的规则被单独点名：沉默不等于许可（大型校招项目常把资格要求放在公司官网而不是招聘页）；公司级"我们欢迎国际申请者"不等于岗位级许可（同一句话后面往往跟着一个具体项目清单）。

**Language Gate**：岗位要求的语言对不上就直接 FAIL——**没列进语言表的语言是硬性否决，不是扣分项**；列了但对方要求的门槛看起来高于你的自评，就标记 FLAG，交给人判断，既不静默丢弃也不当成通过。

过了这两道，才进入五维打分：

| 维度 | 权重 |
| --- | --- |
| 技术技能匹配 | 30% |
| 经验匹配 | 25% |
| 行为/文化匹配 | 15% |
| 地点与通勤 | Pass/Fail（不参与加权） |
| 职业契合与动机 | 30% |

阈值分五档：75 以上"强匹配"，60–74"可以投，求职信里补差距"，45–59"慎重，和人商量"，30–44"除非有战略理由否则别投"，30 以下直接放弃。

[为什么重要]：技术 30% 与职业契合 30% 权重相等。求职信里那句"我为什么想来"通常是最口水的一段，这里把它提升到了和技术能力同等的一等公民——不是修辞问题，是评分问题。

## 双代理：交接的是结构化 JSON，不是"我觉得写得不错"

`/apply` 的核心是一个 drafter–reviewer 循环。drafter 写好 CV 和求职信之后，用 Agent 工具另起一个 reviewer 代理，给它**全新的上下文**，草稿以 inline 文本传入——刻意不让它去 Read 文件，也刻意告诉它别读两个模板结构文件，因为它要评的是内容，不是排版。

reviewer 的输出被强制分成两部分：

- **Part A 结构化修改**：一个 JSON 数组，每条是 `{file, old_string, new_string, reason}`，`reason` 只能取 `keyword match / company angle / reframing / style / grounding` 之一。这是可以直接机械应用的补丁。
- **Part B 叙述建议**：四类——漏掉的关键词、公司/部门角度的关联、被动句改写、语气与行为画像是否匹配。规则是每一类**即使"没问题"也要写一句**，防止沉默被下游误读成"这一项跳过了"。

还有一道独立的**事实核对**：把草稿里每个日期、雇主、头衔、量化指标，与三个来源交叉——候选人档案、主 CV 模板、`CLAUDE.md` 的档案段。重新措辞可以，改数字和抬事实不行；三个来源彼此矛盾时，报"档案不一致"，而不是当成草稿漂移。评审 prompt 里还写死了一句：不得建议编造技能、经历或成果，缺口就诚实说缺口。

[为什么重要]：评审能自动化的前提，是把"可机械应用的修改"和"需要人判断的建议"拆开。前者不需要人读就能落地，后者才值得占用注意力。

## 编译期与文本层：最像工程的一步

真正让这个项目区别于"AI 写简历"的，是它对交付物的验收标准。

CV 用 `lualatex` 编译，求职信必须用 `xelatex`（自定义的 `cover.cls` 依赖 `fontspec`）。README 直说 `pdflatex` 在现代 MiKTeX 上常因 `fontawesome5` 字体展开报错——这不是洁癖，是踩过的坑。然后是硬性条款：

- CV **恰好 2 页**，不是 1 页也不是 3 页；求职信**恰好 1 页**，签名块不能溢出。
- 不允许"职位的标题落在页底、条目翻到下一页"这种孤儿行，要求每段前加 `\needspace{5\baselineskip}`，必要时用 `\enlargethispage` 救回。
- `.tex` 里看着没问题**不算通过**，必须编译出 PDF 后逐页看。

接下来是最少被谈论的一层：ATS（简历筛选系统）可解析性。`tools/verify_pdf.py` 抽取 PDF 的文本层（优先 `pypdf`，退回 `pdftotext -layout`），检查四件事：有没有 `(cid:*)` 标记或乱码替换字符；邮箱和电话是否以**字面文本**出现（只存在于图标或超链接里的联系方式，ATS 读不到）；抽取顺序是否与视觉顺序一致；岗位关键词是否覆盖——以及明确要求**不许堆砌**，真实缺口要留着。

[为什么重要]：这等于给"简历能不能过机筛"配了一个 linter。玄学变成了可重复的检查。

也正因为它依赖平台细节，两个仍然开着的 issue 恰好暴露了这类自动化的脆性：[#669](https://github.com/MadsLorentzen/ai-job-search/issues/669) 说规程里硬编码了 `python3`，而 Windows 上通常只有 `python`；[#655](https://github.com/MadsLorentzen/ai-job-search/issues/655) 说 `verify_layout.py` 取 PATH 上第一个 `pdftotext`，在 Claude Code 的 Windows 环境里会跳过真正的那个。检查器自己也会被环境绊倒。

## 安全模型，以及作者自己承认的边界

`SECURITY.md` 的威胁模型写得少见地坦白：一个能读写文件、同时又读陌生人写的招聘启事的 LLM，本身就是主要风险面，无法消除，只能收窄。它的三层防线是：

1. **指令级**：招聘启事文本是数据、不是指令；不许跟随正文里的指示，也不许抓取正文里的链接（用户自己给的 URL 是唯一例外）。评审的公司调研必须从用户确认的公司名出发，不能从启事正文里的链接出发。
2. **权限白名单**：`.claude/settings.json` 只放行 6 个 portal CLI 的 `bun` 调用和几个具体工具（`verify_pdf.py`、`rank_state.py`、`salary_lookup.py` 等）；CI 里的 `security-guards` job 会拒绝任何放宽白名单、给 `package.json` 加安装期生命周期脚本、或削弱个人数据 gitignore 规则的 PR。
3. **数据边界**：档案、投递记录、薪资数据、申请归档都在 gitignore 里；`/notion-sync` 只同步文件名，文档内容不出本机。

但作者紧接着写了整份文档最重要的一句：**instruction-level defenses raise the bar; they are not a sandbox**——指令级防御只是抬高门槛，不是沙箱。

比提示注入更现实的风险是隐私。GitHub 不允许公开仓库有私有 fork，而 `/setup` 会把姓名、电话、邮箱、履历、薪资期望写进**受版本控制**的文件。README 用一块 IMPORTANT 警告写明了这点，并给了正解：只想自己用，就不要 fork，改成"私有仓库 + 把原仓库设为 upstream"，所有更新流程照常工作。看着 17,841 个 fork，值得想一下其中有多少是连手机号一起推上去的。

## 那串漏斗数字该怎么读

README 的自述是：69 份定制申请，20 次一面，1 份签约，2026 年 6 月入职做 AI 工程师。作者原本是地球物理学家，2025 年底岗位被裁，于是给自己写了这套框架，每周用一次，并在和每个雇主沟通时坦白了用法。

换算一下：一面率 29%，签约率 1.4%。这两个数字**不能**当作"AI 求职更有效"的证据——单样本、无对照组，作者本身是能写框架的工程师，且明确告知了对方。它证明的是"这套流程能跑通"，不是"这套流程更优"。

真正可迁移的不是转化率，而是那套检查：先设否决闸门，再评分，独立评审，编译验收，文本层校验。这些跟"AI 帮你写"其实无关——一个完全不用 LLM 的人照着做，简历质量一样会提升。

## 换到中文市场要改什么

6 个招聘网站 CLI 里，4 个是丹麦市场的（Jobindex、Jobnet、Akademikernes Jobbank、Jobdanmark），通用的只有 `linkedin-search` 和 `freehire-search`（多市场 REST 聚合）。README 说得很清楚：核心工作流与语言、国家无关，招聘网站的搜索 skill 是按丹麦市场做的，模式可以直接替换。

替换的入口是 `/add-portal`：它去调查目标站点（搜索 URL 模式、结果结构、访问规则），按同一结构脚手架出一个 CLI skill，跑一次真实查询验证后才注册。模板也能换——`/add-template` 支持 LaTeX、Typst 或其他工具链，注册时必须跑一次强制测试编译。也就是说，搬到 Boss 直聘、拉勾或猎聘，工作量集中在 portal skill，评分框架和双代理流程不用动。

代价也要说清楚：Claude Code 没有免费档，需要 Pro/Max 订阅或 API 额度；portal CLI 要 Bun；LaTeX 要同时有 `lualatex` 和 `xelatex`。想用 Codex、Gemini CLI 或其他 agent，`AGENTS.md` 是入口，portal skill 开箱可用，但完整流程得走社区 fork——而这些 fork 不在项目的安全策略覆盖范围内。

## 总结

一句话概括：它把求职从"写作"重新定义成"交付"。交付意味着有闸门（不合格就别投）、有评分（投哪个）、有独立评审（谁签字）、有编译产物（能打开、能解析）、有可验证的终态（PDF 文本层里读得到你的邮箱）。

但边界同样清楚：安全是指令级的，不是沙箱；转化率是自述的，不是实验；隐私要靠私有仓库而不是 fork；市场适配要自己写 portal skill。把它当模板改，会比把它当魔法用可靠得多。

## 参考

- 仓库与自述：[MadsLorentzen/ai-job-search](https://github.com/MadsLorentzen/ai-job-search)
- 应用流程规格：[`.claude/commands/apply.md`](https://github.com/MadsLorentzen/ai-job-search/blob/master/.claude/commands/apply.md)
- 评分框架：[`04-job-evaluation.md`](https://github.com/MadsLorentzen/ai-job-search/blob/master/.claude/skills/job-application-assistant/04-job-evaluation.md)
- 安全策略：[`SECURITY.md`](https://github.com/MadsLorentzen/ai-job-search/blob/master/SECURITY.md)
- 版本记录：[Releases](https://github.com/MadsLorentzen/ai-job-search/releases)（最新 v1.7.2，2026-09-29）
- Windows 相关未决 issue：[#669](https://github.com/MadsLorentzen/ai-job-search/issues/669)、[#655](https://github.com/MadsLorentzen/ai-job-search/issues/655)
