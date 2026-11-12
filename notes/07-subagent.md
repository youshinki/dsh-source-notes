---
series: deepseek-harness-source-notes
part: 07
title: "第07篇：subagent，子 agent 怎么起又怎么不污染主上下文 | 源码拆解系列"
version: v0.2.2-rc.1
commit: fcc0ad462172522aaf5c93a7b2e01f6b9f0b0159
date: 2026-11-12
---

# 第07篇：subagent，子 agent 怎么起又怎么不污染主上下文

你让模型"调研一下这个问题"，它说"好，我起个子 agent 去查"。

这个子 agent 到底是什么？是同一个模型再开一个对话窗口？还是真的起了一个新进程？它跑了 20 轮、调了 10 个工具、读了一堆网页——这些脏活会把主对话的上下文撑爆吗？

答案是：子 agent 有自己独立的 Session，主 agent 只在最后收到**一段结算通知**。中间那些工具调用、网页、推理过程，主 agent 一概看不到。这一篇就讲清楚这是怎么做到的。材料来自上游 `packages/subagent/subagent/README.zh.md`。

## 一、subagent 也是一条缝

第04篇讲 seam 的时候我们提过：`ctx.subagent` 是 dsh 上挂着 provider 最多的一条缝，下面同时挂着六个后端：

- `subagent-spawn-in-process`：同进程里新起一个 agent，最常用。
- `subagent-fork-in-process`：同进程里 fork 出一个已经跑过一轮的 agent。
- `subagent-acp`：走 Agent Client Protocol，连外部的 agent 进程。
- `subagent-codex`：直接调用本地装的 Codex CLI。
- `subagent-claude-code`：直接调用本地装的 Claude Code。
- `subagent-dsh-sdk`：通过 SDK 起一个可编程控制的子 agent。

模型调 `tool-subagent` 时会指定用哪一个 provider。也就是说"起个子 agent"在 dsh 里不是一种固定行为，而是一个可替换的策略——你今天用同进程 spawn，明天可以换成真去调一个外部 Claude Code 进程，主 agent 那行代码不动。

## 二、本地子级 vs 外部子级：能不能继续聊

这六个后端其实分两类，行为差别很大。

**本地后端（spawn、fork）**：子 agent 在同一个 dsh 进程里跑，有自己持久化的 Session。它干完一次活不消失，你（或主 agent）后面还能再发消息给它："刚才那个结论，再帮我查一下细节。"它记得之前聊过什么，支持冷恢复——dsh 重启了，能从日志把这个子 agent 重建出来。

**外部后端（ACP、Codex、Claude Code、SDK）**：子 agent 是一个独立运行时。它只接一次任务、跑完返回结果，然后就结束。你不能再发后续消息——它不属于你的进程，没有"接着聊"这个概念。

这个差别在用户视角就是：你看到的"子任务完成"通知，如果是本地子级，会说"它还在，你可以继续给它发消息"；如果是外部子级，会明确告诉你"它已经退出，不支持后续消息"。

## 三、深度和容量：别让 agent 套娃套死

递归最大的风险是套娃——子 agent 又起子 agent，几层下来并行几百个进程。dsh 用两个数字卡住：

**`maxDepth` 默认 1。** 意思是：主 agent 可以起子 agent，但子 agent 默认不能再起孙 agent。你想让它递归，得显式配 `maxDepth=2` 或更高。这是个很保守的默认——大多数"起个子 agent 调研一下"的需求，一层就够了。

**`maxActiveSubagents` 默认 8。** 整棵 subagent 树同时在线的子级总数上限是 8。顶层父 agent 自己不占名额。到了上限，新起子级会直接收到 `ACTIVATION_LIMIT_REACHED` 错误，**不排队**——dsh 故意不排队，因为"父 agent 在等子级，子级又在等自己占的名额"这种循环等待一旦出现就死锁了。名额只在当前进程内存里，不限制磁盘上累计了多少个历史子级。

调参方式也有讲究：调高上限不影响已经在跑的子级；调低上限时，已有的继续跑，新的要等用量降下来才放行。

