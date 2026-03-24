# Mailbox 仓库边界与 Monorepo 判断（2026-03-21）

## 结论

**现阶段不建议为了获得更多 GitHub star，把 mailbox 相关代码硬整合成一个大仓（monorepo / big repo）。**

更合理的路线是：

> **职责分仓 + 一个主品牌入口仓（umbrella repo）**

也就是：

- 代码继续按职责拆分
- 对外用一个主入口仓承接品牌、README、路线图和 star

---

## 为什么不建议“为了 star 合仓”

### 1. star 是结果，不是架构原则

仓库边界首先应服务于：

- 职责是否清晰
- 维护者是否一致
- 发布节奏是否一致
- 用户是否真的需要整体理解/整体安装
- 测试与版本是否高度耦合

而不是先服务于：

- 能不能把 star 集中到一个 repo

**为了 star 改仓库边界，通常会得到更差的工程结构。**

### 2. 当前这些模块天然不是一个单体产品仓

当前 mailbox 体系至少包含：

- mailbox server
- mailbox CLI
- OpenClaw integration
- 未来可能的 runtime-agnostic core
- HappyClaw / standalone integration glue

这些模块的受众并不完全相同：

- 想部署服务端的人主要关心 server
- 想给 agent/runtime 接 mailbox 的人主要关心 CLI
- OpenClaw 用户只关心 OpenClaw integration
- HappyClaw 用户只关心 HappyClaw 接法

把这些代码硬放进一个仓，虽然表面更集中，但**认知负担和维护复杂度也会一起集中**。

### 3. 大仓容易制造“边界错觉”

一旦全部进一个仓，团队会自然倾向于：

- 看到能一起改，就顺手一起改
- 看到能一起发，就顺手一起发
- 看到都在一个 issue 里，就开始跨边界耦合

久而久之，会损害：

- server / cli / integration 的职责边界
- 独立演进能力
- 后续架构清晰度

**对于当前这种还在快速收口边界的阶段，这个风险很大。**

---

## 什么情况下才值得考虑 monorepo

只有在下面这些信号明显成立时，才值得认真考虑大仓：

1. **跨仓改动非常频繁**
   - 每次协议调整都必须同时改 server / cli / integration

2. **版本兼容成本显著上升**
   - 频繁出现 server vX 与 cli vY 不兼容的问题

3. **这些模块必须一起发布**
   - 例如已经形成统一 release train

4. **维护方式已经统一**
   - 同一批维护者
   - 同一套 CI / review / release 节奏

5. **用户必须整体安装和整体理解**
   - 而不是按角色只接触某一部分

**当前还没有充分证据表明已经到了这个阶段。**

---

## 当前更推荐的结构

## 方案：职责分仓 + 主入口仓

### 1. 继续职责分仓

建议保持下面这种拆分思路：

- `agent-mailbox-server`
- `agent-mailbox-cli`
- `agent-mailbox-openclaw`
- 未来如有必要，再考虑：`agent-mailbox-core`

这样做的好处：

- 代码职责清晰
- 各模块可独立迭代
- 不同 runtime/integration 不会互相污染
- 未来是否抽 core 有回旋空间

### 2. 增加一个主品牌入口仓

这个仓的职责不是承载所有代码，而是：

- 讲清整体愿景
- 做统一 README / 架构图 / 快速开始
- 链接各子仓
- 集中承接 GitHub star
- 承载 roadmap / examples / docs index

这个仓可以作为：

- `agent-mailbox`
- 或最终正式品牌名

**这样可以同时获得：品牌集中 + 代码边界清晰。**

---

## 为什么“品牌聚合”比“代码合并”更重要

真正影响传播效果的，通常不是代码是否在一个 repo，而是：

1. **命名是否统一**
2. **README 是否清楚**
3. **主入口仓是否能把故事讲明白**
4. **安装路径是否简单**
5. **新用户是否能快速跑起来**

也就是说：

> GitHub star 更依赖认知清晰度，而不是物理目录是否共址。

---

## 当前建议（可直接执行）

### 不建议做的事

- 不要为了 star 把所有 mailbox 相关代码硬整合成一个大仓
- 不要过早把 integration 和 core 混成一坨
- 不要把“传播诉求”误当成“架构诉求”

### 建议做的事

1. **保持职责分仓**
2. **新增一个主品牌入口仓**
3. **统一 repo 命名风格**，让外部一眼看出它们是一套东西
4. **让主入口仓承接 star、文档和故事**
5. **让代码仓继续保持工程边界清晰**

---

## 建议的 Repo 命名方案

如果后续要做“主品牌入口仓 + 职责分仓”，建议命名风格统一，不要一会儿 `openclaw-mailbox-*`，一会儿 `agent-mailbox-*`，一会儿 `mailbox-plugin-*`。统一命名比想象中更影响传播效果。

### 推荐思路

选择一个稳定的品牌前缀，然后所有仓库沿用同一前缀。

### 方案 A：沿用 `openclaw-mailbox-*`

适合条件：

- 当前阶段主要服务 OpenClaw 生态
- 想强调这套东西最初长在 OpenClaw 体系里

示例：

- `openclaw-mailbox`（主入口仓）
- `openclaw-mailbox-server`
- `openclaw-mailbox-cli`
- `openclaw-mailbox-openclaw`
- 未来如需要：`openclaw-mailbox-core`

### 方案 B：改为更中性的 `agent-mailbox-*`

适合条件：

- 你明确要服务多 runtime（OpenClaw / HappyClaw / standalone）
- 不希望品牌被某个单一 runtime 绑定

示例：

- `agent-mailbox`（主入口仓）
- `agent-mailbox-server`
- `agent-mailbox-cli`
- `agent-mailbox-openclaw`
- 未来如需要：`agent-mailbox-core`

### 我的建议

如果你已经明确这套体系最终要服务：

- OpenClaw
- HappyClaw
- 其他独立 agent runtime

那我更倾向于：

> **主品牌逐步收敛到 `agent-mailbox-*`**

理由：

- 更中性
- 更利于多 runtime 扩展
- 不会让 HappyClaw/standalone 用户天然觉得“这是 OpenClaw 私货”

但如果你短期内还希望借 OpenClaw 语境快速起势，也可以先保留 `openclaw-mailbox-*`，等体系更成熟后再评估品牌迁移。

---

## 最终判断

**现在不应该为了更多 star 而做大仓。**

更成熟、也更适合当前阶段的策略是：

> **职责分仓 + 主品牌入口仓**

这是传播、维护、演进三者之间更平衡的方案。
