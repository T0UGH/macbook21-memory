# Mailbox PRD Phase 1：行为规则与边界

## Phase 1 行为规则与边界

### 1. 目标

Phase 1 的目标是先跑通 mailbox 与 OpenClaw 的基础闭环，优先保证方案可运行、可验证、边界清晰，而不是一开始就建设完整策略系统或通用调度框架。

系统需要能够通过受控的 heartbeat 集成，把目标 mail 推进到“已打开、已理解、可继续处理”的状态。

### 2. 集成方式

Phase 1 采用**插件托管写入 `HEARTBEAT.md`** 的方式完成接入。

托管机制如下：

- 使用 marker 包裹托管 block
- block 带版本号
- block 追加在文件末尾
- 升级时整块替换
- marker 内内容由插件完全托管
- 插件卸载或禁用时自动删除托管 block

托管 block 只写**最小调用指令**，不复制完整行为规则。
该 block 的语义是：**调用 mailbox heartbeat skill**。

### 3. Skill 定位

mailbox heartbeat skill 属于**主 agent 的内生行为能力**。

Phase 1 中，它不是：

- subagent
- 外挂建议层
- 独立长期运行调度器

它的职责是在一次 heartbeat 中完成**有限、受控、可停止**的 mailbox 编排。

### 4. 执行链路

Phase 1 中，heartbeat skill 的允许执行链固定为：

1. `pick-mail`
2. `status`
3. `evaluate`
4. 按规则决定是否 `open`
5. 如有必要，执行最多一次 lightweight `re-check`
6. 停止

Phase 1 不允许将该链路扩展为持续循环、多轮调度或递归执行。

### 5. 行为优先级规则

Phase 1 采用固定优先级规则：

- `open_obligation > requires_reply > recency`

其中：

- `open_obligation`：最高优先级，可触发自动 `open`
- `requires_reply`：次高优先级，可触发自动 `open`
- `recency`：仅作为排序信号，不可单独触发自动 `open`

自动 `open` 仅允许由以下两类原因触发：

- `open_obligation`
- `requires_reply`

如果同一候选同时命中 `open_obligation` 与 `requires_reply`，则统一按 `open_obligation` 解释，并作为行为原因、记录原因和调试原因。

### 6. Evaluate 输出边界

`evaluate` 在 Phase 1 中只负责输出最小结构化判定，不承担完整解释系统或审计系统职责。

其输出应收敛为以下字段：

- `top_candidate`
- `reason_class`
- `should_open`
- `should_recheck`
- `health_warnings`

其中 `reason_class` 在 Phase 1 中为固定枚举，至少包括：

- `open_obligation`
- `requires_reply`
- `recency`
- `none`

Phase 1 中，`evaluate` 不要求：

- 输出完整排序过程
- 输出所有候选排除原因
- 输出完整推理明细
- 承担完整可观测性或审计职责

### 7. Re-check 边界

Phase 1 中，`re-check` 必须满足以下约束：

- 最多执行一次
- 只有本轮已经发生过一次 `open`，才允许执行 `re-check`
- 如果本轮没有发生 `open`，则不得执行 `re-check`

此外，`re-check` 的作用域必须绑定到**刚刚 `open` 的同一 mail / 同一上下文**。

`re-check` 不得：

- 重新全局 `pick-mail`
- 重新发起候选竞争
- 切换到新的 top candidate

`re-check` 的目的仅是确认打开后状态是否发生关键变化，以及当前对象是否进入“已打开、已理解、可继续处理”的稳定状态。

### 8. 停止条件

Phase 1 的 heartbeat skill 必须具有强硬停止边界。

停止条件如下：

- 最多一次 `open`
- 最多一次 `re-check`
- 达到上述边界后必须停止

Phase 1 不允许把 heartbeat skill 演化成小循环引擎或持续调度器。

### 9. 成功结果定义

Phase 1 的成功结果定义为：

> 系统能够把目标 mail 推进到“已打开、已理解、可继续处理”的状态。

这一成功定义在 PRD 中应被视为**行为目标**，而不是严格、完备、机器可验证的 success contract。

Phase 1 不要求 heartbeat skill 成为状态证明器，也不要求提供完整 machine-checkable 成功判定协议。

### 10. Health Warning 地位

`health_warnings` 属于并行暴露的观测信息，不属于主行为决策面。

因此，health warning：

- 不得触发额外 `open`
- 不得触发额外 `re-check`
- 不得改变主优先级
- 不得切换候选

它只用于提示和观测，不参与主执行路径决策。

### 11. 无事静默与内部痕迹

当本轮 heartbeat 没有需要对用户暴露的结果时，系统应对用户保持静默。

同时，系统允许保留内部轻量痕迹，用于最小必要的运行状态记录和后续调试。

这些痕迹的存放位置限定为：

- mailbox 自己的 runtime/state

Phase 1 中，这些痕迹：

- 不写入通用 memory
- 不污染 OpenClaw 公共记忆层

### 12. Non-goals / Out of Scope

Phase 1 明确不包含以下能力：

#### 12.1 策略配置系统

- 不支持用户自定义自动 `open` 条件
- 不支持用户调整 `re-check` 策略
- 不支持用户配置 health warning 对决策的影响

#### 12.2 多轮调度循环

- 不实现持续循环
- 不实现递归推进
- 不实现自驱调度

#### 12.3 多候选切换

- 不在同一轮中切换到新的 top candidate
- `re-check` 不重新进行全局候选竞争

#### 12.4 完整解释 / 审计系统

- 不输出完整排序明细
- 不输出所有候选排除原因
- 不承担完整可观测性平台职责
- 不承担完整审计职责

#### 12.5 通用 memory 持久化

- 不将内部轻量痕迹写入 OpenClaw 通用 memory
- 不将 mailbox runtime/state 扩展成公共记忆层的一部分

#### 12.6 自动 reply 编排

- reply 明确拆出 heartbeat skill
- Phase 1 不在 heartbeat 内直接推进 reply 执行链

#### 12.7 通用插件平台化

- Phase 1 只解决 mailbox 闭环问题
- 不将本版方案扩展成通用 heartbeat orchestration framework
