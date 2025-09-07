+++
title = "Windsurf 入门指南"
date = "2025-06-14T18:30:10+09:00"
updated = "2025-07-28T10:58:23+09:00"
draft = false
description = ""
[taxonomies]
tags = ["AI"]
+++

## 为什么是 Windsurf
曾经叫 `Codeium`, 在 2022 年还是 `Github Copilot` 的时代即提供支持 `VS Code` 和 `JetBrains` 的 AI 编程工具插件; 于 2024 年推出类似于 `Cursor` 的基于 `VS Code` 的 IDE `Windsurf`, 随后将公司更名
* 免费的基础使用和更便宜的订阅价格
* 和 `Cursor` ~~互相抄~~ 功能没太大区别
* 被 `OpenAI` 以约30亿美元收购
> 目前告吹, 公司核心成员被 `Google` 挖走后开发 `Devin AI` 的 `Cognition` 收购了公司, ~~然后开始了疯狂加班~~
* 被 `Anthropic` 断供~~高端的 AI 商战~~, 目前不能使用 `Claude 4`, ~~被收购之后恢复了~~
> `Cursor` 相关的信息污染太厉害了, 乔戈里峰问题

最近我发现 `TREA` 好像更好用, 问题似乎出在 `Windsurf` 配置的上下文太短了, ~~Kiro 大法好~~

## 主要功能
相比插件更好的代码上下文感知 `Context Awareness`, 记忆能力, 调用第三方工具能力
### Tab
补全, 跳转, 导入
### Command
唤出输入框输入指令, 适用于当前文件范围内的修改, 作用于当前游标位置或选中的代码段
> 小技巧, 在注释中写入指令, 然后 `Cmd/Ctrl+I` 随后使用类似 `do it` 这样的简单指令即可
### Cascade
项目范围的大规模理解与调整, 可以运行命令行, 网络检索等工具, 也可以通过 `MCP` 协议调用外部工具
* Planning Mode: 将计划输出到 Markdown 文件, 便于调整和长流程处理
* Browser: 读文档, 获取网页内容; 前端开发
### 总结
三个功能从上到下, 需要的人工的输入越来越多, 上下文感知要求越来越强, 流程和处理时间越来越长, 根据适当情况选择适当的功能

## 原理与局限
神经网络的仿生学原理, 生物神经系统存在大量通过突触连接的神经元, 特定神经活动使关联的神经元突触产生可塑性变化, 是记忆和认知的基础; 神经网络通过数学模型, 用于编码输入与输出之间的概率关联, 本质是数据统计上的关联; 生物的神经系统的学习过程是持续发生的, 学习与使用并非截然分开, 而当前的大语言模型通常将"训练"与"推理"严格分离, 参数在训练后即固定, 所以不要尝试"教会"它, 因为它知道错了也不会改, 而是要通过 `prompt` 提供上下文信息, 使其在有限的上下文窗口中做出合理响应
* 上下文能力长度的限制, 上下文限制不仅在输入端, 同时在中间状态与输出端, 所以大模型能精准利用的有效上下文会明显比标示的上下文 `token` 参数小
* 注意力机制, `<Attention is All You Need>`, 但是 `Attention` 不是 `Understanding`, 无法主动区分重点与细节, 只是根据训练统计模式做 `token` 级别的预测
* 缺乏长期记忆机制, 不能自发积累跨 `Session` 经验和知识, 需要外部记忆模块
> 严格来说模型是可以做轻量微调(PEFT)的

大语言模型是高性能的统计关联引擎, 其输出本质是"有结构的幻觉" - 杨立昆的观点

## 最佳实践
### 比喻
用 minori 的 <ef - a fairy tale of the two.> 的情节来描述大模型非常合适. 新藤千寻在 12 岁时由于事故患上顺行性遗忘症, 从那以后的事情她只能记住从当时往前推算的 13 个小时. 每一个清晨醒来, 她都必须重新阅读自己的日记，从中寻找"我"的线索, 并在日记本上不断续写她自己. 少年麻生莲治在车站与其相遇, 两人的距离在不断的相识与遗忘中逐渐缩短, 经过挣扎和犹豫两个人最终走到了一起. 这段爱情也带着某种悖论式的悲伤, 对于莲治来说, 他无比喜欢千寻, 但是某种意义上来说, 千寻只是在扮演日记本中那个喜欢莲治的自己.
### 日记本
所以归根到底, 是要给~~美少女~~大模型提供优化过的~~日记本~~`prompt`
* 总是使用 Markdown 写文档, 大语言模型对其有优化
* 一致的项目结构与命名规范
* 提供项目的结构和功能的描述, `User Story`, `Product Requirement Document`, 记录决策与约定
* 显式且准确的提供完成任务所需的上下文内容和工具调用, 善用`@`提示
* 积极的使用 `Workflow` 和 `Rule` 功能, 实际上就是模版化标准化的 `prompt` 的应用
* ~~Prompt Engineering is Dead, everything is a Spec.~~
### 自我强化
* 让模型自己进行总结归纳, 做手动调整, 再作为 `prompt` 给到模型上下文, 以此循环
* 压缩 `prompt`, 并且让模型自己来验证它是否理解 `prompt`
* 渐进式的任务分解提示, 分步骤的提示和指令, 让模型自己来做流程的完成情况维护
* 让 Agent 提问题, 写 QA
### 其他注意
* 写通用的规则, 然后用 Markdown 的超链接功能引用, 抹平 IDE 规范差异
* 不要使用大模型做代码格式化 (可以让 `Cascade` 调用外部工具来处理)
* 多 `commit` 和 `commit --amend`, 防止人工~~智障~~智能大规模删改文件
* 复杂操作生成操作清单而非直接执行
* 标记工具生成的代码的位置, 以免大模型直接修改
* 将常用代码封装成可复用的模块, 并写好 `README.md`, 减少大模型上下文负担
* **矛盾**, **模糊**和**不准确**的信息会让大模型偏移
* 当 `Session` 偏离, 立刻开启新的 `Session`, 不要纠结
* 将命令编写结构化的 Markdown 文档, 然后在 `Cascade` 对话框中引用 (如果"几句话就能说明的东西"就调用 `Cascade`, 那很有可能是用的不对)

