# Mailbox PRD Phase 1：行为规则与边界（精修版）

## 1. Objective

Phase 1 的目标是先跑通 mailbox 与 OpenClaw 的基础闭环，优先保证方案可运行、可验证、边界清晰。

本阶段目标是将目标 mail 推进到“已打开、已理解、可继续处理”的状态，而不是在第一版中引入完整策略系统、通用调度框架或完整审计能力。

## 2. Integration Model

Phase 1 通过**插件托管写入 `HEARTBEAT.md`** 完成集成。

托管规则如下：

- 使用 marker 包裹托管 block
- block 带版本号
- block 追加在文件末尾
- 升级时整块替换
- marker 内内容由插件完全托管
- 插件禁用或卸载时自动删除托管 block

托管 block 仅写入**最小调用指令**，不复制完整行为规则。

该 block 的唯一语义是：**调用 mailbox heartbeat skill**。

## 3. Capability Positioning

mailbox heartbeat skill 属于**主 agent 的内生行为能力**。

Phase 1 中，它不是：

- subagent
- 外挂建议层
- 独立长期运行调度器

其职责是在一次 heartbeat 中完成**有限、受控、可停止**的 mailbox 编排。

## 4. Execution Model

Phase 1 中，heartbeat skill 的允许执行链固定为：

1. `pick-mail`
2. `status`
3. `evaluate`
4. 按规则决定是否 `open`
5. 如有必要，执行最多一次 lightweight `re-check`
6. 停止

Phase 1 不允许将该执行链扩展为持续循环、多轮调度或递归执行。

## 5. Decision Rules

### 5.1 Priority Order

Phase 1 采用固定优先级规则：

- `open_obligation > requires_reply > recency`

各类信号语义如下：

- `open_obligation`：最高优先级，可触发自动 `open`
- `requires_reply`：次高优先级，可触发自动 `open`
- `recency`：仅作为排序信号，不可单独触发自动 `open`

自动 `open` 仅允许由以下原因触发：

- `open_obligation`
- `requires_reply`

### 5.2 Overlap Rule

若同一候选同时命中 `open_obligation` 与 `requires_reply`，则统一按 `open_obligation` 解释，并作为：

- 行为原因
- 记录原因
- 调试原因

## 6. Evaluate Contract

`evaluate` 在 Phase 1 中只负责输出最小结构化判定，不承担完整解释系统、审计系统或可观测性平台职责。

其输出应限定为：

- `top_candidate`
- `reason_class`
- `should_open`
- `should_recheck`
- `health_warnings`

其中，`reason_class` 在 Phase 1 中为固定枚举，至少包括：

- `open_obligation`
- `requires_reply`
- `recency`
- `none`

Phase 1 中，`evaluate` 不要求：

- 输出完整排序过程
- 输出所有候选排除原因
- 输出完整推理明细
- 提供完整审计能力

## 7. Re-check Rules

`re-check` 在 Phase 1 中必须满足以下约束：

- 最多执行一次
- 只有本轮已发生一次 `open`，才允许执行 `re-check`
- 若本轮未发生 `open`，则不得执行 `re-check`

`re-check` 的作用域必须绑定到**刚刚 `open` 的同一 mail / 同一上下文**。

`re-check` 不得：

- 重新全局 `pick-mail`
- 重新发起候选竞争
- 切换到新的 top candidate

`re-check` 的目的仅限于确认打开后的关键状态是否发生变化，以及当前对象是否进入“已打开、已理解、可继续处理”的稳定状态。

## 8. Stop Conditions

heartbeat skill 在 Phase 1 中必须具有强硬停止边界。

停止条件如下：

- 最多一次 `open`
- 最多一次 `re-check`
- 达到上述边界后必须停止

Phase 1 不允许将 heartbeat skill 演化为小循环引擎或持续调度器。

## 9. Success Criteria

Phase 1 的成功结果定义为：

> 系统能够把目标 mail 推进到“已打开、已理解、可继续处理”的状态。

该定义在本阶段属于**行为目标**，不是严格、完备、机器可验证的 success contract。

Phase 1 不要求 heartbeat skill 成为状态证明器，也不要求提供完整 machine-checkable 成功判定协议。

## 10. Health Warnings

`health_warnings` 属于并行暴露的观测信息，不属于主行为决策面。

因此，health warning：

- 不得触发额外 `open`
- 不得触发额外 `re-check`
- 不得改变主优先级
- 不得切换候选

其职责仅限于提示和观测，不参与主执行路径决策。

## 11. Silent-by-default and Internal Trace

当本轮 heartbeat 没有需要对用户暴露的结果时，系统应对用户保持静默。

同时，系统允许保留内部轻量痕迹，用于最小必要的运行状态记录和后续调试。

这些痕迹的存放位置限定为：

- mailbox 自己的 runtime/state

Phase 1 中，这些痕迹：

- 不写入通用 memory
- 不污染 OpenClaw 公共记忆层

## 12. Non-goals / Out of Scope

Phase 1 明确不包含以下能力：

### 12.1 Strategy Configuration

- 不支持用户自定义自动 `open` 条件
- 不支持用户调整 `re-check` 策略
- 不支持用户配置 health warning 对决策的影响

### 12.2 Multi-round Scheduling

- 不实现持续循环
- 不实现递归推进
- 不实现自驱调度

### 12.3 Candidate Switching

- 不在同一轮中切换到新的 top candidate
- `re-check` 不重新进行全局候选竞争

### 12.4 Full Explanation / Audit System

- 不输出完整排序明细
- 不输出所有候选排除原因
- 不承担完整可观测性平台职责
- 不承担完整审计职责

### 12.5 General Memory Persistence

- 不将内部轻量痕迹写入 OpenClaw 通用 memory
- 不将 mailbox runtime/state 扩展成公共记忆层的一部分

### 12.6 Automatic Reply Orchestration

- reply 明确拆出 heartbeat skill
- Phase 1 不在 heartbeat 内直接推进 reply 执行链

### 12.7 Generalized Plugin Platforming

- Phase 1 只解决 mailbox 闭环问题
- 不将本版方案扩展成通用 heartbeat orchestration framework
