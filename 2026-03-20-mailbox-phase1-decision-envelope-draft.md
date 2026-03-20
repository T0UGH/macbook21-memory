# Mailbox Phase 1：Decision Envelope Draft

## 1. Purpose

本文档定义 Mailbox Phase 1 的 **decision envelope 第一稿**。

该 envelope 从 `2026-03-20-mailbox-phase1-top-level-action-contract.md` 派生，用于把顶层动作导向设计落成一个可实现、可测试、可 review 的结构化协议。

本稿的目标不是覆盖未来所有 mailbox 决策场景，而是为 Phase 1 提供一个**最小、明确、动作导向**的正式 contract。

## 2. Schema Draft

```ts
type Phase1DecisionEnvelope = {
  target_item: TargetItemRef | null;
  reason_class: "open_obligation" | "requires_reply" | "none";
  should_open: boolean;
  should_recheck: boolean;
  health_warnings: HealthWarning[];
};

type TargetItemRef = {
  message_id: string;
};

type HealthWarning = {
  code: string;
  message: string;
};
```

## 3. Field Semantics

### 3.1 `target_item`

`target_item` 表示本轮 heartbeat 中的**唯一动作对象**。

约束如下：

- `target_item` 使用轻引用语义
- Phase 1 中只保留 `message_id`
- 不承载正文、摘要、上下文或排序细节
- 没有合适对象时，`target_item = null`

### 3.2 `reason_class`

`reason_class` 表示本轮动作决策的**主原因**。

Phase 1 固定为以下枚举：

- `open_obligation`
- `requires_reply`
- `none`

说明：

- `open_obligation`：该对象具备最高优先级的自动打开义务
- `requires_reply`：该对象具备自动打开价值，但优先级低于 `open_obligation`
- `none`：本轮不存在可行动主对象

`recency` 不进入顶层 envelope。它只作为内部排序信号存在。

### 3.3 `should_open`

`should_open` 是显式保留的动作许可位，但它是**派生字段**。

其真值由 `reason_class` 唯一决定：

- `open_obligation` -> `true`
- `requires_reply` -> `true`
- `none` -> `false`

它保留在 envelope 中是为了让下游消费和测试断言更直接。实现中必须保证 `should_open` 与 `reason_class` 的派生关系不可被覆盖。

### 3.4 `should_recheck`

`should_recheck` 是显式保留的动作许可位，用于表达：

> 若本轮已经执行 `open`，是否允许继续执行最多一次 lightweight `re-check`。

`should_recheck` 在 Phase 1 中是**独立字段**，不从 `reason_class` 自动派生。

原因如下：

- `open` 是主动作许可
- `re-check` 是 `open` 后的条件性附加动作
- 并非所有自动 `open` 都需要附带 `re-check`

Phase 1 中，`should_recheck` 的判定规则固定为：

- `reason_class = "open_obligation"` -> `should_recheck = true`
- `reason_class = "requires_reply"` -> `should_recheck = false`
- `reason_class = "none"` -> `should_recheck = false`

这条规则定义的是 **Phase 1 的固定判定表**，并不改变 `should_recheck` 在设计上的独立语义地位。

### 3.5 `health_warnings`

`health_warnings` 属于并行观测信息。

其中：

- `code`：稳定、可比较、可测试的 warning 类型
- `message`：辅助解释字段，不参与任何主决策，不作为逻辑判断依据

Phase 1 中，`health_warnings.code` 固定为以下枚举：

- `failed_outbound`
- `failed_seen`
- `failed_delivery`
- `sync_error`
- `pending_outbound`

warning 的存在：

- 不改变 `target_item`
- 不改变 `reason_class`
- 不改变 `should_open`
- 不改变 `should_recheck`

## 4. Invariants

### 4.1 Target / Reason coupling

- `target_item = null` -> `reason_class = "none"`
- `target_item != null` -> `reason_class ∈ { "open_obligation", "requires_reply" }`

不允许出现：

- `target_item = null` 但 `reason_class != "none"`
- `target_item != null` 但 `reason_class = "none"`

### 4.2 Null target constraints

当 `target_item = null` 时，必须同时满足：

- `should_open = false`
- `should_recheck = false`

这表示“无目标对象”是一种显式且完整的决策结果，而不是 contract 缺项。

### 4.3 Open derivation

`should_open` 必须严格由 `reason_class` 派生：

- `reason_class = "open_obligation"` -> `should_open = true`
- `reason_class = "requires_reply"` -> `should_open = true`
- `reason_class = "none"` -> `should_open = false`

不允许出现 `reason_class` 与 `should_open` 相互冲突的组合。

### 4.4 Re-check gating (runtime constraint)

`should_recheck = true` 仅表示条件性许可，而不是无条件动作。

其成立仍受以下边界约束：

- 本轮必须已经执行过一次 `open`
- `re-check` 最多执行一次
- `re-check` 只能绑定到当前 `target_item`
- `re-check` 不得重新全局 `pick-mail`
- `re-check` 不得切换到新的目标对象

## 5. Example Shapes

### 5.1 Open obligation case

```json
{
  "target_item": { "message_id": "msg_123" },
  "reason_class": "open_obligation",
  "should_open": true,
  "should_recheck": true,
  "health_warnings": []
}
```

### 5.2 Requires reply case

```json
{
  "target_item": { "message_id": "msg_456" },
  "reason_class": "requires_reply",
  "should_open": true,
  "should_recheck": false,
  "health_warnings": [
    {
      "code": "stale_status_snapshot",
      "message": "status snapshot may be outdated"
    }
  ]
}
```

### 5.3 No-target case

```json
{
  "target_item": null,
  "reason_class": "none",
  "should_open": false,
  "should_recheck": false,
  "health_warnings": []
}
```

## 6. What This Envelope Intentionally Excludes

本稿明确不包含以下内容：

- 完整候选列表
- 完整排序过程
- 所有排除原因
- 富对象 payload
- 通用调度建议
- reply 编排信息
- 通用 memory 持久化语义
- `recency` 等内部排序信号的对外暴露

这些内容若未来需要，应在下层结构或后续 phase 中单独设计，而不是塞回 Phase 1 的顶层 envelope。

## 7. Design Consequence

一旦采用本 envelope 草稿，后续工作应遵循以下顺序：

1. 以顶层 action-oriented contract 为设计锚点
2. 用本 envelope 替换旧的状态导向 envelope 思路
3. 对比现有实现与本稿的偏差
4. 决定需要删除、降级或改名的旧字段
5. 以本稿为准重写后续实现与 review 基线

## 8. Summary

Phase 1 的 decision envelope 第一稿已经收敛为一个最小动作协议：

- `target_item`：唯一动作对象
- `reason_class`：主行为原因
- `should_open`：派生动作许可
- `should_recheck`：独立附加许可
- `health_warnings`：并行观测信息

这份 envelope 的核心价值在于：

- 保持顶层设计与 PRD 一致
- 避免状态导向字段污染动作协议
- 为 Phase 1 提供一个足够小、足够硬、足够可测试的正式 contract
