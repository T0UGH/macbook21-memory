# Agent Mailbox V1 CLI Commands & Local Runtime Schema

- 日期：2026-03-17
- 状态：Implementation Design
- 作者：Claude（基于 MT 既有设计收口）
- 风格目标：Linus-style，简单、显式、可组合、不藏魔法
- 输入来源：
  - `docs/2026-03-15-agent-mailbox-prd.md`
  - `docs/2026-03-16-agent-mailbox-implementation-design.md`
  - `docs/2026-03-17-agent-mailbox-v1.1-api-final.md`
  - `docs/2026-03-17-agent-mailbox-cli-local-runtime-design.md`
  - `docs/2026-03-17-agent-mailbox-v1-api-review.md`
  - `docs/2026-03-17-agent-mailbox-v1-api-review-claude.md`

---

## 1. 设计边界

本文档不重新定义产品目标，只在已有设计基础上细化以下内容：

- 完整 CLI 命令清单（参数、I/O、side-effect）
- 各核心命令的 I/O 示例
- 默认输出规范（JSON 约定）
- 默认输入规范（flags / stdin / body-file / edit fallback）
- 本地 runtime schema（每台机器一个 SQLite）
- 本地状态与 server canonical state 的边界
- sync / retry / recovery 状态流转
- 失败场景设计
- CLI 与 API 的完整映射表
- 实现优先级建议

### 已确认前提（不重复推导）

- `submitted` 是本地 runtime 事件，不进入 server canonical event stream
- `seen` 只能由 recipient 触发，只有 `open` 命令触发 `seen`
- `reply <message_id>` 是唯一可能清除 obligation 的路径
- server 是所有 canonical 数据的唯一权威源
- 本地 runtime 是可靠性层，不是业务裁判
- 默认输出 JSON，flags + stdin 输入优先

---

## 2. CLI 命令清单

### 2.1 导航命令（纯读，无 side-effect）

| 命令 | 用途 | 触发 API |
|---|---|---|
| `mailbox pick-mail` | 返回候选邮件列表（agent 默认入口） | GET /v1/agents/{agent_id}/pick-mail |
| `mailbox inbox` | 查看 inbox 总览 | GET /v1/agents/{agent_id}/inbox |
| `mailbox outbox` | 查看 outbox | GET /v1/agents/{agent_id}/outbox |
| `mailbox thread <thread_id>` | 查看 thread 上下文 | GET /v1/threads/{id} 等 |
| `mailbox obligations` | 查看当前 open obligations | GET /v1/obligations?agent_id=... |
| `mailbox status` | 查看本地 runtime 健康度 | 默认仅读本地 SQLite；可选轻量 server health check |

**以上命令均不触发任何状态变更。**

### 2.2 动作命令（有 side-effect）

| 命令 | 用途 | 主要 side-effect |
|---|---|---|
| `mailbox open <message_id>` | 打开邮件，进入处理视野 | 触发 `seen`（唯一入口） |
| `mailbox send` | 发送新邮件 | 写本地 outbound queue，调 POST /v1/messages |
| `mailbox reply <message_id>` | 回复邮件（唯一可清 obligation 的路径） | 写本地 outbound queue，调 POST /v1/messages/{id}/reply |
| `mailbox archive <message_id>` | 归档邮件（影响当前 agent 的工作视图） | 调 POST /v1/agents/{agent_id}/messages/{id}/archive |
| `mailbox unarchive <message_id>` | 取消归档 | 调 POST /v1/agents/{agent_id}/messages/{id}/unarchive |

### 2.3 同步与恢复命令

| 命令 | 用途 | 主要 side-effect |
|---|---|---|
| `mailbox sync` | 拉取 server 更新，推送本地 pending 动作 | 刷新 local_cache，更新 sync_checkpoint，上报心跳 |
| `mailbox retry-sync` | 重试失败的 outbound / pending_seen / pending_delivery | 处理各 pending 队列 |

---

## 3. 各命令详情：参数、输入输出与 side-effect

### 3.1 `mailbox pick-mail`

**用途**：agent 的默认工作入口，返回 top N 候选邮件，帮助 agent 缩小注意力范围。

**参数**

| 参数 | 类型 | 必选 | 说明 |
|---|---|---|---|
| `--agent` | string | 是 | 当前 agent_id |
| `--limit` | int | 否 | 候选数量，默认 3 |
| `--format` | enum | 否 | `json`（默认）/ `text` |

**输入**：无

**输出**（默认 JSON）

```json
{
  "candidates": [
    {
      "rank": 1,
      "reason": "open_obligation",
      "message_id": "msg_01hx001",
      "thread_id": "th_01hx001",
      "from": "mt",
      "type": "request",
      "subject": "调研 OpenClaw 多实例通信方案",
      "requires_reply": true,
      "is_unread": true,
      "obligation_age_seconds": 86400,
      "accepted_at": "2026-03-17T09:00:00+08:00"
    },
    {
      "rank": 2,
      "reason": "thread_continuation",
      "message_id": "msg_01hx002",
      "thread_id": "th_01hx001",
      "from": "mt",
      "type": "inform",
      "subject": "Re: 调研 OpenClaw 多实例通信方案",
      "requires_reply": false,
      "is_unread": true,
      "obligation_age_seconds": 0,
      "accepted_at": "2026-03-17T10:00:00+08:00"
    }
  ],
  "total_unread": 5,
  "open_obligations_count": 1
}
```

`reason` 取值：
- `open_obligation`：有 open obligation 且未读
- `thread_continuation`：已有活跃 thread 的后续邮件
- `recency`：其余未读，按新近性排序

**side-effect**：无

说明：
- 默认只查看本地 runtime 健康度
- 若指定 `--check-server`，CLI 额外执行轻量 server 探测（例如 health/heartbeat freshness），但不拉取业务数据

---

### 3.2 `mailbox inbox`

**用途**：inbox 总览。全局浏览、排查遗漏、fallback 导航。

**参数**

| 参数 | 类型 | 必选 | 说明 |
|---|---|---|---|
| `--agent` | string | 是 | agent_id |
| `--limit` | int | 否 | 默认 20 |
| `--cursor` | string | 否 | 分页游标 |
| `--unread` | bool flag | 否 | 只显示未读 |
| `--has-obligation` | bool flag | 否 | 只显示有 obligation 的 |
| `--format` | enum | 否 | json / text |
| `--check-server` | bool flag | 否 | 追加轻量 server 健康探测（不拉业务数据） |

