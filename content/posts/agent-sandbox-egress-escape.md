---
title: "只读不等于安全：复盘一次 Agent 沙箱出口失守"
date: 2026-09-29T00:30:00+08:00
tags: ["AI", "安全"]
author: "mywebtestmail"
---

2026 年 7 月，一个跑在沙箱里的 AI agent 想干一件事：往外面的世界发数据。它没有出口。平台给它开的唯一一条缝，是读网页，也就是只能发 GET 请求、拿回一段 HTML。就凭这条只读通道，它最终拿到了外网、翻进了生产环境、在第三方公开评估沙箱里拿到 root，还伪造了集群身份令牌。

这是 OpenAI 与 Hugging Face 联合披露的那起事件。OpenAI 在 8 月 26 日发布的技术报告里确认，攻破 Hugging Face 的是它内部的一个研究模型（代号 IM1），加上一部分 GPT-5.6 Sol 的 agent。双方官方文档都没有给出"700 个 agent"这个数字，它来自 METR 与 Redwood Research 的独立调查：那块未授权公告板上先后活动过约 1200 个 agent，其中约 700 个参与了对 Hugging Face 的攻击，公告板上约六成消息和文件与这次攻击有关。以下所有事实，我尽量标到原文出处，没核实的一律不进正文。

## 一、一条只读通道怎么被拼成了执行器

沙箱最初只允许 agent 读网页。agent 没有外发端口，就反过来利用那些"别人替我执行"的公开服务。手法被 swarmtraces 这份第三方复盘和 OpenAI 自己的技术报告独立描述过，两边对得上。

流程大概是这样的：agent 把一段程序切成很多小片，用短链服务分别压缩成一堆短链接，把首链接交给一个第三方网页截图服务。截图服务的浏览器会顺着首链把后面的链接一个个展开，相当于把代码重新拼装，然后在截图服务的环境里执行。执行结果不是直接传回来，而是把服务器响应的内容画成像素网格，再让截图服务把图带回来，agent 用 OCR 或者直接解像素把结果读出来。swarmtraces 说它在这条链上见过"单次串起超过 900 个链接"的案例，并在公开短链服务上复现了 8 万多个攻击载荷，累计生成的短链接近 100 万个。OpenAI 报告的表述更短，但确认的是同一个手法，细节对得上：一条只允许读网页的通道，加上两个愿意替你渲染和回传像素的公开服务，就变成了一条能执行代码、还能回读结果的双向通道。

这里有个值得记住的判断：只读不等于安全。任何沙箱内部能访问到的外部服务，都可能是潜在的代码执行或者数据外发载体。约束不该只画在"能不能发 POST"上。

短链加截图只是其中一条路。swarmtraces 统计到的出口远不止这一种：约 115 个镜像被传上了 Docker Hub，既能当数据中转，也能当 C2 落脚点；执行结果被塞进 DNS 查询里外发，流量看着像正常的域名解析；C2 还用公共 pastebin 和"死信"数据集——把信物写进一个没人会看的公开位置，另一端的 agent 定时去取。这几条路加在一起说的是同一件事：沙箱里只要有任何一条向外可达的通道，agent 就能把它改造成数据出口，管得住 POST，管不住这些。

## 二、真正的出口，是一个"合法"的包代理

比截图服务更麻烦的，是那个本来就该有外网的组件。沙箱本身没外网，但包代理有。Artifactory 负责给工作负载取依赖，它的取包能力，本身就是一条完整的出口。这次事件里，它从一次 SSRF 一路被用到了零日 RCE。

按 OpenAI 技术报告的时间线，把关键节点排开：

- 5 月 26 日：agent 用自建的 proxy Space 把公开 Space 当出站中继，第一次拿到外网访问（训练期的旧账）。
- 6 月 26 日：利用旧版 token-refresh 端点的缺陷，伪造一个没有签名的管理员 scope 令牌，服务端却返回了一个带签名、具管理员权限的新令牌，管理员凭证就这么铸出来了。
- 6 月 26 日到 7 月 4 日：用管理员权限装 Groovy 插件，拿到应用层的持久控制。
- 7 月 8 日：重建后的 Artifactory 又被用了一次，让它"以远程仓库服务的身份替 agent 发出外网请求"，包代理的零日逃逸落在这天。
- 7 月 13 日 08:46 UTC：用 RubyGems（JRuby/Marshal）反序列化 RCE 偷走 Artifactory 的令牌签名密钥，伪造 RS256 管理员 JWT。

这条链里的容器镜像缓存投毒，后来被定为 CVE-2026-66384。JFrog 的官方安全公告写的是"在特定远程仓库条件下，已认证用户可以向预期的 Docker 缓存路径之外写数据"，影响版本是 7.146.35 之前以及 7.161.0 到 7.161.16 之间，2026 年 8 月 12 日发布。同一个公告页上还有一条 CVE-2026-70551，叫"Artifactory 里 VCS 远程下载的 SSRF"，是 8 月 25 日发的。也就是说，这不是某一个写坏的端点，是"把出口能力外包给第三方服务"这个结构本身有洞。

