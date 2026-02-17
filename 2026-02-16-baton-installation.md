# Baton 安装总结

**日期:** 2026-02-16
**项目:** https://github.com/williamfzc/baton

## 安装步骤

### 1. 安装 bun 运行时

```bash
curl -fsSL https://bun.sh/install | bash
```

### 2. 安装项目依赖

```bash
cd ~/workspace/github/baton
bun install
```

**注意:** 不要使用代理，代理会导致 npmjs.org 返回 500 错误。

### 3. 运行 baton

```bash
# CLI 模式
bun run start:cli

# Feishu 模式
bun run start:feishu

# Telegram 模式
bun run start:telegram

# WhatsApp 模式
bun run start:whatsapp

# Slack 模式
bun run start:slack
```

## 支持的 IM 平台

| 平台       | 状态 | 说明                   |
| ---------- | ---- | ---------------------- |
| **Feishu** | ✅   | WebSocket 长连接       |
| **Telegram** | ✅   | Bot API 长轮询        |
| **WhatsApp** | ✅   | Webhook + Graph API    |
| **Slack**  | ✅   | Events API webhook     |
| **CLI**    | ✅   | 本地终端交互          |
| **Discord**| 🔮   | 计划中                |

## 配置文件

创建 `baton.config.json`:

```json
{
  "project": {
    "path": "/path/to/your/project",
    "name": "my-project"
  },
  "feishu": {
    "appId": "cli_xxxxxxxxxxxxxxxx",
    "appSecret": "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
    "domain": "feishu"
  },
  "telegram": {
    "botToken": "123456:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
  },
  "acp": {
    "executor": "opencode"
  }
}
```

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

## 问题排查与修复过程

### 遇到的问题

运行 `bun install` 时遇到以下错误：

```
error: GET https://bnpm.byted.org/xxx - 500
```

bun 始终使用字节内部的 npm 镜像源 `bnpm.byted.org`，无法切换到官方 npm 源。

### 尝试过的方案（均无效）

1. 创建 `.npmrc` 文件设置 `registry=https://registry.npmjs.org/`
2. 创建 `~/.bun/config.toml` 配置文件
3. 设置环境变量 `BUN_CONFIG_REGISTRY_URL`
4. 使用 `--registry https://registry.npmjs.org/` 参数
5. 重新安装官方 bun 二进制文件
6. 尝试安装 Node.js/npm

### 最终解决方案

```bash
# 删除 bun.lock 文件
rm bun.lock

# 不使用代理，直接安装
bun install
```

**原因分析:**

- `bun.lock` 文件中保存了依赖包的下载 URL，这些 URL 指向了 `bnpm.byted.org`
- 删除 `bun.lock` 后，bun 会重新解析依赖并使用正确的 registry
- 之前设置的代理（`http_proxy=http://127.0.0.1:1087`）会导致 npmjs.org 返回 500 错误

## 已知问题

1. **npm registry 问题:** `bun.lock` 文件可能包含过时的 registry 配置，遇到问题可尝试删除
2. **代理问题:** 使用某些代理会导致 npmjs.org 返回 500 错误，建议不使用代理

## ACP Runtime 支持

- **opencode**: 默认，需要 opencode CLI
- **codex**: 需要 `codex-acp` 在 PATH 中
- **claude**: 需要 `claude-code-acp` 在 PATH 中
