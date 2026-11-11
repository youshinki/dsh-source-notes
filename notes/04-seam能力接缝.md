---
series: deepseek-harness-source-notes
part: 04
title: "第04篇：seam 能力接缝，为什么换个模型不用动主循环 | 源码拆解系列"
version: v0.2.2-rc.1
commit: fcc0ad462172522aaf5c93a7b2e01f6b9f0b0159
date: 2026-11-11
---

# 第04篇：seam 能力接缝，为什么换个模型不用动主循环

前三篇我们讲了：dsh 没有内核（第01篇）、一次请求在插件之间怎么流转（第02篇）、插件本身怎么被 Cordis 装上卸下（第03篇）。

但有个问题一直没回答：既然什么都是插件，那"模型"也是个插件吗？如果我想把 DeepSeek 换成另一家模型，要改多少代码？

答案是：**一个字都不用改 agent-loop**。因为模型不是一个"东西"，而是一条叫 `ctx.llm` 的缝（seam）。这一篇就把这条缝剖开看。材料来自上游 `docs/capability-seams.zh.md`。

## 一、seam 不是一个函数，是三个角色

dsh 对"可替换能力"有一个很严格的定义。一个 seam 同时由三种角色组成：

1. **Service Definition（服务定义）**：声明这个能力长什么样——有哪些方法、什么参数、什么返回值。它自己不干活。
2. **Service Provider（服务提供方）**：真正实现接口的代码。一个 seam 可以挂好几个 provider。
3. **Consumer（消费方）**：用这个能力的人，通常是一个面向模型的工具，或者 agent-loop 这种主干模块。

关键规则是：**consumer 只依赖 Definition，从来不 import 某个具体 provider**。换 provider 的时候，挂上新的、撤掉旧的，consumer 那行调用代码纹丝不动。

举个生活里的例子：你的笔记本有个电源口（这是 Definition），插原装充电器是 provider A，插第三方充电头是 provider B，插移动电源是 provider C。笔记本主板（consumer）只认"这个口能供电"这件事，它不在乎你插的是谁家的头。dsh 的 seam 就是把这个电源口提前焊死在架构里，而不是让主板直接连死一块电池。

上游原文说得很直白：一个包可以同时承担多个角色，但单一角色本身不是 seam——你要加一项能力，就得把这三件套一起设计。

这也是为什么 dsh 的目录里你会看到成对出现的包：`llm/llm`（Definition）旁边是 `llm/llm-deepseek`（provider），`fs/fs` 旁边是 `fs/fs-local`、`fs/fs-sandbox`。它们不是随便起的名字，是三个角色各住各的包。

## 二、拿 ctx.llm 解剖一只看看

`ctx.llm` 是最典型的一条缝。Definition 就是一个"适配器注册表"接口：provider 把自己注册进来，声明自己叫什么名字、支持什么模型。

它下面挂着三个 provider：

- `llm-deepseek`：调 DeepSeek 官方接口，这是默认。
- `llm-pi-ai`：另一家模型适配器。
- `llm-replay`：**回放用的假模型**，按录好的脚本返回结果。

消费它的是 `agent-loop`（第02篇讲的那个驱动一次 step 的主循环）和 `compaction-basic`（上下文压缩）。它们代码里写的是"从 `ctx.llm` 拿一个适配器、调 stream"，根本不知道身后是真模型还是录好的回放。

这就是 `llm-replay` 存在的意义——它不服务用户，它服务测试。你想给 agent-loop 写单元测试？挂个 replay provider，模型输出就是确定的，不用真花钱调 API。

顺带一提，`llm-deepseek` 自己又拆出一个小缝 `ctx.deepseekLlmApiExtensions`：官方适配器负责把各插件贡献的请求字段合并到一起。也就是说"主缝"里面还能再开小缝，但 consumer 看到的还是同一个入口。

## 三、整条 dsh 到处都是这种缝

`ctx.llm` 不是孤例。翻一遍上游的服务表，你会发现凡是"可能被换掉"的东西，都被拉成了一条 seam：

