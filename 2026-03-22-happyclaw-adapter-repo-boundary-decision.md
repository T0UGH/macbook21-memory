# HappyClaw 适配插件仓边界决策（Phase 1）

日期：2026-03-22

## 背景

Mailbox Phase 1 讨论中，原本存在一个设想：

> 除了 `openclaw-mailbox-server`、`openclaw-mailbox-cli`、`openclaw-mailbox-plugin` 之外，可能还需要单独再开一个仓库，专门承接与 HappyClaw 的适配插件层。

随着 HappyClaw 侧实现推进，这个设想需要重新评估。

---

## 当前结论

**Phase 1 当前不建议单独开一个 HappyClaw 适配插件仓。**

原因不是“永远不需要”，而是：

> **以当前已经验证出来的真实复杂度，这层还只是一个很薄的宿主注入口，不值得为它支付独立仓库的管理成本。**

换句话说：

- 不是这层没有价值
- 而是它当前的复杂度还没有高到需要一个 repo

---

## 为什么早期会觉得可能需要单独仓

早期之所以会有“HappyClaw 适配插件仓”的设想，是因为默认假设 HappyClaw 侧适配可能会包含一整层相对厚的逻辑，例如：

1. 外部系统与 HappyClaw 之间的协议翻译
2. 主会话注入逻辑
3. 前端展示适配
4. agent 生命周期衔接
5. 配置、部署、升级等额外流程

如果这些逻辑都需要在 HappyClaw 本体之外独立实现，那么把它们放进一个单独 repo 是合理的。

---

## 为什么现在判断“不需要单独仓”

当前 HappyClaw 已经验证出的真实适配形态是：

- 在 HappyClaw 本体内补一个很薄的 internal inject 接口
- 由 mailbox 上游按协议调用这个接口

也就是：

- `POST /internal/inject`
- 独立 secret
- 写消息
- 广播前端
- 尝试 IPC 注入 / 排队
- 正确处理 cursor

这层本质上是：

> **宿主能力补口（host capability patch）**

而不是：

> **一个独立业务系统或独立插件产品**

因此，当前最合理的归属是留在 HappyClaw 本体里，而不是再包一层新仓库。

---

## 架构判断：为什么现在不应硬拆新仓

### 1. 会把“薄接缝”人为做厚

如果为当前这点适配逻辑单独开仓，通常会额外引入：

- 配置结构
- 额外 SDK / client 层
- 适配抽象
- 版本兼容层
- 发布与依赖管理

但目前真实需求只是：

- 一个 internal route
- 一份调用协议

在这种情况下，单独开仓的收益很低，复杂度却会显著上升。

### 2. 容易把职责边界搞混

如果真的存在一个独立的 HappyClaw adapter repo，就会出现典型的边界混乱问题：

- 注入协议到底由谁定义
- HappyClaw route 变化由谁跟进
- sourceMeta / chatJid / 幂等策略归谁负责
- mailbox 侧和 HappyClaw 侧之间多出一层额外协调成本

结果会把原本清晰的两层结构：

- mailbox 侧
- HappyClaw 宿主侧

硬生生拆成三层：

- mailbox 侧
- happyclaw-adapter 仓
- HappyClaw 本体

这是一种典型的过度分层。

### 3. 当前稳定边界已经出现

当前已经比较清晰的边界是：

#### 边界 A：Mailbox 业务闭环
独立仓是合理的：
- `openclaw-mailbox-server`
- `openclaw-mailbox-cli`
- `openclaw-mailbox-plugin`

#### 边界 B：HappyClaw 宿主能力
适合留在宿主本体中：
- `/internal/inject`

这两个边界已经足够稳定。

如果再单独插入一个适配仓，边界反而会更差。

---

## 当前推荐的仓库边界

Phase 1 当前推荐维持如下结构：

- `openclaw-mailbox-server`
- `openclaw-mailbox-cli`
- `openclaw-mailbox-plugin`
- `happyclaw` 本体内提供 `/internal/inject`

也就是：

- mailbox 自己的业务能力与协议，继续独立分仓
- HappyClaw 负责提供宿主注入口
- mailbox 上游根据协议调 HappyClaw

这是当前最简洁、最稳、最少管理摩擦的边界方案。

---

## 什么时候未来可能重新值得单独开仓

当前“不需要单独仓”，不等于未来永远不需要。

以下情况出现时，才可能重新值得抽一个独立适配层仓库：

### 情况 1：不止 HappyClaw，一个接口要适配多个宿主
例如未来还要支持：
- 其他 agent shell
- 其他 web chat 宿主
- 其他会话承载系统

此时“宿主适配层”才具备抽象复用价值。

### 情况 2：上游注入逻辑本身变厚
例如后续上游需要统一处理：
- 重试
- 幂等
- 路由发现
- 多宿主分发
- 统一事件封装
- 监控与追踪

此时可能需要一个 facade / integration 层。

### 情况 3：要把这套能力产品化 / upstream 成通用方案
如果未来要对外表达为：

> 任何外部系统都能向 HappyClaw 主会话注入事件

那么可能需要把协议、client、文档、样例等抽成更正式的一层。

---

## 当前正式判断

> **Phase 1 当前不单开 HappyClaw 适配插件仓，是因为这层已经被验证为“宿主补一个注入口”的薄能力，不值得为它付出独立仓库的管理成本。**

更直白一点：

> **复杂度还没高到需要一个仓。现在硬开，只会把简单问题做复杂。**

---

## 一句话总结

当前 HappyClaw 与 Mailbox 的边界，应理解为：

> **Mailbox 负责业务决策与事件形成，HappyClaw 负责提供主会话注入口；两者之间当前只需要一层薄协议，不需要单独养一个适配仓。**