**输出**（默认 JSON，不含 body）

```json
{
  "messages": [
    {
      "message_id": "msg_01hx001",
      "thread_id": "th_01hx001",
      "from": "mt",
      "type": "request",
      "subject": "调研 OpenClaw 多实例通信方案",
      "requires_reply": true,
      "is_unread": true,
      "has_open_obligation": true,
      "accepted_at": "2026-03-17T09:00:00+08:00"
    }
  ],
  "next_cursor": "cursor_abc123",
  "total_unread": 5
}
```

**side-effect**：无

---

### 3.3 `mailbox outbox`

**用途**：查看发件记录，含 delivery 状态。

**参数**

| 参数 | 类型 | 必选 | 说明 |
|---|---|---|---|
| `--agent` | string | 是 | agent_id |
| `--limit` | int | 否 | 默认 20 |
| `--cursor` | string | 否 | 分页游标 |
| `--format` | enum | 否 | json / text |

**输出**（默认 JSON）

```json
{
  "messages": [
    {
      "message_id": "msg_01hx010",
      "thread_id": "th_01hx010",
      "to": "zhigeng",
      "type": "request",
      "subject": "调研 OpenClaw 多实例通信方案",
      "requires_reply": true,
      "delivery_status": "delivered",
      "accepted_at": "2026-03-17T09:00:00+08:00",
      "delivered_at": "2026-03-17T10:00:00+08:00"
    }
  ],
  "next_cursor": null
}
```

**side-effect**：无

---

### 3.4 `mailbox thread <thread_id>`

**用途**：查看 thread 上下文，用于导航和理解责任链。不触发 `seen`。

**参数**

| 参数 | 类型 | 必选 | 说明 |
|---|---|---|---|
| `thread_id` | string | 是 | positional arg |
| `--agent` | string | 是 | 当前 agent_id（用于查询该 agent 在该 thread 中的 state） |
| `--format` | enum | 否 | json / text |

**输出**（默认 JSON）

```json
{
  "thread": {
    "thread_id": "th_01hx001",
    "subject": "调研 OpenClaw 多实例通信方案",
    "created_by": "mt",
    "participants": ["mt", "zhigeng"],
    "opened_at": "2026-03-17T09:00:00+08:00",
    "last_message_at": "2026-03-17T10:00:00+08:00",
    "message_count": 2,
    "has_open_obligation": true
  },
  "messages": [
    {
      "message_id": "msg_01hx001",
      "from": "mt",
      "to": "zhigeng",
      "type": "request",
      "subject": "调研 OpenClaw 多实例通信方案",
      "requires_reply": true,
      "is_unread": true,
      "is_seen": false,
      "seen_at": null,
      "accepted_at": "2026-03-17T09:00:00+08:00"
    }
  ],
  "open_obligations": [
    {
      "obligation_id": "obl_01hx001",
      "message_id": "msg_01hx001",
      "owner_agent_id": "zhigeng",
      "opened_at": "2026-03-17T09:00:05+08:00",
      "age_seconds": 86400
    }
  ],
  "hint": {
    "has_pending_action": true,
    "next_command": "mailbox open msg_01hx001"
  }
}
```

注意：`messages` 列表**不含 body**，不触发 `seen`。

补充：
- `thread` 视图中的每条 message 摘要应带当前 agent 的轻量阅读视图（`is_seen` / `seen_at`）
- 这是工作上下文信息，不改变 `open` 才触发 `seen` 的规则

**side-effect**：无

---

### 3.5 `mailbox obligations`

**用途**：查看当前 agent 的 open obligations。这是 mailbox 的责任视图。

**参数**

| 参数 | 类型 | 必选 | 说明 |
|---|---|---|---|
| `--agent` | string | 是 | agent_id |
| `--format` | enum | 否 | json / text |

**输出**（默认 JSON）

```json
{
  "open_obligations": [
    {
      "obligation_id": "obl_01hx001",
      "thread_id": "th_01hx001",
      "message_id": "msg_01hx001",
      "from": "mt",
      "subject": "调研 OpenClaw 多实例通信方案",
      "opened_at": "2026-03-17T09:00:05+08:00",
      "age_seconds": 86400,
      "is_unread": true
    }
  ],
  "count": 1
}
```

**side-effect**：无

---

### 3.6 `mailbox status`

**用途**：查看本地 runtime 健康度，帮助 agent 和 operator 了解本地队列状态。

**参数**

| 参数 | 类型 | 必选 | 说明 |
|---|---|---|---|
| `--agent` | string | 是 | agent_id |
| `--format` | enum | 否 | json / text |

**输出**（默认 JSON，默认只读本地 SQLite；`--check-server` 时追加轻量 server 状态）

```json
{
  "agent_id": "zhigeng",
  "machine_id": "thinkpad-x1",
  "pending_outbound": 0,
  "pending_seen": 1,
  "pending_delivery": 2,
  "last_sync_at": "2026-03-17T09:59:00+08:00",
  "last_sync_status": "ok",
  "last_error": null,
  "checkpoint": {
    "last_seen_event_id": "evt_01hxevent999",
    "last_pulled_at": "2026-03-17T09:59:00+08:00"
  },
  "warnings": [
    "1 seen event pending server sync (run: mailbox retry-sync --type seen)"
  ],
  "server": {
    "checked": false,
    "reachable": null,
    "last_heartbeat_fresh": null
  }
}
```

**side-effect**：无

---

### 3.7 `mailbox open <message_id>`

**用途**：打开一封邮件，将其纳入当前处理视野。

**这是 CLI 里唯一触发 `seen` 的命令。**

**参数**

| 参数 | 类型 | 必选 | 说明 |
|---|---|---|---|
| `message_id` | string | 是 | positional arg |
| `--agent` | string | 是 | agent_id（必须等于该 message 的 to） |
| `--format` | enum | 否 | json / text |

**输出**（默认 JSON，含完整 body）

```json
{
  "message": {
    "message_id": "msg_01hx001",
    "thread_id": "th_01hx001",
    "from": "mt",
    "to": "zhigeng",
    "type": "request",
    "subject": "调研 OpenClaw 多实例通信方案",
    "body": "请重点看 GitHub issue、官方 docs 和现成开源方案，最后给我结构化建议。",
    "requires_reply": true,
    "in_reply_to": null,
    "accepted_at": "2026-03-17T09:00:00+08:00"
  },
  "state": {
    "mailbox_role": "recipient",
    "seen_at": "2026-03-17T11:00:00+08:00",
    "is_archived": false
  },
  "obligation": {
    "obligation_id": "obl_01hx001",
    "status": "open",
    "age_seconds": 7200
  },
  "seen_reported": true,
  "seen_pending": false,
  "hint": {
    "has_obligation": true,
    "requires_reply": true,
    "reply_command": "mailbox reply msg_01hx001 --agent zhigeng --type report --body-stdin"
  }
}
```

