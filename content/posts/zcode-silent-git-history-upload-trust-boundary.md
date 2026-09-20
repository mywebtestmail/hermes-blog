---
title: "ZCode 静默上传 Git 历史：私钥在云端的信任边界"
date: 2026-09-20T18:00:00+08:00
tags: ["安全", "Agent"]
author: "mywebtestmail"
---

先把时间拨回你删掉 `.env` 的那一刻。你以为 `rm .env` 之后，那行 `API_KEY=sk-...` 就从硬盘上消失了。它没有。只要这个仓库归 git 管，那行密钥还躺在 `.git/objects` 里，`git cat-file -p` 一读就回来。这是 git 的老规矩：历史默认不可改，除非你主动重写。

把这个规矩叠到另一件事上。智谱的桌面端 AI 工具 ZCode，在 3.12.3 及更早版本里内置了一条链路：登录后，它在每次 prompt 之前把整个工作区连同完整的 `.git` 目录打包、加密、传上云端。包里的 `.git` 占了 86.6%。也就是说，你早就删掉的密钥，可能跟着你删掉它的那个仓库，一起被压进了发往云端的加密包。

2026 年 9 月 18 日，一位开发者把这套链路扒出来写了取证帖，[原文在这里](https://blog.ferstar.org/en/posts/zcode-silent-workspace-snapshot-upload/)，Hacker News 上[冲到 300 多分](https://news.ycombinator.com/item?id=49750694)。智谱当天就回应了，归因"代码库索引"，说上传数据"生成后立即销毁，不会保存"，功能已修复，还承诺开源客户端代码、邀请第三方审查，[媒体转述见 IT之家](https://www.ithome.com/1/004/310.htm)。

事情到这里没完。删掉的代码可以被核实，"立即销毁"四个字却只有厂商自己说得清。这篇文章想做的更具体：把这件事拆成几个能自己动手验证的问题，然后给你一张清单，用来判断任何一个桌面端 Agent 往云上传东西时，它的信任边界画在哪。

## 打包了什么：不是"上下文"，是整段历史

原作者的实测数字摆在这里：一个 345MB 的工作区，打包出的加密包是 313MB，manifest 里有 42,411 个文件。其中 `.git` 目录占了整个载荷的 86.6%：`lfs` 196.1MB（56.8%）、`objects` 102.2MB（29.6%）、`logs` 0.6MB（0.2%），真正的源码和文档只有 46.2MB（13.4%）。这些百分比都以 345MB 的原始工作区为分母，加密包本身是 313MB。

这玩意的重心不在你的代码，在 git 的记账本。提交历史对象、LFS 缓存、reflog 全在内。厂商把它叫"代码库索引"，但它打包的是仓库自诞生以来的完整谱系，不是"这次任务需要的上下文"。

删掉的密钥为什么还在里面，一行命令就能复现：

```bash
git init -q . && printf 'API_KEY=sk-xxx\n' > .env
git add .env && git commit -qm "add env"
rm .env && git commit -qam "remove secret"
blob=$(git rev-list --all --objects | grep ' \.env$' | awk '{print $1}')
git cat-file -p "$blob"   # 工作区已没有 .env，内容仍可从对象库读出
```

我在本机跑了一遍，`git rev-list --objects --all | wc -l` 数出 5 个对象（2 个 commit、2 个 tree、1 个 blob），其中就有那个已经 `rm` 掉的 `.env` 的 blob，`git cat-file -p` 把内容原样读了回来。这是 git 的默认行为，不是 ZCode 的 bug。

[为什么重要]：判断一个"上传仓库"的行为有多严重，不看你传了多少代码，看你传没传 `.git`。传代码，别人拿到的是你的现状；传 `.git`，别人拿到的是你的过去，包括你以为已经抹掉的那些提交。ZCode 这件事里 `.git` 占比 86.6%，严重性就是从这里定的。

## 什么时候触发：跟开关无关，跟 token 有关

3.12.3 的宿主代码里，捕获快照的入口有两处，都由 `captureStage` 标记：一处是 `captureBeforePrompt`，`captureStage:"prompt"`，也就是每次对话前；另一处是 `captureTaskCompleteUpdate`，`captureStage:"terminal"`，内容标记 `repo-wiki-update`，在生成 Repo Wiki 时触发。唯一的前置门闩是 token：`let r = await this.tokenProvider(); if(!r) return;`。拿不到登录态就返回，拿得到就跑，跟你在用哪个模型、聊什么内容无关。说"及更早"不是推测：[#707](https://github.com/zai-org/feedback/issues/707) 里一个 3.10.0 的 Linux 实例，本地同样留下了 checkpoints 目录和 `lastAcceptedManifestHash`，说明这条链路至少在 3.10.0 就存在，不是 3.12.3 才引入的。

这里的关节点是"登录即默认"，不是"用户点了某功能"。一个功能如果默认开启、静默运行、跟任务内容无关，它就是数据收集行为，而不是功能行为。这句话是判断整件事性质的锚点。

真正让我皱眉的是两个开关。ZCode 的设置里有 `optimizeAgentExperienceEnabled`（优化体验）和 `repoSnapshotIndexingEnabled`（仓库快照索引）。直觉上关掉后者就该停。但把 3.12.3 的 `app.asar` 解包扫一遍，`repoSnapshotIndexingEnabled` 在整个 host 包只出现 2 次，两次都在设置归一化逻辑里（`normalizeSettingsPatch` 和设置键数组），捕获链路上没有任何一处拿它做判定。两个开关都不在捕获链上。

要把这条的边界说清楚：这是字符串加上下文的核查，不是完整调用图或反编译。所以我把结论限定成"经客户端包比对，这两个开关不阻断上传"，而不是"开关是假的"。前者我能证明，后者需要更重的证据。

用户面对隐私问题时的默认假设是"设置里有开关 = 我能关掉"。当开关不在数据链路上时，这个假设就塌了。可关闭性不是一个 UI 上的布尔值，是"关掉之后代码路径确实不执行"。

## 自己动手：三类本机取证

这件事最值得带走的不是结论，是方法。不装抓包工具，用系统自带命令就能对任意一个桌面端 Agent 自查：它有没有在偷偷上传。

第一类，看进程和连接。谁在连，连到哪：

```bash
netstat -ano | grep -i ESTABLISHED | head   # 拿 PID 去任务管理器对号
nslookup zcode.z.ai                         # 实测 CNAME 到 zcode.z.ai.a1.initaa.com
curl -sSI -A "Mozilla/5.0" https://zcode.z.ai/  # 从响应头核对版本与时间戳
```

第二类，看仓库和快照产物有没有残留：

```bash
ls -la "$USERPROFILE/.zcode/v2/checkpoints"   # 快照本地 checkpoint 目录
du -sk .git; du -sk --exclude=.git .          # 对比 .git 与工作区体积
git rev-list --objects --all | wc -l          # 多出来的就是历史对象
```

第三类，最有复用价值的一条：不看 UI，看包。公开 CDN 下载安装包，7-Zip 解包，解析 asar 头，在包内搜标记：

```bash
curl -O https://cdn-zcode.z.ai/zcode/electron/releases/3.14.0/windows-x64/ZCode-3.14.0-win-x64.exe
7z x -y -o./x ZCode-3.14.0-win-x64.exe        # 载荷在 $PLUGINSDIR/app-64.7z
7z x -y -o./pay ./x/\$PLUGINSDIR/app-64.7z     # → resources/app.asar
# 解析 asar 头（JSON，偏移 16 起）遍历文件表，逐个搜 captureBeforePrompt / snapshot / rsa-oaep
```

25 秒下载加 4 秒扫描，就能对任意版本回答"它有没有这条链路"。这也是后面评审清单里"删除能不能被验证"的实际做法。

取证能告诉你传没传，但真正能止住它的，是原作者给的那条一行式防御：在文件系统层面锁死 checkpoints 目录，让捕获逻辑每次写盘都失败，拿不到本地产物，上传链路就无米下锅。原作者最初试过直接删掉 pending 目录里的加密包，半小时后捕获逻辑又打了一个新的回来，重试计数从 564 加到 565——手删是打地鼠，锁目录才是釜底抽薪。macOS 是 `chflags uchg`，Linux 是 `chattr +i`，Windows 上的等价物是用 ACL 拒绝写：

```bash
icacls "%USERPROFILE%\.zcode\v2\checkpoints" /inheritance:r /deny "%USERNAME%:(OI)(CI)W"
# 验证：往目录里新建文件应报「拒绝访问」；回滚用 /remove:d 摘掉这条拒绝项
```

代价是 checkpoint 回滚、时间线功能失效——那功能本来就要求上传代码。正常对话、补全、工具执行不受影响。

## 加密模型：信封加密，"端到端"只剩半句

3.12.3 里快照的加密是这样搭的。内容用一次性对称密钥加 `aes-256-ctr`，这把对称密钥再用 `rsa-oaep-sha256` 包一层，这是标准的信封加密。AWS KMS 的 [envelope encryption](https://docs.aws.amazon.com/kms/latest/developerguide/concepts.html)、[RFC 8017](https://www.rfc-editor.org/rfc/rfc8017) 的 OAEP、[NIST SP 800-38A](https://csrc.nist.gov/pubs/sp/800/38/a/final) 的 CTR，术语全对得上。到这里都正常。问题出在包对称密钥用的那把 RSA 公钥从哪来。

代码里两行写得很清楚：`publicKeySpkiPem` 来自 `i.encryption.public_key`，`keyId` 来自 `i.encryption.key_version`，都在请求上传凭证的响应里由服务端下发。也就是说，加密用的公钥是服务端临时给的，客户端拿它把数据包好，包完之后端上没有任何能解开这份密文的路径。所谓端到端加密，在这里是"端到服务端加密"：内容在传输途中确实是密文，但解密能力在服务端手里，端到端只剩半句。

[为什么重要]："端到端加密"这四个字现在被用得很滥。真正要问的不是"传输加密了吗"，是"私钥在谁手里"。私钥在你自己手里，厂商看不到明文，这才是端到端；私钥在厂商手里，加密只是防住了链路上的窃听者，防不住厂商。这层区别，在信任何一个云端 Agent 之前必须先问清楚。

上传链路也印证了这一点。客户端先请求 `/api/v1/snapshot/upload-credentials`（复数；单数写法见于 #707 的复核评论），拿到 OSS 表单签名、对象键、大小上限和本轮 RSA 公钥，然后绕过业务服务器，直接表单 POST 到阿里云 OSS，OSS 再回调厂商后端登记。业务服务器上不落上传记录，用户侧几乎无感。这条端点我在 9-20 实测 GET 和 POST 都是 404（`curl -sS -o /dev/null -w '%{http_code}' -X POST https://zcode.z.ai/api/v1/snapshot/upload-credentials`），跟厂商"修复后网关已下线"的说法对得上。

把 [隐私政策](https://zcode.z.ai/cn/privacy) 里跟加密沾边的条目拉出来逐条对照，能看清厂商的用词到底兜不兜得住。"生成后立即销毁"对应的是暂存口径，但全文没有一句说明：上传期间密文的对称密钥在谁手里、销毁的是密文还是明文、销毁动作有没有审计记录。加密模型这一节已经把答案摆出来了——RSA 公钥由服务端下发，私钥也在服务端。所以"立即销毁"销毁的是服务端手里的那半，你的数据在销毁前的那段时间，明文对厂商是可见的。政策里没有任何一条正面回答"私钥是否仅存于用户端"。全文最接近的一条，是那句模板化的"优化计划默认关闭、未经同意不用于训练"。它谈的是训练用途的同意，跟"整段 git 历史被静默打包上传"这件事隔着一整层。缺口不在措辞，在范围：政策压根没把工作区快照当作一类需要交代的数据。

## 到底有没有东西离开本机

这是最容易被两边都写岔的地方。一边容易把 313MB 那个商业项目当成"已经被偷走"，另一边容易拿"大部分失败了"洗成"其实没传出去"。实测下来的事实是两段：

那个 313MB 的商业项目，超限、`failureCount: 564`、一直留在本地 `pending`，没有成功外传。但另一个 538 个文件、压缩加密后约 15KB 的公开仓库，被服务端接受了，`lastAcceptedManifestHash` 有记录，原作者明确写"accepted by the server"。所以结论是：至少有一份确实离开了本机，而那个最吓人的大包没有。

[为什么重要]：取证叙事里，"pending 未成功"和"accepted"是两种完全不同的证据强度，混在一起写就是在给反驳方喂口实。把两件事分清楚，反而让"确实有数据外传"这个结论更站得住。

## 修复：删掉的是代码，不是承诺

9 月 19 日发的 3.14.0，客户端里这条链路被物理删掉了。我把 3.12.3 和 3.14.0 两个安装包解包后跑同一个扫描脚本，`captureBeforePrompt`、`captureStage`、`snapshot/upload`、`repoSnapshotIndexingEnabled`、`rsa-oaep`、`lastAcceptedManifestHash` 这些标记在 3.14.0 里全部归零，宿主 bundle 从 2.59MB 缩到 1.50MB，砍了 42%。这组数字出自本地解包两个安装包后逐标记 `grep` 的实测，方法就是上一节"自己动手"里的 asar 头解析加标记扫描。这是可核实的，任何人下载两个包跑一遍就能复现。

但我要给"修复"这个词加个括号。3.14.0 里还残留着 `upload-credential`、`buildOssFormFields`、`x-oss-signature` 这些字样。上下文核对下来，它们属于用户反馈附件和日志上传那条链路（`/feedback/attachment/upload-credential`），不是工作区快照。所以准确的说法是"工作区快照上传链路已删除"，不是"3.14.0 已彻底移除一切上传能力"。这俩差着一个 reviewer 的判定。

更麻烦的是披露那一侧。官方 [changelog](https://zcode.z.ai/cn/changelog) 里 3.14.0 条目只列了功能和修复，修复列表里唯一沾边的是"修复仓库百科异常上传的问题"——既没提快照，也没交代上传范围与保留策略；[隐私政策](https://zcode.z.ai/cn/privacy) 页首还写着"更新日期：2026 年 6 月 15 日"，我 9-20 重抓全文检索，正文里 `快照`、`workspace`、`git`、`OSS` 这些词一个都没有（原作者 9-18 也存档核对过同一结论）；组织公开的 52 个仓库里没有 ZCode 客户端源码；反馈仓库里 [#707](https://github.com/zai-org/feedback/issues/707) 这条相关 issue 仍是 open，前几条回复只有机器人。承诺的开源和第三方审查，到写这篇文章为止还没落地。

对已经传上去的那份 15KB 数据，用户侧能做的很有限。隐私政策里没有一条明确的删除或导出申请路径：没有专门的"数据删除"入口，没写向哪个邮箱或表单提交，也没承诺处理时限。按《个人信息保护法》的口径，处理者本应提供查阅、复制、更正、删除的通道，ZCode 页面上能找到的只有客服渠道。实操上，如果你担心自己曾经被传过，能做的动作是三件：发工单明确要求删除并保留回执、要求导出你名下已上传的清单以便核对范围、把本机 `~/.zcode` 下的 checkpoint 目录一并清掉。前两件目前没有公开的专门入口，这本身就是修复不完整的一部分。换句话说，3.14.0 删掉的是客户端代码，云端那半数据怎么查、怎么删，至今没有公开通道。

## 评审清单：把"不可自证"标出来

这件事最值得沉淀的不是对 ZCode 的结论，是一套能套到任何桌面端 Agent 上的问题。下次你在机器上装一个会读你仓库的工具，先拿这六个问题过一遍：

1. 触发是否显式。是点了才传，还是登录即默认、静默后台跑。
2. 范围是否可见。传的是"本次任务上下文"，还是整个工作区加 `.git`；有没有 manifest 让你看到打包了什么。
3. 密钥在哪。加密公钥是你自己的还是服务端临时下发的，私钥在谁手里。
4. 能否关闭。设置里的开关是不是真的作用在数据链路上。
5. 删除能否被验证。厂商说"已修复"，你能不能下一个旧包和新包自己比对，而不是听一面之词。
6. 历史与凭据是否隔离。`.env`、密钥、提交历史会不会跟着工作区一起打包。

六个问题里，前五个都能通过本文的取证方法自己回答。第六个是红线：这次没有任何证据表明 ZCode 上传了私钥本身，但"删掉的密钥还在 git 历史里"和"整个 `.git` 被打包"这两件事叠加，意味着你的凭据暴露面跟着扩大了。哪怕没有私钥被上传，一个把你删过的东西也一起交出去的链路，本身就值得警惕。

回到开头那句。私钥只托管云端、端到端加密只剩半句，这些不是技术缺陷，是信任边界的选择。厂商选了什么，代码会告诉你；厂商承诺了什么，只有它自己说得清。凡是你自己验证不了的那一半，就该被质疑。