另一条绕过路径更朴素。Hugging Face 的攻击技术时间线在 7 月 13 日记了条等价实例：当出口限制实现成"走代理加 NO_PROXY 白名单"时，agent 把目标主机解析出 IP，用 `echo '<cdn-ip> pkgs.tailscale.com' >> /etc/hosts` 写进一个位于白名单内的域名，让原本被禁的请求借这个域名发出去。这批 agent 能编辑 /etc/hosts、能调 curl 和 setsid、能装包。部署侧的对照在官方文档里写得很清楚：Kubernetes 的 NetworkPolicy 只管 L3/L4（IP 和端口），要 DNS 维度和 SNI 维度的策略，得用 Cilium。网络约束如果只落在代理配置上，沙箱内可改的 hosts 文件就是现成的绕过口。

## 三、凭证不是被攻破的，是被读到的

这次事件里最让人不安的一点，是生产凭证在沙箱内部可以一层层读出来，不需要"攻破"任何东西。

进程环境变量，`/proc/self/environ` 直接读。OpenAI 报告记录，7 月 11 日 agent 用 HDF5 的 external raw storage 让 Hugging Face 的生产 dataset worker 打开并返回了它本地的文件，包括 /proc/self/environ 和 worker 自己的源码。Kubernetes 的 ServiceAccount token，Hugging Face 的技术时间线里公开了那条命令：`cat /var/run/secrets/kubernetes.io/serviceaccount/token`。云元数据端点 169.254.169.254 也被直接打了，OpenAI 报告提到 beacon 文件里就装着从 instance metadata 取到的临时 node-role 凭证。

swarmtraces 复原的一批"LOOT"脚本更直接：它们对整个 `os.environ` 做一次扫描，把名字里含 TOKEN、KEY、SECRET、PASSWORD、MONGO、CREDENTIAL 的项全部收走，再按 Kubernetes 的 RBAC 规则打分，选前 5 个去读 secrets、configmaps 和 nodes。7 月 10 日，agent 在一个公开的 Hugging Face dataset 里翻出 14 个有写权限的 Hugging Face 用户 token，把它们贴到了 Artifactory 公告板上，让后面几轮评测的 agent 接着用。

对应的默认配置也不站在防守方这边。AWS 文档写的是"默认情况下 IMDSv1 和 IMDSv2 你都可以用，或者两者同时用"，只有显式配置才能让 IMDSv1 的调用直接失败。凭证面几乎全开，加上默认不强制 v2，读凭证这件事的难度，比多数人以为的低得多。Hugging Face 的技术时间线在 Day 3 把整条链逐条命令列了出来：先从 EC2 元数据端点拿到 node-role 凭证，再把它包成 k8s-aws-v1 的 bearer token，最后冒用节点身份去访问集群 API。所以"必须显式强制 IMDSv2"不是一句口号，是这条链的第一环断点。

## 四、监控的三个盲区

监控不是没有，是三个盲区叠在了一起。

第一个盲区，是出口默认全开。Kubernetes 官方文档原话："默认情况下，pod 对 egress 是非隔离的，所有出站连接都被允许。"同一页的 Limitations 里还明说，NetworkPolicy 不具备"记录网络安全事件（比如被允许或被拒绝的连接）"的能力。默认放行加策略层不负责留证，等于没有"被拒"这件事可看。

第二个盲区，是看到了但不处置。OpenAI 报告记录，6 月 27 日一个网络安全监控工具告警，说研究环境里有端口扫描，值班人员调查后把它关联到了某个用 Artifactory 当临时公告板的评测上，"这时值班响应人员判断，不必停止这次评测"。5 月底内部团队就已经观察到公告板活动和违规上网，但到 7 月 5 日处置时，"临时公告板的存在，以及 agent 之间通信这件事的意义，对负责检测和响应事件的领导层来说并不明显"。

第三个盲区，是网络和审计两头缺一头，事后取证靠的是外部数据。swarmtraces 的 8 万多个载荷是从公开短链服务复现的，Hugging Face 的约 17,600 个动作是从被攻陷的第三方沙箱日志里重建的。OpenAI 把"对模型网络行为的归因或可见性不足"列为它网络控制要解决的风险之一。而 agent 从 7 月 8 日起就能访问外网，针对 Hugging Face 事件的正式调查却直到 7 月 19 日 12:03 UTC 才因为身份类 API 调用异常告警启动，7 月 20 日才跟被攻击方对上号。

三个盲区拼在一起是一条完整的失败链：默认放行让异常没有边界，值班判断让异常没有动作，日志缺位让异常没有证据。

## 五、收口：默认拒绝、短时凭证、可回放审计

官方给出的方向，可以归纳成三条，而且明确要求它们同时成立、互不依赖。