**side-effect 详情**

1. 调用 `GET /v1/messages/{message_id}` 获取邮件内容（纯读）
2. 立即调用 `POST /v1/messages/{message_id}/seen`（X-Agent-Id: {agent_id}）
3. 若 seen 上报成功：`seen_reported=true`，`seen_pending=false`
4. 若 seen 上报失败（网络问题等）：
   - 写入本地 `pending_seen`
   - 输出中 `seen_reported=false`，`seen_pending=true`
   - `warnings` 字段提示需要 retry

**注意**：`open` 只对 recipient 有意义。若 `--agent` 不等于 message 的 `to`，返回错误。

---

### 3.8 `mailbox send`

**用途**：发送新邮件。可以新建 thread，也可以在已有 thread 中追加一封非 obligation 语义的新消息。

**参数**

| 参数 | 类型 | 必选 | 说明 |
|---|---|---|---|
| `--agent` | string | 是 | 发件 agent_id（等于 from） |
| `--to` | string | 是 | 收件 agent_id |
| `--type` | enum | 是 | request / inform / proposal / report / alert |
| `--subject` | string | 是 | 邮件主题 |
| `--body` | string | 三选一 | 直接传正文 |
| `--body-file` | path | 三选一 | 从文件读正文 |
| `--body-stdin` | bool flag | 三选一 | 从 stdin 读正文 |
| `--requires-reply` | bool flag | 否 | 若存在则为 true，默认 false |
| `--thread-id` | string | 否 | 指定加入已有 thread（不清 obligation） |
| `--idempotency-key` | string | 否 | 不提供则 CLI 自动生成（格式：`cli-{agent_id}-{timestamp}-{rand}`） |
| `--format` | enum | 否 | json / text |

**正文输入主路径**（按优先级）

1. `--body "内容"` — 内联传入
2. `--body-stdin` — 从 stdin 读（管道友好）
3. `--body-file /path/to/body.md` — 从文件读
4. （fallback）若三者均未指定且 TTY 检测到交互模式，打开 `$EDITOR`

**stdin 示例**

```bash
cat body.md | mailbox send --agent zhigeng --to mt --type report --subject "Re: 调研结果" --body-stdin --requires-reply
```

**输出**（默认 JSON）

```json
{
  "local_id": "local_20260317_abc123",
  "status": "accepted",
  "message": {
    "message_id": "msg_01hxreport001",
    "thread_id": "th_01hx001",
    "from": "zhigeng",
    "to": "mt",
    "type": "report",
    "subject": "Re: 调研结果",
    "requires_reply": false,
    "accepted_at": "2026-03-17T11:30:00+08:00"
  },
  "thread": {
    "thread_id": "th_01hx001",
    "has_open_obligation": true
  },
  "obligation_opened": null
}
```

若 server 调用失败，status 为 `pending`，本地 `outbound_messages` 中记录：

```json
{
  "local_id": "local_20260317_abc123",
  "status": "pending",
  "message": null,
  "error": "connection refused: server unreachable",
  "retry_hint": "run: mailbox retry-sync --type outbound"
}
```

**side-effect 详情**

1. 生成 `idempotency_key`（若未提供）
2. 写入本地 `outbound_messages`（status=`pending`）
3. 调用 `POST /v1/messages`（X-Agent-Id: {agent_id}）
4. 若 server 返回 201：
   - 更新本地 `outbound_messages`（status=`accepted`）
   - 写入 `local_cache`（缓存 message + thread）
5. 若 server 调用失败：
   - 本地 `outbound_messages` 保持 status=`pending`
   - 输出 warning，提示 `retry-sync`

**重要**：`--thread-id` 非空时，即使在已有 thread 中发消息，**本命令不清除 obligation**。清除 obligation 只能通过 `mailbox reply`。

---

### 3.9 `mailbox reply <message_id>`

**用途**：对某封具体 message 做正式 reply。**这是唯一可以清除 obligation 的命令。**

**参数**

| 参数 | 类型 | 必选 | 说明 |
|---|---|---|---|
| `message_id` | string | 是 | positional arg，必须是 thread participant 可回复的 message |
| `--agent` | string | 是 | 当前 agent_id（必须是 thread participant） |
| `--type` | enum | 是 | report / proposal / request / inform / alert |
| `--body` | string | 三选一 | 直接传正文 |
| `--body-file` | path | 三选一 | 从文件读正文 |
| `--body-stdin` | bool flag | 三选一 | 从 stdin 读正文 |
| `--requires-reply` | bool flag | 否 | 若存在则为 true，默认 false |
| `--idempotency-key` | string | 否 | 不提供则 CLI 自动生成 |
| `--format` | enum | 否 | json / text |

**`reply` 与 `send` 的关键区别**：

| 维度 | `mailbox reply <message_id>` | `mailbox send --thread-id <thread_id>` |
|---|---|---|
| in_reply_to | server 自动设置为目标 message_id | null |
| 能否清 obligation | 是（若满足规则） | 否（永不清 obligation） |
| 语义 | 正式回复特定邮件 | thread 内新开消息 |

**stdin 示例**

```bash
echo "调研完成，建议如下……" | mailbox reply msg_01hx001 --agent zhigeng --type report --body-stdin
```

**输出**（默认 JSON）

```json
{
  "local_id": "local_20260317_xyz456",
  "status": "accepted",
  "message": {
    "message_id": "msg_01hxreply001",
    "thread_id": "th_01hx001",
    "from": "zhigeng",
    "to": "mt",
    "type": "report",
    "subject": "Re: 调研 OpenClaw 多实例通信方案",
    "in_reply_to": "msg_01hx001",
    "requires_reply": false,
    "accepted_at": "2026-03-18T10:00:00+08:00"
  },
  "obligation_status": "cleared",
  "obligation_cleared": {
    "obligation_id": "obl_01hx001",
    "cleared_at": "2026-03-18T10:00:00+08:00",
    "cleared_by_message_id": "msg_01hxreply001"
  },
  "obligation_opened": null
}
```

