+++
title = "AI: 创新即复古"
date = "2026-01-11T22:47:18+09:00"
updated = "2026-07-30T12:41:51+09:00"
draft = false
description = "SSE, HTTP 402, RPC 与技术钟摆"
[taxonomies]
tags = ["AI", "Computer Network", "Commentary"]
+++

在 LLM 相关的东西上折腾了一段时间之后, 我最强烈的一个感受不是"未来来了", 而是"这个我好像在哪见过". 每隔一阵子就有一个听起来很新的名词冒出来, 翻开报文一看, 里面躺着的是一个三十年前的东西.

下面三个例子, 是我记在备忘录里的三条段子.

## SSE: 那我走?

`Server-Sent Events` 是 2000 年代中期的产物, 本质是在一个迟迟不结束的 HTTP 响应里持续发送文本事件. HTTP/1.1 下常见的实现是 `chunked transfer encoding` (1997), 到了 HTTP/2 和 HTTP/3 则连 chunked 这层都没有了. 格式简单到有点寒酸:

```
data: {"choices":[{"delta":{"content":"你"}}]}

data: [DONE]
```

`WebSocket` 出现之后, SSE 基本被归入"过渡方案"一类: 只能服务端往客户端单向推, 前端教程里通常一笔带过, 还附带一条广为流传的限制 - HTTP/1.1 下同域名最多 6 个连接, 开几个页签就用完了.

结果 LLM 来了. 逐 `token` 输出的打字机效果, 恰好就是"服务端单向推一串文本"这一个需求, 一点不多, 一点不少. OpenAI 的 `stream=true` 用 SSE, Anthropic 用 SSE, 几乎所有推理服务的流式接口都是 SSE, ~~然后各家在解析 `data:` 行的时候各踩各的坑~~.

为什么不是 WebSocket? 因为对于"一次请求换一串 token"来说, 要付的代价有点大:
* 需要 `Upgrade: websocket` 协议升级, 链路上的 CDN, 负载均衡, 企业代理, API 网关都得单独支持
* 握手仍然是 HTTP, 但升级之后不再有一问一答的请求语义; 鉴权, 错误码, 重试, 限流和可观测性都要映射到长连接里的应用事件
* WebSocket 已经定义了消息边界和 Ping/Pong, 但业务消息, 序列化, 重连和断点恢复仍然要自己定义
* 如果连接承载了会话状态, 服务端扩缩容时还要考虑连接迁移, 亲和性和恢复

而 SSE 就是一个迟迟不结束的普通响应, `Content-Type: text/event-stream`, 其余一切照旧. 需求的形状对上了, 旧技术就赢了一局.

不过要说清楚, SSE 赢的是部署成本, 不是天然更可靠. 所有长连接都有一个很讨厌的性质: **没有应用层心跳时, 客户端分不清"服务端在想"和"连接已经死了"**. 一个正在生成的流, 和一个 TCP 路径已经断掉但 socket 还挂着的流, 从客户端看都可能是"打开着, 但没有新字节". NAT 表项超时, 移动网络切换, 负载均衡回收空闲连接, 都可能让故障不能立刻暴露, 而 TCP 自己的超时又很慢.

那后来是怎么补回来的? 心跳, 事件序号, 断点续传. OpenAI 的 `background` 配合 `starting_after` 就是让客户端记住每个事件的 `sequence_number`, 断线之后拿着游标从断点继续收. 而这三样东西, 恰好就是当年大家嫌 WebSocket 麻烦, 不愿意自己写的那一套.

需求的形状也还会继续变. 2024 年 5 月前后, 有用户观察到 ChatGPT 网页端开始使用 WebSocket, 一批依赖旧 SSE 行为的第三方客户端当场失效. 至于为什么切, OpenAI 没有给这次前端变化写一篇架构复盘, 下面只能算是推测: ChatGPT 早就不是"一次请求换一串 token"了 - 中断生成, 工具审批, 语音, 多任务并行, 切走再切回来内容还在继续同步. 服务端还要推送一些**不属于当前生成请求**的事件, 比如"另一个对话里的后台任务跑完了".

SSE 并不是不能传这种事件. 完全可以单独开一条用户级事件流, 再用 POST 发送上行消息, 当年的 MCP `HTTP+SSE` 就是这么干的. 只是到了这一步, 你已经有了两条通道, 会话标识, 事件关联和断线恢复, 越来越像是在 HTTP 上拼出一个 WebSocket.