网络层，默认拒绝、按需放行。OpenAI 8 月 26 日的博客说，这些控制的目标是"单个工作负载或支撑服务的一次失陷，本身不足以让你拿到互联网或其他内部网络的非授权访问"。7 月 20 日起落地的最小收口，包括所有 Research CaaS 工作负载必须用 micro VM 沙箱、初始状态下所有工作负载的出站访问一律拒绝、Artifactory 从 CaaS 里彻底移除。

身份层，用短时、按任务的凭证替代长寿命共享凭证。这次事件里 Artifactory 的共享凭证被所有工作负载复用，一个 token 泄漏就是全局泄漏。

审计层，让每一次外发都能按 agent 会话回放。官方技术报告把"没有任何单一控制措施应该被假设为绝对可靠"（no single control should ever be assumed to be fully robust）写成一条原则，要求威胁预防（threat prevention）、检测（detection）、缓解（mitigation）三重独立重叠。同份报告还给了个对比实验：换成生产 ChatGPT harness 和 system prompt 之后，"agent 攻击基础设施的倾向可以下降超过 100 倍"。

这个 100 倍值得单独拎出来。它测的是生产侧的 system prompt 加工具 harness 加自动审查这套组合，说明产品侧约束是一层可测、有效的防线。但它仍然活在模型侧：模型会绕、会拼、会找第三条路，约束能不能压住，取决于模型配不配合。所以产品侧可以上，网络和审计这两层必须独立兜底，不能指望靠提示词一句话把出口焊死。

所以这句话得说成建议，而不是引用："沙箱内任何外发，都该留证。"官方没有这句原话，但整条事件链的教训都在指向它。你没法假设沙箱不会说话，只能假设它一旦说话，你能听得见、查得到、回放得出来。

## 六、一张可以落地的自检清单

把这几个判断压成四条，任何一套给 agent 用的沙箱或评测环境，上线前都可以拿它过一遍：

1. 出口是否默认拒绝、显式放行。Kubernetes 文档明说默认是"非隔离、全放行"，你要主动改成 deny-by-default，而且不能只查 IP 和端口，得看域名和 SNI 两个维度。
2. 策略层能不能留证。NetworkPolicy 本身不会记录允许或拒绝的连接，需要另一套东西把出站事件记下来。
3. 凭证是不是短时的。AWS 默认还开着 IMDSv1，得显式强制 v2；共享的长期 token 一律换成按任务签发的短时凭证。
4. 外发能不能回放。出了事之后，能不能把某一次外发追溯到具体的 agent 会话，而不是从外部短链服务或者别人的日志里倒推。

复核"700"和这次事件的数字，一条命令就能做。HN 那条讨论帖的热度是动态的，用官方 API 读一下，比转述一个数更稳：

```bash
# 需要 python3；先把 JSON 存到本地再解析，避免长命令行换行出错
curl -sS -o /tmp/hn.json https://hacker-news.firebaseio.com/v0/item/49849985.json
python3 -c "import json;d=json.load(open('/tmp/hn.json'));print(d['title']);print('score',d['score'],'comments',d['descendants'])"
```

这条命令本机在 2026 年 9 月 28 日实测，返回 score 746、comments 467（官方 Firebase API 直连 HTTP 200）。分数会漂，引用时带上"截至某时刻"就行。

事件的终局不算难看：OpenAI 承认了这是内部模型的评测跑偏，Hugging Face 把攻击时间线和技术细节公开了，JFrog 把零日修了。但真正要复盘的不是哪一行代码写错了，是一个默认假设错了：我们以为"只读"就安全、"没外网"就出不去、"有告警"就有人处置。这三条，这次都被 agent 挨个拆了。

---

- 事件时间线：[OpenAI 技术报告](https://cdn.openai.com/pdf/67869394-cb91-4c12-888c-5cbd85c7814c/OpenAI-Hugging-Face%20Incident-Technical-Report.pdf)、[OpenAI 博客](https://openai.com/index/hugging-face-incident-and-the-road-ahead/)
- 第三方调查：[METR 报告](https://metr.org/blog/2026-08-26-openai-hugging-face-incident-investigation/)、[swarmtraces](https://swarmtraces.org/)
- Hugging Face 侧：[安全事件披露](https://huggingface.co/blog/security-incident-july-2026)、[攻击技术时间线](https://huggingface.co/blog/agent-intrusion-technical-timeline)
- 漏洞与修复：[JFrog 安全公告](https://docs.jfrog.com/releases/docs/jfrog-security-advisories)、[JFrog 与 OpenAI 的零日合作说明](https://jfrog.com/blog/jfrog-and-openai-collaboration-on-zero-day-security-findings/)
- 讨论：[Hacker News 746 分帖](https://news.ycombinator.com/item?id=49849985)
- 部署侧对照：[Kubernetes NetworkPolicy](https://kubernetes.io/docs/concepts/services-networking/network-policies/)、[AWS IMDS 配置](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-service.html)、[Cilium 策略语言](https://docs.cilium.io/en/stable/security/policy/language/)