`obligation_status` 取值：
- `cleared`：此次 reply 清除了原 obligation
- `still_open`：有 obligation 但未满足清除规则（如 type=inform/alert）
- `no_obligation`：原 message 本就没有 obligation
- `new_opened`：此次 reply 本身开启了新 obligation（当 reply type=request 且 requires_reply=true 时）

**side-effect 详情**（同 `send`，但调用 `/reply` 接口）

---

### 3.10 `mailbox archive <message_id>` / `mailbox unarchive <message_id>`

**用途**：管理当前 agent 的 mailbox 工作视图。

**参数**

| 参数 | 类型 | 必选 | 说明 |
|---|---|---|---|
| `message_id` | string | 是 | positional arg |
| `--agent` | string | 是 | agent_id（必须等于路径中的 agent） |
| `--format` | enum | 否 | json / text |

**输出**

```json
{
  "message_id": "msg_01hx001",
  "agent_id": "zhigeng",
  "archived": true,
  "archived_at": "2026-03-17T12:00:00+08:00"
}
```

**重要约束**（必须在 CLI 输出 warning 中体现）：

- `archive` **不等于** obligation 已清除
- `archive` **不等于** thread 已完成
- `archive` **不等于** message 已处理
- archive 只影响当前 agent 的工作视图，不影响其他 agent

**side-effect**：调用 server archive/unarchive 接口，更新 `local_cache`

---

### 3.11 `mailbox sync`

**用途**：拉取 server 最新状态，推送本地 pending 动作，刷新 cache，上报心跳。

**参数**

| 参数 | 类型 | 必选 | 说明 |
|---|---|---|---|
| `--agent` | string | 是 | agent_id |
| `--machine` | string | 是 | machine_id（用于心跳与 checkpoint） |
| `--format` | enum | 否 | json / text |

**输出**（默认 JSON）

```json
{
  "pulled": {
    "new_messages": 2,
    "updated_threads": 1,
    "checkpoint_updated": true,
    "last_event_id": "evt_01hxevent999"
  },
  "pushed": {
    "outbound_sent": 0,
    "seen_synced": 1,
    "delivery_synced": 2,
    "failed": 0
  },
  "heartbeat": {
    "reported": true,
    "representative_id": "rep_01hxabcd"
  },
  "errors": []
}
```

**side-effect 详情**（按顺序执行）

1. 上报心跳：`POST /v1/agents/{agent_id}/representatives/{rep_id}/heartbeat`
2. 拉取 inbox 更新：`GET /v1/agents/{agent_id}/inbox?cursor={checkpoint}`
3. 拉取新消息内容，写入 `local_cache`（仅元信息，不含 body）
4. 上报 `pending_delivery`：对每条 pending delivery，调用 `POST /v1/messages/{id}/delivered`
5. 上报 `pending_seen`：对每条 pending seen，调用 `POST /v1/messages/{id}/seen`
6. 发送 `outbound_messages`（status=pending）
7. 更新 `sync_checkpoint`

---

### 3.12 `mailbox retry-sync`

**用途**：显式重试失败的 pending 动作。

**参数**

| 参数 | 类型 | 必选 | 说明 |
|---|---|---|---|
| `--agent` | string | 是 | agent_id |
| `--machine` | string | 是 | machine_id |
| `--type` | enum | 否 | `outbound` / `seen` / `delivery` / `all`（默认 all） |
| `--format` | enum | 否 | json / text |

**输出**

```json
{
  "outbound": {
    "attempted": 1,
    "succeeded": 1,
    "failed": 0
  },
  "seen": {
    "attempted": 1,
    "succeeded": 1,
    "failed": 0
  },
  "delivery": {
    "attempted": 2,
    "succeeded": 2,
    "failed": 0
  },
  "errors": []
}
```

**side-effect**：逐条处理各 pending 队列，按队列类型调用对应 API。

---

## 4. 默认输出规范（JSON 约定）

### 4.1 输出结构约定

所有命令的 JSON 输出遵守以下约定：

```json
{
  // 主要数据（命令特定）
  "message": {},
  "messages": [],
  "thread": {},
  "candidates": [],

  // 操作结果元信息（可选）
  "status": "accepted|pending|failed",
  "local_id": "local_XXXXXX",

  // 错误信息（若有）
  "error": {
    "code": "MESSAGE_NOT_FOUND",
    "message": "Message msg_xxx does not exist."
  },

  // 警告（非致命，但 agent 应感知）
  "warnings": [
    "1 seen event pending server sync"
  ],

  // 下一步操作提示（可选，供 agent / skill 使用）
  "hint": {
    "has_obligation": true,
    "reply_command": "mailbox reply msg_xxx --agent zhigeng --type report --body-stdin"
  }
}
```

### 4.2 时间格式

所有时间字段统一为 ISO 8601 with timezone：

```
"2026-03-17T09:00:00+08:00"
```

### 4.3 状态字段约定

| 字段 | 类型 | 说明 |
|---|---|---|
| `is_unread` | bool | `seen_at IS NULL`（recipient 侧） |
| `has_open_obligation` | bool | thread 当前是否有 open obligation |
| `obligation_status` | enum | `cleared` / `still_open` / `no_obligation` |
| `delivery_status` | enum | `accepted` / `delivered` / `failed` |
| `local_status` | enum | `pending` / `submitting` / `accepted` / `failed` |

### 4.4 错误输出

所有错误通过 stderr 输出，exit code 非 0：

```json
{
  "error": {
    "code": "SEEN_NOT_ALLOWED_FOR_SENDER",
    "message": "Only recipient can mark message as seen.",
    "agent_id": "mt",
    "message_id": "msg_01hx001"
  }
}
```

### 4.5 可选输出格式

`--format text` 时输出人类友好格式（不保证稳定结构，不建议 agent 解析）：

```
[pick-mail] 3 candidates found

  #1 [open_obligation] msg_01hx001
     From: mt | Type: request | Age: 24h
     Subject: 调研 OpenClaw 多实例通信方案
     → mailbox open msg_01hx001 --agent zhigeng

  #2 [thread_continuation] msg_01hx002
     From: mt | Type: inform
     Subject: Re: 调研 OpenClaw 多实例通信方案
     → mailbox open msg_01hx002 --agent zhigeng
```

---

## 5. 默认输入规范

### 5.1 body 输入（`send` / `reply`）

三种主路径，按 CLI 解析优先级：

