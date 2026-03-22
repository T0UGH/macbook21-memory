# 2026-03-19 Mailbox Heartbeat Bootstrap Phase 1 拍板稿

这份文档记录贵平与 MT 对 **mailbox 插件如何通过托管写入 `HEARTBEAT.md` 来快速跑通第一版集成** 的正式拍板结论。

目标不是最终架构，而是：

> **先让这套系统 run 起来，同时保持可升级、可回滚、可替换。**

---

## 1. 总体定位

当前阶段不追求正式 heartbeat extension seam。

Phase 1 采用：

> **mailbox plugin 安装时，向 `HEARTBEAT.md` 托管写入一个 mailbox block。**

这个方案的定位是：

- bootstrap 方案
- MVP 方案
- 过渡性集成方案

它不是最终 heartbeat registration model。

---

## 2. 托管 block 的 marker 形态

托管 block 采用固定 marker：

```md
<!-- OPENCLAW:MAILBOX:BEGIN version=1 -->
...
<!-- OPENCLAW:MAILBOX:END -->
```

### 目的

这个 marker 必须支持：
- 幂等安装
- 检测是否已存在
- 整块升级替换
- 精确删除
- 版本演进

---

## 3. 安装策略

### 3.1 文件不存在
如果 `HEARTBEAT.md` 不存在，则直接创建文件并写入托管 block。

### 3.2 文件已存在
如果 `HEARTBEAT.md` 已存在，则：

- 不尝试智能插位
- 不猜测用户结构
- **直接追加到文件末尾**

这是当前最稳的 Phase 1 策略。

---

## 4. 升级策略

如果再次安装或插件升级时，发现已存在：

```md
<!-- OPENCLAW:MAILBOX:BEGIN version=1 -->
...
<!-- OPENCLAW:MAILBOX:END -->
```

则默认行为为：

> **整块替换升级**

即：
- 保留 marker
- 用新内容整体覆盖 block 内部内容
- 版本号升级到新版本

当前不采用“只提示不替换”的保守策略。

---

## 5. 用户手改策略

当前拍板：

> **marker 内就是插件托管区。**

这意味着：
- 用户即使手改了 block 内内容
- 后续安装 / 升级时也仍然按托管内容直接覆盖

用户自定义应写在 marker 外部，而不是 marker 内部。

当前不支持：
- diff merge
- 半托管
- 用户改了就退出托管

---

## 6. 卸载 / 禁用策略

mailbox plugin 禁用或卸载时：

> **自动删除整个托管 block**

形成完整生命周期：

- 安装 → 写入
- 升级 → 替换
- 禁用/卸载 → 删除

当前不采用“保留并提示用户手工清理”的策略。

---

## 7. Block 内容策略

Phase 1 的托管 block 只写：

> **最小调用指令**

不把完整 mailbox 规则复制到 `HEARTBEAT.md`。

### 不应写入的内容

不应在 block 中写入：
- 完整 urgency 规则
- 完整 recommended_actions 规则
- 完整 health warning 规则
- 大段设计哲学或实现说明

原因：
- 这些真相应尽量留在 plugin / skill / CLI 内部
- `HEARTBEAT.md` 只是 bootstrap 接入口，不应成为第二份规范文档

---

## 8. Block 中不写“插件”，而写“skill”

本轮讨论的重要纠偏：

`HEARTBEAT.md` 是给 agent 看的运行指令，不是给系统看的架构说明。

因此，block 中不应写：
- “调用 mailbox 插件”
- “执行某个抽象插件能力”
- “执行某个过胖的单一 CLI 命令”

而应写成：

> **执行 mailbox heartbeat skill**

原因：
- skill 在 agent 语义上更顺
- skill 适合作为编排层
- 插件负责平台接入，skill 负责行为编排

---

## 9. mailbox heartbeat skill 的职责定位

当前拍板：

> **mailbox heartbeat flow 应该是一个 skill。**

但它不是整个 mailbox 系统本身，而是：

> **mailbox plugin 内部提供的编排 skill。**

它负责在语义层编排：
- CLI 调用
- evaluate / decision
- 单次 lightweight re-check

### skill 内部编排的对象

- `mailbox pick-mail`
- `mailbox status`
- snapshot 组装
- evaluate / decision envelope
- 有事继续 / 无事静默
- 最多一次 lightweight re-check

### 不应做的事

- 不应演化成多轮自治循环
- 不应变成隐式 scheduler
- 不应承担整个插件平台职责

---

## 10. 最小 heartbeat block 语义

Phase 1 中，托管 block 的核心语义正式定为：

> 每次 heartbeat 时，执行一次 **mailbox heartbeat skill**；
> 如果该 skill 判断当前无事，则静默；
> 如果存在 open obligation 或需要上浮的 mailbox 事项，则按 skill 返回的建议继续处理；
> 单次 heartbeat 内最多只允许一次 lightweight re-check，避免循环。

这里的关键点是：
- heartbeat block 不绑定单个 CLI 命令
- 不直接暴露插件架构名词
- 只声明执行一个 mailbox 编排 skill

---

## 11. 当前阶段的正确分层

### plugin
负责：
- 安装/卸载
- 托管写入 `HEARTBEAT.md`
- 平台接入
- runtime glue
- 后续正式 heartbeat seam 的演进

### skill
负责：
- mailbox heartbeat flow 的编排语义
- 如何组合 CLI + evaluate
- 如何得出建议

### CLI
负责：
- 细粒度动作
- 本地可靠性层
- `pick-mail / status / open / reply / retry-sync`

### HEARTBEAT.md
负责：
- 托管 bootstrap 接入口
- 调用 mailbox heartbeat skill

---

## 12. 一句话总结

当前 Phase 1 的正式判断是：

> **先由 mailbox plugin 托管写入一个带版本号的 heartbeat block；该 block 在每次 heartbeat 时调用 mailbox heartbeat skill；skill 作为插件内部的编排层去组合 CLI 与 decision logic，从而先把系统跑起来。**