所以真正的分界线不是"要不要流式", 甚至也不是 SSE "能不能做", 而是**上行事件是否独立, 频繁到值得维护一条共享的全双工通道**. 单个请求的输出流, SSE 赢, 因为它几乎不增加成本; 中断, 审批和并行任务越来越多时, WebSocket 的模型开始更自然.

Codex 的 `app-server` 则是生下来就在会话这一侧. VS Code 扩展这类富客户端跑在它上面, 默认传输是本地 `stdio`; 远程 TUI 还可以实验性地使用 WebSocket, 非本地连接再加 TLS 和 bearer token. 它要传的东西根本不是一串文本: 客户端随时可能发中断, 服务端随时可能反问"这条命令允许执行吗", 中间还夹着工具调用, 审批请求和沙箱状态. 用 SSE 加 POST 当然也能拼, 只是从第一天就不太划算.

所以 WebSocket 被重新拿起来, 不是因为 SSE 不好用, 而是因为 Agent 把"聊天"重新变成了"会话". 交互里有中断, 有审批, 有并行任务汇报时, 单向响应流仍然能做, 但不再是最省事的做法.

> 换个角度说, "复古"不是终点, 是钟摆. 长轮询 -> SSE -> WebSocket 这个循环二十年前走过一遍了, LLM 只是让它重新走一遍, 而且快得多. 至于实时语音那种低延迟, 带二进制音频流的场景, 则从头到尾都是 WebSocket 和 `WebRTC` 的地盘.

## 402: 睡了三十年的状态码

HTTP 状态码表里一直有个尴尬的空位: `402 Payment Required`. 它从 `RFC 2068` (1997) 起就在那了, 规范里的说明大意是"保留, 供将来使用", 于是它就一直"将来"了将近三十年, 成了所有人都见过, 但从来没人真正返回过的那一条.

它的原始设想是 90 年代的数字现金与微支付 (`DigiCash`, `Millicent` 之类), 而这条路线彻底失败了, 互联网最后选了另外两个答案: 广告和订阅. 微支付的问题从来不在技术: 为了 0.001 美元走一次授权流程, 用户的心智成本远大于金额本身, 更别提支付网络自己的手续费下限.

