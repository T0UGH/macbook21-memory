# Agent Mailbox V1 CLI & Local Runtime Design

- 日期：2026-03-17
- 状态：Draft for Implementation
- 作者：MT
- 风格目标：Linus-style CLI judgment
- 关联文档：
  - `docs/2026-03-15-agent-mailbox-prd.md`
  - `docs/2026-03-16-agent-mailbox-implementation-design.md`
  - `docs/2026-03-17-agent-mailbox-v1.1-api-final.md`

---

## 1. 设计结论

V1 客户端明确定位为：

> **一个面向 agent 的、纯 CLI 的、默认结构化输出的 mailbox runtime。**

它不是：
- 人类交互界面
- TUI
- GUI
- 只包一层 HTTP 请求的薄 wrapper
- 会偷偷替 agent 做业务判断的“智能壳”

它应该是：
- **纯 CLI**
- **默认给 agent 用**
- **默认 JSON 输出**
- **flags + stdin 输入优先**
- **带本地 runtime 可靠性层**
- **但不持有业务真相定义权**

一句话总结：

> **厚在可靠性，薄在语义。**

---

## 2. 核心设计原则

## 2.1 服务对象是 agent，不是人

因此客户端的默认行为必须围绕以下目标：
- 易脚本化
- 易组合
- 易解析
- 可稳定接入 Claude Code / shell / automation runtime

这意味着：
- 默认输出不追求人类友好，而追求稳定结构
- 默认输入不依赖 editor，而优先 flags / stdin
- 命令语义必须硬，不靠模糊推断

## 2.2 默认入口不是 inbox，而是注意力导航

Mailbox 不是简单列表系统。

如果每次都让 agent 从 `inbox` 开始扫，它得到的是：
- 原始数据堆
- 重新排序注意力的负担
- 高噪声入口

所以 V1 客户端的默认入口应该是：

> **`mailbox pick-mail`**

而不是 `mailbox inbox`。

`inbox` 仍然存在，但定位是：
- 全局总览
- 排查
- 回顾
- fallback 导航

## 2.3 只有显式动作改变关键状态

Linus 风格会非常在意这点：

> **不要把副作用藏进看起来像读取的命令里。**

因此 V1 客户端必须遵守：
- 只有 `open` 触发 `seen`
- 只有 `reply <message_id>` 可能清除 obligation
- `thread` 只做上下文导航，不偷偷改状态
- `inbox` / `pick-mail` / `status` 都是纯读

## 2.4 CLI 可以厚，但只能厚在可靠性

CLI 不应该替 agent 决定：
- reply target
- obligation 是否清除
- thread 是否 closed
- 什么是业务真相

CLI 应该负责：
- 本地 pending 队列
- retry
- checkpoint
- cache
- 网络失败恢复

所以：

> **CLI 可以是厚客户端，但不能是魔法客户端。**

---

## 3. 默认工作流定稿

V1 推荐工作流：

1. `mailbox pick-mail`
2. agent 从候选中选择一封
3. `mailbox open <message_id>`
4. 如需上下文：`mailbox thread <thread_id>`
5. 如需回复：`mailbox reply <message_id>`
6. 后台 / 显式执行：`mailbox sync` / `mailbox retry-sync`

这条路径的分工非常明确：

- `pick-mail`：缩小注意力范围
- `open`：正式进入处理视野
- `thread`：看上下文结构
- `reply`：发出与某封邮件严格绑定的回信

这是 V1 客户端最应该守住的主路径。

---

## 4. 命令模型

## 4.1 导航命令

### `mailbox pick-mail`

用途：
- 返回 top N 候选邮件
- 作为 agent 的默认入口

设计要求：
- 无 side-effect
- 默认返回结构化候选列表
- 每条候选至少包含：
  - `message_id`
  - `thread_id`
  - `reason`
  - `requires_reply`
  - `is_unread`
  - `obligation_age_seconds`
  - `accepted_at`

V1 默认建议：
- 返回 top 3

语义定位：

> 它不是替 agent 做决定，而是替 agent 缩小注意力范围。

---

### `mailbox inbox`

用途：
- 查看 inbox 总览
- 排查遗漏
- 做全局浏览

设计要求：
- 无 side-effect
- 默认不返回完整 body
- 支持分页、筛选、排序

语义定位：

> `inbox` 是总览入口，不是默认工作入口。

---

### `mailbox thread <thread_id>`

用途：
- 查看 thread 上下文
- 了解消息关系、义务状态、最近活动

默认输出：
- thread 元信息
- message 列表摘要
- open obligations
- 最近事件摘要

默认不做：
- 不返回全文阅读体验的主视图
- 不自动触发 `seen`

语义定位：

