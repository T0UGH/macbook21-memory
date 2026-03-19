# 2026-03-19 OpenClaw Mailbox Plugin / Heartbeat 讨论总结

这份文档总结了贵平与 MT 围绕 **OpenClaw Mailbox 如何真正接入 OpenClaw 机制** 的讨论。

重点不是 mailbox skill 本身，而是：

- mailbox 到底应该被理解成什么
- 它和 skill / plugin / heartbeat / hook 的边界是什么
- 现阶段最现实的接入路径是什么

---

## 1. 关键结论：mailbox 不是一个 skill，而是一个插件能力

本轮讨论里，最重要的概念升级是：

> **mailbox 不是一个独立 skill。mailbox 应该被理解为一个 OpenClaw 插件能力。**

更准确的结构应该是：

- `openclaw-mailbox-server`
  - 业务真相
  - 状态流转
  - obligation 语义
  - API

- `openclaw-mailbox-cli`
  - agent 调用入口
  - 本地可靠性层
  - retry / pending / checkpoint / cache

- `openclaw-mailbox-skill`
  - 决策协议层
  - 负责理解 snapshot
  - 输出 decision envelope
  - 告诉主 agent 如何使用 mailbox

- **mailbox plugin / extension**
  - 才是真正把 mailbox 接进 OpenClaw 的平台扩展层
  - 负责 heartbeat / safe-point integration
  - 负责 CLI 调用适配
  - 负责把 mailbox 变成“装上就能用”的能力

结论：

> **skill 只是插件中的一层，不应承担整个 mailbox 系统的接入职责。**

---

## 2. 为什么会觉得 skill 撑不住

贵平明确指出：

- 到 CLI 层为止都没偏
- 但 skill 的语义太小，撑不住整套邮件系统

MT 认可这个判断。

原因是：

如果把这些责任都压到 skill 上：
- heartbeat 接入
- 调度触发
- runtime glue
- lifecycle 行为
- OpenClaw 平台级集成

那 skill 就会变成一个被迫承担平台职责的概念容器。

而 skill 天然更适合负责的是：
- 决策协议
- 结构化输出
- 如何解释 mailbox 状态
- 如何给主 agent 建议动作

所以问题不是 skill 做错了，而是：

> **之前把 skill 放在了过大的位置上。**

---

## 3. 对 OpenClaw heartbeat / cron / hook / plugin 的理解更新

MT 查阅 OpenClaw 官方文档与 upstream 仓库后，当前理解如下。

### 3.1 heartbeat

heartbeat 是 OpenClaw core 的内建周期性机制。

官方默认 prompt：

`Read HEARTBEAT.md if it exists (workspace context). Follow it strictly. Do not infer or repeat old tasks from prior chats. If nothing needs attention, reply HEARTBEAT_OK.`

heartbeat 的定位：
- 周期性 awareness
- 适合 inbox / calendar / notifications 等“定期抬头看一眼”的场景
- 运行在 main session 中，天然有上下文

### 3.2 cron

cron 用于：
- 精确时间调度
- 隔离 session 运行
- 一次性提醒
- 精准定时任务

### 3.3 hook

hook 用于：
- command events
- lifecycle events
- session events
- message events

hook 是事件驱动自动化，不是 heartbeat 注册机制。

### 3.4 plugin

plugin 是 OpenClaw 的平台扩展单元，负责：
- 提供能力
- 注册 channel/provider/tools/services/skills
- 把一整类能力以可安装、可升级、可隔离的方式接入 OpenClaw

当前最重要的理解更新是：

> **heartbeat 是 core 机制；plugin 提供能力；skill 定义行为协议；hook 处理事件。**

---

## 4. 对“自动注册 heartbeat behavior”的调研结论

贵平关心：

- GitHub 上是否存在 OpenClaw 插件自动注册 heartbeat behavior 的成熟例子

MT 调研后的结论：

### 4.1 找到了的东西

在 OpenClaw upstream 中，可以明确看到：

- heartbeat 本身是 core 机制
- heartbeat 主要通过 `HEARTBEAT.md` 和 `heartbeat.prompt` 定制行为
- plugin manifest 支持插件携带 `skills`
- hooks 支持事件驱动自动化

### 4.2 没找到的东西

MT 没有找到一个成熟现成的官方模式是：

- 插件安装后
- 自动向 heartbeat scheduler 正式注册一个新的 behavior contributor
- 且无需依赖 `HEARTBEAT.md`

也就是说：

> **目前没有找到可直接照抄的“plugin auto-register heartbeat behavior”成熟例子。**

### 4.3 这意味着什么

这说明 mailbox 如果要实现“安装插件后自动获得 heartbeat 行为”，很可能有两种路径：

1. **临时 bootstrap 方案**
   - 插件安装时自动写入 / 更新 `HEARTBEAT.md`

2. **长期正式方案**
   - 给 OpenClaw core 增加一个 heartbeat integration seam
   - 让插件提供 mailbox heartbeat adapter / behavior contributor

