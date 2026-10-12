# DeepSeek Harness 技术入门 · Task03 打卡：Preset / Scope / Host Plane / Agent Plane

> 课程：[Datawhale 组队学习《DeepSeek Harness 技术入门》](https://www.datawhale.cn/learn/summary/381)
> 对应内容：第 3 章前半（拆解"一切皆插件"原则·上）+ Task3 概念辨析
> 实践环境：macOS 26.6 · Node.js v24.16.0 · DeepSeek Harness 0.2.0-rc.2
> 打卡日期：2026-10-12

---

## 一、学习内容归纳

**四个核心概念，一句话版**：

- **Preset**：一份声明式的插件组合清单（patch YAML），描述一个会话的能力面。
- **Scope**：组合的边界。Preset 的贡献挂进会话自己的 Scope，可见性和生命周期由 Scope 决定——这就是"会话级能力从哪里来"以及"动态组合为什么必须有边界"。
- **Host Plane（宿主层）**：进程级单例——模型路由/LLM 适配器、Tool Registry 基座、会话日志/持久化、子进程树、设置/凭证。不随会话复制，共享但中立。
- **Agent Plane（会话层）**：每个会话一片，靠 Scope 挂载 Preset 贡献，靠 `isolate` 拿到私有服务实例。

**为什么不是每个 Session 复制一套完整插件树**：复制 = 资源浪费 + 状态不一致 + 热更新爆炸；Host 单例 + Scope 挂载让 4 套 Preset 共享同一个 Runtime 且互不污染。

**三类组合项**（Standard 基线的拆法）：面向模型的工具（工具面）/ 会话级状态（isolate 服务）/ Host 能力适配项（基座）。

**一个意外的收获——版本敏感的实锤**：对照书和本机 0.2.0-rc.2 的 preset 定义发现两处差异：书里 Tool Presentation 写 `mode: code`，本机实测 `mode: ptc`；书里 Minimal 包含 `str_replace_editor`，本机的 minimal.patch.yml 里没有这个插件行（包存在于 npm 但未进组合）。Task01 自测题问"哪些东西版本敏感"，这就是活例子——具体参数和默认组合会变，组合语义（patch 行、scope、isolate）不变。

## 二、作业 A：Host Plane / Agent Plane 分层图

![Host Plane / Agent Plane 分层图](./task03/img/task3-diagram.png)

标注的 Standard 能力（10 项 ≥ 8）及归属：

| # | 能力 | 归属 | 提供者 | 开放对象 |
|---|---|---|---|---|
| 1 | Persona 提示段落 | Agent Plane（Scope 贡献） | dsh-persona | 模型（上下文） |
| 2 | 文件读写/搜索工具 | Agent Plane（模型工具） | tool-fs / tool-fs-search | 模型（tools 数组） |
| 3 | Shell 工具 | Agent Plane（模型工具）+ Host 子进程树 | tool-bash（能力缝隙后端） | 模型 |
| 4 | Skills | Agent Plane（渐进披露） | skill-filesystem + tool-skill | 模型（按需加载） |
| 5 | Plan Mode | Agent Plane（**isolate: planMode**） | dsh-plan-mode | 模型＋会话 |
| 6 | Compaction 三件套 | Agent Plane（**isolate: compaction**） | dsh-compaction-* | 会话（上下文存活） |
| 7 | Subagent / Workflow | Agent Plane（**isolate: workflowEngine**） | dsh-tool-subagent / workflow | 模型（委托） |
| 8 | 模型路由 / LLM 适配 | **Host Plane（单例共享）** | llm 适配器注册表 | 所有会话 |
| 9 | 会话日志 / 持久化 | **Host Plane** | session-persistence | 所有会话 |
| 10 | 设置 / 凭证 | **Host Plane** | settings / credentials | 所有会话 |

组合树对照（书里的 Persona/Filesystem/Shell/Search/Skills/Plan/Subagent/Workflow/Interaction 九项）：1-4、7 即前七项的工具面，Interaction 对应 ask-user/todo/present，全部落在 Agent Plane；Model Route、Persistence、Settings 是 Host 侧的适配项——与"三类组合项"的分类完全对得上。

## 三、作业 B：改 Agent Plane vs 改 Host Plane（自己的例子）

**改 Agent Plane**（只影响一个会话的组合）：给"对外演示"会话写一个 preset 补丁，禁用 `tool-web` 和 `present`、把 persona 换成"只读代码讲解员"——同一台 DSH 里，我的工作会话完全不受影响。对应书内"禁用写文件工具"：一个 patch 行 `disabled: true` 的事。

**改 Host Plane**（所有会话同时生效）：把 bash 能力缝隙的后端从 `bash-local` 换成 `bash-sandbox`——从此任何会话、任何 preset 的 shell 都进沙箱；或者把模型路由切到本地 vllm 端点——所有会话的下一个请求都走新路由。对应书内"所有文件操作进远程容器"：改的是**能力的实现**，不是能力的组合。

判别口诀：问"这个改动让**谁**看见"——一个会话看见 → Agent Plane；所有会话看见 → Host Plane。

## 四、自测问题

**1. DSH 的分层避免了哪两种极端？各自代价是什么？**
极端一：每会话完整复制插件树——隔离彻底但资源浪费、多份状态难一致、插件更新逐会话重放。极端二：一切全局单例——省资源但会话互相污染（A 的工具漏进 B 的目录）、无法按会话定制。DSH 取中：Host 单例 + Scope 挂载 + isolate 私有实例。

**2. "会话 A 新增的 Tool 影响到会话 B"——问题出在哪一层？**
作用域层：该贡献被注册进了全局/宿主层而不是 A 的 Agent Scope（或服务没 isolate，实例泄漏到 Host）。修法：挪回 Scope 贡献或补 isolate 映射。

**3. PTC 与 Standard 在源码层面的差异有多小？改变的是什么？**
3 行 YAML：`workflow-ptc`、`tool-workflow` 两行 disabled，加一行 `tool-presentation`。改变的不是能力集合，而是**结果的呈现方式**。所以"PTC 不一定更快"：round-trip 压缩利于批量调用场景，单任务可能反而多一次汇总呈现。

## 五、收获与卡点

- **收获**：用 preset 定义里的 `group + isolate`（planMode / compaction / workflowEngine / terminals）当"会话级状态"的实证，把四个概念落到同一张图上；拿到一个判别口诀——"这个改动让谁看见"。另外抓到两处书 vs 当前版本的差异（mode 取值、Minimal 是否含 editor），"版本敏感"从抽象提醒变成了亲身经历。
- **卡点**：个别能力的归属要翻定义确认（终端服务是 isolate 的会话级，但子进程树是 Host 共享——同一个"shell"字眼横跨两层），初学容易想当然。