## 总结
`Windsurf` 的人工智能, 将补全, 重构, 规划, 搜索和执行结合为统一体验, 从局部函数到全局架构, 都可以以更自然的方式与其协同. 但它并不魔法般全能, 它依赖上下文, 依赖清晰的项目结构, 依赖好的 `prompt` 提示与任务规划. 因此, 真正的生产力提升, 并非来自"你来把这个代码写了", 而是来自构建了一个让它理解, 协作, 执行的良好环境. 也许可以把 `Windsurf` 看作每天早上醒来要读你笔记的~~美少女~~程序员, 而开发者的职责, 就是写好那本"日记本", 明确的目标, 清晰的上下文, 有逻辑的步骤和合理的调用.

## 其他一些看法
`Vibe coding` 译为氛围编程, 在实践中, 对编码能力的要求下降了, 但是对项目规划和产品设计, 团队协作和组织能力, 团队积累与结构化知识的要求反而是提高了, 某种意义上来说, 反而拉大了组织与组织之间的差距. 人工智能更像一个每天都是新入职的, 代码能力扎实的实习生, 如何让其能快速了解代码规范, 项目架构, 周边工具, 沟通方法才是发挥其作用的关键. 开发者角色也从"代码生产者"转变为"意图定义者", 核心能力迁移到问题验证与处理, 领域建模, 知识结构化和精准表达.
```
组织差距:
强的组织 -> 精准的领域语言 -> 结构化知识库 -> AI 高效产出
弱的组织 -> 模糊的需求 -> 碎片化的信息 -> AI 混乱输出
开发模式:
传统模式 [记忆 -> 检索 -> 编码]
AI 模式 [意图 -> 验证 -> 优化]
```
大语言模型的原理和局限也决定了它在 `Day 0` 即代码从头开始的效果比较好, 而 `Day 1+` 即后续的开发和维护上, 会需要开发者更多的指引.~~身为程序员最讨厌的就是写文档和别人不写文档~~拒绝 `YOLO coding`

## 更进一步
什么才能称得上思维, 思考的本质, 观点与偏见, 错了也不改与反例的权重, 波普尔的 `可证伪性(Falsifiability)`
- 乔戈里峰
- 杨立昆

## 参考资料
* [Windsurf创始人：我们对Java工程师做了很多优化和适配](https://mp.weixin.qq.com/s/1xbOMGlFI0No2AuixK6vbw)
* [ef - a fairy tale of the two.](https://zh.wikipedia.org/zh-cn/Ef_-_a_fairy_tale_of_the_two.)
* [Vibe Coding: Automated Research Digest & Windsurf Standards](https://github.com/PaulDuvall/vibecoding)
* [Getting started with Windsurf](https://docs.windsurf.com/windsurf/getting-started)
* [Windsurf best practices](https://www.reddit.com/r/Codeium/comments/1h2psgy/windsurf_best_practices/)
* [Vibe coding](https://en.wikipedia.org/wiki/Vibe_coding)
* [Windsurf CEO: Betting On AI Agents, Pivoting In 48 Hours, And The Future of Coding](https://www.ycombinator.com/library/MO-windsurf-ceo-betting-on-ai-agents-pivoting-in-48-hours-and-the-future-of-coding)
* [Real-world engineering challenges: building Cursor](https://newsletter.pragmaticengineer.com/p/cursor)
* [解析Yann LeCun的自主机器智能构想](https://zhuanlan.zhihu.com/p/657425622)
* [Zed](https://zed.dev/)
* [TREA](https://www.trae.ai/)
* [feat(cli): Add context management feature with profiles #834](https://github.com/aws/amazon-q-developer-cli/pull/834)
* [The Hidden Algorithms Powering Your Coding Assistant](https://diamantai.substack.com/p/the-hidden-algorithms-powering-your)
* [Conducting smarter intelligences than me: new orchestras](https://southbridge-research.notion.site/conducting-smarter-intelligences-than-me)
* [Gemini CLI](https://github.com/google-gemini/gemini-cli)
* [Introducing Kiro](https://kiro.dev/blog/introducing-kiro/)
* [Spirit of Kiro](https://github.com/kirodotdev/spirit-of-kiro)
* [My LLM codegen workflow atm](https://harper.blog/2025/02/16/my-llm-codegen-workflow-atm/)
* [The New Code — Sean Grove, OpenAI](https://www.youtube.com/watch?v=8rABwKRsec4)
* [Claude Code: Best practices for agentic coding](https://www.anthropic.com/engineering/claude-code-best-practices)
* [Coding with LLMs in the summer of 2025](https://antirez.com/news/154)