---

## 5. 对 `HEARTBEAT.md` 的理解更新

### 5.1 一般是谁写

OpenClaw 语义下，`HEARTBEAT.md` 通常是：

- 人先定义意图和边界
- agent 代写并维护

现实中的常见模式：
- 人手写
- agent 起草，人拍板
- agent 后续微调维护

### 5.2 为什么 mailbox 不能依赖用户自己写 HEARTBEAT.md

贵平指出一个核心问题：

> 如果 mailbox 接入依赖每个用户自己写 heartbeat，那它就不可复制。

MT 认可这个判断。

如果每个人都要手写 heartbeat，问题会变成：
- 各自行为不一致
- obligation 规则不一致
- 升级困难
- 无法产品化
- 安装后不能直接得到能力，只能得到说明书

因此得出关键判断：

> **如果 mailbox 想成为真正可复用的 OpenClaw 能力，`HEARTBEAT.md` 不能是主接入路径。**

它最多只能是：
- 覆盖层
- 策略层
- 用户自定义层

而不该是 mailbox 的 primary integration path。

---

## 6. 当前最现实的最简方案：插件安装时自动写入 HEARTBEAT.md

在没有现成 heartbeat extension seam 的情况下，贵平提出了一个最简方案：

> 插件安装时，自动往 `HEARTBEAT.md` 里加几句话。

MT 对这个方案的判断：

### 6.1 作为短期方案：可行

它是一个可接受的：
- Phase 0
- MVP
- bootstrap strategy

优点：
- 不需要先改 OpenClaw core
- 插件装上即可马上让 heartbeat 开始参与 mailbox check
- 比继续抽象讨论更容易快速验证产品闭环

### 6.2 作为长期方案：不够好

原因：
- 它本质上仍是文本注入，而不是平台 contract
- 会和用户自定义产生冲突
- 升级 / merge / 回滚会麻烦
- 容易误把临时方案当最终架构

### 6.3 如果要这么做，必须满足的机制约束

MT 提出，如果走这条路，必须：

#### (1) 使用明确 marker 包裹托管片段
例如：

```md
<!-- OPENCLAW:MAILBOX:BEGIN -->
...
<!-- OPENCLAW:MAILBOX:END -->
```

这样才能：
- 幂等插入
- 可更新
- 可删除
- 避免重复写入

#### (2) 注入最小 block
只注入最小 mailbox heartbeat 逻辑，不写大段说明书。

#### (3) 明确这是“插件托管内容”
用户可以看，但 marker 不应随意破坏。

#### (4) 卸载/禁用时可回滚
插件不应只写不收。

因此，当前阶段最准确的定义是：

> **插件自动写入 HEARTBEAT.md 是一个可接受的 bootstrap 方案，但不是最终 heartbeat registration model。**

---

## 7. 当前最稳定的架构判断

综合本轮讨论，当前最稳定的判断是：

### 7.1 mailbox 的正确定位

mailbox 应被理解为：

> **一个 OpenClaw 插件能力**

而不是：
- 一个独立 skill
- 一个 HEARTBEAT.md 模板
- 一个 agent 私有流程手册

### 7.2 skill 的正确定位

mailbox skill 只是插件中的“决策协议层”。

负责：
- 定义 snapshot
- 定义 decision envelope
- 定义 how to use mailbox

不负责：
- heartbeat 注册
- 平台调度
- lifecycle glue
- runtime 编排

### 7.3 heartbeat 的正确定位

heartbeat 是：
- OpenClaw core 已有的周期性入口
- mailbox 插件应该利用它，而不是重写它

### 7.4 HEARTBEAT.md 的正确定位

`HEARTBEAT.md` 应降级成：
- 用户策略覆盖层
- agent 可维护的 checklist
- 插件 bootstrap 过渡层

而不应成为 mailbox 的最终主接入方式。

---

## 8. 当前阶段建议

MT 当前建议：

### 短期（可落地）
- 允许 mailbox plugin 通过托管 marker 向 `HEARTBEAT.md` 注入最小 mailbox block
- 用来快速验证 heartbeat 接入闭环

### 中期（更合理）
- 继续研究 OpenClaw core 是否应补一个正式的 heartbeat integration seam
- 例如：
  - heartbeat behavior contributor
  - mailbox heartbeat adapter
  - plugin-owned heartbeat service invoked by core

### 长期（正确方向）
- mailbox 插件安装后，应自动获得默认 heartbeat 行为
- `HEARTBEAT.md` 仅作为用户 override
- 而不是 primary integration path

---

## 9. 一句话总结

本轮讨论最重要的共识是：

> **mailbox 不是一个 skill，而是一个插件能力；skill 只是这个插件里的决策协议层。**

以及：

> **在没有正式 heartbeat extension seam 之前，插件自动托管写入 `HEARTBEAT.md` 可以作为 bootstrap 方案，但不应被误认为最终架构。**
