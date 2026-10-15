---
title: "当 AI 开始撤回论文：Lean 证明成为验证层"
date: 2026-10-15T21:25:00+08:00
tags: ["AI", "Lean"]
author: "mywebtestmail"
---

2026 年 10 月 7 日，OpenAI 的数学仓库更新了它的[变更日志](https://raw.githubusercontent.com/openai/math/main/history.md)。同一条记录里，三篇手稿被撤回，十四篇被修订，另有十三篇因为要引用修订后的伙伴论文而跟着改了版本和引文。一次更新，三十篇稿子被触动。

先把这个仓库的体量摆出来。它的 [README](https://raw.githubusercontent.com/openai/math/main/README.md) 自述：绝大多数结果由同一个未发布的内部模型产出，平均每个结论消耗约 3 小时 ChatGPT Pro thinking 的算力，整套评测约 4000 道题。[目录文件](https://raw.githubusercontent.com/openai/math/main/CONTENTS.md)收录了 719 篇手稿、372 个家族。这些数字全部来自仓库自述，没有第三方复核过，后文我会再标一次。

## 井喷：产出速度与随之而来的信任赤字

三篇撤回放在 719 篇里，比例上只是几个千分点，看着不大。但它暴露的东西跟比例无关。

能把这个尺度说清楚的材料，是 Thomas Hales 发在 Terence Tao 博客上的一篇[客座文章](https://terrytao.wordpress.com/2026/10/09/what-mathematicians-should-know-about-the-lean-theorem-proverquestions-of-reliability-and-ai/)（2026 年 10 月 9 日，挂的是 Tao 的博客与作者标签，正文作者是 Hales，引用的时候别搞混）。他给了两组数字：Kepler 猜想的人工形式化大约花掉 20 人年、50 万行；而 mathlib 现在有近 30 万条定理、10 万多个定义、250 万行、700 多名贡献者。同一篇里还列了自动化那一侧的时间线：2025 年 9 月 PNT 的准自动形式化，2026 年 1 月一份两周、13 万行的形式拓扑，2026 年 3 月 24 维球填充的形式化（初版约 50 万行，golf 之后压到约 20 万行），2026 年 9 月 4 日 Anthropic 宣布 FLT 的自动形式化用 11 天产出 1300 万行 Lean。

一边是 3 小时一个结论、一批 700 篇；另一边是 20 人年一个定理。写证明早就不是瓶颈了，这一点不需要预测，它已经发生了。

## 撤回复盘：一个符号错误如何带走三篇论文

三篇里最关键的是《Algebraicity of Weil classes on split abelian eightfolds》，它自己的[撤回通知](https://raw.githubusercontent.com/openai/math/main/preprints/Algebraicity-of-Weil-classes-on-split-abelian-eightfolds-September-18-2026/README.md)把错误写得很具体：stabilization-trace 论证里有个符号错误。按手稿的有符号二重点约定记 I(f1) = −m（m 为正），原文给每条逆向稳定化迹记 +1，于是声称插入 m 条就能把有符号计数做成零。可标准尖点的两个分支源定向相反，在这个约定下每条逆向迹的符号应该是 −1，算下来 I_new = I(f1) − m = −2m ≠ 0。它这一步调用的 Eliashberg–Murphy 消灭定理要求有符号二重点计数为零，前提不成立，后面那串定向手术与嵌入 brane 的构造就失去了支撑。

另外两篇是《Algebraicity of Kuga–Satake Correspondences for K3 Surfaces》和《The rational Hodge conjecture for products of K3 surfaces》。[后一篇的通知](https://raw.githubusercontent.com/openai/math/main/preprints/The-rational-Hodge-conjecture-for-products-of-K3-surfaces-October-4-2026/README.md)直说它依赖前一篇那套有缺陷构造的改编版，[Kuga 那篇](https://raw.githubusercontent.com/openai/math/main/preprints/Algebraicity-of-Kuga-Satake-Correspondences-for-K3-Surfaces-October-3-2026/README.md)的说法一样：那个符号错误挡掉了本应成立的有符号二重点消灭，缺口让这篇声称的代数性定理处于未证明状态。

三份通知都带一句容易读漏的声明：撤回针对的是证明，不代表相应的数学命题为假。这句话是实话，也是这类事件的常见落点，但读的人更容易记住"撤回"两个字。

目录变化能对上号。三篇的标题在撤回前的 [CONTENTS.md](https://raw.githubusercontent.com/openai/math/adc7f1241b42e322a6451854ab7e4b4c146bf78a/CONTENTS.md) 里各出现一次，在当前版本里是 0 次。preprints/ 下一共 749 个目录，其中 719 个在目录文件里有条目，未被收录的 30 个里 27 个是旧日期版本的目录（新版本仍在目录内），真正一条目录都不占的只有 3 个，就是这三篇。

时间口径有点乱：三份通知的落款都是"Withdrawn on October 6, 2026"，变更日志却把它们归在 October 7 那一条下面。我不打算断言哪个更准，能确定的是这套对外时间线本身不太整齐。

同一批修订也值得看。十四篇分成几组：Lipschitz heights 与 Ashkin–Teller 四篇，修了交叉、边界贴合、条件性与收敛论证，还补了实 Lipschitz 接口的证明；Kähler minimal model programs 六篇，扩写了正性与收缩论证，并写明哪些结论是外部输入、需要什么假设；Taming 与 hypersymplectic 两篇，改正了锥相等的断言、加了严格包含的例子、去掉一处不必要的锥比较依赖；Incompressible Box Transport 与 Finite Computation 一篇重做了 torus 投影与公共时钟估计；Exact BSD 一篇删掉一条指向已移除支撑稿的过时引文。紧跟着，十三篇因为引文指向修订版而改版。一个符号错误，成本沿着依赖链外溢到了三十篇稿子上。这才是它真正贵的地方。

## 同行评议兜不住的地方

把这件事读成"审稿人不负责"是错的，问题在带宽差了几个数量级。

一个人认真读一篇 60 页的代数几何论文，看穿一个符号错误的前提是他先手推那条稳定化迹的符号约定，再对照编译式的尖点分支定向。审稿人靠启发式判断推导"看起来对不对"，而这类错误长得很普通：定义一致、形式规范、每一步单看都站得住，错在约定与符号的错位上。它恰好在人类最不擅长、机器最擅长的那一层。

反过来，AI 产出的规模是 3 小时一个结论、一批 700 篇，还能一次修订 30 篇。要求人工评审按这个速率给出同等强度的核查，是不现实的。这不是态度问题，是数量级问题。

仓库自己也没装作没事。[README](https://raw.githubusercontent.com/openai/math/main/README.md) 里写着不是所有结果都有对应的 Lean 形式化，并承认部分未形式化的结果可能有问题。形式化清单的元数据更直接：[formalization.yaml](https://raw.githubusercontent.com/openai/math/main/lean/formalization.yaml) 尾部写着 `automation.methods: [agent]`、`review.status: unchecked`，范围一栏也只写 partial progress。变更日志那边自报的覆盖率是 top-line 结果的形式化比例 300/719，约 42%。

Hales 那篇客座文里把这个缺口说得很干脆：Lean 证明在通过内核检查之前不应该被相信，而在内核之外，还需要人工审计来保证 statement fidelity，也就是"被证明的命题确实是你想说的那个"。他把 comparator 点名为做成这件事的工具。

## 验证层：把信任换成检查

Lean 的检查链条是分级的，这一点[官方手册里写得很清楚](https://lean-lang.org/doc/reference/latest/ValidatingProofs/)。最省事的一档是编辑器里的蓝勾，它只说明文件按当时的依赖编译过；往上一档是 `#print axioms`，看结论依赖了哪些公理；再往上是用 `lean4checker --fresh` 从零重导并重检；最高一档是 `lake comparator`：在沙箱里构建、导出证明项，拿到沙箱外重放，再跟一份可信的 challenge 文件比对陈述是否一致。手册里还老实列出，即便做到这一步，仍然剩下几项假设，比如没有同时影响所有被用检查器的实现 bug，以及 challenge 文件里没有人有意无意地把命题写错。

这里有个我原本低估的点：形式化锁住的是"命题被证明了"，它不负责"命题是你想说的那个"。Lean 官方文档甚至把关照不足的 AI 生成证明与程序直接归到恶意代码那一类去讨论，理由是未经审查的代码可以无害地编译通过却做了别的事。

这个问题在 2026 年 10 月 6 日挂出来的[一篇题为 Navier-Stokes lost in translation 的预印本](https://arxiv.org/abs/2610.08144)里被推到更硬的层面。作者 Bastounis、Circelli 与 Hansen 讨论的是 autoformalization：把自然语言数学翻译成 Lean 时，消解歧义所需的保真条件在算术层级上没有可计算的界，比停机问题更难。他们直接指出，某次自动形式化的 Lean 产物并不对应其自然语言证明所宣称的内容。这篇是预印本，未经同行评审，结论的强度我不替它背书，但它指出的方向跟上面那个"命题是不是你要的那个"是同一件事。

还有一层更少被提到：验证器自己也会翻车。Lean 的负责人 Leonardo de Moura 在 2026 年 8 月 24 日写了一篇[内核可靠性 bug 追猎的复盘](https://leodemoura.github.io/blog/2026-8-24-postmortem-for-the-kernel-soundness-bug-hunt/)。与 OpenAI 的 Daniel Selsam 合作的那轮排查从 7 月 30 日跑到 8 月 20 日，结束条件很朴素：内部 AI 报不出新问题了。找出来的问题包括引用计数溢出后被改写来构造出 False 的证明，以及 Linux 版 Lean 用上有已知 bug 的 GMP 6.1.2 构造 False。8 月 21 日发布 v4.33.1 修掉，仓库新增 `soundness` 与 `runtime-soundness` 标签。后续动作是把 `lake check`、`--paranoid` 与 comparator 逐步收进官方发行版。这篇文章的立场值得记住：它把 comparator 加外部检查器称为金标准，同时把 AI 生成的证明视为潜在的恶意证明来源。

顺带说一句，验证工具本身的作者结构也在变。同一批材料里提到的 [con-leche](https://raw.githubusercontent.com/leanprover/con-leche/master/README.md) 是一个自带一致性证明的外部检查器，能处理 mathlib；它的 README 自述实现与证明是在 Lean 团队 Breitner 的监督下由 Claude 完成的，而 README 本身是仓库里唯一由人写的东西。检查器跑分的公开榜在 [Lean Kernel Arena](https://arena.lean-lang.org/)（清单未逐条核实）。

## 生成代码 vs 验证代码：预算该往哪投

我的判断是：验证代码比生成代码更值得投入；没有形式化证明的结果不该进正式库。这不是说生成不值钱，是说两边的边际收益已经换位了。

支持这个判断的是四组对照。

第一组是成本对照。生成侧已经被压到 3 小时一个结论、一批 700 篇；验证侧还是人工陈述保真、外部检查器、42% 的自报覆盖率。缺口明摆着在验证侧。这组数字出自仓库自述，没有第三方核实，强度到此为止。

第二组是错误成本的外溢方式。一个符号错误带走三篇、连带修订十四篇、再改十三篇引文。生成侧多跑一次很便宜，验证侧漏一次很贵，而且贵在下游。

第三组来自验证器的迭代节奏。2026 年 7 月底到 8 月下旬这段时间里，发生了内核可靠性 bug 的排查与修复（8 月 21 日发 v4.33.1），随后是把 `lake check` 与 comparator 收进官方发行版的收尾动作。验证层的每一次加固都会复用到所有下游结论上，生成侧的一次提升只惠及那一次产出。

第四组是覆盖率这个数字本身。42% 意味着超过一半的主结果是靠人读的。这个缺口不投钱只会继续留着。

落到具体动作上，我会做三件事：把 comparator 与 `lake check` 挂进 CI，让"命题被证明"这件事每次提交都跑一遍；把 challenge 文件冻结成可信基线，改它就等于改命题，必须有第二个人签字；把每份形式化附带的假设清单留在仓库里，而不是藏在验证者的脑子里——仓库在这个方向上已经有起步，[lean/docs](https://raw.githubusercontent.com/openai/math/main/lean/docs/003.md) 下按家族编号写的 scope note 就在干这件事，它连"论文后续的应用没有包含在形式化里"这种边界都会写明。

## 落地边界：团队的门槛与红线

给要做类似事情的团队，我会把门槛写成可执行的四条：

1. 没通过 `lake comparator` 的结果不进正式知识库，也不做论文主结论，最多进草稿区并显式标"待验证"。
2. 形式化通过不等于可以引用，statement fidelity 要有人审，审的是 challenge 文件里那句话和你想说的话是不是同一句。
3. 覆盖不到的部分要写清覆盖到哪一步，"有形式化"和"主定理被形式化"是两件事，仓库的 [ComparatorChallenges](https://raw.githubusercontent.com/openai/math/main/lean/ComparatorChallenges/README.md) 里就有只验证支撑引理、不验证主定理的配置。
4. 未形式化的结论允许存在，但必须带标记，且禁止被下游结论引用——撤回那三篇的问题不在于错，而在于它们曾经被当成支撑用了出去。

### 查证边界

这篇里有些东西我没能核实，列在这里，读的时候请当它们是自述或未定：

- 官方公告页不存在，能查到的只有仓库的变更日志和三份撤回通知。
- 内部模型的名称与版本、3 小时算力这个口径，都只有仓库自述。
- 42%（300/719）与 formalization.yaml 里 200 条主结果声明、173 条论文条目之间的口径差异，没有官方解释——两者统计基数不同，我没找到把它们对齐的说明。
- 撤回的三篇历史上是否曾经有过 Lean 形式化，无法反推，仓库只公布当前版本。
- lean/docs 下按家族编号的 scope note 一共两百多份，撤回稿对应家族的编号与是否留有说明，我没确定。
- con-leche 的检查结果被十几个检查器复核过这个说法，只见于 Hales 那篇文章，是一面之词。
- Kernel Arena 上的检查器清单我没有逐条核实。

## 参考

- [OpenAI math：变更日志（2026-10-07 的撤回与修订）](https://github.com/openai/math/blob/main/history.md)
- [OpenAI math：README（体量、算力与形式化覆盖率自述）](https://github.com/openai/math/blob/main/README.md)
- [OpenAI math：CONTENTS.md（当前目录，719 篇手稿）](https://github.com/openai/math/blob/main/CONTENTS.md)
- [撤回前 revision 的 CONTENTS.md（对照用）](https://github.com/openai/math/blob/adc7f1241b42e322a6451854ab7e4b4c146bf78a/CONTENTS.md)
- [撤回通知：Algebraicity of Weil classes on split abelian eightfolds](https://github.com/openai/math/blob/main/preprints/Algebraicity-of-Weil-classes-on-split-abelian-eightfolds-September-18-2026/README.md)
- [撤回通知：Algebraicity of Kuga–Satake Correspondences for K3 Surfaces](https://github.com/openai/math/blob/main/preprints/Algebraicity-of-Kuga-Satake-Correspondences-for-K3-Surfaces-October-3-2026/README.md)
- [撤回通知：The rational Hodge conjecture for products of K3 surfaces](https://github.com/openai/math/blob/main/preprints/The-rational-Hodge-conjecture-for-products-of-K3-surfaces-October-4-2026/README.md)
- [OpenAI math：lean/formalization.yaml（形式化声明与元数据）](https://github.com/openai/math/blob/main/lean/formalization.yaml)
- [OpenAI math：lean/ComparatorChallenges/README.md（只验证支撑引理的配置）](https://github.com/openai/math/blob/main/lean/ComparatorChallenges/README.md)
- [OpenAI math：lean/docs/003.md（按家族编号的范围说明示例）](https://github.com/openai/math/blob/main/lean/docs/003.md)
- [Hales 客座文：What mathematicians should know about the Lean Theorem Prover](https://terrytao.wordpress.com/2026/10/09/what-mathematicians-should-know-about-the-lean-theorem-proverquestions-of-reliability-and-ai/)
- [Lean 官方手册：Validating a Lean Proof](https://lean-lang.org/doc/reference/latest/ValidatingProofs/)
- [de Moura：内核可靠性 bug 追猎复盘（2026-08-24）](https://leodemoura.github.io/blog/2026-8-24-postmortem-for-the-kernel-soundness-bug-hunt/)
- [arXiv:2610.08144：Navier-Stokes lost in translation（预印本）](https://arxiv.org/abs/2610.08144)
- [con-leche：带一致性证明的外部检查器](https://github.com/leanprover/con-leche/blob/master/README.md)
- [Lean Kernel Arena](https://arena.lean-lang.org/)