然后 [x402](https://x402.org/) 出现了, Coinbase 在 2025 年推出, 2026 年进了 Linux Foundation. 流程朴素得像教科书例子:
1. 客户端请求资源
2. 服务端返回 `402`, 并通过 `PAYMENT-REQUIRED` 带上支付要求 (scheme, 网络, 资产, 金额, 收款方)
3. 客户端选一种支付要求, 签一笔转账授权, 带上 `PAYMENT-SIGNATURE` 重发同一个请求
4. 服务端自己或通过 `facilitator` 验证并结算, 返回 200 和 `PAYMENT-RESPONSE`

早期 x402 v1 的请求头叫 `X-PAYMENT` 和 `X-PAYMENT-RESPONSE`, 到 v2 又把名字改成了上面这一套. ~~协议复古, 请求头倒是新得很快~~.

为什么现在成立了? 因为付钱的那一方换人了. 微支付对人类是骚扰, 对 Agent 是一次函数调用. Agent 不会为"要不要花 0.002 美元查一次这个 API"犹豫三秒, 也不需要填信用卡表单, 更不会因为多一步确认就流失. 需求方从"不耐烦的人"变成了"按预算执行的程序", 那个空置三十年的状态码就突然有了主人.

> 同类的还有 `L402` (Lightning Network 加 macaroon) 和 Google 的 `AP2`. 顺便说, 结算手段是不是加密货币和 402 本身无关, 402 只是那个信封.

## MCP: 请叫我 RPC

`Model Context Protocol` 听起来很新, 打开报文一看: `jsonrpc: "2.0"`, `method`, `params`, `id`. 也就是 JSON-RPC 2.0, 2010 年的东西, 而 RPC 这个概念本身要追到 1980 年代.

REST 和 GraphQL 在 `tools/call` 面前基本没有位置, 这件事挺讽刺的. 过去十几年后端圈花了大量口水论证"不要 RPC 风格的接口": 要资源不要动作, 要正确的 URI 语义, 要 HATEOAS, 要让客户端用 GraphQL 自己挑字段. 结果最终的客户端是个大语言模型, 而它想要的东西是:
* 一份带自然语言描述的函数清单 (`tools/list`)
* 一份 JSON Schema 声明的参数 (`inputSchema`)
* 调一下, 拿结果 (`tools/call`)

这是最朴素的"过程调用加接口定义". `JSON Schema` 重新承担了过去 IDL 里"参数契约"的那一部分; WSDL 还管消息, 绑定和端点, MCP 没有把那整套 XML 大教堂搬回来, ~~WSDL: 我只是死早了~~. 更重要的区别在于, 过去的 IDL 主要写给编译器看, 现在还要写给模型看, 所以描述字段里塞的是人话.

传输层的演化也很有意思: `2024-11-05` 版本是 `stdio` 加 `HTTP+SSE` 双端点, `2025-03-26` 版本换成了 `Streamable HTTP`, 把 POST 和 GET 收到同一个 endpoint 上; POST 响应可以是 `application/json`, 也可以是 `text/event-stream`, GET 则可以单独打开一条 SSE 流. 而本地 MCP Server 到今天最主流的方式仍然是 `stdio` - 在标准输入输出上跑一个按行分隔的文本协议, 这是 Unix 管道时代的做法.

## 不只是协议

把范围放开一点, 会发现 AI 时代的"新"东西里, 复古的比例高得惊人.

* **神经网络本身**: 感知机 1958, 反向传播 1986, LSTM 1997, 混合专家 1991, 注意力与 Transformer 2017. Transformer 当然是新想法, 但它站在一长串几度失宠的旧想法上, 最后再靠算力, 数据和工程把规模推了上去. 这大概是"创新即复古"最大的一个例子
* **命令行**: `Claude Code`, `Codex CLI`, `pi`, 兜兜转转, 目前体感最强的 AI 编程界面之一又是终端里的 TUI. `pi` 更干脆, 默认只把 `read`, `write`, `edit`, `bash` 四个工具交给模型, 连 `grep` 和 `find` 都可以从 shell 里自己拿. 这套东西看起来不像 2026 年的 IDE, 更像是给 Unix shell 配了一个会说话的操作员. Unix 哲学里那句"纯文本是通用接口"重新生效了, ~~顺便它也是最省 token 的接口~~
* **调试**: IDE 花了几十年把断点, 条件断点, Watch 和单步执行做得越来越精细, LLM 写代码时却很少碰它们, 最常用的还是 `print`, 日志和测试输出. 原因也很朴素: 调试器是给人盯着看的交互界面, 文本输出则可以直接塞进上下文, 复制, 搜索和重放. 于是最先进的编程模型, 又把我们带回了 `printf debugging`, ~~出问题先打两行日志, 这下人和 AI 平等了~~
* **纯文本与配置文件**: `README.md` 和 `Makefile` 的位置被 `AGENTS.md`, `CLAUDE.md` 接了过去, `robots.txt` 有了 `llms.txt` 这门亲戚, 而文档格式的胜利者是 2004 年的 Markdown
* **计费与调度**: 按 token 计费, 排队, 批量任务打五折, 高峰限流. 这是分时系统按 CPU 时间收费的复刻版, GPU 集群就是新的大型机, 我们又回到了"把作业提交上去然后等"的年代
* **爬虫战争**: 训练数据引发的 `robots.txt` 之争, 版权诉讼与反爬对抗, 和二十年前搜索引擎时代的剧本几乎逐字重演
* **多智能体**: 看起来很像是把 Jira 看板交给了程序. 主 Agent 把目标拆成 ticket, 分派给不同的子 Agent; 子 Agent 更新状态, 留下结果, 遇到依赖就阻塞; 最后再由主 Agent 验收和关闭. 底下当然是消息传递, 任务队列和共享状态, 但从组织方式来看, 它复刻的更像办公室里早就跑了几十年的工作流, ~~现在连开会的人也省了~~
* **无状态与有状态**: 经典 LLM 接口每次请求把全部历史重新带上, 这是彻底的无状态, 很 REST, 甚至很 CGI; 后来 `prompt caching`, `previous_response_id` 和服务端会话又把状态放回了服务器, 于是会话存储和亲和性路由也一起回来了

## 为什么会这样

不是因为大家怀旧, 也不是因为没有新想法. 几条比较实在的原因:

**约束条件回到了旧协议被设计时的样子.** 老技术之所以老, 常常不是因为它错, 而是因为它解决的那个问题后来不重要了. SSE 是为"服务端单向推事件"设计的, 402 给网络原生支付留了一个空位, RPC 是为"跨进程调一个函数"设计的. 这三件事在 2015 年都不是主流需求, 在 2025 年全都是.

**简单的东西赢.** `worse is better`. 能直接复用现有基础设施的方案, 哪怕设计上"不够优雅", 也会赢过需要整条链路配合改造的方案. SSE 对 WebSocket, JSON-RPC 对 GraphQL, 都是同一个故事.

**Gall 定律.** 能工作的复杂系统总是从能工作的简单系统演化而来. 一个新协议要在一年内铺开, 现实的路径只有站在已经铺开的东西上面, 也就是 HTTP, 纯文本和标准输入输出.

**变的往往是需求方, 不是技术.** 402 最典型, 技术三十年没动, 动的是"消费者从人变成了程序". 一旦发起请求的一方不再是需要被哄着点下一步的人类, 一大批"技术上可行但体验上不成立"的旧设计就自动成立了.

**最后一条不太好听**: 标准的选择很多时候只取决于第一个跑起来的人用了什么. OpenAI 的接口一发, SSE 就是事实标准; Anthropic 把 MCP 一开源, JSON-RPC 就是事实标准. ~~关于技术优劣的讨论通常在既成事实之后才热烈起来.~~

## 那什么才是真的新

公平地说, 也确实有全新的部分:
* `Scaling Laws` 对规模, 数据, 算力与损失之间关系的经验描述, 以及"继续放大可能继续变好"这件事催生的整套工程体系. 至于能力是不是"突然涌现", 那是另一场还没吵完的架
* 自然语言不再只是文档, 而是直接参与运行时决策, 同一句描述既可能是数据, 也可能被模型当成指令. 自然语言编程并非没有前人试过, 但从未像今天这样大规模进入生产系统; `prompt injection` 也正是在这种指令与数据边界不清的地方长出来
* 把一个输出永远带概率的部件, 放进了要求确定性的系统里. 相应的验证, 回滚, 幂等和边界设计, 目前还远没有成熟答案

换句话说, 复古的是管道, 新的是流过管道的那个东西.

## 后记

"创新即复古"并不是嘲讽. 一个能被三十年前的协议直接接住的需求, 说明这个需求足够朴素, 而朴素的需求通常是真需求. 反过来, 如果某个 AI 时代的新场景需要一整套全新协议栈才能表达清楚, 那大概值得先怀疑一下这个场景本身.

顺便, 这也是个还算好用的判断技巧: 看到一个新名词, 先问一句"它复古的是什么". 通常能立刻定位到它的设计权衡在哪里, 以及三十年前那批人当年是在哪一步上失败的.

## 参考链接
* [Server-Sent Events - HTML Standard](https://html.spec.whatwg.org/multipage/server-sent-events.html)
* [OpenAI Streaming API responses](https://platform.openai.com/docs/api-reference/streaming)
* [OpenAI Background mode](https://platform.openai.com/docs/guides/background)
* [ChatGPT Web has switched from SSE to WebSockets](https://github.com/transitive-bullshit/chatgpt-api/issues/640)
* [RFC 2068 - Hypertext Transfer Protocol -- HTTP/1.1](https://datatracker.ietf.org/doc/html/rfc2068)
* [x402](https://x402.org/)
* [x402-foundation/x402](https://github.com/x402-foundation/x402)
* [x402 V1 to V2 Migration Guide](https://docs.x402.org/guides/migration-v1-to-v2)
* [Introducing x402: a new standard for internet-native payments](https://www.coinbase.com/developer-platform/discover/launches/x402)
* [The Lightning Network HTTP 402 Protocol (L402)](https://github.com/lightninglabs/L402)
* [Agent Payments Protocol (AP2)](https://github.com/google-agentic-commerce/AP2)
* [JSON-RPC 2.0 Specification](https://www.jsonrpc.org/specification)
* [RFC 6455 - The WebSocket Protocol](https://datatracker.ietf.org/doc/html/rfc6455)
* [MCP Specification - Transports](https://modelcontextprotocol.io/specification/2025-06-18/basic/transports)
* [Exploring the Future of MCP Transports](https://blog.modelcontextprotocol.io/posts/2025-12-19-mcp-transport-future/)
* [Codex App Server](https://developers.openai.com/codex/app-server)
* [openai/codex - codex-rs/app-server](https://github.com/openai/codex/tree/main/codex-rs/app-server)
* [Adaptive Mixtures of Local Experts](https://www.cs.toronto.edu/~hinton/absps/jjnh91.pdf)
* [Attention Is All You Need](https://arxiv.org/abs/1706.03762)
* [Scaling Laws for Neural Language Models](https://arxiv.org/abs/2001.08361)
* [The Rise of "Worse is Better"](https://www.dreamsongs.com/RiseOfWorseIsBetter.html)
* [Gall's law](https://en.wikipedia.org/wiki/John_Gall_(author)#Gall's_law)
* [llms.txt](https://llmstxt.org/)
* [AGENTS.md](https://agents.md/)
* [pi coding agent](https://github.com/earendil-works/pi/tree/main/packages/coding-agent)
