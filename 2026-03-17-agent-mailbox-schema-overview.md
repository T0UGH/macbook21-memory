# Agent Mailbox Schema Overview

- 日期：2026-03-17
- 作者：MT
- 目的：把 Agent Mailbox V1 的核心数据模型压缩成一页“表职责总览 + 字段硬规则”，方便实现时快速对齐

---

## 一、总体设计原则

### 1. Server 是唯一权威源
- canonical mail / thread / event / obligation / registry 都以 server 为准
- 本地 mailbox 是工作缓存，不是最终真相

### 2. 产品像邮箱，存储像状态系统
- 对外是 mailbox 语义
- 对内以 SQLite 为 canonical store
- `.mail.md` 是 mirror / export，不是唯一真相

### 3. 事实与状态分层
- **事实（facts）** → `events`
- **专项状态对象（state objects）** → `obligations` / `deliveries` / `representatives`
- **摘要投影（projection / summary）** → `threads.has_open_obligation` 等

### 4. 所有表必须有审计时间
以下字段是硬规则：
- `created_at`：记录创建时间
- `updated_at`：记录更新时间

注意：
- 业务语义时间仍需单独保留，如 `accepted_at` / `opened_at` / `delivered_at` / `cleared_at`
- `created_at` / `updated_at` 不替代业务时间

### 5. ID 策略
- canonical id 由 server 生成
- 使用带前缀的 ULID
  - `msg_<ulid>`
  - `th_<ulid>`
  - `evt_<ulid>`
  - `obl_<ulid>`
  - `dlv_<ulid>`
- `agent_id` 保持稳定短字符串，如 `mt` / `zhigeng`

---

## 二、8 张表的职责总览

### 1. `agents`
**角色：** agent registry / 工作身份定义表  
**它回答：** 这个 agent 是谁、叫什么、做什么、对外地址是什么

#### 建议关键字段
- `agent_id`
- `primary_address`
- `name`
- `role`
- `node_label`
- `capabilities_json`
- `preferences_json`
- `work_boundaries_json`
- `is_active`
- `created_at`
- `updated_at`

#### 硬规则
- `agent_id` 是内部稳定身份，不是邮箱地址
- `primary_address` 承载邮箱语义，如 `mt@local.agent`
- 不把运行时状态塞进这张表

#### 不该放的东西
- 当前 active representative
- inbox 计数
- 长篇 prompt
- 复杂组织关系图

---

### 2. `threads`
**角色：** 一段持续协作关系的摘要视图  
**它回答：** 这个 thread 的主题是什么、何时开始、最近一次互动是什么、当前是否还欠回复

#### 建议关键字段
- `thread_id`
- `subject`
- `created_by_agent_id`
- `opened_at`
- `last_message_at`
- `last_message_id`
- `has_open_obligation`
- `archived_at`
- `metadata_json`
- `created_at`
- `updated_at`

#### 硬规则
- thread 是一等对象，不是 message 的附属
- `has_open_obligation` 可作为 projection 保留
- thread 不应滑向 ticket / 工单模型

#### 不该放的东西
- owner
- priority
- 复杂 status flow
- resolved_by / closed_reason

---

### 3. `messages`
**角色：** canonical mail store / 邮件本体表  
**它回答：** 这封邮件是谁发给谁、属于哪个 thread、内容是什么、是否要求回复、何时被 server 接纳

#### 建议关键字段
- `message_id`
- `thread_id`
- `from_agent_id`
- `to_agent_id`
- `type`
- `subject`
- `body_markdown`
- `requires_reply`
- `in_reply_to_message_id`
- `submitted_at`（如需保留本地提交时间）
- `accepted_at`
- `canonical_mail_path`
- `metadata_json`
- `created_at`
- `updated_at`

#### 硬规则
- `messages` 只保存邮件本体和邮件级业务语义
- `.mail.md` 是 mirror / export，不是权威存储
- `requires_reply` 是 obligation 规则入口
- `accepted_at` 是 canonical 生命周期关键时间点

#### 不该放的东西
- `delivered`
- `seen`
- `replied`
- `obligation_open`
- 其他运行时状态

---

### 4. `events`
**角色：** mailbox 生命周期事实流  
**它回答：** 系统里究竟发生过什么事

#### 建议关键字段
- `event_id`
- `thread_id`
- `message_id`
- `event_type`
- `actor_type`
- `actor_id`
- `target_agent_id`
- `occurred_at`
- `metadata_json`
- `created_at`
- `updated_at`

#### 建议事件类型
- `submitted`
- `accepted`
- `opened`
- `delivered`
- `seen`
- `replied`
- `obligation-open`
- `obligation-cleared`

#### 硬规则
- events 记录事实，不承担全部当前状态表达
- `occurred_at` 是业务发生时间，不等于记录创建时间
- event 应尽量 append-only

#### 不该放的东西
- 复杂 status
- 同步成功/失败状态机
- 只用于 debug 的非结构化日志

