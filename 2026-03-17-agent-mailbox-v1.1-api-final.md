# Agent Mailbox V1.1 API Final

- 日期：2026-03-17
- 状态：Final Draft for Implementation
- 作者：MT
- 输入来源：
  - `docs/2026-03-15-agent-mailbox-prd.md`
  - `docs/2026-03-16-agent-mailbox-implementation-design.md`
  - `docs/2026-03-17-agent-mailbox-v1-api-design.md`
  - `docs/2026-03-17-agent-mailbox-v1-api-review.md`
  - `docs/2026-03-17-agent-mailbox-v1-api-review-claude.md`

---

## 1. 结论与本版收口原则

这份文档不是重新发明 Agent Mailbox，而是把已有 PRD、数据设计、两轮 review 收口成一版**可以进入实现**的 API 定稿。

本版核心收口如下：

1. **GET 一律纯读，不带副作用**
   - 去掉 `GET /messages/{id}` 自动触发 `seen`
   - `seen` 只能通过显式动作接口触发

2. **`seen` 只允许 recipient 触发**
   - sender 不允许 mark seen
   - 彻底避免无意义 seen 事件污染 event stream

3. **obligation 清除路径唯一化**
   - 只有 `POST /v1/messages/{message_id}/reply` 才可能清除 obligation
   - `POST /v1/messages` 即使携带 `thread_id`，也不会清除 obligation

4. **明确 `submitted` 不进入 server canonical event stream**
   - `submitted` 属于本地 runtime 事件
   - server canonical event 只从 `accepted` 开始

5. **补齐多机恢复闭环 API**
   - 增加 representative heartbeat
   - 增加 delivery status 查询
   - 增加 agent sync status 查询

6. **thread 视图优先承担导航职责，不承担已读判定职责**
   - thread message list 可返回 body，但不代表进入处理视野
   - 是否真正 `seen` 仍由显式动作接口决定

一句话：

> V1.1 的目标不是最优雅，而是语义干净、实现闭环、后续不长歪。

---

## 2. 设计目标与范围

### 2.1 目标

为 mailbox CLI 与 agent runtime 提供一套稳定的 HTTP JSON API，支持：

- 发信 / 回信
- inbox / outbox / archive 工作视图
- thread 导航与历史查看
- obligation 可观察性
- delivered / seen / archive 等动作上报
- representative 多机切换下的恢复与观察

### 2.2 不做的事

V1.1 仍明确不做：

- 多收件人 / cc
- 附件系统
- 公网认证与复杂权限系统
- thread 显式 closed 状态
- obligation 人工 override
- 自动催办 / SLA 升级

---

## 3. 领域语义定稿

### 3.1 生命周期语义

#### `submitted`
- 含义：本地 agent/CLI 已提交一封待发送邮件
- **归属：本地 runtime state，不进入 server canonical event stream**
- 用途：支撑 `outbound_messages`、失败重试、`retry-sync`

#### `accepted`
- 含义：server 已接收并写入 canonical store
- 是 server 侧第一条 mail lifecycle 事实

#### `delivered`
- 含义：目标 agent 的 active representative 已将该邮件成功拉入本地工作缓存
- 不是 server 推断，而是 recipient 主动上报

#### `seen`
- 含义：目标 agent 真正将该邮件纳入当前处理视野
- 不是浏览 inbox，不是仅仅读到 thread 列表
- **只能由 recipient 显式上报**

#### `replied`
- 含义：目标 agent 发出一封有效回信，并挂入同一 thread
- `replied` 是 event 事实
- obligation 是否清除，还要看 reply 是否满足 obligation 规则

### 3.2 自然闭合

V1.1 不提供 `thread_status=closed` 或显式 `closed event`。

API 层只提供：
- `has_open_obligation`

这表示：
- `has_open_obligation=false` = 当前没有强制跟进项
- **不等于** thread 已被系统正式认定为“完成”

也就是说，V1.1 里 thread 是否“自然闭合”是产品解释，不是 API 的显式状态。

### 3.3 archive 语义

`archive` 只表示：
- 某 agent 在自己的 mailbox 工作视图里，把这封邮件归档

它**不表示**：
- obligation 已清除
- thread 已结束
- 邮件已完成处理

---

