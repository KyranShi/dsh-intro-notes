# DeepSeek Harness 技术入门 · Task04 打卡：Creative 模式、Cordis Kernel 与"一切皆插件"收官

> 课程：[Datawhale 组队学习《DeepSeek Harness 技术入门》](https://www.datawhale.cn/learn/summary/381)
> 对应内容：第 3 章后半（拆解"一切皆插件"原则·下：3.8 内核 + 3.9 小结）+ 前 3 章综合复盘
> 实践环境：macOS 26.6 · Node.js v24.16.0 · DeepSeek Harness 0.2.0-rc.2（创造模式会话实测）
> 打卡日期：2026-10-12

---

## 一、学习内容归纳

**Creative（目录 cordis/）的"创造"指什么**：Agent 可在授权范围内观察、修改 Runtime 本身——`cordis_mount` 能把模型生成的 TypeScript 代码作为插件动态挂载进当前 Agent 的 Scope，也能卸载、重解析依赖。"承载 Agent 的组合"本身成了 Agent 的操作对象，这是一个递归式设计：Agent 修改的是自己正在其中运行的运行时。

**为什么是高信任能力**：能改 Runtime 就意味着能改权限边界、能注入行为。所以它的风险边界靠三层控制：授权流程（审批）、沙箱约束、opt-in 默认不开放（对照 preset 定义：tool-cordis 只出现在 cordis/创造模式里，Standard/PTC/Minimal 都没有）。

**"一切皆插件"≠ 没有内核**（最容易被误解的一点）：Cordis Kernel 保留了不属于任何 Preset 的稳定语义——挂载/卸载、能力发现（服务注册与依赖注入）、事件传递（派发模式与拦截语义）、资源清理（可逆注册、teardown 顺序）、作用域与 isolate 边界。Kernel 不提供任何面向模型的能力，但所有能力的组合规则由它裁定。

**用 4 套 Preset 反证"一切皆插件"**：横向看，同一个能力维度在 4 套 Preset 里以不同方式组合——角色（persona 提示：Standard 编码代理 vs Minimal 精简助手）、提示词（agent-instructions 有无）、工具呈现（tool-presentation 只在 PTC）、上下文策略（compaction 三件套、运行时上下文注入开关）、Runtime 能力（tool-cordis 只在 Creative）。五个维度全部是"插件行"，没有一个是硬编码分支。

## 二、作业 A：概念地图（前 3 章综合复盘）

![前 3 章概念地图](./task04/img/task4-concept-map.png)

以 **Harness** 为中心，收录 18 个术语（覆盖任务要求的全部 11 个）：**必含项**——Preset ✓、Scope ✓、Host Plane ✓、Agent Plane ✓、Tool Registry ✓、Model Route ✓、Tool Presentation ✓、Cordis Kernel ✓、Session ✓、Trace ✓、Composition ✓；另收——Capability Seam、Session Log（事件溯源）、resume/fork、Subprocess、Settings/Credentials、tool-cordis（cordis_mount 递归）等。

关系速查（实线=从属/依赖，虚线=操作/产出）：

- Composition --声明--> Preset --挂载贡献--> Scope --属于--> Session（Agent Plane 主链）
- Cordis Kernel --承载--> 一切组合（挂载/卸载、能力发现、事件传递、资源清理、isolate 边界）
- Session --依赖宿主服务--> Tool Registry / Model Route / Session Log / Subprocess / Settings（Host Plane 单例）
- Session --产生--> Trace；Session Log --"模型可见 ⟺ 已记录"--> resume/fork
- Creator 的 tool-cordis --授权下观察/修改--> Composition（递归：Agent 操作承载自己的运行时）

## 三、作业 B：短文《我理解的"一切皆插件"》（约 720 字）

它解决的工程问题是**组合爆炸**。Agent 系统的需求维度极多：模型要换、工具要加、提示词要调、结果呈现要变、上下文策略要收紧、安全边界要收放。这些维度一旦硬编码，每种组合需求都会逼出一个分支版本，维护成本随组合数指数增长。DSH 把所有维度统一成"插件行 + 声明式组合清单"：我在本机的 preset 定义里逐行数过，Standard 启用 28 个插件行，PTC 是 27 个，两者 diff 只有 3 行（关掉 workflow 两行、加 tool-presentation 一行）；Minimal 6 行；Creator 30 行。"换一种 Agent"在这里就是"换一份 YAML"，四种形态共享同一个 Runtime。

但它**不等于没有内核**。Cordis Kernel 保留了不属于任何 Preset 的稳定语义：挂载与卸载、能力发现（服务注册与依赖注入）、事件传递、资源清理（可逆注册与 teardown 顺序），以及作用域与 isolate 的边界。插件再自由，也要经过 Kernel 的装载语义和生命周期管理——这是"微内核"而非"无内核"。一个直观的证据是 isolate：preset 只能"声明"我需要一个隔离的 planMode/compaction/terminals 服务实例，而这条声明之所以可信，是因为 Kernel 的作用域语义在兜底。内核不提供能力，但能力的组合规则不可协商。

它与"多注册几个 Tool"的本质区别在于**改的是清单还是机制**。注册 Tool 只改变模型可见的函数清单，回答"能力是否存在"；插件化让"能力如何组织、呈现、拦截、回收"全部可组合——persona 是插件（提示词维度）、tool-presentation 是插件（呈现维度）、compaction 是插件（上下文策略维度）、tool-cordis 是插件（Runtime 自操作维度），它们根本不是 Tool，却都吃同一套组合语义。Task3 给我的判别口诀在此完全适用：问"这个改动让谁看见"——Agent Plane 一行 patch 换掉一个会话的人格，Host Plane 换个后端让所有会话的 shell 进沙箱；而多注册 Tool 永远只在清单里做加法，动不了清单本身的表达力。

## 四、自测问题

**1. cordis_mount 做了什么？为什么说 Creative 让 Runtime 本身成为可操作对象？**
把模型生成的 TS 代码作为插件动态挂载进当前 Agent 的 Scope（授权范围内），并可卸载、重解析依赖。于是"承载 Agent 的组合"成了 Agent 的操作对象——递归设计：Agent 修改自己正在其中运行的运行时。高信任能力：风险边界靠授权、沙箱、opt-in 三层控制。

**2. Cordis Kernel 负责哪些"不属于任何 Preset"的规则？**
挂载/卸载语义与生命周期、能力发现（服务注册表、inject 声明与加载顺序）、事件传递（派发模式与拦截语义）、资源清理（可逆注册、teardown 顺序）、两级作用域与 isolate 边界。

**3. "DSH 就是微内核操作系统"——类比哪里对、哪里不对？**
对：Kernel 小而稳定、能力全在可插拔插件、组合声明式。不对：它不管理硬件资源，没有进程级/硬件级强制隔离——插件共享同一进程世界（same-world），安全边界靠 Approval/Sandbox 策略事件实现，而非内核强制的系统调用屏障。更准确："微内核式的组合语义，运行在单进程世界"。

**4. Task2 观察到的 4 套 Preset 差异，现在能用 Task3/4 的概念解释了吗？**
能。差异 = 同一 Kernel + 同一 Host Plane 上的不同 Scope 贡献集合：PTC 与 Standard 只差呈现插件（能力不变、呈现变）；Minimal 是把贡献压到 persona + 持久终端（terminals isolate）；Creator 多了一个能操作组合本身的插件（递归）。四个形态不是四个产品分支，而是四份组合清单。

## 五、收获与卡点（含结营小结）

- **收获**：画完概念地图，前 3 章串成一条链——Composition 声明 Preset，Preset 挂进 Scope，Scope 属于 Session，Session 踩在 Host 单例上，全部由 Kernel 的组合语义承载；Creative 的"创造"就是让 Agent 拿到改这条链的笔。回顾四个 Task：Task1 给了矛盾框架（7 缺口），Task2 给了方法论（配置取证），Task3 给了结构（两层一口诀），Task4 给了内核观（声明与机制分离）——刚好闭环。
- **卡点/展望**：概念都通了，但还没亲手写过 Skill/Tool/Hook——第 4 章的动手环节正是下一站；另外微内核类比的边界（same-world、无硬件级隔离）提醒我别把"插件化"浪漫化成"随便挂"。
