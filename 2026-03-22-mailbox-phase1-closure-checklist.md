# Mailbox Phase 1 收口清单（总控表）

日期：2026-03-22

目标：把 Agent Mailbox Phase 1 当前进度、剩余缺口、阻塞关系与建议负责人一次性拉平，作为后续收口的统一总控视图。

---

## 一、总体判断

当前状态可以概括为：

- **底层协议闭环基本完成**：server / CLI / OpenClaw mailbox plugin 三条线已经基本打通
- **HappyClaw 主会话注入能力已进入最后收口阶段**：代码已接入，但运行时验证和边界收紧尚未完成
- **更上层门面/一体化集成面尚未正式收口**：是否需要单独门面仓库、由谁承接上游注入逻辑，还需要明确

一句话：

> 现在不是“从 0 到 1 没做完”，而是“底层 1 已经有了，上层最后一层封口还没彻底做完”。

---

## 二、总控表

| 模块 | 当前状态 | 已完成 | 未完成 | 是否阻塞 Phase 1 | 建议负责人 | 优先级 |
|------|----------|--------|--------|------------------|------------|--------|
| `openclaw-mailbox-server` | 基本完成 | `pick-mail` 已对齐 CLI 聚合格式；`candidates / total_unread / open_obligations_count` 已落地；e2e 已更新 | 其他 API 契约审查、文档/观测性增强 | 否 | Hex | P2 |
| `openclaw-mailbox-cli` | 核心完成 | `status / pick-mail / open` 等核心命令已可支撑 Phase 1；与 plugin 已完成真实联调 | `mailbox --help` / 子命令帮助与文档收尾 | 否 | Hex | P2 |
| `openclaw-mailbox-plugin`（skill/plugin） | 基本完成 | evaluate / heartbeat-block / heartbeat-skill / runtime-state / adapter / plugin lifecycle 已完成；测试通过；真实 CLI + server 集成已跑通 | 更正式的 OpenClaw SDK 接缝产品化（如需要） | 否（对当前 Phase 1） | Hex | P2 |
| OpenClaw 正式安装/产品化接缝 | 部分完成 | 已有 `openclaw-entry.ts`、`resolveConfig()`、`register()` 等接缝抽象 | 是否已真正接到正式 SDK / 正式安装路径，尚未看到最终收口 | 视 Phase 1 定义而定；对“逻辑跑通”不阻塞，对“正式可安装交付”可能阻塞 | 贵平拍板 / Hex 落地 | P2 |
| HappyClaw `/internal/inject` 能力 | 已有代码，未完全关门 | route / schema / secret / cursor 分流 / build 均已完成 | 运行时 curl 验证未做；前端可见性未验；活跃/排队两条路径未实测 | **是** | Hex | **P0** |
| HappyClaw 主会话约束 | 未收口 | 已有通用 inject API 雏形 | 仍允许任意 registered group 注入，尚未把“只注入主会话”约束收死 | **是** | Hex | **P0** |
| HappyClaw inject 幂等性 | 未完成 | 无 | 缺少 requestId / mailbox_message_id 去重；重试会重复注入 | 严格说不一定阻塞 demo，但阻塞“可放心使用” | Hex | P1 |
| mailbox → HappyClaw inject 上游调用协议 | 未正式收口 | HappyClaw 接口已存在 | 谁调用、何时调用、chatJid 如何决定、失败怎么处理、是否重试，尚未形成正式协议 | **是** | 贵平拍板 / Hex 落地 | **P0** |
| 门面仓库 / facade integration | 未明确 | 分仓策略已明确（server / cli / plugin 分仓） | 是否还需要单独门面层仓库，职责边界尚未拍板 | 视你是否把它纳入 Phase 1 范围；当前不一定阻塞核心闭环 | 贵平 | P1 |
| 文档与收口报告 | 部分完成 | 多份阶段文档和交付邮件已存在 | 缺一份真正面向最终集成的“Phase 1 最终交付视图” | 否，但影响后续交接效率 | 贵平 / MT | P1 |

---

## 三、按优先级拆解的剩余事项

### P0：不做就不能说 Phase 1 真正关门

#### 1. HappyClaw 运行时验证
**目标：** 证明 `/internal/inject` 不是“能编译”，而是“能真正工作”。

