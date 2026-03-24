# 2026-03-15 claude-mem 架构讨论总结

## 背景

这次围绕 GitHub 项目 `thedotmack/claude-mem` 做了一次较完整的架构讨论，重点不是安装和配置，而是理解它的系统设计、数据流、记忆抽象方式，以及这套 memory policy 的优缺点。

目标是回答几个核心问题：

- claude-mem 本质上是什么
- 为什么它不是简单存聊天记录
- observation 和 summary 分别是什么
- SessionStart / session-init / PostToolUse / Stop 各自负责什么
- 这套规则系统怎么决定“该注入什么记忆”
- 这套设计的优点、缺点和可改进点是什么

---

## 一、对 claude-mem 的总体判断

结论：

**claude-mem 不是聊天记录保存器，而是一套给 Claude Code 用的跨会话记忆系统。**

它的核心目标不是“存住所有对话”，而是：

1. 采集 Claude 在工作过程中的关键行动痕迹
2. 把这些痕迹压缩成可复用的 observation
3. 在阶段结束时生成高层 summary
4. 存入本地数据库
5. 在未来新会话开始时，把最有价值的历史重新注入给 Claude

一句话概括：

**把 Claude Code 的短期上下文，转成可持续、可检索、可压缩的项目记忆层。**

---

## 二、整体架构理解

讨论中把它拆成四层：

### 1. Hooks
挂在 Claude Code 生命周期上。

作用：
- 感知会话事件
- 收集输入
- 调 worker API

### 2. Worker
本地常驻服务，是整个系统的大脑。

作用：
- 管理 session
- 接收 hook 数据
- 处理 observation / summary
- 提供 context injection
- 提供搜索接口
- 负责数据落库

### 3. SQLite
主存储层。

保存：
- sessions
- prompts
- observations
- summaries

### 4. MCP / Search
给 Claude 提供主动搜索记忆的能力。

例如：
- search
- timeline
- get_observations

一句话总结：

**hook 是入口，worker 是执行引擎，SQLite 是事实存储层，MCP/search 是检索层。**

---

## 三、整体流程（高层版）

这次讨论把整体流程归纳成 6 步：

### 1. SessionStart
新会话开始。

系统从历史中拉取 observation + summary，生成压缩上下文注入给 Claude。

### 2. UserPromptSubmit / session-init
用户发出 prompt。

系统把本轮 prompt 接入 claude-mem：
- 记录当前项目
- 记录 prompt
- 记录当前会话
- 准备启动后续处理链路

### 3. PostToolUse
Claude 每调用一次工具，就触发一次工具后处理。

这一步会把：
- toolName
- toolInput
- toolResponse
- cwd
- sessionId

交给 worker。

### 4. Worker 处理 observation
worker 不会简单原样存工具日志，而是会：
- 找到对应 session
- 进入队列
- 交给当前 provider 的 agent
- 把原始工具痕迹提炼成 observation
- 落入数据库

### 5. Stop
一轮工作停下来时，尝试生成 summary。

不是每次都保证成功落库，但会触发一次 summary 尝试。

### 6. 下次会话再利用
未来新 session 开始时，再次从 observation + summary 中恢复上下文，形成闭环。

一句话版：

**先读历史，再登记当前 prompt，再记录工具行为，再尝试生成总结，最后下一次再把这些记忆喂回去。**

---

## 四、SessionStart 的理解

我们明确了一个很重要的点：

**SessionStart 的职责不是创建会话，而是注入历史上下文。**

它做的事情：
- 确保 worker 在运行
- 获取当前项目范围
- 请求 `/api/context/inject`
- worker 从本地数据库拿 observation 和 summary
- 经过规则系统筛选后，拼装成上下文文本
- 注入给 Claude Code

### 是否调用 LLM
讨论结论是：

**SessionStart 这一步通常不需要额外调用 LLM。**

它更多是：
- 本地 SQL 查询
- 本地筛选
- 本地格式化

### token 成本
这里也澄清过：

- **不是额外一次独立的 LLM token 开销**
- 但会**占用当前会话上下文 token**

也就是说，它主要消耗的是当前 prompt 窗口，而不是单独发起一次新推理。

### 是否会无限大
结论是不会无限大。

原因：
- 有项目过滤
- 有数量裁剪
- 有分层展示
- 有上下文窗口上限

但如果配置不保守、历史太多，仍然可能膨胀。

---

## 五、为什么系统必须记录用户输入