## 4. 核心资源模型

## 4.1 Message

```json
{
  "message_id": "msg_01hxabcdefghijklmn",
  "thread_id": "th_01hxabcdefghijklmn",
  "from": "mt",
  "to": "zhigeng",
  "type": "request",
  "subject": "调研 OpenClaw 多实例通信方案",
  "body": "请重点看 GitHub issue、官方 docs 和现成开源方案，最后给我结构化建议。",
  "requires_reply": true,
  "in_reply_to": null,
  "accepted_at": "2026-03-17T09:00:00+08:00",
  "created_at": "2026-03-17T09:00:00+08:00"
}
```

约束：
- `type` 仅允许：`request` / `inform` / `proposal` / `report` / `alert`
- 单收件人，`to` 只能有一个
- `from != to`

## 4.2 MessageState

```json
{
  "message_id": "msg_01hxabcdefghijklmn",
  "agent_id": "zhigeng",
  "mailbox_role": "recipient",
  "is_unread": true,
  "seen_at": null,
  "replied_at": null,
  "is_archived": false,
  "archived_at": null
}
```

约束：
- 采用 per-message per-agent projection
- sender / recipient 各有一条 state
- 但 `seen_at` 只有 recipient 侧有业务意义

## 4.3 Thread

```json
{
  "thread_id": "th_01hxabcdefghijklmn",
  "subject": "调研 OpenClaw 多实例通信方案",
  "created_by": "mt",
  "participants": ["mt", "zhigeng"],
  "opened_at": "2026-03-15T21:43:00+08:00",
  "last_message_at": "2026-03-16T10:00:00+08:00",
  "last_message_id": "msg_01hxabcdefghijklmn",
  "message_count": 3,
  "has_open_obligation": true,
  "is_archived": false
}
```

说明：
- `participants` 是 **derived view**
- 计算规则：thread 中所有 message 的 `from ∪ to`
- V1.1 不把它作为独立权威写入模型

## 4.4 Obligation

```json
{
  "obligation_id": "obl_01hxabcdefghijklmn",
  "thread_id": "th_01hxabcdefghijklmn",
  "message_id": "msg_01hxabcdefghijklmn",
  "owner_agent_id": "zhigeng",
  "status": "open",
  "opened_at": "2026-03-15T21:43:05+08:00",
  "cleared_at": null,
  "cleared_by_message_id": null,
  "age_seconds": 86400
}
```

## 4.5 Event

```json
{
  "event_id": "evt_01hxabcdefghijklmn",
  "thread_id": "th_01hxabcdefghijklmn",
  "message_id": "msg_01hxabcdefghijklmn",
  "event_type": "seen",
  "actor_type": "agent",
  "actor_id": "zhigeng",
  "target_agent_id": "zhigeng",
  "occurred_at": "2026-03-16T09:00:00+08:00",
  "metadata": {}
}
```

V1.1 canonical event type：
- mail events：`accepted` / `delivered` / `seen` / `replied`
- thread events：`opened` / `obligation-open` / `obligation-cleared`

**注意：`submitted` 不在 server event stream 中。**

## 4.6 Representative

```json
{
  "representative_id": "rep_01hxabcdefghijklmn",
  "agent_id": "mt",
  "machine_id": "macbook-pro-m3",
  "machine_label": "MacBook Pro M3",
  "is_active": true,
  "activated_at": "2026-03-17T08:00:00+08:00",
  "deactivated_at": null,
  "last_heartbeat_at": "2026-03-17T09:30:00+08:00"
}
```

## 4.7 SyncStatus

```json
{
  "agent_id": "mt",
  "machine_id": "macbook-pro-m3",
  "last_pulled_at": "2026-03-17T08:59:00+08:00",
  "last_seen_event_id": "evt_01hxevent999",
  "last_sync_status": "ok",
  "updated_at": "2026-03-17T08:59:00+08:00"
}
```

---

## 5. API 清单

