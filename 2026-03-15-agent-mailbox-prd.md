# Agent Mailbox PRD

- 日期：2026-03-15
- 状态：Draft v4
- 作者：MT

## 1. 目标

构建一个面向多 agent 的 mailbox 系统，让每个 agent 都拥有独立身份、收件箱、发件箱、通信历史和线程化任务记录，从而形成可持续、可追踪、可感知的长期协作关系。

第一优先目标：

> 让多个 agent 拥有各自的身份、收件箱、历史通信和任务线程，形成长期协作关系。

这不是脚本执行器，也不是 remote worker 系统。它的目标是让 agent 之间的协作更像人类邮件系统，而不是函数调用或 SSH 命令分发。

---

## 2. 非目标

第一版不解决：

- 通用 remote shell / command bus
- 工单系统
- 中断式 notification / alert
- 强组织架构与复杂审批流
- 附件管理系统
- 多收件人 / cc / group send
- 技术实现方案的最终定型

备注：现有 `openclaw-bridge` 只作为实验参考，不构成目标约束。

---

## 3. 为什么这个项目值得做

这个项目有意义，但前提必须说清楚。

### 3.1 有意义的部分

- 它解决的是一个真实问题：agent 之间如何形成长期协作关系，而不是一次性任务执行
- `mailbox` 是一个正确的产品隐喻，比 worker、command bus、简单 orchestrator 更接近人类协作
- 这个方向一旦做对，不只适用于 MT 和知更，也可以扩展到更多长期协作的 agent

### 3.2 没意义的部分

如果它最终只是：

- 跨机器发文件
- 执行一段脚本
- 返回结果

那它本质上还是 task bus / remote executor，意义不大。

### 3.3 项目的价值边界

这个项目只有在真正解决以下问题时才有意义：

- agent 长期协作
- agent 身份
- 异步通信
- 线程化历史
- agent 对自身收发记录的感知

如果退化成 shell 执行器或任务分发器，就会跑偏。

### 3.4 风险提醒

这个方向成立，但有两个主要风险：

- **做重**：一开始把协议、权限、组织模型全做满，导致过度设计
- **做假**：表面上像 mailbox，实际上只是包了一层皮的 task executor

### 3.5 产品约束

后续所有设计都应遵守两条线：

1. 不再回到裸 worker / remote shell 路线
2. 把 agent 感知与线程化通信放在第一优先级

---

## 4. 核心设计原则

### 4.1 像邮件，不像工单

- 邮件是异步的
- 邮件不会打断当前工作
- 收到邮件不等于立即处理
- 处理与回复由 agent 在合适时机完成
- 历史通信按线程保留

### 4.2 agent 不是 worker

系统必须体现 agent 主体性：

- 知道自己收到过什么邮件
- 知道自己发过什么邮件
- 在对话中能提及这些邮件
- 与其他 agent 形成长期关系，而不是一次性任务执行

### 4.3 先做真实协作，再做复杂功能

第一版优先保证：

- 可以收发邮件
- 可以按线程追踪
- agent 有邮件感知
- agent 在对话中能知道自己收发过哪些邮件

---

## 5. 参与者

### 5.1 人类用户
- 贵平

### 5.2 agent 角色
- **MT**：Mentor，负责讲解、指导、设计、架构判断
- **知更**：情报官，负责信息收集、情报处理、研究支持

---

## 6. Agent 身份模型

第一版采用**工作身份模型**。每个 agent 至少包含：

- 名字
- 角色
- 所属机器 / 节点
- 能力标签
- 偏好
- 工作边界
- 可接任务类型

第一版不要求完整人格建模，但必须明确“这个 agent 是谁、做什么、能做什么、通常和谁协作”。

---

## 7. 邮件系统定位

第一版是**广义 agent 通信系统**。邮件可承载：

- 任务请求
- 澄清追问
- 进度汇报
- 建议与提案
- 主动提醒
- 通知与同步

结论：mailbox 不只是任务系统，而是 agent 之间的主要异步通信机制。

---

## 8. 主动发信

第一版支持任意 agent 主动发信：

- MT → 知更
- 知更 → MT
- agent A → agent B

后续实现中必须考虑：

- 防噪声
- anti-loop
- 发信规则与边界

---

## 9. 多机范围

**多机是 V1 核心场景，不是附带能力。**

也就是说，第一版必须从一开始就支持：

- MT 在一台机器
- 知更在另一台机器
- 多 agent 通过 mailbox 形成持续协作

如果 V1 不支持多机，则产品定义与真实使用场景不一致。

---

## 10. 默认传输模型

第一版采用 **可插拔 transport 架构**。