> `thread` 是上下文导航页，不是“正式阅读邮件”的入口。

---

### `mailbox open <message_id>`

用途：
- 真正打开一封邮件
- 将其纳入当前处理视野

这是客户端里最关键的命令。

它必须承担两件事：

#### 1) 触发 `seen`
- 成功打开后，CLI 负责上报 seen
- 若上报失败，写入本地 `pending_seen`

#### 2) 输出下一步提示
打开后不应只返回正文，还应附带：
- 当前是否存在 open obligation
- 当前 message 的 `type` / `requires_reply`
- 若要回复，应执行哪条命令
- thread 最近几条摘要

语义定位：

> `open` = 正式阅读入口 + 行动提示入口。

---

## 4.2 动作命令

### `mailbox send`

用途：
- 发送新邮件
- 可以新建 thread
- 也可以往已有 thread 中追加一封非 reply 语义的新消息

输入主路径：
- flags + stdin

例如：
- `mailbox send --to zhigeng --type request --subject "..." --body-stdin`
- `cat body.md | mailbox send --to zhigeng --type report --subject "..." --body-stdin`

支持但不作为主路径：
- `--body-file`
- `--edit`

设计判断：
- `--edit` 可以存在，但只是 human fallback
- 主路径必须对 agent / shell 友好

---

### `mailbox reply <message_id>`

用途：
- 对某封具体 message 做 reply

这是 obligation 语义最关键的命令，因此必须收得很硬：

- reply 必须绑定到具体 `message_id`
- 不支持“对整个 thread 模糊回复”
- CLI 不猜测目标 message
- obligation 清除只可能经由这条路径发生

输入主路径：
- flags + stdin

语义定位：

> reply 是明确动作，不是 CLI 的猜测结果。

---

### `mailbox archive <message_id>`
### `mailbox unarchive <message_id>`

用途：
- 管理某 agent 自己的工作视图

明确约束：
- 不影响 obligation
- 不影响 thread 是否自然闭合
- 不代表完成处理

---

## 4.3 同步与恢复命令

### `mailbox sync`

用途：
- 拉取 server 更新
- 推送本地 pending 动作
- 刷新 cache / checkpoint

说明：
- 这不是 agent 的主业务命令
- 但它是 CLI runtime 的核心系统命令

---

### `mailbox retry-sync`

用途：
- 重试失败的 outbound / pending_seen / pending_delivery 等动作

说明：
- 可以由 agent 显式调用
- 更合理的是由定时任务 / 后台流程调用

---

### `mailbox status`

用途：
- 查看本地 runtime 健康度与待处理项

建议输出：
- pending outbound 数
- pending seen 数
- pending delivery 数
- 最近 sync 时间
- 最近错误摘要
- 当前 representative / machine 信息（若存在）

---

### `mailbox obligations`

用途：
- 查看当前 open obligation
- 帮 agent 理解“有哪些明确欠回复项”

语义定位：

> 这是 mailbox 的责任视图，不是普通列表功能。

---

## 5. 状态变化规则定稿

## 5.1 `seen` 规则

V1 客户端规则：

> **只有 `mailbox open <message_id>` 才触发 `seen`。**

以下命令都不触发 `seen`：
- `mailbox pick-mail`
- `mailbox inbox`
- `mailbox thread`
- `mailbox status`
- `mailbox obligations`

这样做的原因：
- 简单
- 明确
- 不藏副作用
- 与“进入处理视野”的定义严格对齐

---

## 5.2 obligation 规则

CLI 不负责判断 obligation 业务真相。

CLI 只负责：
- 明确调用 `reply <message_id>`
- 把这个动作发送给 server
- 展示 server 返回的 obligation 状态变化

也就是说：
- CLI 不自己清 obligation
- CLI 不根据 thread 内容猜 obligation 是否已结束
- CLI 不根据 archive 判断 obligation 是否已清

---

## 5.3 thread 规则

`thread` 的职责是：
- 展示上下文
- 帮助导航
- 帮助理解责任链

它不是：
- 读取即记 seen 的入口
- 自动决定回复目标的入口
- 业务状态推导器

---

## 6. 输入输出设计

## 6.1 默认输出：JSON

V1 默认输出必须是：

> **结构化、稳定、可解析优先。**

默认：
- JSON

可选：
- `--pretty`
- `--text`

但人类展示模式不能反客为主。

原因：
- agent 不该猜格式
- shell/脚本/Claude Code 更容易接
- 可组合性更强

---

## 6.2 默认输入：flags + stdin

正文输入主路径建议为：
- `--body`
- `--body-file`
- `--body-stdin`

其中主推荐路径是：
- flags + stdin