---

### 5. `obligations`
**角色：** 回复义务状态对象  
**它回答：** 哪封消息要求回复、谁欠回复、是否已被明确回复清除

#### 建议关键字段
- `obligation_id`
- `thread_id`
- `message_id`
- `owner_agent_id`
- `status`
- `opened_at`
- `cleared_at`
- `cleared_by_message_id`
- `metadata_json`
- `created_at`
- `updated_at`

#### 硬规则
- 一封 message 在 V1 最多开启一条 obligation
- `requires_reply = true` 是 obligation 的开启入口
- obligation 的清除必须由明确回复完成
- `status` 在这张表里是合理的，V1 先只保留 `open / cleared`

#### 不该放的东西
- 复杂任务拆解
- 多层状态流
- 子任务体系

---

### 6. `representatives`
**角色：** agent 与运行机器之间的代表关系表  
**它回答：** 某个 agent 当前由哪台机器代表运行

#### 建议关键字段
- `agent_id`
- `machine_id`
- `machine_label`
- `is_active`
- `activated_at`
- `deactivated_at`
- `last_heartbeat_at`
- `metadata_json`
- `created_at`
- `updated_at`

#### 硬规则
- 同一 `agent_id` 同一时刻只能有一个 `is_active = 1`
- representative 是独立关系对象，不应塞进 `agents`
- V1 先手动切换，不做自动接管

#### 不该放的东西
- 完整机器监控系统数据
- 复杂主备切换逻辑

---

### 7. `deliveries`
**角色：** 邮件投递状态对象  
**它回答：** 某封 canonical message 是否已正式送达目标 agent 的工作台

#### 建议关键字段
- `delivery_id`
- `message_id`
- `target_agent_id`
- `representative_machine_id`
- `status`
- `delivered_at`
- `last_attempt_at`
- `attempt_count`
- `metadata_json`
- `created_at`
- `updated_at`

#### 硬规则
- delivery 是 **per-agent**，不是 per-machine
- machine 只是 delivery 的执行上下文
- `status` 在这张表里合理，V1 先保留 `pending / delivered`
- `retry-delivery` 依赖这张表

#### 不该放的东西
- 过重的失败状态机
- 与 message 本体混在一起

---

### 8. `agent_checkpoints`
**角色：** 纯拉同步的进度与健康度记录  
**它回答：** 某个 agent 在某台机器上同步到哪了、最近同步是否正常

#### 建议关键字段
- `agent_id`
- `machine_id`
- `last_pulled_at`
- `last_seen_event_id`
- `last_sync_status`
- `last_error`
- `created_at`
- `updated_at`

#### 硬规则
- checkpoint 建议按 `agent + machine` 维度记录
- 它是同步控制对象，不是业务对象
- server 端需要可观察 checkpoint，不能只存在本地文件

#### 不该放的东西
- 复杂同步状态机
- 和 representative 混在一张表里

---

## 三、表与表之间的关系（文字版）

### `agents`
定义稳定身份。

### `threads`
承载一段持续协作关系。

### `messages`
挂在 `threads` 下，保存邮件本体。

### `events`
记录 `messages` / `threads` 相关生命周期事实。

### `obligations`
由 `messages.requires_reply` 触发，绑定到 `thread + message + owner_agent`。

### `representatives`
决定当前哪个 machine 代表某个 agent。

### `deliveries`
描述某封 `message` 对某个目标 agent 的投递状态，并与 representative 执行上下文相关。

### `agent_checkpoints`
记录 agent 在某机器上的 pull 同步进度。

---

## 四、最核心的分层判断

### 哪些表保存“本体”
- `agents`
- `threads`
- `messages`

### 哪些表保存“事实”
- `events`

### 哪些表保存“状态对象”
- `obligations`
- `representatives`
- `deliveries`
- `agent_checkpoints`

### 哪些字段属于“摘要投影”
- `threads.has_open_obligation`
- `threads.last_message_at`
- `threads.last_message_id`

---

## 五、实现时最重要的 10 条纪律

1. **不要把 mailbox 做回 task executor**
2. **不要把 `messages` 做成运行时状态大杂烩**
3. **不要把 `agents` 做成 prompt / 人格配置中心**
4. **不要把 `threads` 做成工单系统**
5. **不要让 `events` 退化成 debug 日志**
6. **不要省略 `obligations`，它是协作密度来源**
7. **不要把 representative 和 machine 监控混在一起**
8. **不要把 delivery 视为 message 的附属布尔值**
9. **不要忽视 checkpoint，纯拉系统必须显式建模同步进度**
10. **不要混淆审计时间（created_at/updated_at）和业务时间（accepted_at/opened_at/delivered_at 等）**

---

## 六、一句话总结

Agent Mailbox V1 的 schema 应坚持这样一个骨架：

> **邮件本体、事件事实、专项状态对象、线程摘要投影四层分离；server 保持权威，客户端保持工作台语义。**
