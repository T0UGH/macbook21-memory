# HappyClaw：SIGTERM 排查与 launchd 托管落地（2026-03-20）

## 背景

2026-03-20 中午，Hex 出现“不回消息”的现象。排查后发现，不是 Hex 本身卡住，而是其宿主链路 HappyClaw 被外部终止，导致整条执行链掉线。

## 关键结论

### 1. HappyClaw 不是自崩，而是收到外部 SIGTERM

HappyClaw 日志中明确出现：

- `Shutdown signal received, cleaning up...`
- `signal: "SIGTERM"`
- `Force exit (second signal)`

时间点为：

- `2026-03-20 12:44:06`

这说明 HappyClaw 是被外部终止，而不是自身异常崩溃。

### 2. 同一时间 OpenClaw gateway 也收到 SIGTERM

`~/.openclaw/logs/gateway.log` 中同一时间窗口存在：

- `2026-03-20T12:44:06.122+08:00 [gateway] signal SIGTERM received`
- `2026-03-20T12:44:06.126+08:00 [gateway] received SIGTERM; shutting down`

随后 gateway 又重新启动。

### 3. gateway 能自动恢复，是因为它被 launchd 托管

通过 `launchctl print gui/$(id -u)/ai.openclaw.gateway` 可确认：

- gateway 是 `LaunchAgent`
- plist 路径：`/Users/wangguiping/Library/LaunchAgents/ai.openclaw.gateway.plist`
- 具备：
  - `RunAtLoad = true`
  - `KeepAlive = true`
- `last terminating signal = Terminated: 15`

这说明 gateway 在收到 SIGTERM 后由 launchd 自动拉起。

### 4. HappyClaw 当时不能自动恢复，是因为没有托管

在本次处理前，HappyClaw 没有对应的：

- launchd
- crontab
- pm2 / forever

因此它和 gateway 一起被上层 SIGTERM 波及时：

- gateway 会自动恢复
- HappyClaw 不会自动恢复

这就是 Hex 掉线但 OpenClaw 还活着的直接原因。

## 本次落地动作

### 1. 为 HappyClaw 创建 launchd 服务

新建 plist：

- `/Users/wangguiping/Library/LaunchAgents/ai.happyclaw.service.plist`

核心配置：

- `Label = ai.happyclaw.service`
- `RunAtLoad = true`
- `KeepAlive = true`
- `WorkingDirectory = /Users/wangguiping/happyclaw`
- `ProgramArguments = /usr/local/bin/npm run start`
- 标准输出：
  - `/Users/wangguiping/happyclaw/data/logs/happyclaw.out.log`
- 标准错误：
  - `/Users/wangguiping/happyclaw/data/logs/happyclaw.err.log`

### 2. 启用并接管服务

执行：

- `launchctl bootstrap gui/$(id -u) /Users/wangguiping/Library/LaunchAgents/ai.happyclaw.service.plist`
- `launchctl kickstart -k gui/$(id -u)/ai.happyclaw.service`

### 3. 处理中间的端口冲突

第一次启动失败，原因为：

- `EADDRINUSE: address already in use :::3000`

根因不是 launchd 配置错误，而是之前手工拉起的 HappyClaw 旧进程仍占用 3000 端口。

当时残留的手工进程主要是：

- `50958` → `npm run start`
- `50968` → `node dist/index.js`

停掉旧手工实例后，launchd 新实例成功接管。

## 最终状态

最终由 launchd 托管的 HappyClaw 服务状态：

- launchd label：`ai.happyclaw.service`
- launchd pid：`52404`
- 实际 node 进程 pid：`52423`
- 3000 端口恢复正常监听

日志确认成功启动：

- `happyclaw running`
- `Web server started`
- `port: 3000`
- `Feishu WebSocket client started`
- `IM channel connected`
- `WebSocket client connected`

## 后续操作规范

以后 **HappyClaw 的默认运行方式改为 launchd**，不要再把手工 `nohup ... npm run start` 作为主路径。

### 推荐操作

#### 看状态
```bash
launchctl print gui/$(id -u)/ai.happyclaw.service
```

#### 手动重启
```bash
launchctl kickstart -k gui/$(id -u)/ai.happyclaw.service
```

#### 停止服务
```bash
launchctl bootout gui/$(id -u)/ai.happyclaw.service
```

#### 查看日志
```bash
tail -f /Users/wangguiping/happyclaw/data/logs/happyclaw.out.log
tail -f /Users/wangguiping/happyclaw/data/logs/happyclaw.err.log
```

## 工程判断

这次事件的系统性问题不是 HappyClaw 自己有 bug，而是：

- OpenClaw gateway 是“有托管的服务”
- HappyClaw 只是“裸进程”

因此两者在面对同一波 SIGTERM 时恢复能力完全不对称。

本次通过 launchd 托管，已经把 HappyClaw 从“裸进程”升级成“服务”。

后续如果再遇到类似上层 SIGTERM，HappyClaw 也应具备自动恢复能力。