| 缝 | provider 举例 | 谁在用 |
|---|---|---|
| `ctx.shell` | `bash-local`（本地）、`bash-sandbox`（沙箱）、`pwsh-local`（PowerShell） | `tool-bash`、`tool-pwsh`、hooks 桥接 |
| `ctx.fs` | `fs-local`、`fs-sandbox`、`fs-ssh` | `tool-fs` |
| `ctx.web` | `web-search-deepseek`、`web-deepseek-api`、`web-fetch-http` | `tool-web` |
| `ctx.sessionPersistence` | `session-persistence-jsonl` | agent-loop、session-query |
| `ctx.subagent` | 进程内 spawn、fork、ACP、Codex、Claude Code、dsh-sdk | `tool-subagent` |
| `ctx.browserUse` | Playwright MCP、Chrome DevTools MCP、Stagehand | 浏览器工具 |

看出来了吗？**执行 bash 命令的工具 `tool-bash` 永远只调 `ctx.shell`**。你想把它从"直接在本机跑"换成"先过沙箱"，只要把 `bash-sandbox` provider 挂上去就行，`tool-bash` 一行不改。文件读写、网页抓取、子 agent、持久化日志，全是这个套路。

连"钥匙放哪"都是一条缝：`ctx.credentials` 下面有本地凭据实现，配置里只存对密钥的引用，真正的 token 由 provider 持有，consumer 按操作现取现用。凭据换了、轮换了，下一次请求自动生效，不用改业务代码。

## 四、为什么非要绕这一层

你可能会想：直接 `import llm-deepseek` 不就行了，为什么多搞一个接口？

三个实际收益：

**第一，可测试。** 刚才说的 `llm-replay` 就是——没有 seam，主循环和真模型死绑，自动化测试根本写不了。

**第二，可换环境。** 开发时用 `bash-local` 直接在本机跑；发布时换成 `bash-sandbox` 加一层沙箱；连远程机器时换成 `fs-ssh`。消费者不知道、也不需要知道脚下跑的是哪台机器。

**第三，第三方插件有位置可进。** 你想给 dsh 加一个自己的模型供应商？不用给主循环提 PR，按 Definition 写一个 provider 注册进 `ctx.llm` 就行。这才是"一切皆插件"能长成生态的真正原因——不是谁都能改内核，而是每个可扩展点都提前留好了缝。

还有一个容易被忽略的好处：**同一份 Definition 可以有多个 provider 同时挂载**。`ctx.subagent` 下面同时挂着进程内 spawn、fork、ACP、Codex、Claude Code、dsh-sdk 六个 provider，模型用工具时可以指定走哪一个。缝不是"二选一"，而是"谁都能注册，按需选择"。

## 五、四种角色，别混了

上游把所有服务分成四类，加新东西之前先对号入座：

- **core**：主干服务，只有一份实现，换了整个产品就不是它了。比如 `ctx.sessions`（会话存储本身）、`ctx.agents`。
- **seam**：本节主角，同一份 Definition、多份 provider。比如 `ctx.llm`、`ctx.shell`。
- **service**：就是一个普通服务，不强调可替换，但也挂在 ctx 上给大家用。
- **bundle**：组合点，把一堆插件打包成一个产品形态（桌面版、SDK 最小版、headless 版）。

判断标准很简单：这个东西**应不应该允许用户换一个实现**？应该，就做成 seam；不应该，就老老实实写 core。

反过来说，如果你在 dsh 里看到一个包名字里带 experimental，先去能力图里查它的角色——大概率是一条正在试的缝，provider 可能还在换。这种位置就是第三方插件最容易插进来的地方。

---

串一下：第02篇的主循环为什么能这么干净？因为它只跟 `ctx.llm`、`ctx.tools`、`ctx.sessions` 这些缝打交道，不认识任何具体 provider。第03篇讲的插件加载机制，就是用来把这些 provider 挂上去、再拆下来的。

下一篇（第05篇）我们钻进工具系统：模型说"我要调 bash"，从 tool call 到命令真正执行，中间那道 pre / execute / post 流水线是怎么走的，沙箱和审批又是怎么插进来的。

> 配图1 放这里：seam 三角色关系图（Definition / Provider / Consumer）
> 配图2 放这里：ctx.llm 的一个 seam 三个 provider 实例
> 配图3 放这里：dsh 里常见 seam 家族对照