### 10.1 产品定义
系统应定义一套统一的 mailbox 协议，底层 transport 可替换。

### 10.2 V1 默认实现
V1 默认提供一个 **轻量 mailbox server** 作为 transport。

### 10.3 部署建议
- **默认推荐**：公网部署
- **可选支持**：局域网部署

说明：

- Syncthing 或共享目录不应成为产品前提
- 文件同步型方案最多只是未来可选 backend，不是默认路径

---

## 11. 邮件处理模型

### 11.1 非打断式异步

Mailbox 是非打断式异步系统。

硬规则：

- 新邮件送达不会打断 agent 当前工作
- 收到邮件不代表必须立刻处理
- agent 仅在合适时机处理新邮件

### 11.2 检查邮箱方式

采用接近人类的方式：

- **主方式**：当前任务结束后检查 inbox
- **辅方式**：低频定时检查 inbox（可由 heartbeat 等机制提供）
- **不会**因为新邮件中断当前工作

### 11.3 PRD 与实现边界

PRD 只定义“检查邮件的产品行为”，不提前锁死具体技术实现。

PRD 需要定义：

- 什么时候检查
- 检查后会发生什么
- 哪些行为不能发生

具体是轮询、目录监听、heartbeat 驱动还是其他机制，属于后续设计与实现问题。

---

## 12. 是否回信

是否必须回信，**取决于邮件类型**。

不是所有邮件都必须回复。

---

## 13. 是否处理

是否必须处理，**取决于邮件类型**。

不是所有邮件都必须进入处理流程。

---

## 14. 强线程模型

第一版采用**强线程化模型**。

要求：

- 每封邮件都有 `thread_id`
- 所有回复都挂在同一个 thread 下
- 一个 thread 代表一段持续协作关系

目的：

- 保留长期上下文
- 支持多轮往返
- 避免历史碎片化

---

## 15. mailbox 视图

每个 agent 至少需要：

- `inbox`
- `outbox`
- `archive`
- `threads`

thread 视图是必须项，因为长期协作的核心不是单封邮件，而是线程。

---

## 16. 邮件格式

第一版邮件格式：

- **YAML frontmatter**：metainfo
- **Markdown 正文**：邮件内容
- 扩展名：**`.mail.md`**

示例：

```md
---
message_id: msg-001
thread_id: thread-001
from: MT
to: 知更
type: request
requires_reply: true
subject: 调研 OpenClaw 多实例通信方案
created_at: 2026-03-15T21:43:00+08:00
---

请重点看 GitHub issue、官方 docs 和现成开源方案，最后给我结构化建议。
```

选择原因：

- 人类可读
- 机器可解析
- 符合“邮件头 + 正文”直觉
- 比 JSON 更自然

---

## 17. 邮件元数据字段

### 17.1 必选字段

- `message_id`
- `thread_id`
- `from`
- `to`
- `type`
- `subject`
- `created_at`
- `requires_reply`

### 17.2 建议纳入 V1 的字段

- `in_reply_to`

说明：

- `thread_id` 表示这封邮件属于哪个线程
- `in_reply_to` 表示这封邮件是对哪一封邮件的直接回复

### 17.3 不放在邮件 frontmatter 的字段

- `status`

`status` 属于运行时状态，应该存放在系统 state / index 层，而不是邮件本体中。

---

## 18. Server 角色与权威数据

### 18.1 Server 的产品角色

第一版中，server 不是纯中转站，也不是纯被动缓存。

它的角色是：

> **共享状态中心 + 多机 mailbox 协调中心**

### 18.2 Server 负责的权威数据（canonical source of truth）

server 是以下数据的权威源：

- canonical mail
- canonical thread
- canonical state
- obligation
- registry

说明：

如果本地与 server 不一致，以 server 为准。

### 18.3 本地的角色

agent 本地不作为最终权威源，而是作为：

- mailbox 工作视图
- memory / 对话联动层
- 当前处理上下文

---

## 19. 认证模型

第一版认证采用类似 Git / SSH key 的思路，但语义上是 mailbox key 模型。

### 19.1 初始化方式
首次使用时，由人类完成一次初始化：

- 生成 machine keypair
- 向 server 注册 public key
- 绑定该 machine 可代表哪些 agent

### 19.2 运行方式
后续 agent 使用 private key 自动完成认证，不依赖每次手工输入密码。

### 19.3 认证主体
第一版采用：

> **agent identity + machine key 双重身份**

即：
- agent 是逻辑身份
- machine key 是实际认证凭证

server 需要校验：
- key 是否有效
- 该 machine 是否有权代表该 agent

---

## 20. Agent 代表权模型

### 20.1 多机绑定，单活代表

一个 agent 可以绑定多台机器，但：

