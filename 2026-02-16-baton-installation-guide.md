# Baton 安装完整指南

**日期:** 2026-02-16
**项目:** https://github.com/T0UGH/baton (forked from williamfzc/baton)

## 项目位置

```
~/workspace/github/baton
```

## 安装步骤

### 1. 安装 bun 运行时

```bash
curl -fsSL https://bun.sh/install | bash
```

安装完成后，bun 位于 `~/.bun/bin/bun`

### 2. 安装项目依赖

```bash
cd ~/workspace/github/baton
bun install
```

**注意：**
- **不要使用代理**，会导致 npmjs.org 返回 500 错误
- 如果遇到 `bnpm.byted.org` 错误，删除 `bun.lock` 后重新安装

### 3. 安装 ACP Runtime

#### 安装 Claude Code ACP

```bash
bun install -g @zed-industries/claude-code-acp
```

### 4. 配置 PATH

bun 安装脚本会自动添加到 `~/.zshrc`，但在当前终端需要手动加载：

```bash
source ~/.zshrc
```

## 全局命令配置

### 创建 baton 全局命令

创建了 `~/.bun/bin/baton` 脚本，支持以下命令：

```bash
baton feishu     # 启动飞书模式
baton cli        # 启动 CLI 模式
baton telegram    # 启动 Telegram 模式
baton whatsapp    # 启动 WhatsApp 模式
baton slack       # 启动 Slack 模式
```

**脚本位置：** `~/.bun/bin/baton`

**脚本内容：**
```bash
#!/bin/bash
cd /Users/wangguiping/workspace/github/baton

case "$1" in
  "feishu")
    ~/.bun/bin/bun src/feishu-server.ts
    ;;
  "telegram")
    ~/.bun/bin/bun src/telegram-server.ts
    ;;
  "whatsapp")
    ~/.bun/bin/bun src/whatsapp-server.ts
    ;;
  "slack")
    ~/.bun/bin/bun src/slack-server.ts
    ;;
  *)
    ~/.bun/bin/bun src/cli.ts "$@"
    ;;
esac
```

## 项目配置

### 配置文件位置

```
~/workspace/baton.config.json
```

### 支持的配置文件名

1. `baton.config.json`（推荐）
2. `.batonrc.json`
3. `baton.json`

### 配置查找规则

- 从当前工作目录开始查找
- 向上最多查找 5 层目录
- 找到第一个匹配的配置文件即使用

### 当前配置

```json
{
  "project": {
    "path": ".",
    "name": "default"
  },
  "feishu": {
    "appId": "cli_a918e3194038dcd5",
    "appSecret": "ftrQd33b8onq4mGUOv0rWfE2MaRCQQls",
    "domain": "feishu"
  },
  "acp": {
    "executor": "claude-code"
  }
}
```

## ACP Runtime 支持

| Runtime | Executor | 安装状态 | 备注 |
| ------- | --------- | --------- | ---- |
| **opencode** | `opencode` | 未安装 | 默认，需要 opencode CLI |
| **claude** | `claude-code` | ✅ 已安装 | 需要 `claude-code-acp` 在 PATH 中 |
| **codex** | `codex` | 未安装 | 需要 `codex-acp` 在 PATH 中 |

## 运行方式

### 方式一：使用全局命令

```bash
baton feishu     # 飞书模式
baton cli        # CLI 模式
```

### 方式二：使用 bun 运行

```bash
cd ~/workspace/github/baton
bun run start:feishu  # 飞书
bun run start:cli     # CLI
```

### 方式三：直接运行服务器文件

```bash
cd ~/workspace/github/baton
bun src/feishu-server.ts
```

## 支持的 IM 平台

| 平台       | 状态 | 模式               |
| ---------- | ---- | ------------------ |
| **Feishu** | ✅   | WebSocket 长连接   |
| **Telegram** | ✅   | Bot API 长轮询    |
| **WhatsApp** | ✅   | Webhook + Graph API |
| **Slack**  | ✅   | Events API webhook  |
| **CLI**    | ✅   | 本地终端交互        |

## 常用命令

| 命令       | 用途              |
| ---------- | ----------------- |
| `/repo`    | 列出并选择仓库    |
| `/current` | 显示当前状态      |
| `/stop`    | 停止当前任务     |
| `/reset`   | 重置当前会话     |
| `/mode`    | 选择 Agent 模式   |
| `/model`   | 选择模型          |
| `/help`    | 显示帮助          |

## 项目结构

```
~/workspace/github/baton/
├── src/
│   ├── cli.ts                  # CLI 交互模式入口
│   ├── feishu-server.ts        # 飞书服务器入口
│   ├── telegram-server.ts       # Telegram 服务器入口
│   ├── whatsapp-server.ts       # WhatsApp 服务器入口
│   ├── slack-server.ts          # Slack 服务器入口
│   ├── config/                 # 配置加载
│   ├── core/                   # 核心逻辑
│   ├── im/                     # IM 适配器
│   └── utils/                  # 工具函数
├── package.json
├── tsconfig.json
└── bun.lock                   # 依赖锁定文件
```

## 常见问题

### Q1: 运行 `baton feishu` 没有输出

**A:** 检查以下项：
1. 确认 `~/workspace/baton.config.json` 存在
2. 确认配置文件的 key 是 `feishu`（不是 `feishu`）
3. 确认依赖已安装：检查 `node_modules` 目录

### Q2: 安装依赖时遇到 `bnpm.byted.org` 错误

**A:** 删除 `bun.lock` 后重新安装：
```bash
rm bun.lock
bun install
```

### Q3: 使用代理后 npmjs.org 返回 500

**A:** 不要使用代理，直接运行 `bun install`

### Q4: `claude-code-acp` 命令找不到

**A:** 重新加载 shell 配置：
```bash
source ~/.zshrc
```

## 更新记录

### 2026-02-16

- ✅ 安装 bun 运行时
- ✅ 安装项目依赖
- ✅ 安装 Claude Code ACP (`@zed-industries/claude-code-acp`)
- ✅ 创建全局 `baton` 命令
- ✅ 创建飞书配置文件
- ✅ 成功启动飞书服务器并连接
- ✅ 推送到 GitHub fork: https://github.com/T0UGH/baton