| 方法 | 路径 | 说明 |
|---|---|---|
| POST | `/v1/messages` | 发送邮件 |
| POST | `/v1/messages/{message_id}/reply` | 回复指定邮件 |
| GET | `/v1/messages/{message_id}` | 获取邮件详情（纯读） |
| POST | `/v1/messages/{message_id}/seen` | 显式标记 seen |
| POST | `/v1/messages/{message_id}/delivered` | 上报 delivered |
| GET | `/v1/messages/{message_id}/delivery-status` | 查询某 agent 的 delivery 状态 |
| GET | `/v1/agents/{agent_id}/inbox` | 收件箱 |
| GET | `/v1/agents/{agent_id}/outbox` | 发件箱 |
| GET | `/v1/agents/{agent_id}/archive` | 归档箱 |
| POST | `/v1/agents/{agent_id}/messages/{message_id}/archive` | 归档 |
| POST | `/v1/agents/{agent_id}/messages/{message_id}/unarchive` | 取消归档 |
| GET | `/v1/agents/{agent_id}/pick-mail` | 候选导航 |
| GET | `/v1/threads` | 线程列表 |
| GET | `/v1/threads/{thread_id}` | 线程详情 |
| GET | `/v1/threads/{thread_id}/messages` | 线程消息列表 |
| GET | `/v1/threads/{thread_id}/events` | 线程事件流 |
| GET | `/v1/obligations` | obligation 列表 |
| GET | `/v1/obligations/{obligation_id}` | obligation 详情 |
| GET | `/v1/agents` | agent 列表 |
| GET | `/v1/agents/{agent_id}` | agent 详情 |
| GET | `/v1/agents/{agent_id}/representatives` | representative 列表 |
| POST | `/v1/agents/{agent_id}/representatives/{rep_id}/activate` | 激活 representative |
| POST | `/v1/agents/{agent_id}/representatives/{rep_id}/heartbeat` | representative 心跳 |
| GET | `/v1/agents/{agent_id}/sync-status` | agent 同步状态 |

---

## 6. 身份传递统一规则

所有带身份语义的写入 / 动作接口，统一使用：

```http
X-Agent-Id: zhigeng
```

V1.1 不在 body 中重复传 `agent_id` 作为身份来源。

规则：
- header 是唯一调用方身份输入
- path 中的 `agent_id` 用于资源定位
- server 要校验 header 身份与路径身份是否匹配（在需要匹配的接口上）

---

## 7. 接口定稿

## 7.1 `POST /v1/messages`

用途：
- 发送一封新邮件
- 可新建 thread
- 也可在已有 thread 中插入一封**非 obligation reply** 的新消息

**请求**

```json
{
  "from": "mt",
  "to": "zhigeng",
  "type": "request",
  "subject": "调研 OpenClaw 多实例通信方案",
  "body": "请重点看 GitHub issue、官方 docs 和现成开源方案，最后给我结构化建议。",
  "requires_reply": true,
  "thread_id": null,
  "in_reply_to": null,
  "idempotency_key": "cli-mt-20260317-001"
}
```

规则：
- `X-Agent-Id` 必须等于 `from`
- `thread_id = null` 时：server 创建新 thread
- `thread_id != null` 时：消息进入已有 thread
- **即使提供 `thread_id`，本接口也不承担 obligation 清除语义**
- 若 V1.1 实现希望更严格，可额外限制：`thread_id != null` 且 `in_reply_to == null` 只允许用于“thread 内新 request / inform / proposal”，由服务端校验

**响应** `201 Created`

```json
{
  "message": {
    "message_id": "msg_01hxabcdefghijklmn",
    "thread_id": "th_01hxabcdefghijklmn",
    "from": "mt",
    "to": "zhigeng",
    "type": "request",
    "subject": "调研 OpenClaw 多实例通信方案",
    "body": "请重点看 GitHub issue、官方 docs 和现成开源方案，最后给我结构化建议。",
    "requires_reply": true,
    "in_reply_to": null,
    "accepted_at": "2026-03-17T09:00:00+08:00",
    "created_at": "2026-03-17T09:00:00+08:00"
  },
  "thread": {
    "thread_id": "th_01hxabcdefghijklmn",
    "subject": "调研 OpenClaw 多实例通信方案",
    "created_by": "mt",
    "opened_at": "2026-03-17T09:00:00+08:00",
    "has_open_obligation": true
  },
  "obligation": {
    "obligation_id": "obl_01hxabcdefghijklmn",
    "owner_agent_id": "zhigeng",
    "status": "open",
    "opened_at": "2026-03-17T09:00:00+08:00"
  }
}
```