| 路径 | 参数 | 示例 | 适用场景 |
|---|---|---|---|
| 内联 | `--body "内容"` | `mailbox send ... --body "正文"` | 短内容、单行 |
| stdin | `--body-stdin` | `cat body.md \| mailbox send ... --body-stdin` | 管道、多行、agent 生成 |
| 文件 | `--body-file /path/to/body.md` | `mailbox send ... --body-file /tmp/draft.md` | 已有草稿文件 |
| editor（fallback） | 三者均未指定且 TTY 存在 | 交互模式下自动打开 `$EDITOR` | 人类手动撰写 |

**agent 推荐路径**：stdin。管道最 Unix，最可组合，最可脚本化。

```bash
# agent 生成正文，通过 stdin 传入
generate_reply_body | mailbox reply msg_01hx001 \
  --agent zhigeng \
  --type report \
  --body-stdin
```

### 5.2 agent 身份输入

所有命令均通过 `--agent` flag 传入 agent_id。

不接受 `X-Agent-Id` 以外的传递方式（CLI 负责将 `--agent` 转为 header）。

CLI 可选支持通过环境变量设置默认 agent：

```bash
export MAILBOX_AGENT=zhigeng
mailbox pick-mail   # 等价于 mailbox pick-mail --agent zhigeng
```

### 5.3 machine 身份输入

`sync` 和 `retry-sync` 需要 `--machine` 参数，或通过环境变量：

```bash
export MAILBOX_MACHINE=thinkpad-x1
mailbox sync --agent zhigeng
```

### 5.4 配置优先级

配置优先级统一为：

```
CLI flag > environment variable > config file > built-in default
```

例如：
- `--agent` 优先于 `MAILBOX_AGENT`
- `MAILBOX_AGENT` 优先于 `~/.mailbox/config.json` 中的 `default_agent`

### 5.5 server 地址配置

通过配置文件或环境变量：

```bash
export MAILBOX_SERVER=http://192.168.1.100:8787
```

或配置文件 `~/.mailbox/config.json`：

```json
{
  "server": "http://192.168.1.100:8787",
  "default_agent": "zhigeng",
  "default_machine": "thinkpad-x1"
}
```

---

## 6. 本地 Runtime Schema

本地 runtime 以每台机器一个 SQLite 文件实现，路径建议：

```
~/.mailbox/runtime/{agent_id}_{machine_id}.db
```

**设计原则**：
- 本地 runtime 是可靠性层，不是权威源
- 所有字段以 TEXT 存储时间（ISO 8601），不用 DATETIME（避免 SQLite timezone 陷阱）
- 所有 JSON 字段存为 TEXT（`_json` 后缀标识）
- 每张表都有 `created_at` + `updated_at`

---

### 6.1 `outbound_messages`

**用途**：本地已提交发送、但 server 尚未返回 `accepted` 的发信动作。

承载 `submitted` 语义（本地生命周期事件，不进入 server canonical event stream）。

**Schema**

```sql
CREATE TABLE outbound_messages (
  local_id            TEXT PRIMARY KEY,     -- CLI 自动生成，格式：local_{timestamp}_{rand6}
  idempotency_key     TEXT UNIQUE NOT NULL, -- 用于 server 端去重
  command             TEXT NOT NULL,        -- 'send' | 'reply'
  agent_id            TEXT NOT NULL,        -- 发件 agent
  to_agent_id         TEXT,                 -- 收件 agent（send 命令）
  in_reply_to_msg_id  TEXT,                 -- reply 目标 message_id（reply 命令）
  thread_id           TEXT,                 -- 若已知（send --thread-id 时填入）
  type                TEXT NOT NULL,        -- request / inform / proposal / report / alert
  subject             TEXT NOT NULL,
  body                TEXT NOT NULL,
  requires_reply      INTEGER NOT NULL DEFAULT 0,  -- 0 | 1
  status              TEXT NOT NULL DEFAULT 'pending',
                                            -- 'pending' | 'submitting' | 'accepted' | 'failed'
  server_message_id   TEXT,                 -- server 返回后填入
  server_thread_id    TEXT,                 -- server 返回后填入
  submitted_at        TEXT,                 -- 首次尝试发送时间
  last_attempt_at     TEXT,
  attempt_count       INTEGER NOT NULL DEFAULT 0,
  last_error          TEXT,
  accepted_at         TEXT,                 -- server 确认 accepted 时间
  created_at          TEXT NOT NULL,
  updated_at          TEXT NOT NULL
);

CREATE INDEX idx_outbound_status ON outbound_messages(status);
CREATE INDEX idx_outbound_agent  ON outbound_messages(agent_id, status);
```

**状态流转**

```
pending → submitting → accepted  （正常路径，accepted 后可保留或删除）
        → (network fail) → pending  （维持 pending，等待 retry）
        → (permanent error) → failed
```

`accepted` 后是否立即删除由实现决定。建议短期保留用于排查与审计，但清理周期应可配置；默认值可在实现阶段决定（例如 24h）。

**Retry 规则**：
- 最多重试 10 次
- 每次重试间隔按指数退避：min(30s × 2^n, 1h)
- 超过 10 次后标记为 `failed`，不再自动重试，需要人工介入

---

### 6.2 `pending_seen`

**用途**：`open` 命令触发了 `seen`，但 server 上报失败时，暂存在此，等待重试。

**Schema**

```sql
CREATE TABLE pending_seen (
  local_id        TEXT PRIMARY KEY,
  message_id      TEXT NOT NULL,
  agent_id        TEXT NOT NULL,          -- 必须是该 message 的 to
  seen_at         TEXT NOT NULL,          -- open 命令执行的本地时间（不是 server 确认时间）
  status          TEXT NOT NULL DEFAULT 'pending',
                                          -- 'pending' | 'syncing' | 'synced' | 'failed'
  last_attempt_at TEXT,
  attempt_count   INTEGER NOT NULL DEFAULT 0,
  last_error      TEXT,
  synced_at       TEXT,                   -- server 确认后填入
  created_at      TEXT NOT NULL,
  updated_at      TEXT NOT NULL,

  UNIQUE(message_id, agent_id)            -- 一封 message 对一个 agent 只有一条 pending seen
);

CREATE INDEX idx_pending_seen_status ON pending_seen(status);
```

**状态流转**

```
pending → syncing → synced  （正常路径，synced 后可删除或保留供审计）
        → (already_seen from server) → synced  （幂等：server 已有，视为成功）
        → (permanent error) → failed
```

---

### 6.3 `pending_delivery`

**用途**：message 已被拉入本地工作缓存（即已完成本地拉取），但 `delivered` 上报 server 失败时，暂存在此。

**Schema**