这是中间专门讨论过的一个关键点。

问题是：
为什么记忆系统不仅记录工具调用，还要记录用户 prompt？

结论：

**因为只看动作，不记录用户输入，就只知道做了什么，不知道为什么做。**

记录用户输入的必要性在于：

1. 让系统知道当前任务目标
2. 给 observation / summary 提供语义锚点
3. 帮助后续总结时理解“这些动作是在服务什么目标”
4. 未来恢复时，不仅恢复动作，还恢复任务意图
5. 支持多轮 prompt 导致的任务方向变化

一句话：

**不记录用户输入，系统看到的是动作；记录了用户输入，系统才能理解意图。**

---

## 六、session-init 的理解

session-init 被定义为：

**把当前这轮 prompt 正式接入记忆系统。**

它不是只针对第一条 prompt，而是每次用户提交 prompt 都会经过的入口。

它主要记录这些信息：
- `contentSessionId`：Claude Code 外部会话 ID
- `project / cwd`：当前项目
- `prompt`：用户输入
- `promptNumber`：该 session 内第几次 prompt
- `sessionDbId`：claude-mem 内部数据库主键

它的意义有两个：

1. 登记当前这一轮工作
2. 决定是否启动后续处理链路

我们还讨论过一个工程点：

- `contentSessionId` 是外部业务会话 ID
- `sessionDbId` 是内部数据库主键

这种“外部 ID + 内部 ID”的双层设计是正常的工程做法。

---

## 七、observation 是什么

我们对 observation 下了一个统一定义：

**observation = Claude 在工作过程中留下的关键行动痕迹。**

特点：
- 不是普通聊天
- 主要来自工具调用
- 会记录 toolName / input / response / cwd
- 比 transcript 更接近“做事证据”

举的例子包括：
- grep
- git diff
- 读文件
- 搜代码

只要这些是通过工具层执行的，都可能成为 observation 的来源。

### observation 的作用
它不是日志，而是 memory 的底层事实材料。

更准确说：
- transcript 太长太脏
- 原始工具输出太噪
- observation 是从中提炼出的“对未来可能有价值的行动痕迹”

---

## 八、PostToolUse 是什么

PostToolUse 是 observation 采集链路的入口。

每次 Claude 调完一个工具，就会触发一次 PostToolUse。

这一步会把：
- `toolName`
- `toolInput`
- `toolResponse`
- `cwd`
- `sessionId`

发给 worker。

### 为什么不直接存原始工具输出
这是一个重点问题。

讨论结论：

**因为原始工具日志太脏，不适合直接作为长期记忆。**

原始输入/输出的问题：
- 太长
- 太重复
- 噪音多
- 不适合未来直接注入上下文

所以系统真正想存的不是原始日志，而是：
- 这次动作在干什么
- 它的发现是什么
- 对当前任务有没有价值

一句话：

**日志系统追求全，memory 系统追求有用。**

---

## 九、worker 如何处理 observation

这是讨论过的关键链路。

worker 在收到工具事件后，并不是直接写数据库，而是会：

1. 接收这条工具事件
2. 找到对应 session
3. 先进入处理队列
4. 交给当前 provider 的 agent（Claude / Gemini / OpenRouter）
5. 由 agent 把原始工具痕迹提炼成 observation
6. 最后再落库

这里的核心思想是：

**worker 不是简单转存，而是“先排队，再提炼，再落库”。**

agent 扮演的角色是：
- 把工具行为包装成 prompt
- 交给 provider 处理
- 返回结构化 observation
- 让 worker 写入数据库

### 最终 observation 落库会包含什么
当时给出的抽象字段包括：
- 属于哪个 session
- 属于哪个 project
- 时间信息
- observation 的核心内容（标题、narrative、结果等）
- 可能的类型或标签

一句话：

**最终落库的不是原始命令回显，而是“这次动作在这个项目里产生了什么有意义的记忆”。**

---

## 十、summary 是什么

我们给 summary 的定义是：

**summary = 比 observation 更高层的阶段性工作总结。**

作用：
- 说明这一轮主要做了什么
- 给未来 session 一个高层状态恢复点
- 提供方向，而不是所有细节

也就是说：
- observation 是底层行动痕迹
- summary 是高层工作抽象

### 两者区别
在调研 GitHub 项目的例子里，我们用过这样的区分：