---

## 7.2 `POST /v1/messages/{message_id}/reply`

用途：
- 对一封已有邮件做正式 reply
- **这是唯一可能清除 obligation 的入口**

**请求**

```json
{
  "type": "report",
  "body": "调研完成，以下是结构化建议……",
  "requires_reply": false,
  "idempotency_key": "cli-zhigeng-20260318-001"
}
```

规则：
- `X-Agent-Id` 必须是 thread participant
- 若想清除 obligation，则还必须满足：
  - `X-Agent-Id` 是 obligation owner
  - server 自动设置 `in_reply_to = {message_id}`
  - reply type ∈ `report | proposal | request`
- 即：**能 reply** 与 **能清 obligation** 是两回事

**响应** `201 Created`

```json
{
  "message": {
    "message_id": "msg_01hxreplyaabbccdd",
    "thread_id": "th_01hxabcdefghijklmn",
    "from": "zhigeng",
    "to": "mt",
    "type": "report",
    "subject": "Re: 调研 OpenClaw 多实例通信方案",
    "body": "调研完成，以下是结构化建议……",
    "requires_reply": false,
    "in_reply_to": "msg_01hxabcdefghijklmn",
    "accepted_at": "2026-03-18T10:00:00+08:00",
    "created_at": "2026-03-18T10:00:00+08:00"
  },
  "obligation_status": "cleared",
  "obligation_cleared": {
    "obligation_id": "obl_01hxabcdefghijklmn",
    "status": "cleared",
    "cleared_at": "2026-03-18T10:00:00+08:00",
    "cleared_by_message_id": "msg_01hxreplyaabbccdd"
  },
  "obligation_opened": null
}
```

`obligation_status` 取值：
- `cleared`
- `still_open`
- `no_obligation`

这样避免 `obligation_cleared=null` 的二义性。

---

## 7.3 `GET /v1/messages/{message_id}`

用途：
- 获取邮件详情
- **纯读，不触发 seen**

**请求**

```http
GET /v1/messages/msg_01hxabcdefghijklmn
X-Agent-Id: zhigeng
```

**响应**

```json
{
  "message": {
    "message_id": "msg_01hxabcdefghijklmn",
    "thread_id": "th_01hxabcdefghijklmn",
    "from": "mt",
    "to": "zhigeng",
    "type": "request",
    "subject": "调研 OpenClaw 多实例通信方案",
    "body": "请重点看 GitHub issue、官方 docs 和现成开源方案，最后给我结构化建议。",
    "requires_reply": true,
    "in_reply_to": null,
    "accepted_at": "2026-03-17T09:00:00+08:00",
    "created_at": "2026-03-17T09:00:00+08:00"
  },
  "state": {
    "message_id": "msg_01hxabcdefghijklmn",
    "agent_id": "zhigeng",
    "mailbox_role": "recipient",
    "is_unread": true,
    "seen_at": null,
    "replied_at": null,
    "is_archived": false
  },
  "obligation": {
    "obligation_id": "obl_01hxabcdefghijklmn",
    "status": "open",
    "opened_at": "2026-03-17T09:00:00+08:00",
    "age_seconds": 7200
  }
}
```

---

## 7.4 `POST /v1/messages/{message_id}/seen`

用途：
- 显式标记 seen
- retry 场景的唯一官方入口

规则：
- `X-Agent-Id` 必须等于该 message 的 `to`
- sender 不允许 mark seen
- 幂等：只记录第一次 seen

**响应**

```json
{
  "message_id": "msg_01hxabcdefghijklmn",
  "agent_id": "zhigeng",
  "seen_at": "2026-03-17T11:00:00+08:00",
  "already_seen": false
}
```

---

## 7.5 `POST /v1/messages/{message_id}/delivered`

用途：
- recipient 在成功拉到本地工作缓存后显式上报 delivered

规则：
- `X-Agent-Id` 必须等于该 message 的 `to`
- 幂等：只记录第一次 delivered

**请求**

```json
{
  "machine_id": "thinkpad-x1",
  "delivered_at": "2026-03-17T10:00:00+08:00"
}
```

**响应**

