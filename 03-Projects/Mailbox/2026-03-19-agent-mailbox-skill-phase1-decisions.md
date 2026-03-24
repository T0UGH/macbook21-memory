# 2026-03-19 Agent Mailbox Skill Phase 1 拍板稿

这份文档记录贵平对 `openclaw-mailbox-skill` Phase 1 的正式拍板结论。

目标：作为后续 schema、规则文档、实现与 review 的统一锚点。

---

## 1. Phase 1 总定义

`openclaw-mailbox-skill` 的 Phase 1 定义为：

> **一个无状态、无副作用、可直接调用 mailbox CLI、输入最小 snapshot、输出 decision envelope、但不形成多轮自治循环的主 agent 内部决策技能。**

这句话是后续所有实现与 review 的最高边界。

---

## 2. 输入边界

Phase 1 的输入严格收敛为两个来源：

- `mailbox pick-mail`
- `mailbox status`

即：先只用这两个来源组装最小 `MailboxSnapshot`。

### Phase 1 暂不引入

- inbox 全量
- obligations 详情列表
- thread 详情

后续如需扩展，进入 Phase 2 再讨论。

---

## 3. 输出边界

Phase 1 的 `MailboxDecisionEnvelope` 保留以下 6 个核心字段：

1. `should_process`
2. `urgency`
3. `priority_item`
4. `recommended_actions`
5. `defer_reason`
6. `health_warnings`

这 6 个字段共同覆盖：
- 是否处理
- 处理优先级
- 先处理谁
- 推荐动作
- 为什么延后
- 本地健康异常

---

## 4. CLI 调用边界

Phase 1 **允许 skill 内部直接调用 mailbox CLI**。

但边界明确如下：

### 允许
- 调用 `mailbox pick-mail`
- 调用 `mailbox status`
- 在单次 skill 调用内执行有限步骤
- 在明确边界内执行一次 `mailbox open <message_id>`，用于把候选消息推进到可处理上下文

### 禁止
- 演化成自动多轮自治循环
- 隐式连续推进多个 mailbox 状态转移直到“自己收敛”
- 变成隐藏的二级调度器

换句话说：

> skill 可以直接调 CLI，也可以在 Phase 1 中执行一次受控的 `open`，但不能借此把自己做成 mini-agent loop。

---

## 5. 状态管理边界

Phase 1 **明确禁止跨调用状态**。

即：
- 不记“上次建议了什么”
- 不记“主 agent 上次是否采纳”
- 不做“升级语气 / 连续催办”
- 每次都只基于当前 snapshot 做单次判断

要求：
- 无状态
- 无副作用
- 单次决策

---

## 6. urgency 设计

Phase 1 的 `urgency` 先只保留 4 档：

- `high`
- `normal`
- `low`
- `none`

当前不引入更复杂的分数制或细粒度枚举。

原则：
- 先保证可解释
- 不做黑盒评分器

---

## 7. 双通道设计

Phase 1 明确分成两条通道：

### 7.1 业务通道
回答：
- 现在是否该处理 mailbox
- 应该优先处理哪一项
- 原因是什么

### 7.2 健康通道
回答：
- 是否存在 failed outbound
- 是否存在 failed seen / delivery
- 是否存在 sync error
- 是否存在应优先暴露的本地可靠性异常

这两条通道必须分开，避免把“消息优先级”和“本地可靠性故障”混成一锅。

---

## 8. priority_item.reason 枚举

Phase 1 的 `priority_item.reason` 先只保留 3 类：

- `open_obligation`
- `thread_continuation`
- `recency`

当前不扩展更多 reason 枚举。

原则：
- 第一版只保留核心决策来源
- 更细原因留到后续迭代

---

## 9. recommended_actions 设计

Phase 1 **允许输出具体 CLI 命令**。

同时：
- 允许输出短动作序列
- 建议控制在 **1~2 步**
- 最多不要超过 3 步

这保证 skill 的建议是可执行的，同时不越界成自动执行脚本。

---

## 10. next_check_hint 设计

Phase 1 的 `next_check_hint` 先只保留 3 种：

- `immediate`
- `after_current_task`
- `next_heartbeat`

当前不引入更复杂的时间语义，例如：
- defer_until timestamp
- snooze duration
- retry window

---

## 11. 触发点设计

Phase 1 的触发点先只定义 4 类：

- `heartbeat`
- `coarse_safe_point`
- `idle_gap`
- `post_mailbox_action_recheck`

这 4 类触发点已足够支撑第一阶段集成。

---

## 12. 文档产物拆分

Phase 1 的设计产物明确拆成两个：

### 产物 1：schema
用于定义：
- `MailboxSnapshot`
- `MailboxDecisionEnvelope`
- 字段、枚举、约束

### 产物 2：决策规则文档
用于定义：
- obligation 何时为 `high`
- health warnings 如何提升关注度
- defer_reason 如何生成
- next_check_hint 如何选择
- recommended_actions 如何生成

要求：
- schema 与规则文档不得混写成一个模糊文档

---

## 13. 实现顺序

Phase 1 的推进顺序正式定为：

1. `MailboxSnapshot` schema
2. `MailboxDecisionEnvelope` schema
3. 决策规则文档
4. skill 最小实现
5. skill 测试
6. mailbox `--help`

---

## 14. mailbox --help

`mailbox --help` / 子命令 `--help` 需要补充，但优先级定为：

> **schema 稳定之后尽快补齐**

也就是：
- 不是当前最先做的事
- 但也不是可以无限后推的事

建议作为 schema / 规则文档之后的并行小项纳入计划。

---

## 15. 对实现与 review 的要求

后续 Hex 或其他实现者推进时，应严格按以下标准 review：

### 必须满足
- 仍然符合总定义
- 输入未越界膨胀
- 输出字段完整
- 无状态
- 无副作用
- 不形成多轮自治循环
- 如需执行 `open`，必须停留在“单次状态推进”层面，不能继续隐式串联 reply / re-check / 下一封处理
- 业务通道 / 健康通道清晰分离
- schema 与规则文档分离

### 不应出现
- skill 自己偷偷维护历史状态
- skill 变成隐式 scheduler
- 输出只剩自然语言 prose
- recommended_actions 演化成长脚本
- Phase 1 过早引入 obligations/thread/inbox 全量细节

---

## 16. 一句话总结

`openclaw-mailbox-skill` 的 Phase 1，已经正式收敛为：

> **一个用最小 snapshot 做单次结构化决策的主 agent 内部技能，而不是一个自治邮箱代理。**
