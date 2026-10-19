---
title: "handwired：把判断写成减法，让 JavaScript 跑成神经网络"
date: 2026-10-19T22:20:00+08:00
tags: ["JavaScript", "神经网络", "WebGPU", "开源项目"]
author: "mywebtestmail"
---

[chenglou/handwired](https://github.com/chenglou/handwired) 的 [README](https://github.com/chenglou/handwired/blob/main/README.md) 第一句很像标题党：Turn your regular JavaScript code into a neural net; no training, exact, yet also fully differentiable。我第一遍读完，以为又是那种「丢进去一个 React 组件、吐出一份 ONNX」的自动转换器。

翻到 [docs/README.md](https://github.com/chenglou/handwired/blob/main/docs/README.md) 第三行，作者自己把这个误会堵死了：No tool converts existing JavaScript source into a program。你写的是 builder code，普通 JavaScript，跑一次，把操作 append 进一个 `Circuit`；运行时在 CPU 上用 JS 求值，只有 json-search 那个 demo 走 WebGPU。

所以「把现有 JS 变成神经网络」是目标，不是它此刻做的转换。这个落差得先摆出来，不然后面全是误会。

## 一个 if，两个 ReLU

README 给的最小例子是可以拿纸笔算的。条件判断 `(a > 4) ? 10 : 12` 被写成两行：

```js
const above4 = Math.max(0, a - 4) - Math.max(0, a - 5);
const out = above4 * 10 + (1 - above4) * 12;
```

a 小于等于 4 时两个 ReLU 都是 0，相减得 0；a 在 4 和 5 之间，第一项爬起来、第二项还压着，得小数；a 大于 5，两项一起平移，差回到 1。于是 `above4` 就是一个 0/1 的判定位。两条分支都算出来，再用这个位去选。

这就是全部戏法。if 没有了，比较没有了，剩下的全是加法、乘法、`max(0, x)`。

这种写法不是手写的特例，runtime 认的就是这个形状。`Circuit` 按顺序存下每次 append 的操作，求值时沿拓扑序重放一遍，源码里 `evaluateReference(source: Float32Array)` 走的就是那条最朴素的参考路径：一张操作表、一次遍历、纯 JS。

## 四条指令撑起整个 runtime

[docs/execution-model.md](https://github.com/chenglou/handwired/blob/main/docs/execution-model.md) 把 runtime 的指令集定死成四种：

affine 是加权和；reglu 写成 `f(left * max(0, gate))`，可以理解成带门控的线性层；boolean 是那个更邪门的 `f(f(f(score) + K) - K)`；index 是查表。

boolean 那条值得多说一句。`K` 取 8,388,609，源码在 [src/runtime/numeric-ui-core.ts](https://github.com/chenglou/handwired/blob/main/src/runtime/numeric-ui-core.ts#L1915-L1922)，注释原话是「FP32 rounds to whole numbers at this size; the odd base makes 0.5 round up」。FP32 在这个量级上已经没有小数位了，偏移再减回来等于把浮点值吸附到整数上；基数取奇数，正好让 0.5 往上进位。用一加一减做出阶跃函数，再拿它当布尔值用。

四种指令，全部 FP32 逐步舍入。所谓「可微」，不是额外挂了什么特殊算子，而是整条链路本来就只剩可微的四则运算加 clamp。反传梯度因此是现成的。不过这句话只对四则运算和 clamp 成立：[docs/execution-model.md](https://github.com/chenglou/handwired/blob/main/docs/execution-model.md) 里写着布尔判定和 index 读取默认拒绝梯度，`backpropagateReference` 碰到这两种操作会直接抛错，得显式传 `{booleanGradient: 'straight-through'}` 或 `{indexGradient: 'zero'}` 才训得下去。也正因为每一步都是固定形状的矩阵运算，它才可能被融成一个巨大的 matmul、被稀疏化、被重排到 GPU 上。

指令集小到这个程度，好处是解释器几十行能写完，坏处是所有表达力都得从这四块里拼出来。README 里那句 collect the + and * and relu into a giant matmul 说的也是这件事：基座越窄，整段程序才越有机会被合成一个矩阵。

## 代价：没有提前退出这回事

真正上手才知道疼的地方在这。[docs/patterns.md](https://github.com/chenglou/handwired/blob/main/docs/patterns.md) 第三行写得很直白：The program runs every operation on every frame and can't stop early, so a loop becomes a count, a tree, a table, or the same work on every item.

每帧每条操作都要跑，没有分支、没有 early exit、没有 runtime 循环。`findIndex` 找到就跑这种写法直接作废，`items.sort(compare)` 也是。得改成计数、平衡树、查表，或者「对每个元素做同样的事」。

同一份文档里有可以核对的账。一个只翻转一次的 `findIndex`（64 行）写成 195 条操作、3 层；换成树是 447 条、34 层。同一个 `findIndex`，命中点任意时，316 条、31 层，普通循环是 127 条、127 层。`items.sort(compare)` 排 64 项，4,096 条操作、3 层。

这些数字挺说明问题：树版本的操作数比原来的扫描版本还多一倍，可它换来了固定的形状。排序之所以飙到四千条，是因为两两比较本来就被摊成了全连接。省下来的不是算力，是「数据结构要不要在运行时变形」这件事本身。

## 三个 demo，各证一件事

[HandwiredOS](https://chenglou.me/handwired/os/) 是拿这个 runtime 驱动一整个桌面：焦点、层叠、拖拽、缩放、窗口控制、弹簧、相册布局，还有 Monitor 的自省面板。它的 [README](https://github.com/chenglou/handwired/blob/main/src/handwiredos/README.md) 写的是 234 个 source cell、4,646 条 live operations、130 层、9,909 个存储权重。官网首页那几个大数字也是 weights 9,909、layers 130、training 0，与 [项目主页](https://chenglou.me/handwired/) 上挂的一致。我在本机跑 `bun lab/runtime-cost/report.ts`，输出是 sources 234、operations 5,028、liveOperations 4,646、layers 130、storedWeights 9,909、roots 727，与文档逐项对得上。规模这个卖点是真的。

[learned-gallery](https://chenglou.me/handwired/learned-gallery/) 走的是另一条路：[README](https://github.com/chenglou/handwired/blob/main/src/learned-gallery/README.md) 说 244 个 source cell 就是被训练出来的系数，Adam 更新，梯度来自 `circuit.backpropagateReference`，整个程序 6,320 条操作、8 层。这条演示的是「权重可以解冻」，也就是它为什么能叫神经网络。

[json-search](https://chenglou.me/handwired/json-search/) 每次按键在 10 万份生成的 JSON 上重搜一遍，两套 GPU 代码生成器：[loop-emit.ts](https://github.com/chenglou/handwired/blob/main/src/json-search/loop-emit.ts) 给每条操作发一句 f32，[bit-emit.ts](https://github.com/chenglou/handwired/blob/main/src/json-search/bit-emit.ts) 把值压成整数或者 1 bit。只有这个 demo 有 WebGPU 路径，[src/json-search/README.md](https://github.com/chenglou/handwired/blob/main/src/json-search/README.md) 里的 11–15 ms 也是作者自测，同机 RegExp 8–28 ms，CPU 路径大约 13 秒。我没有独立复测，这组数字只能当作者自述看。

三件事摆在一起，handwired 真正在卖的不是某个 demo，而是一种写法：状态永远是张量里的一格，界面逻辑永远是固定层数上的线性运算。收益是这套逻辑能被 GPU 整段吞下去，也能整段求导；代价前面说过，没有提前退出。

## 自己跑一遍

我在 Windows + bun 1.4.3 上从零 clone，装依赖 34 个包、41 秒。四组检查都过了：

- `bun test tests/runtime`：174 pass / 0 fail / 155,853 个 expect，4.92 秒跑完（文档说大约 2 秒，我这台慢，算环境差异）。
- `bun test template`：三条全过，其中一条是「the program and plain JavaScript agree on every frame」，拿 [template/toggle.ts](https://github.com/chenglou/handwired/blob/main/template/toggle.ts) 里的模板开关和等价 JS 逐帧比对。
- `bun run check`（tsc 加 oxlint）：退出码 0。
- `bun scripts/serve-demo-site.ts`：起来以后 `/`、`/os/`、`/json-search/`、`/learned-gallery/` 四个路由都是 200。

装完就能跑，没有配置文件，没有环境变量，这一点比多数 ML 仓库友好。

整个仓库的体量也不像周末玩具：src 下 18,011 行 TypeScript（runtime 4,266、handwiredos 5,473、json-search 6,184、learned-gallery 2,088），tests 21,748 行、30 个测试文件，全仓 141 份 Markdown 文档、1,101 个 .ts 文件。GitHub 上的数字是 825 次提交、1 位贡献者（contributors API 只返回 chenglou 一个人），9 个 open issue 里 5 个是文档语法修正，其余是功能请求（通用 WebGPU 后端、Doom demo、Neural Lander），没有崩溃类报告。质量靠仓库自带的测试，不靠社区当回归测试机。

顺带提一句建仓时间。GitHub API 显示仓库创建于 2026-10-15，看着像刚生出来四天；但 [tags 页](https://github.com/chenglou/handwired/tags)里有个 `exploration-2026-09`，指向 9 月 16 日的提交，tag message 写的是「Repository state on September 16, 2026」。代码历史至少能追到那会儿，别被创建日期骗了。

## 它不是 npm 包，精度也有边界

`package.json` 里写着 `"private": true`，没有 main、module、exports、bin 任何一个字段。`npm install` 装不到，除非你把仓库 clone 下来当参考实现读。这和它的定位是一致的。

更该看的是 [docs/execution-model.md](https://github.com/chenglou/handwired/blob/main/docs/execution-model.md) 里那几段「看起来能算、实际会翻车」的边界：`emitMaximum(2^24, -1)` 得 16,777,215，对应的 minimum 得 0 而不是 -1；钳到 0..1 在 `2^24 + 2` 处可能得到 2，在 `2^25` 处得 0；index 读要求索引严格落在 ±2^24 以内且是整数；boolean atom 的 `(x + K) - K` 依赖后端不把它化简掉，换个编译器把这两步消掉，整个判定就没了。

这些都是 FP32 的硬约束，不是 bug，但它决定了你没法把这套东西当成「随便什么 JS 都行」的通用加速器。围观的时候可以跳过，真要往上写东西，这几段得先读。

规模上的成本作者写在 [lab/status.md](https://github.com/chenglou/handwired/blob/main/lab/status.md)：236 万条 built operations，构建 3.1 秒、编译 1.8 秒、峰值内存 4.2 到 4.5 GB；到 690 万级，构建 9.7 秒、编译 5.5 秒、峰值 9 到 11 GB。都是 M5 Max 上的数。

## 我会把它推荐给谁

先说结论：它不适合当框架用。想拿它给现有前端提速，基本走不通，光是「不能提前退出」这一条就能把大半业务逻辑挡在门外。只是给一个表格加筛选的话，写普通 JS 更快，也更好维护。

它适合两类场景。一类是某种固定、高吞吐的判定管线，比如筛选、打分、排序，结构能在构建期摊平，并且你愿意为一次求值付出几千条操作。另一类是把它当教材读：if 能被写成两个 ReLU 相减，比较能被写成一次偏移取整，条件选择能被写成一次乘加。这套东西平时藏在框架里，这里全摊在桌面上，141 份文档和 2 万多行测试都在解释同一件事。

作者 825 次提交一个人写完，MIT 授权，截至 2026-10-19 是 1,783 stars、147 forks。长期维护没法保证，在线 demo 我也只验证了 HTTP 200 和静态资源可达，真实帧率没测。把它当参考实现围观是安全的，当生产依赖还早。

## 参考

- [chenglou/handwired 仓库](https://github.com/chenglou/handwired)
- [README：口号与可手算的 if 例子](https://github.com/chenglou/handwired/blob/main/README.md)
- [docs/README.md：builder code 与运行时的实际边界](https://github.com/chenglou/handwired/blob/main/docs/README.md)
- [docs/execution-model.md：四条指令、FP32 舍入与精度边界](https://github.com/chenglou/handwired/blob/main/docs/execution-model.md)
- [docs/patterns.md：不能提前退出的代价与操作数对照表](https://github.com/chenglou/handwired/blob/main/docs/patterns.md)
- [docs/tutorial.md：上手教程](https://github.com/chenglou/handwired/blob/main/docs/tutorial.md)
- [template/toggle.ts：模板开关的 builder 写法](https://github.com/chenglou/handwired/blob/main/template/toggle.ts)
- [src/handwiredos/README.md：234 cells / 4,646 live ops / 130 layers](https://github.com/chenglou/handwired/blob/main/src/handwiredos/README.md)
- [src/learned-gallery/README.md：Adam 训练出来的系数](https://github.com/chenglou/handwired/blob/main/src/learned-gallery/README.md)
- [src/json-search/README.md：10 万文档与两套 GPU 代码生成器](https://github.com/chenglou/handwired/blob/main/src/json-search/README.md)
- [lab/status.md：规模与构建成本实测](https://github.com/chenglou/handwired/blob/main/lab/status.md)
- [tags：exploration-2026-09 与 9 月的代码历史](https://github.com/chenglou/handwired/tags)
- [项目主页](https://chenglou.me/handwired/)
- [HandwiredOS 在线演示](https://chenglou.me/handwired/os/)
- [learned-gallery 在线演示](https://chenglou.me/handwired/learned-gallery/)
- [json-search 在线演示](https://chenglou.me/handwired/json-search/)