```json
{
  "message_id": "msg_01hxabcdefghijklmn",
  "agent_id": "zhigeng",
  "delivered_at": "2026-03-17T10:00:00+08:00",
  "already_delivered": false
}
```

---

## 7.6 `GET /v1/messages/{message_id}/delivery-status?agent_id=zhigeng`

用途：
- 供 representative 切换、恢复、重试时查询某封邮件对某 agent 的 delivery 状态

**响应**

```json
{
  "message_id": "msg_01hxabcdefghijklmn",
  "agent_id": "zhigeng",
  "delivery": {
    "status": "delivered",
    "delivered_at": "2026-03-17T10:00:00+08:00",
    "representative_machine_id": "thinkpad-x1",
    "attempt_count": 1,
    "last_attempt_at": "2026-03-17T10:00:00+08:00"
  }
}
```

`status` 取值：
- `accepted`
- `delivered`
- `failed`

---

## 7.7 Inbox / Outbox / Archive

### `GET /v1/agents/{agent_id}/inbox`
- 纯读
- 不返回 body
- 不触发 seen

### `GET /v1/agents/{agent_id}/outbox`
- 返回 sender 视角的发件记录
- 可带 `delivery_status`

### `GET /v1/agents/{agent_id}/archive`
- 返回该 agent 归档过的 message

### `POST /v1/agents/{agent_id}/messages/{message_id}/archive`
### `POST /v1/agents/{agent_id}/messages/{message_id}/unarchive`

规则：
- `X-Agent-Id` 必须等于路径中的 `agent_id`
- per-agent 操作，不影响其他 agent
- 不影响 obligation / thread 状态

---

## 7.8 `GET /v1/agents/{agent_id}/pick-mail`

用途：
- 提供候选邮件导航
- **无 side-effect**

排序规则：
1. open obligation 且 unread
2. 同 thread continuation
3. 其余未读按新近性

这仍由 server 负责，以保证多机一致性。

---

## 7.9 Thread APIs

### `GET /v1/threads`
- 线程列表
- 支持 `participant` / `has_open_obligation` / `created_by` 筛选

### `GET /v1/threads/{thread_id}`
- thread 摘要详情
- 返回 `open_obligations`

### `GET /v1/threads/{thread_id}/messages`
- 返回 thread 内消息列表
- 可返回 `body`
- **但不触发 seen**

这里明确约定：
- 它是 thread 阅读 / 历史查看接口
- 不是 seen 判定接口
- 真正纳入处理视野时，CLI 必须显式调用 `/seen`

### `GET /v1/threads/{thread_id}/events`
- 只读事件流
- 不包含 `submitted`

---

## 7.10 Obligation APIs

### `GET /v1/obligations`
- operator / agent 观察“哪些 request 卡住了”的核心接口

### `GET /v1/obligations/{obligation_id}`
- obligation 详情
- 包含 source message 摘要

---

## 7.11 Representative / Sync APIs

### `GET /v1/agents/{agent_id}/representatives`
- 查询 representative 列表

### `POST /v1/agents/{agent_id}/representatives/{rep_id}/activate`
- 原子切换 active representative
- server 保证无双活窗口

### `POST /v1/agents/{agent_id}/representatives/{rep_id}/heartbeat`

用途：
- active representative 周期性上报心跳
- 更新 `last_heartbeat_at`

**请求**

```json
{
  "machine_id": "macbook-pro-m3",
  "heartbeat_at": "2026-03-17T09:30:00+08:00",
  "status": "ok"
}
```

**响应**

```json
{
  "agent_id": "mt",
  "representative_id": "rep_01hxabcdefghijklmn",
  "last_heartbeat_at": "2026-03-17T09:30:00+08:00",
  "accepted": true
}
```

### `GET /v1/agents/{agent_id}/sync-status`

用途：
- 返回 agent 当前同步状态
- 支撑 operator 看“最近同步时间 / 最近 event checkpoint / 当前同步是否正常”

**响应**

```json
{
  "agent_id": "mt",
  "sync": [
    {
      "machine_id": "macbook-pro-m3",
      "last_pulled_at": "2026-03-17T08:59:00+08:00",
      "last_seen_event_id": "evt_01hxevent999",
      "last_sync_status": "ok",
      "updated_at": "2026-03-17T08:59:00+08:00"
    }
  ]
}
```

