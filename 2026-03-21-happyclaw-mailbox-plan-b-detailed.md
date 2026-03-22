# HappyClaw 接入 Mailbox：方案 B 细化稿（最小 Internal Injection API）

> 目标：细化方案 B。前提是：**mailbox 事件必须注入 HappyClaw 主会话**，不能走独立会话；同时尽量保持改动最小，便于后续维护 patch 或 upstream。

---

## 1. 方案 B 的一句话定义

> **在 HappyClaw 中增加一个极小的本地 mailbox injection 入口，让外部 mailbox tick runner 可以把结构化 mailbox 事件注入主会话，并复用 HappyClaw 现有消息处理链。**

这里的关键词有 3 个：

- **本地入口**
- **主会话注入**
- **复用现有链路**

这不是重写 HappyClaw 的消息系统，也不是把 mailbox 深耦合进 HappyClaw 核心。

---

## 2. 为什么方案 B 是当前主方案

相比 A/C/D，B 的优势在于：

1. **行为模型最对**
   - mailbox 进入主会话
   - agent 在原上下文里继续工作
   - 与 OpenClaw heartbeat 的心智模型一致

2. **工程边界最干净**
   - mailbox 决策仍在外部 tick runner
   - HappyClaw 只负责“接受一条系统来源的消息并送进主会话”

3. **改动面最可控**
   - 不需要重写 scheduler / queue / processGroupMessages
   - 只是在边缘开一个入口

4. **后续最容易 upstream**
   - 因为这个入口本质上是一种通用能力：
     - 外部系统注入系统消息到主会话
     - 不仅 mailbox 可用，未来其他系统事件也可用

---

## 3. 方案 B 的核心设计原则

## 原则 1：入口只负责“注入”，不负责 mailbox 决策

HappyClaw 不应该知道：

- 什么叫 open obligation
- pick-mail 怎么排序
- 为什么这封信要进来

这些都应该在外部 mailbox tick runner 完成。

HappyClaw 入口只做：

- 接收一个要注入主会话的 payload
- 校验是否合法
- 走现有主会话消息链

## 原则 2：消息要有明确 source 标记

注入消息不能伪装成真人消息。

建议保留：

- `source = mailbox`
- 可选的 `source_meta`

这样前端、日志、排障、后续统计都清晰。

## 原则 3：复用现有消息链，不重造轮子

理想路径是复用现有这些能力：

- `storeMessageDirect(...)`
- `broadcastNewMessage(...)`
- `queue.sendMessage(...)`
- `enqueueMessageCheck(...)`
- `processGroupMessages(...)`

不要另写一套“mailbox 专用消息投递系统”。

## 原则 4：local-only / internal-first

这个入口应优先定义成：

- 本机调用
- 内部用途
- 非公开多租户 API

至少 Phase 1 不要把它设计成对互联网暴露的正式外部能力。

---

## 4. 建议的总体结构

```text
mailbox tick runner
  -> evaluate / decide
  -> POST local HappyClaw mailbox injection endpoint
  -> HappyClaw injection handler
      -> normalize mailbox-injected message
      -> storeMessageDirect(...)
      -> broadcastNewMessage(...)
      -> queue.sendMessage(...) or enqueueMessageCheck(...)
  -> 主会话 agent 继续处理
```

---

## 5. 入口设计

## 5.1 推荐形态

我建议做一个 **internal local HTTP endpoint**。

例如：

```text
POST /internal/mailbox/inject
```

为什么优先 HTTP 而不是别的：

- 边界清楚
- 调用方实现简单
- 便于调试和日志采样
- 比 WebSocket 模拟前端更稳
- 比 scheduler 内部直耦合更解耦

## 5.2 调用约束

建议只允许：

- `localhost`
- 或 Unix domain socket
- 或要求一个本地 shared secret/header

至少做到：

- 不是公网 API
- 不是普通用户前端入口

---

## 6. 请求/响应契约建议

## 6.1 请求体

建议最小请求体如下：

```json
{
  "chatJid": "web:main",
  "content": "[Mailbox] 你收到一条 open obligation，需要在当前上下文中处理。\n\nFrom: sender_agent\nSubject: ...\nSummary: ...",
  "source": "mailbox",
  "sourceMeta": {
    "mailbox_message_id": "msg_123",
    "reason_class": "open_obligation",
    "requires_reply": true,
    "injected_at": "2026-03-21T03:20:00Z"
  }
}
```

## 6.2 为什么字段要这么少

Phase 1 不要做太重：

- `chatJid`：注入到哪个主会话
- `content`：最终要送给 agent 的文本
- `source`：明确标记来源
- `sourceMeta`：保留排障与追踪所需信息

不要一开始就塞太多 mailbox 专用字段进 HappyClaw 内核。

## 6.3 响应体

建议返回：

```json
{
  "ok": true,
  "messageId": "local_msg_456",
  "delivery": "injected"
}
```

其中 `delivery` 可取：

