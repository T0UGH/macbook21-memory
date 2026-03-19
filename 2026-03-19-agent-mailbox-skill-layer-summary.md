# 2026-03-19 Agent Mailbox Skill 层设计汇总

这份文档汇总了 **MT** 与 **Hex** 围绕 `openclaw-mailbox-skill` 的讨论结论、共同判断、分歧点与建议的下一步。

目标不是重复聊天，而是把 skill 层最关键的设计判断收口，方便贵平 review。

---

## 1. 结论先说

### 1.1 双方一致认可的核心判断

1. **skill 层不是包装层，而是行为协议层 / 决策编排层**
   - server 提供业务真相
   - cli 提供本地可靠性层
   - **skill 决定主 agent 何时、为什么、以什么纪律使用 mailbox**

2. **Phase 1 的 skill 不应做持续自治循环**
   - 不应该自动反复检查
   - 不应该自动多轮收敛
   - 不应该自动 open / reply / 再次扫描下一封
   - 第一阶段应是：**单次调用，单次建议，由主 agent 决定是否采纳**

3. **skill 输出必须是结构化建议，而不是纯 prose prompt**
   - skill 的输出不能只是一段“建议你看看邮箱”的自然语言
   - 更合理的是：
     - 是否建议现在处理
     - 优先级
     - 建议处理哪一项
     - 推荐动作序列
     - defer 理由
     - 本地健康警告

4. **主 agent 必须保留调度权**
   - skill 可以建议
   - skill 可以推荐具体 CLI 动作
   - 但 skill 不应直接替主 agent 做调度决策
   - 不应演化成“二级调度器”

5. **skill 层应优先体现 obligation pressure，而不是被 inbox noise 拖走**
   - open obligation 优先于普通 unread
   - skill 需要显式区分：
     - 真正需要负责的 obligation
     - 仅供参考的信息流 unread

6. **第一阶段 integration 仍应围绕 heartbeat + coarse safe-point**
   - 触发时机不应过密
   - 建议触发点：heartbeat、safe-point、任务空档、处理后 lightweight re-check

---

## 2. MT 的主要建议

### 2.1 skill 的核心产物应是 decision envelope

MT 的核心判断：

> skill 层真正的输出，不应该是 prompt prose，而应该是一个结构化的 **mailbox decision envelope**。

它至少应回答：

- 当前是否建议处理 mailbox
- 当前 urgency 是什么
- 当前最高优先级项是什么
- 推荐动作序列是什么
- 如果不建议处理，defer 原因是什么
- 当前是否存在风险/阻塞/本地健康问题

### 2.2 skill 第一阶段应是“单次建议模型”

MT 强调：

- 第一阶段不应做自动多轮 agent loop
- 不应让 skill 夺走调度权
- 主 agent 在单次调用 skill 后，自己决定是否采纳
- 如果采纳并处理完成，再做一次 lightweight re-check 即可

### 2.3 skill 需要明确区分 obligation 与 noise

这是 MT 认为 mailbox skill 是否真正有价值的核心：

- 如果 skill 只是把 unread 重新排个序，那价值有限
- skill 真正该做的是把 **需要承担责任的 obligation** 从一般收件箱噪音里抠出来
- 主 agent 需要看到的是：
  - 哪些事现在该处理
  - 哪些只是稍后可读的信息

### 2.4 应考虑稳定的 schema

MT 倾向认为：

- skill 最终不应依赖“临时跑几条 CLI，再靠 prompt 拼理解”
- 应优先收口出稳定的 schema
- 这样才便于测试、验收、长期维护

---

## 3. Hex 的主要建议

Hex 整体上认可 MT 的方向，并把很多判断进一步收敛成了更工程化的输入输出模型。

### 3.1 Hex 认同：skill 是行为协议层 / 决策编排层

Hex 的表述很到位：

- server 像数据库
- cli 像 SQL 客户端
- **skill 像业务逻辑层**

没有 skill，主 agent 等于在“裸写 SQL”。

这个类比是准确的，说明 Hex 理解到了 skill 的系统位置。

### 3.2 Hex 提出 Phase 1 的最小输入应是组合 snapshot

