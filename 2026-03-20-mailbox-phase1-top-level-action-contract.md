# Mailbox Phase 1：Top-level Action-oriented Decision Contract

## 1. Purpose

本文档定义 Mailbox Phase 1 的**顶层动作导向决策协议**，作为后续实现、schema 设计与 review 的上层锚点。

该协议的目标不是描述完整系统状态，而是为一次 heartbeat 中的有限编排提供一个**窄、硬、可推导**的决策接口。

Phase 1 的顶层设计以 PRD 为事实源，不反向受既有实现期 envelope 约束。后续 contract 和 schema 应从该顶层设计派生，而不是让局部实现字段反向塑造上层行为模型。

## 2. Design Position

Phase 1 的 decision contract 采用**动作导向（action-oriented）**建模，而不是状态导向（state-oriented）建模。

这意味着该协议优先回答以下问题：

- 本轮唯一主对象是谁
- 本轮主原因是什么
- 本轮是否允许自动 `open`
- 本轮是否允许在 `open` 后执行一次 lightweight `re-check`
- 当前有哪些并行 health warnings

该协议**不负责**输出完整处理状态、完整候选解释、完整排序明细或通用调度建议。

## 3. Contract Shape

Phase 1 的顶层决策协议收敛为以下 5 个核心字段：

- `target_item`
- `reason_class`
- `should_open`
- `should_recheck`
- `health_warnings`

### 3.1 `target_item`

`target_item` 表示本轮 heartbeat 中的**唯一动作对象**。

约束如下：

- `target_item` 采用**轻引用**语义
- `target_item` 只负责标识本轮唯一动作对象
- `target_item` 不承载候选详情、正文内容、解释性上下文或排序细节
- 当本轮没有合适对象时，`target_item = null`

本阶段不允许：

- 多目标并列
- 用富对象承载大段候选信息
- 用“状态描述”替代对象选择

### 3.2 `reason_class`

`reason_class` 表示本轮动作决策的**主原因**。

Phase 1 中，`reason_class` 为固定枚举，仅包括：

- `open_obligation`
- `requires_reply`
- `none`

其中：

- `open_obligation` 表示该对象具备最高优先级的自动打开义务
- `requires_reply` 表示该对象具备自动打开价值，但优先级低于 `open_obligation`
- `none` 表示本轮不存在可行动主对象

`recency` 不进入顶层 decision contract。它可以作为内部排序信号参与 `pick-mail` / `evaluate`，但不作为顶层主行为原因对外表达。

### 3.3 `should_open`

`should_open` 是显式保留的动作许可位，但其语义为**派生字段**，并不拥有独立判断空间。

其真值由 `reason_class` 唯一决定：

- `reason_class = open_obligation` → `should_open = true`
- `reason_class = requires_reply` → `should_open = true`
- `reason_class = none` → `should_open = false`

因此，`should_open` 的存在是为了下游消费便利，而不是为了引入额外判定自由度。

### 3.4 `should_recheck`

`should_recheck` 是显式保留的动作许可位，用于表示：

> 若本轮已经执行 `open`，是否允许继续执行最多一次 lightweight `re-check`。

`should_recheck` 保持为**独立字段**，不从 `reason_class` 自动派生。

原因如下：

- `open` 是主动作许可
- `re-check` 是 `open` 后的条件性附加动作
- 并非所有自动 `open` 都必须附带 `re-check`

因此，Phase 1 不将 `should_recheck` 设计为 `should_open` 或 `reason_class` 的机械派生结果。

### 3.5 `health_warnings`

`health_warnings` 属于**并行观测信息**。

其职责仅限于暴露健康性提示，不参与主决策，不改变动作优先级，也不触发额外动作。

因此：

- `health_warnings` 不改变 `target_item`
- `health_warnings` 不改变 `reason_class`
- `health_warnings` 不触发 `open`
- `health_warnings` 不触发 `re-check`

## 4. Semantic Constraints

Phase 1 的顶层动作协议满足以下语义约束：

### 4.1 Target / Reason coupling

- `target_item = null` → `reason_class = none`
- `target_item != null` → `reason_class ∈ { open_obligation, requires_reply }`

也就是说：

- 没有目标对象时，必须显式表示为 `none`
- 一旦存在目标对象，就必须给出明确的动作主因
- 不允许出现“选中了对象，但没有动作主因”的脏状态

### 4.2 Null target constraints

当 `target_item = null` 时，必须同时满足：

- `reason_class = none`
- `should_open = false`
- `should_recheck = false`

这表示“无目标对象”本身也是一种完整且显式的决策结果，而不是 contract 缺项。

### 4.3 Open derivation

`should_open` 的值必须严格由 `reason_class` 派生：

- `open_obligation` → `true`
- `requires_reply` → `true`
- `none` → `false`

不允许出现 `reason_class` 与 `should_open` 相互冲突的组合。

### 4.4 Re-check gating

`should_recheck = true` 并不意味着系统可以脱离上下文直接执行 re-check。

其成立前提仍受 Phase 1 的行为边界约束：

- 本轮必须已经执行过一次 `open`
- `re-check` 最多执行一次
- `re-check` 只能绑定到刚刚 `open` 的同一对象
- `re-check` 不得重新全局 `pick-mail`
- `re-check` 不得切换到新的目标对象

因此，`should_recheck` 表示的是**条件性许可**，不是无条件动作。

## 5. What This Contract Intentionally Excludes

该顶层协议明确不承载以下内容：

- 完整候选列表
- 完整排序过程
- 所有排除原因
- 通用调度建议
- 富对象 payload
- reply 编排信息
- 通用 memory 持久化语义

这些能力若未来需要，应在下层 schema、辅助结构或后续 phase 中单独设计，而不是塞回顶层动作协议。

## 6. Design Consequence

一旦采用该 top-level action-oriented contract，后续工作应遵循以下顺序：

1. 以该文档作为 Phase 1 的上层设计锚点
2. 从该协议派生新的 decision envelope / schema
3. 校验现有实现与新协议的偏差
4. 决定哪些字段保留、哪些字段降级、哪些字段删除
5. 以新协议为准重写后续实现与 review 基线

换句话说，当前的优先级不是“让 PRD 迁就旧 schema”，而是“让 schema 回到顶层行为设计之下”。

## 7. Summary

Phase 1 的顶层决策协议已经收敛为一个**窄、硬、动作导向**的模型：

- 一个唯一目标：`target_item`
- 一个主行为原因：`reason_class`
- 一个派生动作许可：`should_open`
- 一个独立附加许可：`should_recheck`
- 一组并行观测信息：`health_warnings`

该模型的核心价值在于：

- 不把排序信号误抬成动作原因
- 不把状态描述误写成动作协议
- 不让实现期 schema 反向绑架顶层设计
- 为 Phase 1 保持最小、清晰、可收敛的行为边界