原因：
- 最 Unix
- 最可脚本化
- 最适合 agent

`$EDITOR` 可以有，但只作为：
- 人类 fallback
- 调试便利能力

而不是默认路径。

---

## 7. 本地 runtime state 设计

这是客户端最关键的工程层。

如果没有这一层，本 CLI 只会退化成：
- 一组 HTTP 请求 wrapper

而 mailbox 客户端真正的价值就在于：
- 在 agent 侧承接异步失败、重试、恢复、同步状态

---

## 7.1 本地状态对象建议

### `outbound_messages`

表示：
- 本地已发起发送
- 但 server 尚未确认 `accepted`

用途：
- 表达 `submitted`
- 支撑断网重试
- 支撑 `retry-sync`

---

### `pending_seen`

表示：
- 某 message 已被 `open`
- 但 seen 上报尚未成功

用途：
- 避免因为网络错误导致“实际上看了，但 server 不知道”

---

### `pending_delivery`

表示：
- 某 message 已被拉入本地工作缓存
- 但 delivered 尚未成功同步回 server

用途：
- 支撑多机 / failover 恢复
- 支撑 retry-delivery

---

### `local_cache`

表示：
- inbox/thread/message 的本地缓存

用途：
- 提高查询效率
- 降低频繁 HTTP 拉取
- 支撑弱网 / 离线边界下的工作连续性

---

### `sync_checkpoint`

表示：
- 本机上次同步到的 event / 时间点 / 状态

用途：
- 增量同步
- 错误恢复
- operator 可观察性

---

## 7.2 本地 runtime 的职责边界

### CLI 负责
- pending 队列
- retry
- checkpoint
- cache
- 网络失败恢复
- 机械性的状态补偿

### CLI 不负责
- 定义业务真相
- 自己清 obligation
- 自己判断 thread 是否 closed
- 自己猜回复目标
- 自己从上下文里“智能推断”业务状态

也就是说：

> **本地 runtime 是可靠性层，不是业务裁判。**

---

## 8. 推荐最小命令集

V1 第一批建议只做这些命令：

- `mailbox pick-mail`
- `mailbox inbox`
- `mailbox thread <thread_id>`
- `mailbox open <message_id>`
- `mailbox send`
- `mailbox reply <message_id>`
- `mailbox archive <message_id>`
- `mailbox unarchive <message_id>`
- `mailbox obligations`
- `mailbox sync`
- `mailbox retry-sync`
- `mailbox status`

这组命令已经足够形成完整闭环：
- 导航
- 阅读
- 回复
- 责任查看
- 本地同步
- 失败恢复

---

## 9. 方案对比

## 方案 A：超薄 CLI

特点：
- 几乎只是 API wrapper
- 本地不维护多少状态

优点：
- 简单
- 代码少

缺点：
- 价值低
- 多机恢复能力弱
- mailbox 客户端层形同虚设

---

## 方案 B：中等厚度 CLI（推荐）

特点：
- API wrapper + 本地 runtime state
- 带 retry / checkpoint / pending / cache
- 语义动作仍然显式

优点：
- 工程上最平衡
- 符合 mailbox 的本质
- 更适合 agent / 多机 / 弱网场景

缺点：
- 本地状态设计与维护成本更高

---

## 方案 C：厚客户端工作流引擎

特点：
- CLI 主动编排太多 agent 行为
- 自动选目标、自动推断状态、自动推进业务流

优点：
- 看起来更“智能”

缺点：
- 极易长歪
- 语义最容易脏
- 调试困难
- 不符合“显式优先”的工程纪律

---

## 10. 最终推荐

V1 客户端最终推荐采用：

### 10.1 形态
- 纯 CLI
- 面向 agent

### 10.2 默认入口
- `mailbox pick-mail`

### 10.3 关键阅读语义
- 只有 `open` 触发 `seen`
- `thread` 不触发 `seen`

### 10.4 回复语义
- reply 必须绑定具体 `message_id`
- obligation 清除只可能经由 `reply <message_id>`

### 10.5 输入输出
- 默认 JSON
- flags + stdin 优先
- editor 只是 fallback

### 10.6 本地能力
- CLI 持有本地 runtime state
- 负责 retry / pending / checkpoint / cache
- 不持有业务真相定义权

---

## 11. 一句话结论

> **Agent Mailbox V1 客户端应该是一个纯 CLI、默认 JSON、以 `pick-mail -> open -> reply` 为主路径、只有显式动作才改状态、并带本地可靠性层的 mailbox runtime。**

这版设计既符合 agent 使用方式，也符合 Linus 风格的 CLI judgment：
- 简单
- 显式
- 可组合
- 不藏魔法
- 对失败和恢复足够认真
