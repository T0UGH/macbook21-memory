# HappyClaw 接入 Mailbox 的假设性实现方案（2026-03-21）

> 目标：讨论 **HappyClaw 如何把 mailbox 事件注入主会话**。这里不讨论当前是否立刻实现，只讨论：**如果现在要做，我会怎么改。**

---

## 前提共识

这份讨论建立在几个已经明确的前提上：

1. **独立会话方案不合适**
   - 因为看不到主会话上下文
   - 也和 OpenClaw heartbeat 的行为模型不一致

2. **目标必须是主会话注入**
   - mailbox 事件应进入 HappyClaw 的主会话消息流
   - 让 agent 在原有上下文里继续处理

3. **HappyClaw 已经有现成主会话消息处理链**
   - `storeMessageDirect(...)`
   - `broadcastNewMessage(...)`
   - `queue.sendMessage(...)`
   - `enqueueMessageCheck(...)`
   - `processGroupMessages(...)`

4. **问题不在“能不能处理消息”，而在“从哪里把 mailbox 事件接进去”**

---

## 设计目标

如果要接 mailbox，我会坚持下面 5 个目标：

1. **注入主会话，而不是新开会话**
2. **复用 HappyClaw 现有消息链路，而不是重写一套 mailbox 特供处理流**
3. **保留 mailbox source 标记，不能伪装成人类消息**
4. **改动尽量小，便于做薄 patch / 后续 upstream**
5. **把 mailbox 触发和消息注入分层，不把 mailbox 业务深度塞进 HappyClaw 核心**

---

# 方案 A：模拟合法 Web 客户端，通过现有 WebSocket `send_message` 注入

## 思路

不改 HappyClaw 核心，只做一个外部 mailbox adapter：

- scheduler 定时触发 mailbox tick
- 如果命中事项
- adapter 以一个合法 web client 身份连上 HappyClaw WebSocket
- 调用现有 `send_message`
- 让消息走现有主会话链路

## 数据流

```text
mailbox tick
  -> adapter decision
  -> websocket send_message(chatJid, content)
  -> handleWebUserMessage(...)
  -> store/broadcast/queue/processGroupMessages
  -> 主会话 agent 继续处理
```

## 优点

- **零改 HappyClaw 核心代码**
- 完全走现有前门
- 最容易快速验证可行性
- 适合先做 PoC

## 缺点

- 本质上是“模拟前端客户端”
- 依赖 session/auth 管理
- mailbox 注入语义会比较别扭
- source 标记可能不够干净
- 长期看工程气质一般

## 什么时候适合

适合：

- 需要最快验证
- 暂时不想维护 HappyClaw patch
- 接受这是过渡方案

## 我的判断

**可以作为 PoC，但不适合作为长期主方案。**

---

# 方案 B：在 HappyClaw 增加一个最小 internal injection API（推荐）

## 思路

在 HappyClaw fork 上补一个**很薄的本地入口**，比如：

- local-only HTTP endpoint
- internal route
- 或 mailbox-specific injection handler

这个入口不做复杂业务，只做一件事：

> 接收一条 mailbox injected message，然后复用现有主会话消息处理链。

## 数据流

```text
mailbox tick
  -> adapter decision
  -> POST /internal/mailbox/inject
  -> internal handler
  -> storeMessageDirect / broadcastNewMessage / queue.sendMessage / enqueueMessageCheck
  -> processGroupMessages
  -> 主会话 agent 继续处理
```

## 建议入口语义

### 示例请求

```json
{
  "chatJid": "web:main",
  "source": "mailbox",
  "sourceMeta": {
    "messageId": "msg_123",
    "reason": "open_obligation"
  },
  "content": "[Mailbox] 你收到一条需要处理的消息 ..."
}
```

### 核心要求

- 不伪装成人类消息
- 对前端/日志可见 source=mailbox
- 对 agent 来说它是一条真实可处理输入

## 优点

- **最符合工程直觉**
- 复用 HappyClaw 现有链路
- source 边界清晰
- 便于审计/调试
- patch 面可控
- 后续最容易 upstream

## 缺点

- 需要维护一个薄 patch / fork
- 不是零侵入
- 需要设计 local-only 安全边界

## 什么时候适合

适合：

- 想长期稳定跑 mailbox
- 愿意维护一个很小的 HappyClaw patch
- 未来希望找作者 upstream

## 我的判断

**这是我当前最推荐的主方案。**

---

# 方案 C：直接在 scheduler task 内部绕过前门，调用内部函数注入

## 思路

不新开 HTTP/local route，而是在 HappyClaw 内部 scheduler 执行 mailbox tick 后，直接调用内部函数，例如：

