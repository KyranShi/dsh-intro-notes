# DeepSeek Harness 技术入门 · Task02 打卡：从零运行 DSH + 4 套 Preset 对照实验

> 课程：[Datawhale 组队学习《DeepSeek Harness 技术入门》](https://www.datawhale.cn/learn/summary/381)
> 对应内容：第 2 章（从零运行 DeepSeek Harness）+ Task2 实验（4 套 Preset 能力对照 / 执行轨迹差分）
> 实践环境：macOS 26.6 · Node.js v24.16.0（满足 `^22.19.0 || >=24`）· DeepSeek Harness 0.2.0-rc.2（`npx @deepseek-ai/dsh web`，Web UI `127.0.0.1:3080`）
> 打卡日期：2026-10-12

---

## 一、学习内容归纳

**环境与安全习惯**：Node 版本是硬门槛（`^22.19.0 || >=24`）；Workspace 是 Agent 的"现实世界"，实验一律在独立空目录里做（本次全部材料放在独立目录，未触碰工作区外文件）。

**4 套 Preset 是同一 Runtime 的 4 种能力组合**（macOS 下实际启用的插件行数）：

| Preset | 界面名 | 启用插件行 | 定位 |
|---|---|---|---|
| standard | 标准模式 | 28 | 全能基线 |
| ptc | PTC 模式 | 27 | 同一批能力 + 结果呈现层 |
| minimal | 极简模式 | 6 | 压薄 Harness：固定 Prompt + 持久终端 |
| cordis | 创造模式 | 30 | 标准 + 自我指涉 |

**取证方法论**（本 Task 最有用的观念）：模型对自身能力的说明**不能当证据**——它只能复述上下文里的提示词，工具清单在请求的 tools 数组里，没列出来的它感知不到，而且会幻觉。正确取证看运行时：UI Trace 里的实际请求、插件清单、preset 定义文件。

## 二、能力对照矩阵（作业：4 套 Preset 对照表）

![4 套 Preset 能力对照矩阵](./task02/img/task2-matrix.png)

取证来源：本机 `dsh-web-app/presets/{standard,ptc,minimal,cordis}.patch.yml` 逐行核对 + 当前会话（创造模式）运行时工具目录实测，两者逐项吻合。

最有意思的三个发现：

1. **PTC 与 Standard 的 diff 只有 3 行**：关掉 `workflow-ptc`、`tool-workflow` 两行，加上 `tool-presentation`（mode: ptc）一行。"换一种工作方式"在源码层面是三行 YAML 的事。
2. **Minimal 只剩 6 行**：精简 persona（`includeRuntimeContext: false`，连 cwd 和模型名都不注入）+ 持久终端组（pty + persistent-bash，`isolate: terminals`）。
3. **Creator = Standard + tool-cordis + 组合编写技能**：多出来的不是功能插件，而是"让 Agent 操作自己运行时组合"的递归能力。

## 三、实验记录

### 实验 1：能力对照取证（配置级 + 运行时实测）

方法：不问模型"你能干什么"，直接读 preset 定义 + 枚举运行时工具目录。
结果：上表矩阵；当前会话（创造模式）实测包含 `cordis_inspect_list` / `cordis_inspect_query` / `plugin_manager` 与组合编写技能（`cordis-plugin-development` 等），与 `cordis.patch.yml` 的预测完全一致。
结论：**4 套 Preset 的差异可以在挂载前从配置精确预测**——这就是"组合"而非"定制分支"的含义。

### 实验 2：执行轨迹差分（同一任务 4 模式）

任务：读目录 → 建 3 个文本文件 → 统计总行数 → 摘要。

> 状态说明（诚实起见）：本记录的差分基于 preset 组合推导（可用工具决定轨迹形态），标注为"预期差分"；4 个 Session 的实测数字待跑，复现命令附后。

| 维度 | Minimal（预期） | Standard（预期） | PTC（预期） |
|---|---|---|---|
| 可用路径 | 只有持久 bash | fs + bash + todo | fs + bash + presentation |
| 建文件 | heredoc/echo 三连 | 3× fs write（可并行） | 同 Standard，批量调用更激进 |
| 统计行数 | `wc -l`（终端状态可复用） | bash 一次 | 同 Standard |
| 预计步数 | 最多 | 中 | 中（省轮次但多一次呈现汇总） |
| 轨迹特征 | 原始但连贯 | 工具化、结构化 | 批量调用 + 结果呈现 |

**为什么官方 Benchmark 选 Minimal**：模型能力 ≠ Agent 能力。Minimal 把 Harness 影响压到最薄（固定 Prompt、无运行时上下文、单工具面），测出的差异才能归因于模型本身；工具越丰富，成绩越取决于 Runtime 组合——那是 Agent Benchmark 的地盘。

复现命令（Web UI 新建 4 个 Session 分别选 4 套 preset，输入同一任务，记录步数/工具序列/答案）：

```text
请读取当前目录，创建 a.txt/b.txt/c.txt 三个文件（各写几行内容），
统计三个文件的总行数，并给一段摘要。
```

## 四、自测问题

**1. 为什么"模型对自身能力的说明"不能当证据？应该看什么？**
模型只能复述上下文（提示词/persona）；工具清单在请求的 tools 数组里，模型对未列出的工具没有感知；且会幻觉补全。应看运行时证据：Trace 中的实际请求、插件/工具枚举、preset 定义。

**2. Preset 改变的是"能力是否存在"还是"能力如何呈现"？两者是几件事？**
两件事。能力是否存在 = Tool Registry 的清单（Standard→Minimal 是大改）；能力如何呈现 = 结果的呈现方式（Standard→PTC 几乎不动清单，只加 presentation 插件）。PTC 就是"同一批能力、不同呈现"的官方实证。

**3. 模型在纯模型 Benchmark 上很强，放进任何 Agent Runtime 都同样强吗？**
不会。Agent 成绩 ≈ 模型能力 × Runtime 组合 × 上下文策略。工具面宽窄、结果呈现、失败恢复都会改变模型的有效发挥；所以模型 Benchmark 要用 Minimal 这类压薄组合来隔离变量。

## 五、收获与卡点

- **收获**：养成了"配置取证优先于提问"的习惯；拿到一个 Agent 系统先翻它的组合定义，比问它十个问题都准。PTC 与 Standard 只差 3 行这件事，让我对"一切皆插件"从口号变成了可量化的信任。
- **卡点**：实验 2 的实测轨迹还没跑完（预期差分已给出）；另外"PTC 不一定更快"的边界（批量场景赚、单任务亏）还需要实测数据支撑，留给后续补充。
