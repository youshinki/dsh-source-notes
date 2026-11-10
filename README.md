# dsh-source-notes

**DeepSeek Harness（dsh）源码拆解系列**的配套仓库：每篇文章对应的代码片段、架构图、时序图和可复现脚本都沉淀在这里。

> 头条系列名：《工业级 Agent 是怎么造的：DeepSeek Harness 源码拆解》

## 这个仓库解决什么问题

官方文档讲的是「怎么用」和「概念是什么」，官方教程已经把 Cordis、插件开发、子系统参考写得很全。这个系列只做一件事：

**拆实现——为什么这么设计、边界情况怎么处理、换个做法会怎样。**

所以这里记录的是文档里查不到、只有读源码才能确认的东西。

## 版本锚定

dsh 处于开发者预览阶段，源码会持续重构。**本仓库所有笔记锁定在一个具体提交上**，避免文章写完就对不上代码：

| 项 | 值 |
|---|---|
| 上游仓库 | https://github.com/deepseek-ai/deepseek-harness |
| 版本 | v0.2.2-rc.1 |
| 锁定提交 | `fcc0ad462172522aaf5c93a7b2e01f6b9f0b0159` |
| 快照日期 | 2026-11-04 |
| 协议 | MIT |

复现环境：

```sh
git clone https://github.com/deepseek-ai/deepseek-harness.git
cd deepseek-harness
git checkout fcc0ad462172522aaf5c93a7b2e01f6b9f0b0159
pnpm install
pnpm run build
pnpm dsh web
```

每个笔记文件顶部也会单独标注它所对应的提交，单独复现某一篇时以文件内标注为准。

## 系列目录（共 13 篇）

| # | 标题 | 核心问题 | 状态 |
|---|---|---|---|
| 01 | 开篇：一天烧掉 92 万亿 Token，dsh 凭什么扛得住 | 三个架构取舍 + 全景图 | 初稿（含 3 张配图） |
| 02 | 一次请求的完整生命周期 | 输入到工具调用再到落盘，链路怎么走 | 已写（含 3 张配图） |
| 03 | Cordis 内核：插件、服务、事件怎么串起来 | loader 挂载机制与依赖关系 | 已写（含 3 张配图） |
| 04 | Agent 循环：调度、流式工具调用、中断与重试 | 循环层与执行层如何解耦 | 待写 |
| 05 | 上下文管理：会话、压缩、追加式 trace | Agent 为什么会「忘记」前文 | 待写 |
| 06 | 工具与权限：schema、自动授权审查、沙箱 | 危险操作怎么被拦住 | 待写 |
| 07 | Subagent：递归层级、任务分派、上下文隔离 | 子智能体为什么不会污染主上下文 | 待写 |
| 08 | 模型层：多模型路由与流式协议适配 | 换模型要改哪里 | 待写 |
| 09 | 插件三层结构：plugin / bundle / patch | `cordis.patch.yml` 覆盖机制 | 待写 |
| 10 | 可观测性：trace 与回放 | 「每次运行都有迹可循」怎么实现 | 待写 |
| 11 | 桌面端和 Web UI 为什么也是插件 | UI 的可替换边界在哪 | 待写 |
| 12 | 实战：读完源码，写一个自己的插件 | 把理解变成产出 | 待写 |
| 13 | 横向对比：dsh vs LangGraph vs Claude Code | 工业级 harness 的设计取舍 | 待写 |

## 目录结构

```
notes/      每篇一篇 markdown，命名 01-xxx.md，正文顶部标 commit
code/       每篇可运行的最小示例，按篇号分目录
diagrams/   架构图与时序图源文件（drawio / mermaid）
```

## 引用约定

- 引用上游代码时注明文件路径与提交号，遵守 MIT 协议保留原始版权声明。
- 本仓库内的文章正文版权归作者所有，转载需授权；代码示例可自由使用。

## 关于作者

杨振辉（youshinki），XR / 音视频全栈架构师。上一个开源系列：[ai-agent-zero-to-hero](https://github.com/youshinki/ai-agent-zero-to-hero)。