- `injected`：已注入当前活跃 agent
- `queued`：当前没有活跃 agent，已排队等待处理
- `rejected`：目标会话不可注入

这个结果对外部 tick runner 很有用。

---

## 7. Handler 内部推荐行为

如果让我写，我会让 injection handler 按下面步骤执行：

### Step 1: 校验请求

校验：

- `chatJid` 必填
- `content` 非空
- `source` 必须是 `mailbox`
- `sourceMeta` 可选但建议保留

### Step 2: 做权限/目标校验

确认：

- 目标 `chatJid` 存在
- 是允许注入的主会话
- 不是 agent conversation 虚拟会话
- 不是非法/未知 jid

### Step 3: 持久化消息

调用等价于：

- `ensureChatExists(chatJid)`
- `storeMessageDirect(...)`

但要把 source 写进去，避免与普通用户消息混淆。

### Step 4: 广播前端事件

调用：

- `broadcastNewMessage(...)`

保证：

- Web UI 能看到这条 mailbox 注入消息
- source=mailbox 可见

### Step 5: 尝试注入当前活跃 agent

调用：

- `queue.sendMessage(...)`

如果当前有活跃处理器，就让 agent 在当前上下文里直接看到新输入。

### Step 6: 若无活跃 agent，则排队

调用：

- `enqueueMessageCheck(chatJid)`

确保消息不会丢，只是延后到下一轮处理。

---

## 8. 消息内容怎么组织

这是方案 B 很关键的一点。

## 8.1 不要伪装成真人自然语言闲聊

不要把它伪装成：

> 王贵平：你有一封邮件去看下

这会污染消息语义。

## 8.2 建议采用“系统可读、agent 可处理”的注入格式

例如：

```text
[Mailbox Injected Event]
Reason: open_obligation
From: sender_agent
Mailbox-Message-Id: msg_123

You have received a mailbox item that should be handled in the current main conversation context.

Summary:
- subject: xxx
- requires_reply: true
- note: ...

Please decide whether to open/process/reply based on the current conversation context.
```

### 为什么这样更好

- agent 能理解
- 不是普通人类消息
- 不会误导审计/日志
- 可持续演进

---

## 9. 最小 patch 应改哪些层

## 应改的层

### 1. route / handler 层

新增一个极小 internal endpoint。

### 2. source metadata 层

让 `new_message` / DB 记录里能保留 `source=mailbox`。

### 3. glue code 层

把 injection handler 接到现有主会话消息链。

## 尽量不要改的层

### 1. `group-queue.ts` 核心状态机

不要为了 mailbox 改其主语义。

### 2. `processGroupMessages(...)` 主流程

不要把 mailbox 特判深塞进去。

### 3. HappyClaw scheduler 主循环

scheduler 不是 mailbox 的归宿。

### 4. agent 会话模型

不要为了 mailbox 改动“主会话 vs agent conversation”基础模型。

---

## 10. 风险与对策

## 风险 1：source=mailbox 进入 DB/前端链路不够顺滑

### 对策

Phase 1 可以先做到：

- DB 里有标记
- WebSocket `new_message` 里有 `source`
- 前端先不做复杂 UI，只要别丢标记

## 风险 2：活跃 agent 注入和队列处理语义不一致

### 对策

复用现有：

- `queue.sendMessage(...)`
- `enqueueMessageCheck(...)`

不要自创新的投递逻辑。

## 风险 3：入口做成公网能力，边界失控

### 对策

Phase 1 明确为：

- local-only
- internal-only
- mailbox use-case first

## 风险 4：patch 逐渐扩散成重度 fork

### 对策

给 patch 设纪律：

- 只为“主会话注入 mailbox 事件”服务
- 一旦需要改核心行为，先停下来重新评估

---

## 11. 分阶段实施建议

## Phase 1：最小可用

目标：先跑通主会话注入。

交付：

- internal injection endpoint
- source=mailbox 标记
- store/broadcast/queue 主链打通
- 基础日志

## Phase 2：调试与可观测性

补：

- 注入成功/queued/rejected 统计
- mailbox source 前端展示优化
- 更清晰的 trace

## Phase 3：考虑 upstream-friendly 抽象

如果验证稳定，再考虑把这个入口抽象成更通用的：

- external system message injection capability
- 而不局限于 mailbox

---

## 12. 如果让我现在拍板

如果现在要做，我会明确这样定：

### 采用
- **方案 B**
- **internal local HTTP endpoint**
- **主会话注入**
- **source=mailbox 明确标记**
- **patch 尽量薄，准备未来 upstream**

### 不采用
- 独立 mailbox 会话
- 深耦合 scheduler
- sidecar 双写
- 伪装成人类消息

---

## 最终判断

方案 B 的关键不在于“新增一个接口”，而在于：

> **用最小的 HappyClaw patch，把外部 mailbox 决策结果送进它已有的主会话消息处理链。**

这是当前最符合：

- 主会话一致性
- OpenClaw/HappyClaw 行为一致性
- 最小维护成本
- 后续 upstream 可能性

这四个目标的方案。
