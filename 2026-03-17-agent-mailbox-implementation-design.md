# Agent Mailbox Implementation Design

- 日期：2026-03-16
- 状态：Draft v1
- 作者：MT
- 关联文档：`docs/2026-03-15-agent-mailbox-prd.md`

---

## 1. 目标

在既有 PRD 的基础上，把 Agent Mailbox 从“产品定义”推进到“可实现的技术设计基线”。

本文档当前聚焦：

1. server 权威存储模型
2. ID 策略
3. 通信协议基线
4. 同步模型基线
5. V1 最小 schema 草图

暂不在本轮定死：

- 最终 API 全量 contract
- 本地 cache 详细目录结构
- CLI 完整返回体
- 错误码全集
- 重试与恢复全流程

---

## 2. 已确认的实现前提

### 2.1 Canonical store
采用**混合模式**：

- server 的 canonical source of truth 使用 **SQLite**
- 邮件保留 **`.mail.md` 镜像 / 导出能力**
- thread / event / obligation / registry / index 都以 SQLite 为准

### 2.2 CLI ↔ server 通信
采用 **HTTP JSON API**：

- CLI 通过 HTTP 调 server
- server 返回 JSON
- V1 不采用共享目录 / 文件同步方案
- V1 不采用自定义 socket 长连接协议

### 2.3 同步模型
采用**纯拉模式（pull-based sync）**：

- agent 在合适时机主动拉取 inbox / thread / state 更新
- 发信、回信通过主动 push 完成
- 失败后通过 retry 机制补偿
- server 不主动中断式推送新邮件

### 2.4 ID 策略
采用 **server 生成 canonical id + 带前缀的 ULID**：

- `message_id = msg_<ulid>`
- `thread_id = th_<ulid>`
- `event_id = evt_<ulid>`
- `obligation_id = obl_<ulid>`

说明：

- V1 中 canonical id 由 server 生成
- client 不负责生成最终 canonical id
- agent 身份 id 可继续使用固定字符串，如 `mt` / `zhigeng`

---

## 3. 设计原则

### 3.1 产品隐喻与存储实现解耦
系统对用户和 agent 来说像邮件系统，但 server 内部不要求以文件系统充当 canonical source。

结论：

- 产品层像邮件
- 存储层像可靠的状态系统

### 3.2 事件是事实，状态是投影
系统内部保留事件流（events），对外主要呈现状态视图（projection / summary）。

结论：

- 不能只存最终状态
- 也不建议所有查询都临时从 events 现场推导
- V1 采用 **event + projection** 的平衡路线

### 3.3 先保证闭环与可观察性，再追求优雅
V1 的首要目标不是做最纯粹的 event-sourcing 系统，而是做一个：

- 能稳定收发
- 能查 thread
- 能看 obligation
- 能恢复与重试
- 能支撑 agent 感知和 operator 观察

### 3.4 多机前提下，server 是唯一权威源
既然 PRD 已确定 server 是 canonical source，那么：

- canonical mail / thread / event / obligation / registry 应以 server 为准
- 本地 mailbox 视图是工作缓存，不是最终真相

---

## 4. V1 最小 schema 草图

V1 建议至少包含以下核心表：

1. `agents`
2. `threads`
3. `messages`
4. `events`
5. `obligations`
6. `representatives`

建议再加入两个辅助表：

7. `deliveries`
8. `agent_checkpoints`

---

## 5. 表结构草图

### 5.1 `agents`

用于描述 agent 的稳定身份与工作属性。

建议字段：

- `agent_id`
- `name`
- `role`
- `node_label`
- `capabilities_json`
- `preferences_json`
- `work_boundaries_json`
- `is_active`
- `created_at`
- `updated_at`

说明：

- `agent_id` 建议使用稳定字符串，如 `mt` / `zhigeng`
- 这张表本质上是轻量 registry，而不是复杂的人格建模系统

---

### 5.2 `threads`

表示一段持续协作关系，是 mailbox 的核心对象之一。

建议字段：