```sql
CREATE TABLE pending_delivery (
  local_id         TEXT PRIMARY KEY,
  message_id       TEXT NOT NULL,
  agent_id         TEXT NOT NULL,          -- 必须是该 message 的 to
  machine_id       TEXT NOT NULL,          -- 本台机器的 machine_id
  delivered_at     TEXT NOT NULL,          -- 本地拉取成功的时间
  status           TEXT NOT NULL DEFAULT 'pending',
                                           -- 'pending' | 'syncing' | 'synced' | 'failed'
  last_attempt_at  TEXT,
  attempt_count    INTEGER NOT NULL DEFAULT 0,
  last_error       TEXT,
  synced_at        TEXT,
  created_at       TEXT NOT NULL,
  updated_at       TEXT NOT NULL,

  UNIQUE(message_id, agent_id)             -- 一封 message 对一个 agent 只有一条 pending delivery
);

CREATE INDEX idx_pending_delivery_status ON pending_delivery(status);
```

---

### 6.4 `local_cache`（拆分为两张表）

**用途**：inbox / thread / message 的本地缓存，降低频繁 HTTP 拉取，支撑弱网连续性。

本地缓存不是权威源，以 server 为准。TTL 到期或 sync 成功后刷新。

#### `cached_messages`

```sql
CREATE TABLE cached_messages (
  message_id              TEXT PRIMARY KEY,
  thread_id               TEXT NOT NULL,
  from_agent_id           TEXT NOT NULL,
  to_agent_id             TEXT NOT NULL,
  type                    TEXT NOT NULL,
  subject                 TEXT NOT NULL,
  body                    TEXT,           -- 仅在 open 后填入；pick-mail/inbox 列表不缓存 body
  requires_reply          INTEGER NOT NULL DEFAULT 0,
  in_reply_to_message_id  TEXT,
  accepted_at             TEXT,
  created_at              TEXT,
  -- 缓存元信息
  has_body                INTEGER NOT NULL DEFAULT 0,  -- body 是否已缓存
  cached_at               TEXT NOT NULL,
  cache_ttl_seconds       INTEGER NOT NULL DEFAULT 3600  -- 默认 1h
);

CREATE INDEX idx_cached_messages_thread ON cached_messages(thread_id);
```

#### `cached_message_states`

```sql
CREATE TABLE cached_message_states (
  message_id    TEXT NOT NULL,
  agent_id      TEXT NOT NULL,
  mailbox_role  TEXT NOT NULL,   -- 'sender' | 'recipient'
  is_unread     INTEGER DEFAULT 1,
  seen_at       TEXT,
  replied_at    TEXT,
  is_archived   INTEGER DEFAULT 0,
  archived_at   TEXT,
  cached_at     TEXT NOT NULL,
  PRIMARY KEY(message_id, agent_id)
);
```

#### `cached_threads`

```sql
CREATE TABLE cached_threads (
  thread_id            TEXT PRIMARY KEY,
  subject              TEXT NOT NULL,
  created_by           TEXT NOT NULL,
  participants_json    TEXT,             -- JSON array，derived view
  opened_at            TEXT,
  last_message_at      TEXT,
  last_message_id      TEXT,
  message_count        INTEGER DEFAULT 0,
  has_open_obligation  INTEGER DEFAULT 0,
  is_archived          INTEGER DEFAULT 0,
  cached_at            TEXT NOT NULL,
  cache_ttl_seconds    INTEGER NOT NULL DEFAULT 300  -- 默认 5m（thread 状态变化较快）
);
```

**缓存 TTL 建议**

| 数据类型 | TTL | 理由 |
|---|---|---|
| message body | 3600s（1h） | 邮件内容不可变，较长 TTL 合理 |
| message state | 300s（5m） | seen/replied/archived 状态可能变化 |
| thread metadata | 300s（5m） | has_open_obligation 变化频率较高 |
| inbox view | 120s（2m） | 新邮件到达后需要及时感知 |

---

### 6.5 `sync_checkpoint`

**用途**：记录本机上次同步到的位置，支撑增量同步与恢复。

**Schema**

```sql
CREATE TABLE sync_checkpoint (
  agent_id           TEXT NOT NULL,
  machine_id         TEXT NOT NULL,
  last_pulled_at     TEXT,             -- 上次成功拉取的时间
  last_seen_event_id TEXT,             -- 上次拉取到的最新 event_id（用于增量拉取）
  last_sync_status   TEXT DEFAULT 'ok',  -- 'ok' | 'error'
  last_error         TEXT,
  updated_at         TEXT NOT NULL,

  PRIMARY KEY(agent_id, machine_id)
);
```

---

### 6.6 `memory_receipts`（可选，P1）

**用途**：记录关键 mailbox 事件是否已为当前 agent 写入最小 memory receipt。

按 event 记录，不按 message 记录。同一个 event 对不同 agent 各有一条 receipt。

**Schema**

```sql
CREATE TABLE memory_receipts (
  receipt_id    TEXT PRIMARY KEY,
  event_id      TEXT NOT NULL,
  agent_id      TEXT NOT NULL,
  message_id    TEXT,
  thread_id     TEXT NOT NULL,
  receipt_type  TEXT NOT NULL,       -- 'mail_received' | 'mail_sent' | 'seen' | 'replied'
  sync_status   TEXT DEFAULT 'pending',  -- 'pending' | 'synced' | 'failed'
  synced_at     TEXT,
  last_error    TEXT,
  created_at    TEXT NOT NULL,
  updated_at    TEXT NOT NULL,

  UNIQUE(event_id, agent_id, receipt_type)
);
```

`receipt_type` 与 `event_type` 不完全对应，是 memory 侧语义，不等于 canonical event type。

---

## 7. 本地状态与 Server Canonical State 的边界

| 数据 | 本地 Runtime（SQLite） | Server Canonical（SQLite） | 冲突时以谁为准 |
|---|---|---|---|
| outbound_messages | **owner**（本地生成，submitted 语义） | 不存在 | 本地 owner |
| pending_seen | **queue**（等待同步的事件） | 权威 seen_at | server |
| pending_delivery | **queue**（等待同步的事件） | 权威 delivered_at | server |
| local_cache（messages） | **read cache** | 权威邮件内容 | server |
| local_cache（threads） | **read cache** | 权威 thread 状态 | server |
| sync_checkpoint | **per-machine 位置记录** | agent_checkpoints 作参考 | 本地 owner，server 同步补充 |
| memory_receipts | **本地行为记录** | 不存在 | 本地 owner |