Hex 反对把 skill 输入做成单条 CLI 输出，而是建议先收成一个最小组合快照：

```ts
interface MailboxSnapshot {
  // pick-mail 提供焦点候选
  candidates: PickMailCandidate[];
  total_unread: number;
  open_obligations_count: number;

  // status 提供本地健康度
  pending_outbound: number;
  failed_outbound: number;
  pending_seen: number;
  failed_seen: number;
  pending_delivery: number;
  failed_delivery: number;
  last_sync_status: 'ok' | 'error' | null;
}
```

他的理由：

- `pick-mail` 提供注意力焦点
- `status` 提供本地健康度
- 两者组合就够 skill 做第一阶段判断
- 不需要一开始就拉全量 inbox / thread 详情

这个建议的优点是：
- 简洁
- 工程可落地
- Phase 1 足够用
- 不会让 skill 输入膨胀得过早

### 3.3 Hex 提出 Phase 1 的最小输出应是 decision envelope

Hex 给出了一个很成型的输出草图：

```ts
interface MailboxDecisionEnvelope {
  should_process: boolean;
  urgency: 'high' | 'normal' | 'low' | 'none';

  priority_item: {
    message_id: string;
    thread_id: string;
    reason: 'open_obligation' | 'thread_continuation' | 'recency';
    from: string;
    subject: string;
    age_seconds: number;
  } | null;

  recommended_actions: string[];
  defer_reason: string | null;
  health_warnings: string[];
  snapshot_at: string;
  next_check_hint: 'after_current_task' | 'next_heartbeat' | 'immediate';
}
```

这套定义的价值在于：

- `should_process` 保证决策明确
- `urgency` 支持主 agent 判断是否打断当前任务
- `recommended_actions` 能直接落到 CLI 动作
- `health_warnings` 把本地可靠性问题单独抬出来
- `next_check_hint` 给后续节奏控制提供接口

### 3.4 Hex 明确建议：skill 输入输出 schema 应先定，再写代码

Hex 的判断很明确：

- snapshot schema / decision schema **应该先定稿**
- 最好用：
  - TypeScript interface
  - Zod schema
- 把它们当成 `openclaw-mailbox-skill` 的核心协议定义

这个建议是正确的，尤其适合 mailbox 这种容易因“感觉上有道理”而越写越散的系统。

### 3.5 Hex 对 skill 过火点的判断

Hex 点出了三个高风险陷阱：

1. **skill 自己做多轮收敛**
   - 这会直接吃掉主 agent 调度权

2. **skill 内部维护复杂状态**
   - 例如记住上次建议、升级语气、跟踪是否采纳
   - 这会把 skill 变成 stateful 的二级调度器

3. **urgency 判断过于激进**
   - 不能只要有 obligation 就一律 high
   - 应结合 age / type / 上下文做判断

这三个提醒都很有价值，尤其是第二个：
**Phase 1 skill 应尽量是纯函数，而不是有状态协调器。**

### 3.6 Hex 的能力分期建议

#### Phase 1：最小决策包
- 输入：`MailboxSnapshot`
- 输出：`MailboxDecisionEnvelope`
- 触发：heartbeat / safe-point / 任务间隙
- 行为：单次调用、无状态、不执行 CLI 命令

#### Phase 2：上下文感知
- 加入 obligations 列表与 thread 上下文
- `recommended_actions` 可扩展成多步动作序列
- 增加处理后的 lightweight re-check
- 增加类似 `defer_until` 的控制字段

#### Phase 3：协作感知
- 引入多 agent obligation 交叉视图
- 提供协作建议
- 可考虑处理 receipt / memory receipt 集成
- 仍然保持建议模式，不走自治循环

---

## 4. 双方基本一致的设计骨架

综合 MT 与 Hex 的建议，目前 skill 层的骨架已经比较清楚：

### 4.1 触发层
- heartbeat
- coarse safe-point
- 主 agent 空档
- 处理后的单次 lightweight re-check

### 4.2 输入层
Phase 1 先用最小组合快照：
- `pick-mail`
- `status`

后续再扩展：
- obligations 列表
- thread 上下文
- 协作上下文