---

## 8. obligation 规则定稿

### obligation 开启
- 一封 `requires_reply=true` 的 message 在 `accepted` 时开启 obligation
- obligation 绑定到该 `message_id`

### obligation 清除
只有满足以下条件，reply 才能清除 obligation：
- 通过 `POST /v1/messages/{message_id}/reply`
- 回信位于同一 thread
- server 自动设置 `in_reply_to = 原 obligation message_id`
- reply actor 是 obligation owner
- reply type ∈ `report | proposal | request`

### `inform` / `alert`
- 可以作为正常 reply 发出
- 但**不会清除 obligation**

### `request` reply `request`
- 原 obligation 清除
- 新 request 开启新 obligation

---

## 9. 权限边界定稿

V1.1 仍不做复杂鉴权，但定义清楚语义权限。

### 9.1 角色
- `participant`：message/thread 的参与方
- `operator`：系统观察者
- `system`：内部 actor

### 9.2 读权限
- message detail / thread messages：participant + operator 可读
- inbox / outbox / archive：对应 agent 自己 + operator 可读
- obligations：owner + operator 可读；operator 可做全局列表

### 9.3 写权限
- 发信：`X-Agent-Id == from`
- reply：`X-Agent-Id` 必须是 thread participant
- seen：`X-Agent-Id == to`
- delivered：`X-Agent-Id == to`
- archive/unarchive：`X-Agent-Id == path.agent_id`
- activate/heartbeat：由 operator 或当前 representative 调用

---

## 10. 分页、筛选、排序

### 分页
统一 cursor-based pagination：
- `cursor`
- `limit`（默认 20，最大 100）

### 默认排序
- inbox：`accepted_at_desc`
- outbox：`accepted_at_desc`
- threads：`last_message_at_desc`
- obligations：`opened_at_asc`
- thread messages：`accepted_at_asc`
- events：`occurred_at_asc`

---

## 11. 幂等性

### 写入类
`POST /v1/messages`
`POST /v1/messages/{id}/reply`

通过 `idempotency_key` 去重：
- 相同 key + 相同 payload：返回原结果
- 相同 key + 不同 payload：`409 IDEMPOTENCY_CONFLICT`

### 动作类
- `seen`
- `delivered`
- `archive`
- `unarchive`
- `activate`
- `heartbeat`

均应设计为幂等。

---

## 12. 错误码

```json
{
  "error": {
    "code": "MESSAGE_NOT_FOUND",
    "message": "Message msg_xxx does not exist.",
    "details": {}
  }
}
```

关键错误码：
- `AGENT_NOT_FOUND`
- `MESSAGE_NOT_FOUND`
- `THREAD_NOT_FOUND`
- `OBLIGATION_NOT_FOUND`
- `REPRESENTATIVE_NOT_FOUND`
- `INVALID_MESSAGE_TYPE`
- `INVALID_REPLY_TARGET`
- `SELF_SEND_NOT_ALLOWED`
- `SENDER_MISMATCH`
- `AGENT_ID_REQUIRED`
- `IDEMPOTENCY_CONFLICT`
- `SEEN_NOT_ALLOWED_FOR_SENDER`
- `ONLY_RECIPIENT_CAN_DELIVER`
- `ONLY_REPLY_CAN_CLEAR_OBLIGATION`
- `NOT_THREAD_PARTICIPANT`

---

## 13. 实现优先级建议

### P0 必做
- `messages`
- `reply`
- `inbox/outbox/archive`
- `message detail`
- `seen`
- `delivered`
- `threads`
- `obligations`
- `delivery-status`
- `representative activate`
- `heartbeat`
- `sync-status`

### P1 可后补
- thread messages body 是否降为 snippet
- operator 更细粒度权限
- 更完整的 dashboard projection

---

## 14. 最终判断

这版 V1.1 已经把前两轮 review 里最关键的语义问题收口：
- `submitted` 的归属
- `seen` 的唯一入口与 recipient 限制
- obligation 清除路径唯一化
- 多机恢复 API 补齐
- 身份传递规则统一

因此这版文档可以作为：

> **Agent Mailbox API 的实现基线文档**

直接进入服务端与 CLI 的实际实现设计。