- 直接构造消息对象
- 直接调用 `storeMessageDirect(...)`
- 直接 `broadcastNewMessage(...)`
- 直接 `queue.sendMessage(...)` / `enqueueMessageCheck(...)`

也就是把 mailbox scheduler 和注入逻辑直接耦在一起。

## 数据流

```text
scheduler task
  -> mailbox tick
  -> direct internal injection
  -> store/broadcast/queue/processGroupMessages
```

## 优点

- 不需要额外入口协议
- 路径短
- 内部实现可能最快

## 缺点

- mailbox 和 HappyClaw scheduler 强耦合
- 改动面容易扩散
- 复用性差
- 不利于多 runtime 统一抽象
- upstream 时更难讲清楚边界

## 什么时候适合

适合：

- 只想做本机一次性实验
- 完全不考虑后续抽象和复用

## 我的判断

**不推荐作为正式方案。**

原因不是做不到，而是边界太脏。

---

# 方案 D：让 HappyClaw 只负责调度，真正注入走外部 sidecar + DB/WS 双写

## 思路

单独做一个 sidecar：

- HappyClaw scheduler 只负责周期调用 sidecar
- sidecar 负责 mailbox tick
- 然后 sidecar 通过更底层方式把消息写进 HappyClaw（DB + WS / API 等）

## 优点

- HappyClaw 主代码改动可能最少
- mailbox 逻辑基本独立

## 缺点

- 系统复杂度上升
- 容易引入“双写一致性”问题
- 比 B 更黑盒
- 调试更麻烦

## 我的判断

**不推荐当前阶段采用。**

这是典型为了少改主系统，引入更多系统复杂度。

---

# 方案对比

| 方案 | 是否改 HappyClaw | 是否复用现有主会话链路 | 是否适合长期 | 是否利于 upstream | 我的建议 |
|------|------------------|------------------------|--------------|-------------------|----------|
| A. WebSocket 模拟注入 | 否 | 是 | 一般 | 一般 | 可做 PoC |
| B. 最小 internal injection API | 是，少量 | 是 | **强** | **强** | **最推荐** |
| C. scheduler 内部直耦合 | 是，中等 | 是 | 弱 | 弱 | 不推荐 |
| D. sidecar + 底层双写 | 可能少 | 间接 | 弱 | 弱 | 不推荐 |

---

# 如果让我现在实现，我会怎么选

## 第一选择：B

### 我会这样做

1. **fork 一个 HappyClaw 仓**
2. 增加一个**极小的本地 mailbox injection 入口**
3. 这个入口只做：
   - 参数校验
   - source=mailbox 标记
   - 复用现有主会话消息链路
4. 不改：
   - group queue 核心语义
   - processGroupMessages 主逻辑
   - scheduler 主架构
5. mailbox tick adapter 作为外部组件存在，HappyClaw 只暴露“注入消息”能力

### 原因

这是最符合下面三点的方案：

- **主会话一致性**
- **工程边界清晰**
- **未来可 upstream**

---

## 第二选择：A（仅 PoC）

如果想最快验证，不想先动 HappyClaw fork，我会先用 A 做个短期 PoC。

但我会明确把它定义成：

> **验证注入模型的临时方案，不是最终架构。**

---

# 我不会怎么做

如果是我来主导，我不会：

1. **做独立 mailbox 会话**
   - 不符合上下文连续性要求

2. **让 mailbox 只变成通知系统**
   - 价值太低

3. **把 mailbox 深耦合进 HappyClaw scheduler 核心**
   - 未来维护很痛

4. **一开始就做重度 fork**
   - 成本高，且不必要

---

# Patch 边界建议

如果后续走方案 B，我会严格控制 patch 边界：

## 可以改的层

- 新增一个小 injection route / handler
- source=mailbox 的消息元数据定义
- 调用现有主会话消息链的 glue code

## 尽量不要改的层

- `group-queue.ts` 的核心状态机
- `processGroupMessages(...)` 的主行为
- 现有 scheduler 主循环
- agent conversation / 主会话整体模型

一句话：

> **在边缘加入口，不在核心改语义。**

---

# 最终判断

如果现在纯假设“让我来做”，我的排序会是：

1. **B：最小 internal injection API（主方案）**
2. **A：WebSocket 模拟注入（PoC/过渡）**
3. **C：scheduler 直耦合（不推荐）**
4. **D：sidecar + 双写（不推荐）**

最核心的判断不是“能不能把 mailbox 接进去”，而是：

> **要以最小改动，把 mailbox 事件送进 HappyClaw 已有的主会话消息处理链。**