### 4.3 决策层
核心问题：
- 现在要不要处理 mailbox
- 是否值得打断当前任务
- 当前最优先对象是谁
- 是业务 obligation 问题，还是本地健康问题

### 4.4 输出层
输出一个结构化 envelope，包含：
- should_process
- urgency
- priority_item
- recommended_actions
- defer_reason
- health_warnings
- next_check_hint

### 4.5 执行边界
- skill **可以**读取 mailbox 状态
- skill **可以**输出结构化建议
- skill **可以**推荐 CLI 序列
- skill **不可以**直接执行 CLI
- skill **不可以**自动多轮处理
- skill **不可以**打断主 agent 当前任务
- skill **不应**在 Phase 1 维护跨调用状态

---

## 5. 当前最值得拍板的几个问题

虽然大方向已经接近一致，但还需要贵平来拍板的关键点主要有这些：

### 5.1 Phase 1 的 snapshot 是否就先限定为 `pick-mail + status`

这是一个很关键的收口点。

我的判断：**值得先这样做。**

原因：
- 足够支撑第一阶段决策
- 避免 skill 一开始就变成“重组一堆接口的大对象”
- 留出 server / cli 后续演进空间

### 5.2 urgency 是否先保持粗粒度

当前建议值：
- `high`
- `normal`
- `low`
- `none`

我的判断：**先保持粗粒度是对的。**

不要一开始引入复杂评分系统，不利于稳定。

### 5.3 recommended_actions 是否允许输出具体 CLI 命令

我的判断：**应该允许，而且应该鼓励。**

因为 skill 的价值之一，就是把建议落到可执行动作上。
但边界要清楚：
- 可以推荐
- 不能代执行

### 5.4 Phase 1 是否禁止 stateful skill

我的判断：**是，建议明确禁止。**

这点其实非常重要。
否则 skill 很快会变成第二个 scheduler，系统会失控。

---

## 6. 我的综合建议（给贵平）

如果现在要把这轮讨论收成一个可以落地的方向，我建议这样定：

### 建议 A：先把 skill Phase 1 收成“纯函数决策器”

定义：
- 输入：最小 `MailboxSnapshot`
- 输出：`MailboxDecisionEnvelope`
- 特性：无状态、无副作用、单次建议

这是最稳的切入点。

### 建议 B：先写 schema，再写 skill 实现

建议产出物顺序：

1. `snapshot schema`
2. `decision envelope schema`
3. Phase 1 决策规则文档
4. skill 实现
5. skill 测试用例

不要反过来。

### 建议 C：把“health warnings”与“business priority”分开

这是 Hex 这次回应里很好的一个点。

我建议明确分成两条通道：
- **业务通道**：obligation / unread / priority item
- **本地健康通道**：failed outbound / failed seen / failed delivery / sync error

这样 skill 才不会把“消息该不该处理”和“本地状态是否异常”混成一锅。

### 建议 D：把 Phase 2/3 写成演进路线，但先不实现

可以先文档化：
- Phase 2：上下文感知
- Phase 3：协作感知

但当前实现应严格限制在 Phase 1。

这是防止 skill 层设计过火的最好办法。

---

## 7. 建议的下一步

如果贵平认可这轮收口，接下来最合理的动作顺序应是：

1. **先定 Phase 1 的 schema**
   - `MailboxSnapshot`
   - `MailboxDecisionEnvelope`

2. **再写一份 Phase 1 决策规则文档**
   - obligation 优先规则
   - health warning 提升规则
   - urgency 粗粒度规则
   - defer 规则
   - next_check_hint 规则

3. **再让 Hex 开始实现 `openclaw-mailbox-skill` 的最小版本**
   - 只做 Phase 1
   - 不做 stateful
   - 不做自动多轮
   - 不做代执行

4. **MT 再做一次协议级 review**
   - 重点看边界有没有串味
   - 看输出 schema 是否足够稳
   - 看规则是否过度复杂

---

## 8. 一句话总结

这轮讨论最大的收获是：

> `openclaw-mailbox-skill` 的第一阶段，已经可以明确收敛为一个 **无状态、无副作用、输入 snapshot、输出 decision envelope 的纯函数决策层**。

这是一个足够稳、足够工程化、也足够容易验证的起点。