#### observation 像什么
- 读了 README 发现项目定位是 persistent memory system
- 看了 worker-service 确认架构是 hook + worker + SQLite
- 看了 ContextBuilder 确认策略是 recent-first + count cap
- 看了 OpenRouterAgent 确认 provider 支持 OpenRouter

特点：
- 细
- 贴近动作
- 是“过程中的发现”

#### summary 像什么
- claude-mem 是给 Claude Code 用的跨会话记忆系统
- 核心价值在 observation / summary 分层和 context 编排
- SessionStart 主要走本地检索
- 架构方向对，但规则系统还比较朴素

特点：
- 高层
- 压缩
- 是“阶段性结论”

一句话：

**observation 是发现，summary 是判断。**

---

## 十一、为什么两个都要有，不能只留 summary

这个问题也有明确结论。

### 只留 summary 的问题
- 太粗
- 只有结论，没有证据
- 无法回看推导过程

### 只留 observation 的问题
- 太碎
- 只有细节，没有方向
- 恢复状态成本高

所以两者必须结合：

- summary 负责快
- observation 负责准

在 SessionStart 注入时：
- summary 先给高层方向
- observation 再补关键证据和上下文

一句话：

**summary 定方向，observation 补证据。**

---

## 十二、summary 是怎么产生的

讨论里确认：

**summary 主要在 Stop 阶段尝试生成。**

流程：
1. Stop hook 触发
2. hook 从 transcript 中提取最后一个 assistant message
3. 调用 `/api/sessions/summarize`
4. worker 收到请求后处理 summary
5. 成功则落库

### 是否每次 Stop 都一定落 summary
不是。

更准确说法是：

**每次 Stop 都会尝试触发一次 summary，但未必成功落一条。**

不落库的情况包括：
- 没有有效材料
- 材料太弱
- 被隐私规则跳过
- worker / provider 调用失败

### 是否要调用 LLM
这一步通常会。

所以讨论里也明确：

- SessionStart 更偏本地检索
- summary 生成更偏模型抽象

因此 claude-mem 的 token 消耗重点之一就在 summary 生成。

---

## 十三、provider 和模型配置的理解

我们确认过当前源码支持的 provider：
- `claude`
- `gemini`
- `openrouter`

默认配置是：
- `CLAUDE_MEM_PROVIDER=claude`
- `CLAUDE_MEM_MODEL=claude-sonnet-4-5`

其他默认模型：
- Gemini：`gemini-2.5-flash-lite`
- OpenRouter：`xiaomi/mimo-v2-flash:free`

配置位置：
- `~/.claude-mem/settings.json`

同时支持环境变量覆盖。

### 关于 minimax
讨论结论：
- 没看到原生 minimax provider
- 但可以理论上通过 OpenRouter 间接使用 minimax 模型
- 这通常不是只让 summary 用 minimax，而是 worker 侧相关生成任务都会走当前 provider

---

## 十四、规则系统：它到底怎么决定注入什么

这是这次讨论的核心。

我们的结论是：

**claude-mem 的关键不在“存了多少”，而在“下一次拿什么给 Claude 看”。**

当前看下来，它的主规则大概是 4 条：

### 1. 按项目筛
先确定当前项目范围。

原则：
- 当前项目优先
- 只看这个项目的记忆
- 有时会合并相关 worktree / parent project

这是第一道过滤，避免项目污染。

### 2. 按时间排
在项目范围内，采用 recent-first。

原则：
- 最近的 observation / summary 优先
- 更适合恢复“刚刚做到哪”

### 3. 按数量裁剪
不会把候选结果全塞进去。

原则：
- 只取前 N 条
- N 受配置控制
- 目的是控 token

### 4. 按层级展示
不是每条都全文展开。

原则：
- 前面少数 observation 才全文
- 其余降级成 timeline / 索引层
- summary 给高层
- observation 给细节
- 真要细查，再用 search / get_observations

一句话概括当前主策略：

**项目过滤 + recent-first + 数量截断 + 分层展示。**

---

## 十五、对这套规则系统的评价

我们对它做过一轮正反面判断。

### 优点
1. 简单
2. 快
3. token 成本可控
4. 很适合“继续刚才做的工作”

### 缺点
1. 可能忽略老但重要的信息
2. recent-first 不一定等于 most-important
3. 如果最近几条 observation 很脏，系统会放大噪音
4. 智能性有限，更像稳妥的规则系统，而不是高级 relevance engine

### 总结判断
这是一个：