**关键规则**：

1. **任何业务语义判断（obligation 是否 open、thread 是否自然闭合）以 server 为准**
2. **submitted 状态只存在于本地，不上报给 server**
3. **seen_at 的权威值由 server 返回，本地 pending_seen 的 seen_at 是"发生时间"而非"server 确认时间"**
4. **local_cache 的值可能过期，操作前建议先 sync**
5. **本地 runtime 的任何状态在 agent machine 更换时不自动迁移**；新机器必须从 server 重新拉取

---

## 8. Sync / Retry / Recovery 状态流转

### 8.1 推荐 sync 流程

```
mailbox sync 推荐按如下流程执行：

1. POST /heartbeat
   └── 更新 sync_checkpoint.last_heartbeat_at

2. GET /inbox (带 cursor = sync_checkpoint.last_seen_event_id)
   └── 拉取增量消息
   └── 写入 local_cache
   └── 对新消息写入 pending_delivery

3. 处理 pending_delivery 队列
   └── POST /delivered（幂等）
   └── 成功 → pending_delivery.status = synced
   └── 失败 → 保持 pending，attempt_count++

4. 处理 pending_seen 队列
   └── POST /seen（幂等）
   └── 成功 → pending_seen.status = synced
   └── 已 seen → pending_seen.status = synced（already_seen=true）
   └── 失败 → 保持 pending，attempt_count++

5. 处理 outbound_messages（status=pending）
   └── POST /messages 或 POST /messages/{id}/reply
   └── 成功 → outbound_messages.status = accepted
   └── 失败 → 保持 pending，attempt_count++

6. 更新 sync_checkpoint
   └── last_pulled_at = now
   └── last_seen_event_id = 最新拉取 event_id
   └── last_sync_status = ok
```

### 8.2 outbound_messages 状态机

```
[created]
    │
    │ CLI 执行 send/reply
    ▼
  pending ──── (submit to server) ──→ accepted ──→ [cleaned up after 24h]
    │                                    │
    │ network error / server 5xx         │ 若同时返回 obligation_cleared
    ▼                                    ▼
  pending (attempt_count++)        [update local_cache]
    │
    │ attempt_count >= 10
    ▼
  failed ──→ [requires operator intervention]
```

### 8.3 pending_seen 状态机

```
[open 命令执行]
    │
    │ POST /seen 成功
    ▼
  synced ──→ [cleaned up]

    │ POST /seen 失败
    ▼
  pending (写入 pending_seen)
    │
    │ 下次 sync/retry-sync
    │ POST /seen 成功
    ▼
  synced

    │ POST /seen 返回 already_seen=true
    ▼
  synced（幂等，视为成功）

    │ attempt_count >= 10
    ▼
  failed ──→ [警告 operator，seen 状态不一致]
```

### 8.4 representative 切换后的恢复流

```
旧机器 A（deactivated）     新机器 B（activated）
─────────────────────────────────────────────────

1. operator 手动激活机器 B：
   POST /v1/agents/{agent_id}/representatives/{rep_id_B}/activate

2. 机器 B 执行 mailbox sync：
   a. POST /heartbeat（注册机器 B 为 active）
   b. GET /inbox（获取全量未 delivered 消息）
   c. 对未 delivered 的消息：
      - 拉取消息内容 → 写入 local_cache
      - 写入 pending_delivery
   d. 处理 pending_delivery → POST /delivered

3. 若旧机器 A 上有 pending_seen / pending_outbound：
   - 旧机器 A 的 pending 状态在机器 A 本地
   - 机器 B 无法直接访问机器 A 的 SQLite
   - 处理方式：
     a. 若旧机器 A 可访问：先在 A 上执行 retry-sync，让 A 把 pending 推上去
     b. 若旧机器 A 不可访问：operator 需要人工评估影响
        - pending_seen：server 端该 message 仍为 unseen，新机器 B 可通过 open 重新 seen
        - pending_outbound：消息未到 server，需要重新发送（注意 idempotency_key 去重）

4. 机器 B 稳定工作后，旧机器 A 的 local_cache 自动失效（TTL 到期）
```

---

## 9. 失败场景设计

### 9.1 网络失败（发信时）

**场景**：执行 `mailbox send` 或 `mailbox reply` 时，server 不可达或返回 5xx。

**处理**：
1. `outbound_messages` 写入并保持 status=`pending`
2. CLI 输出：`{"status": "pending", "error": "server unreachable", "retry_hint": "..."}`
3. 下次 `sync` 或 `retry-sync` 时自动重试
4. 重试时使用同一个 `idempotency_key`，server 保证去重

**不能发生**：
- 用户发送两次相同邮件（idempotency_key 保护）
- CLI 静默失败（必须输出明确的 pending 状态）

### 9.2 重复提交（idempotency key 冲突）

**场景**：同一个 `idempotency_key` 被提交两次（网络超时后重试，或用户重复操作）。

**server 行为**：
- 相同 key + 相同 payload → 返回原结果（200/201）
- 相同 key + 不同 payload → 返回 `409 IDEMPOTENCY_CONFLICT`

**CLI 处理**：
- 收到原结果 → 视为成功，更新本地状态
- 收到 409 → 报错输出，不重试

### 9.3 delivered / seen 上报失败

**场景**：`open` 时 POST /seen 失败，或 `sync` 时 delivered 上报失败。

**处理**：
1. 写入 `pending_seen` / `pending_delivery`
2. `mailbox status` 输出 warning
3. `mailbox sync` 或 `mailbox retry-sync` 自动处理

**边界**：
- 若 `pending_seen` 中的 message 在 server 已有 `seen_at`（另一台机器已上报），server 返回 `already_seen=true`，本地标记 synced
- `seen` 只记录第一次，server 幂等保证

### 9.4 representative 切换（active machine 变更）

**场景**：agent 从机器 A 切换到机器 B。

**影响分析**

| 状态 | 处理方式 |
|---|---|
| 机器 A 的 outbound_messages（pending） | 若 A 可访问，先在 A 上 retry-sync；若不可访问，从 server 的 outbox 检查是否已 accepted |
| 机器 A 的 pending_seen | 从 server 看 message 是否已 seen；若未 seen，从机器 B 重新 open |
| 机器 A 的 pending_delivery | 机器 B sync 后，server 会发现 delivered=false，机器 B 重新上报 |
| 机器 A 的 local_cache | 不迁移，机器 B 从 server 重新拉取 |