- `thread_id`
- `subject`
- `created_by_agent_id`
- `opened_at`
- `last_message_at`
- `last_message_id`
- `has_open_obligation`
- `archived_at`
- `metadata_json`

说明：

- `has_open_obligation` 建议作为 projection 字段保留
- `last_message_at` / `last_message_id` 用于 thread list 与 inbox 导航

---

### 5.3 `messages`

保存邮件本体。

建议字段：

- `message_id`
- `thread_id`
- `from_agent_id`
- `to_agent_id`
- `type`
- `subject`
- `body_markdown`
- `requires_reply`
- `in_reply_to_message_id`
- `created_at`
- `accepted_at`
- `canonical_mail_path`
- `metadata_json`

说明：

- `body_markdown` 直接保存正文 markdown
- `.mail.md` 可作为 mirror / export，而不是唯一权威存储
- `accepted_at` 是 canonical 生命周期的关键时间点，值得单独保留

---

### 5.4 `events`

保存系统事实流，是 lifecycle 的底层记录。

建议字段：

- `event_id`
- `thread_id`
- `message_id`
- `event_type`
- `actor_type`
- `actor_id`
- `target_agent_id`
- `occurred_at`
- `metadata_json`

说明：

- 建议统一一张 `events` 表
- thread event 可允许 `message_id` 为空
- `target_agent_id` 对 delivered / seen / replied 等事件很有价值

---

### 5.5 `obligations`

显式记录“哪些邮件欠回复”。

建议字段：

- `obligation_id`
- `thread_id`
- `message_id`
- `owner_agent_id`
- `status`
- `opened_at`
- `cleared_at`
- `cleared_by_message_id`
- `metadata_json`

说明：

- 虽然 obligation 可由规则推导，但工程上建议保留显式表
- operator 视图、inbox 排序、thread 状态都依赖它

---

### 5.6 `representatives`

记录 agent 当前 active representative。

建议字段：

- `agent_id`
- `machine_id`
- `machine_label`
- `is_active`
- `activated_at`
- `deactivated_at`
- `last_heartbeat_at`
- `metadata_json`

说明：

- V1 是手动切换，但这张表仍然必要
- 一个 agent 同时只能存在一个 active representative

---

### 5.7 `deliveries`

记录投递状态与重试基础数据。

建议字段：

- `delivery_id`
- `message_id`
- `target_agent_id`
- `representative_machine_id`
- `status`
- `delivered_at`
- `last_attempt_at`
- `attempt_count`
- `metadata_json`

说明：

- delivery 在多机场景下值得显式建模
- 这张表能直接支撑 `retry-delivery`、operator 观察与 representative 切换后的恢复

---

### 5.8 `agent_checkpoints`

记录 agent / representative 的拉取同步位置。

建议字段：

- `agent_id`
- `machine_id`
- `last_pulled_at`
- `last_seen_event_id`
- `last_sync_status`
- `updated_at`

说明：

- 纯拉模式下需要 checkpoint 概念
- 该表用于增量拉取、恢复与排查同步问题

---

## 6. 当前技术路线总结

V1 Agent Mailbox 当前建议实现路线可概括为：

> **一个以 SQLite 为 canonical source、通过 HTTP JSON API 提供局域网多机访问、客户端按纯拉模型同步，并使用 event + projection 组织状态的异步 agent mailbox 系统。**

---

## 7. 下一步待细化的设计主题

后续建议按以下顺序继续细化：

1. 逐表讲清字段语义、约束与索引
2. 设计 V1 最小 API contract
3. 设计本地 mailbox cache / pending queue
4. 设计 `.mail.md` mirror 生成策略
5. 设计 retry / recovery / representative switch 细节

---

## 8. 当前决策状态

### 已基本确定
- SQLite + `.mail.md` 镜像
- HTTP JSON API
- 纯拉模式同步
- server 生成 canonical id
- 前缀 + ULID
- event + projection 路线
- 8 张表的 V1 schema 草图

### 尚未定稿
- 每张表的最终字段类型与约束
- 索引策略
- API 请求与响应体
- 本地 cache 的详细数据结构
- `.mail.md` 的生成时机
- 失败重试与恢复流程