名字也是这套体系的一部分。根 agent 固定叫 `lead`，后面起的默认叫 `agent-1`、`agent-2`……如果 `agent-1` 又起了个子级，完整路径就是 `agent-1/checker`。同级名字唯一，整棵树里不会撞名。主 agent 想给某个子级发消息，用这个路径就能找到它——不用记那串 36 字符的 Session id。

## 四、父子通信：父只收到一段结算通知

这是 subagent 设计里最关键的一条：**主 agent 看不到子 agent 的过程**。

子 agent 自己跑了多少轮、调了什么工具、读了哪些文件、中间想了什么——全部留在它自己的 Session 里。干完之后，系统向父 Session 追加一条 `user-role` 的结算通知，大致长这样：

> Background subagent `reviewer` finished and will do no further work unless you send it more.
> [子级最终输出里的非空文本块]
> Structured result: { ... }

就这一段。父 agent 看到的不是子级的完整 transcript，而是它**最后那一段总结 + 一个结构化 JSON**。子级的工具调用日志、网页内容、中间错误，全部不带回来。

这是有意为之。如果把子级的所有过程都塞回主上下文，那"起子 agent"就失去意义了——本来想隔离出来的脏活，结果全堆在主对话里，token 照样爆。隔离的收益就在这里：父级上下文只增加几百 token 的一段通知，子级那几千上万 token 的过程不进来。

还有个细节叫结构化结果。如果子 agent 被要求返回 JSON，那段结算通知里会单独附一个 `Structured result: {...}` 块，主 agent 可以直接解析它做下游决策，不用再从自然语言里抠数字。这就是为什么 dsh 里"起个子 agent 调研后返回结论"是个可编程的动作，不是聊天。

子级那边也被明确告知这一点。每个本地子级在被创建时都会收到一条系统提示：

> 你的最终答案会在父级在线时自动送达。把结果和证据写进最终回答里；父级看不到你的完整 transcript、工具输出和推理过程。如果你最后一个动作是给父级发消息，那完成通知会省略这段收尾文本。

也就是说子模型从一开始就知道"我写的东西只有最后一段会被看到"，它自己会把关键结论压缩进最终回答。

## 五、隔离和回收：拆的时候别留半截

子 agent 不是起完就不管了。dsh 对回收有严格要求：

**每个子 agent 都有自己的 Session 文件**，和父 Session 是分开的。父子关系通过一个叫 `subagent/descriptor` 的事件记在子级日志里——这个 descriptor 不进模型历史，第06篇讲的压缩也不会动它，冷恢复时靠它把子级重建出来。

**取消一个子级要调 `dispose()`**，它会等子级自己的清理函数跑完、再把它挂的所有子子级一起拆掉。你想把整棵树停掉，调 `drainDescendants([root])`，它会等所有正在跑的子级都结束、再统一收摊。dsh 不会"父进程退出、子进程变孤儿"。

**权限也隔离**：父 agent 被授权做什么，不代表它起的子 agent 自动有同样权限。给子 agent 发消息、打断它、看它的输出，都要单独过一遍权限校验。这在安全上很重要——你不想让一个被诱导的主 agent 通过"起个子 agent"绕开你设的限制。

---

串一下：第04篇讲了 seam，subagent 就是 seam；第05篇讲了工具流水线，`tool-subagent` 就是其中一个工具；第06篇讲了会话压缩，子 agent 有自己独立的压缩生命周期。这三层叠起来，才让"起个子 agent 干脏活"在工业上真的可用——不是概念演示，是真能跑几百轮、真能回收、真不污染主上下文。

下一篇（第08篇）讲模型层：多模型路由、流式协议适配、不同 provider 的请求字段怎么合并。为什么你能在同一个 dsh 里同时挂 DeepSeek、豆包、本地 Ollama，主循环不用改。

> 配图1 放这里：subagent 树（lead → agent-1, agent-2; agent-1/child）
> 配图2 放这里：本地 vs 外部 provider 对比
> 配图3 放这里：父子通信流（父派任务 → 子跑 N 轮 → 一段结算通知回父）