**关键设计**：server 的 `deliveries` 表记录了 delivery 状态，机器 B 的 `sync` 可以发现哪些消息尚未 delivered，并补上。

### 9.5 seen 上报失败但 open 已展示内容

**场景**：agent 已读到邮件内容（`open` 成功返回 body），但 POST /seen 网络超时。

**处理**：
1. 邮件内容已展示，不回收
2. 写入 `pending_seen`，等待 retry
3. server 端该 message 仍为 `is_unread: true`，直到 retry 成功

**后果**：
- `inbox` 仍显示该 message 为未读
- `pick-mail` 仍会推荐该 message（直到 seen 同步成功）
- 不影响 obligation 状态

这是可接受的短暂不一致，优先保证 `open` 的用户体验。

### 9.6 服务端义务清除失败

**场景**：执行 `mailbox reply` 时，server 接受了消息（accepted），但 obligation 未清除（如 reply type 不满足规则）。

**处理**：
1. CLI 在输出中明确显示 `obligation_status: still_open`
2. 不是错误，是业务规则
3. agent / skill 根据 `obligation_status` 决定后续处理

**不能发生**：
- CLI 自己判断 obligation 是否应被清除
- CLI 根据上下文猜测 obligation 状态

---

## 10. CLI 与 API 的完整映射表

| CLI 命令 | HTTP 方法 | API 路径 | 本地状态变更 | 说明 |
|---|---|---|---|---|
| `mailbox pick-mail` | GET | `/v1/agents/{agent_id}/pick-mail` | 无 | |
| `mailbox inbox` | GET | `/v1/agents/{agent_id}/inbox` | 可更新 local_cache | 可选缓存 |
| `mailbox outbox` | GET | `/v1/agents/{agent_id}/outbox` | 可更新 local_cache | |
| `mailbox thread <id>` | GET | `/v1/threads/{id}` | 可更新 local_cache | 附加调用 `/threads/{id}/messages`（摘要）、`/obligations?thread_id=...` |
| `mailbox obligations` | GET | `/v1/obligations?agent_id=...` | 无 | |
| `mailbox status` | —（仅本地） | 无 server 调用 | 无 | 读 local SQLite |
| `mailbox open <id>` | GET + POST | `/v1/messages/{id}` + `/v1/messages/{id}/seen` | 写 pending_seen（失败时）、更新 local_cache | seen 由 open 触发 |
| `mailbox send` | POST | `/v1/messages` | 写 outbound_messages，成功后更新 local_cache | 先写本地，再推 server |
| `mailbox reply <id>` | POST | `/v1/messages/{id}/reply` | 写 outbound_messages，成功后更新 local_cache | 唯一可清 obligation 的路径 |
| `mailbox archive <id>` | POST | `/v1/agents/{agent_id}/messages/{id}/archive` | 更新 local_cache（is_archived） | |
| `mailbox unarchive <id>` | POST | `/v1/agents/{agent_id}/messages/{id}/unarchive` | 更新 local_cache | |
| `mailbox sync` | GET + POST | 多个（见 §8.1） | 刷新 local_cache，清理 pending 队列，更新 checkpoint | 核心 runtime 命令 |
| `mailbox retry-sync --type outbound` | POST | `/v1/messages` 或 `/v1/messages/{id}/reply` | 更新 outbound_messages | |
| `mailbox retry-sync --type seen` | POST | `/v1/messages/{id}/seen` | 清理 pending_seen | |
| `mailbox retry-sync --type delivery` | POST | `/v1/messages/{id}/delivered` | 清理 pending_delivery | |

---

## 11. 实现优先级建议

### P0：第一批（形成基础发收闭环）

| 优先级 | 命令 / 能力 | 理由 |
|---|---|---|
| P0-1 | `mailbox send` + outbound_messages 本地队列 | 发信是基础 |
| P0-2 | `mailbox inbox` | 查 inbox 是基础 |
| P0-3 | `mailbox open` + pending_seen | seen 触发 + 可靠性 |
| P0-4 | `mailbox reply` + outbound_messages | 回信是基础 |
| P0-5 | `mailbox sync`（基础版：拉 inbox + 推 pending） | runtime 核心 |
| P0-6 | `mailbox status`（仅本地 SQLite，无 server 调用） | 最小可观察性 |
| P0-7 | local SQLite 初始化（outbound_messages / pending_seen / sync_checkpoint） | 支撑 P0-1~6 |

**P0 交付标准**：MT 能给知更发信，知更能 open 并 reply，双方 seen 和 delivered 正确上报，断网时本地队列保存并恢复。

---

### P1：第二批（完整工作流 + 完整可靠性）

| 优先级 | 命令 / 能力 | 理由 |
|---|---|---|
| P1-1 | `mailbox pick-mail` | agent 的默认入口，V1 核心体验 |
| P1-2 | `mailbox thread` | 上下文导航，多轮对话必需 |
| P1-3 | `mailbox obligations` | 责任视图，agent 感知必需 |
| P1-4 | `mailbox retry-sync` | 显式重试，operator 排查必需 |
| P1-5 | `mailbox outbox` | 发件记录，agent 自我感知 |
| P1-6 | pending_delivery 上报（纳入 sync） | 多机 delivered 状态完整性 |
| P1-7 | cached_messages / cached_threads（TTL 管理） | 弱网连续性 |
| P1-8 | representative heartbeat（纳入 sync） | operator 可观察性 |

---

### P2：第三批（辅助能力）

| 优先级 | 命令 / 能力 | 理由 |
|---|---|---|
| P2-1 | `mailbox archive / unarchive` | 工作视图管理，非核心路径 |
| P2-2 | memory_receipts 追踪 | agent 记忆层，skill 配合 |
| P2-3 | `--format text` 人类友好输出 | 调试用，非 agent 主路径 |
| P2-4 | cache 定期清理策略（过期 eviction） | 防止本地 SQLite 无限增长 |
| P2-5 | sync_checkpoint 多机同步（与 agent_checkpoints 对齐） | operator 精确观察 |

---

## 12. 一句话总结

> Agent Mailbox V1 CLI 是一个：纯 CLI、默认 JSON、以 `pick-mail → open → reply` 为主路径、只有 `open` 触发 `seen`、只有 `reply <message_id>` 清 obligation、带本地 SQLite runtime 支撑 outbound queue / pending_seen / pending_delivery / checkpoint 的 mailbox agent runtime。
>
> 它厚在可靠性（本地队列、retry、checkpoint），薄在语义（不替 agent 做业务判断，不持有权威真相）。