**靠谱、务实、能跑，但不算高级的 memory policy。**

也就是说：
- 架构方向对
- 成本控制意识对
- 但策略上限不高

---

## 十六、如果重新设计，我会怎么改

这是讨论中最深入的一部分。

结论是：

**我不会推翻现有架构，但会重点升级 memory policy。**

提出了 6 个方向：

### 1. recent-first 升级为 hybrid ranking
不要只看最近，还要看：
- 当前任务相关性
- 信息密度
- 长期重要度
- 时间衰减

让系统能找回“老但关键”的记忆。

### 2. 给 observation 做质量分层
将 observation 分成：
- ephemeral（短命）
- working memory（中期）
- durable memory（长期高价值）

避免低价值工具痕迹抢占上下文预算。

### 3. summary 不要只依赖 Stop
建议做双层摘要：
- 会话中的 micro-summary
- Stop 时的 session summary

避免过度依赖最后一个 assistant message。

### 4. query-aware retrieval
不仅根据项目和时间检索，还要考虑当前用户这轮 prompt 的任务意图。

例如同一项目内，如果用户切换了任务方向，就不该只注入 recent memory。

### 5. 给记忆增加状态字段
例如：
- exploring
- decided
- blocked
- verified
- outdated

让系统知道哪些旧结论已经过时或被推翻。

### 6. 做记忆治理
不是只增量写入，而是定期：
- 去重
- 归并
- 过期
- 升维

否则长期下来，memory 系统会从资产变成噪音堆积器。

### 如果只能优先改 3 件事
最终给出的优先级是：

1. query-aware retrieval
2. observation 质量分层
3. 记忆治理

一句话评价：

**claude-mem 已经从“能记”走到了“能用”，但还没走到“很聪明地记、很聪明地取”。**

---

## 十七、用真实场景帮助理解

### 场景一：登录后被踢下线
用户输入：
- “帮我看下为什么用户登录后马上被踢下线”

Claude 过程中会：
- grep refreshToken
- read auth.ts
- git diff session.ts
- grep cookie options
- read middleware.ts

这些 PostToolUse 会被提炼成 observation，例如：
- 检查了 refresh token 刷新逻辑
- 发现 session.ts 改动影响 cookie 写入
- 中间件可能在 token 刷新后立即判定 session 无效

Stop 时 summary 可能变成：
- 本轮排查集中在 refresh token 和 session cookie
- 初步怀疑 session.ts 的 cookie 配置变更导致登录后 session 丢失
- 下一步验证 SameSite / domain / maxAge

### 场景二：调研 GitHub 项目
用户输入：
- “帮我看看 claude-mem 值不值得跟”

Claude 过程中会：
- 读 README
- 看 package.json
- 看 worker-service.ts
- 看 ContextBuilder.ts
- 看 OpenRouterAgent.ts

这些动作形成 observation，例如：
- 项目核心架构是 hook + worker + SQLite
- context injection 走本地检索
- summary 支持多 provider
- observation 策略偏 recent-first

最终 summary 可能是：
- claude-mem 是面向 Claude Code 的跨会话记忆系统
- 架构方向正确，关键价值在 context 编排
- 优点是分层设计清晰，缺点是系统复杂度高、策略不够聪明

这些场景帮助明确：

- observation 更像“过程中的发现”
- summary 更像“阶段性判断”

---

## 十八、这次讨论的最终共识

这次聊天里形成的关键共识可以压缩成几句话：

1. **claude-mem 不是日志系统，而是记忆系统。**
2. **它的核心不是存储，而是筛选和注入。**
3. **SessionStart 读历史，session-init 接当前 prompt。**
4. **observation 是行动痕迹，summary 是高层总结。**
5. **当前规则系统靠谱，但偏朴素，适合连续工作流，不算高级检索系统。**
6. **如果要变强，关键在 query-aware retrieval、observation 分层和记忆治理。**

---

## 最后一段总结

从工程视角看，claude-mem 最大的价值，不是“终于把聊天存下来了”，而是：

**它试图把 agent 的工作过程转化成可复用的项目记忆，并在未来低成本恢复给模型。**

这是对的方向。

但系统真正的上限，不在 SQLite、MCP 或 provider，而在：

- observation 提炼质量
- summary 质量
- context 注入策略
- 长期记忆治理能力

所以最终判断是：

**这是一个方向正确、工程扎实、但 memory policy 还可以明显升级的系统。**