**至少要验证：**
- injected 路径：有活跃 agent 时，消息进入当前主会话
- queued 路径：无活跃 agent 时，消息入库并触发后续处理
- 前端消息可见性：用户在 HappyClaw Web 界面可看到注入消息
- cursor 行为：不会重复吃消息，也不会丢消息

**建议动作：**
- 重启 HappyClaw
- curl 调 `/internal/inject`
- 各跑一遍 injected / queued 场景
- 出一份最终验证报告

**负责人：** Hex

---

#### 2. 把“主会话注入”约束收死
**问题：** 当前实现是“通用 inject API”，不是“严格主会话注入”。

**风险：** 调用方传错 `chatJid` 时，系统会静默注入错误上下文。

**建议收口方式（三选一，建议前两种）：**
- 只允许 `group.is_home === true`
- 只允许明确配置的主会话 JID
- 最差也要在上游调用层把目标固定死，不能外部任意传

**负责人：** Hex（实现），贵平（拍板边界）

---

#### 3. 定义 mailbox → HappyClaw inject 的正式调用协议
**当前缺口：** HappyClaw 已开口，但上游如何使用这条口子还没正式收口。

**至少要定清楚：**
- 谁负责发起 inject 调用
- 在什么触发点调用
- 请求体字段是什么
- `chatJid` 如何确定
- 失败是否重试
- 如何避免重复注入
- 成功后如何记录或回传结果

**负责人：** 贵平先拍板，Hex 落地

---

### P1：不一定阻塞 demo，但会阻塞“可放心使用”

#### 4. inject 幂等性
**问题：** 当前请求重试会重复注入。

**建议：**
- 增加 `requestId`
- 或至少基于 `sourceMeta.mailbox_message_id` 做短期去重

**负责人：** Hex

---

#### 5. 门面层 / facade 是否存在，需不需要单独仓库
**当前情况：** 三个核心仓库已经明确拆分，但“门面层”是否要单独存在还没正式拍板。

**要回答的问题：**
- Phase 1 是否需要单独 facade repo
- 如果不需要，由谁承担跨仓串联逻辑
- 如果需要，这个 repo 的职责是什么：
  - 仅配置编排？
  - 注入适配？
  - 最终部署与样例？

**负责人：** 贵平

---

#### 6. 最终交付视图文档
**问题：** 目前文档很多，但还缺一份“外部读者一看就懂现在做到哪、怎么用、还差什么”的总文档。

**建议内容：**
- Phase 1 目标
- 已完成能力
- 仓库边界
- HappyClaw / OpenClaw / mailbox server / CLI 的关系
- 当前已知限制
- 下一步演进方向

**负责人：** 贵平 / MT

---

### P2：不阻塞 Phase 1，但建议尽快补齐

#### 7. CLI help / 使用文档收尾
**现状：** CLI 能用，但帮助信息与最终用户导向文档还可以补。

**负责人：** Hex

---

#### 8. OpenClaw 正式 SDK / 安装接缝产品化
**现状：** plugin 逻辑已收口，但是否已达到“正式安装交付”状态，还没看到最终证据。

**负责人：** 贵平拍板范围，Hex 落地

---

#### 9. server 其他 API 契约与观测性增强
**现状：** `pick-mail` 已完成，但其余 API 是否需要同级别契约审查，还没系统推进。

**负责人：** Hex

---

## 四、建议的最终关门顺序

### 推荐顺序
1. **HappyClaw 运行时验证**
2. **把主会话边界收死**
3. **定义 mailbox → inject 上游调用协议**
4. **决定幂等性这轮做不做**
5. **决定是否需要 facade repo**
6. **补最终交付视图文档**
7. **补 CLI help / OpenClaw 产品化接缝等尾项**

---

## 五、我当前的阶段性判断

### 如果你问：底层核心闭环是否已经具备？
**答案：是。**

- server：基本完成
- CLI：基本完成
- OpenClaw plugin：基本完成

### 如果你问：Phase 1 是否已经能无争议宣布全部关门？
**答案：还不能。**

主要还差：
- HappyClaw 真机运行时验证
- 主会话注入边界收死
- mailbox → inject 上游协议正式收口
- 门面层是否存在的最后拍板

---

## 六、一句话总结

> Mailbox Phase 1 当前已完成“协议层与技能层闭环”，正在收最后一层“主会话注入与集成交付”封口。