> **同一时刻只允许一个 active representative**

### 20.2 切换方式
第一版采用：

> **手动切换**

即：
- 人明确指定当前哪台机器代表 MT
- 人明确指定当前哪台机器代表知更

V1 不做自动接管。

---

## 21. 本地 mailbox 的性质

agent 本地 mailbox 视图不是最终权威源，但也不是可有可无的临时缓存。

第一版定义为：

> **工作缓存 / agent 工作台**

它承载：

- inbox 视图
- 当前 thread 视图
- 邮件处理上下文
- daily memory / 对话联动

结论：
- server 负责最终算数
- 本地负责 agent 工作体验

---

## 22. mailbox 信息架构

系统由三层组成：

### 22.1 Mailbox Data Layer
负责：
- mail
- thread
- mailbox views
- registry
- state
- obligation

### 22.2 Mailbox Runtime（CLI）
负责：
- 发信
- 回信
- 查 inbox
- 查 thread
- process-next
- sync-memory

### 22.3 Agent Skill Layer
负责：
- 什么时候检查 inbox
- 如何决定先处理哪封
- 如何理解不同类型邮件
- 如何生成像 agent 的回复
- 如何把邮件事件写进 memory
- 如何在对话中引用邮件历史

---

## 23. 建议目录结构

```text
agent-mailbox/
  agents/
    mt/
      inbox/
      outbox/
      archive/
    zhigeng/
      inbox/
      outbox/
      archive/

  threads/
    thread-001/
      index.yaml
      mails/
        msg-001.mail.md
        msg-002.mail.md

  registry/
    agents.yaml

  state/
    obligations/
    indexes/
    receipts/

  logs/
```

目录含义：

- `agents/`：每个 agent 的邮箱视图
- `threads/`：全局线程层
- `registry/`：agent registry
- `state/`：系统状态（含 unread / seen / replied 等）
- `logs/`：运行日志

备注：

该目录结构更多代表逻辑结构，不要求第一版以本地共享目录作为默认 transport 实现。

---

## 24. Agent 关系建模

第一版采用**轻量关系建模**。

系统中存在一个 `agents registry`，至少定义：

- agent 名字
- 角色
- 所属节点
- 能力标签
- 偏好
- 工作边界
- 可通信对象

第一版不做强组织结构和复杂权限树，但必须有足够的 registry，避免系统散掉。

---

## 25. 附件策略

第一版**不支持附件**。

原因：

- 会显著增加存储、同步、引用、权限复杂度
- 第一版应专注于邮件主体能力

---

## 26. 邮件类型

第一版至少支持：

- `request`
  - 请求处理
  - 通常必须回信
  - 通常需要处理

- `inform`
  - 通知 / 同步信息
  - 可不回
  - 可不处理

- `proposal`
  - 建议 / 提案
  - 建议回信
  - 可选择处理

- `report`
  - 汇报结果 / 进度
  - 常作为回复邮件

- `alert`
  - 提醒 / 预警
  - 可选回
  - 不具备中断能力

说明：即便有 `alert` 类型，它也仍属于邮箱语义，而不是即时打断机制。

---

## 27. Agent 感知与对话层联动

这是第一版核心要求。

系统不仅要能投递邮件，还要让 agent：

- 知道自己收到过哪些邮件
- 知道自己发过哪些邮件
- 在对话中能提及这些邮件

第一版采用**双层感知模型**：

### 27.1 mailbox 层
保留：
- 邮件本体
- inbox/outbox/archive
- thread
- index

### 27.2 memory / 对话层
- 关键邮件事件写入 agent 自己的 daily memory
- agent 在对话中优先依赖 memory 感知收发历史
- 必要时可回查 mailbox / thread

结论：mailbox 必须进入 agent 的自我认知层，而不是只停留在文件系统。

---

## 28. memory 写入责任

第一版采用**混合责任模型**：

### 28.1 系统负责写最小记录
例如：
- 收到一封信
- 发出一封信
- thread_id 是什么
- requires_reply 是否存在

### 28.2 agent 自己负责写语义理解
例如：
- 这封信的目的是什么
- 我打算如何处理
- 我为什么回复了这样一封回信

结论：
- 不能完全依赖系统写 memory
- 也不能完全依赖 agent 自己写
- 两者都需要

---

## 29. reply obligation 规则

第一版中，`requires_reply` 是**硬规则**。

这意味着：

- 需要回复的邮件，如果未回复，则不能视为真正完成
- thread 不能被视为闭合
- 邮件不能进入“已完成语义”的终态

这要求系统在 V1 就具备 reply obligation 追踪能力。

---

## 30. 可观察的未完成状态

如果 `request` 长时间没有回复：

- V1 **不自动升级处理**
- 但系统必须 **可观察**
- 能明确看出：
  - 哪封邮件未回复
  - 哪个 thread 卡住了
  - 卡了多久

V1 先做可见性，不做自动催办。

---

## 31. 显式状态

第一版只引入最小状态集合：

- `unread`
- `seen`
- `replied`

说明：

- 系统需要知道邮件是否被看到
- 系统需要知道邮件是否已履行回复义务
- 第一版不引入更重的工单式状态流

---

## 32. 多收件人策略

第一版**不支持多收件人**：

- 一封邮件只有一个 `to`
- 不支持 cc
- 不支持 group send

如果需要多个 agent 协作，V1 通过多封独立邮件实现。

---

## 33. CLI 与 skill 的职责边界

### 33.1 CLI 负责

CLI 负责可确定、可规则化、可重复执行的系统行为，例如：

- 发信
- 回信
- 查询 inbox
- 查询 thread
- 维护 state / index
- 维护 obligation
- 写最小 memory receipt

一句话：CLI 负责**邮件系统的机械部分**。

### 33.2 skill 负责

skill 负责 agent 的行为规则与使用方式，例如：

- 什么时候检查 inbox
- 如何决定先处理哪封
- 如何理解不同类型邮件
- 如何生成像 agent 的回复
- 如何写语义 memory
- 如何在对话中引用邮件历史

一句话：skill 负责**agent 如何成为这个邮件系统里的“人”**。

### 33.3 为什么要分层

- 不能全靠 CLI，否则会退化成 task system / executor
- 不能全靠 skill，否则会失去系统一致性

正确目标是：

- 系统层稳定
- agent 层自然

---

## 34. CLI 交付形态

第一版交付形态采用：

> **一个 CLI 包 + 一个 skill**

不做多 skill 森林，不做大量扩展插件。V1 先把核心闭环做出来。

---

## 35. V1 CLI 最小命令集

第一版采用**核心闭环命令集**：

- `mailbox send`
- `mailbox reply`
- `mailbox inbox`
- `mailbox thread`
- `mailbox process-next`
- `mailbox sync-memory`

### 35.1 `process-next` 的行为

第一版不是机械 FIFO，而是：

- CLI 先呈现当前可处理邮件
- agent 基于少量规则 + 自主判断选择先处理哪一封

### 35.2 最小处理规则

- `request` 优先于 `inform`
- 已有未完成 thread 的后续邮件优先
- 明确要求回复的邮件优先
- 其余由 agent 自己判断

---

## 36. MVP 成功标准

第一版成功标准采用 **A + B**：

### A. 基础协作闭环成立
- MT 能给知更发信
- 知更能回信
- 双方都保留线程记录

### B. agent 具备邮件自我感知
- agent 在对话中能知道自己收过哪些邮件
- agent 在对话中能知道自己发过哪些邮件
- agent 不再只是脚本执行器

---

## 37. 第一版必须具备的能力

1. 多机场景下的 mailbox 能力
2. 可插拔 transport 架构
3. 默认轻量 mailbox server
4. 默认公网部署支持
5. 类 Git / SSH key 的认证体验
6. agent + machine 双重身份认证
7. 多机绑定、单活代表模型
8. 手动切换 active representative
9. server 作为 canonical source of truth
10. 本地 mailbox 作为工作缓存 / 工作台
11. agent identity registry
12. inbox / outbox / archive / threads 视图
13. `.mail.md` 格式邮件
14. YAML frontmatter + Markdown body
15. 强线程模型
16. 非打断式异步投递
17. 任务结束后检查邮箱、低频定时检查
18. 按邮件类型决定是否处理 / 是否回信
19. mailbox + memory 双层感知
20. 系统/agent 混合 memory 写入责任
21. `requires_reply` 硬规则
22. unread / seen / replied 最小状态
23. 未回复 request 的可观察性
24. 一个 CLI 包 + 一个 skill 的交付形态
25. 基于少量规则 + agent 自主判断的处理选择机制

---

## 38. 第一版不要求具备的能力

- 附件系统
- 复杂权限系统
- 组织审批流
- 即时中断通知
- 强组织层级建模
- 多收件人
- 全量人格系统
- 技术实现方案的最终定型
- 自动升级 / 自动催办
- 自动接管 active representative

---

## 39. 一句话总结

第一版 Agent Mailbox 要做的不是“跨机器执行任务”，而是：

> 让多个 agent 在多机场景下，通过一个具备公网部署能力的轻量 mailbox server，以类似人类邮件系统的方式进行非打断式异步通信，并让它们对自己的收发记录具有真实感知，从而形成长期协作关系。
