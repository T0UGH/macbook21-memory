# Baton 飞书连接配置

**日期:** 2026-02-16

## 飞书应用配置

### 1. 创建飞书应用

1. 访问 [飞书开放平台](https://open.feishu.cn/)
2. 登录并创建应用
3. 进入 **凭证与基础信息** 获取：
   - `App ID`
   - `App Secret`

### 2. 配置应用权限

在应用权限页面添加以下权限：
- 获取与发送单聊消息
- 获取与发送群组消息

### 3. 配置订阅模式

在应用配置中选择：
- **开发者后台** 或 **事件与回调**
- **订阅方式**：使用长连接接收事件/回调

### 4. 配置 Baton

创建 `baton.config.json`：

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

**配置文件位置：**
- 文件：`~/workspace/baton.config.json`
- 支持的文件名：`baton.config.json`、`.batonrc.json`、`baton.json`
- 查找规则：从当前工作目录向上查找 5 层

### 5. 启动飞书服务

```bash
# 方式一：使用全局命令
baton feishu

# 方式二：从项目目录运行
cd ~/workspace/github/baton
bun run start:feishu
```

### 6. 验证连接

启动成功后应该看到：

```
╔════════════════════════════════════════╗
║        Baton Feishu Server             ║
║        (WebSocket Long Connection)     ║
╚════════════════════════════════════════╝
Project: .
App ID: cli_a918e3194038dcd5
Domain: feishu

Connecting to Feishu via WebSocket...

[FeishuAdapter] Starting WebSocket client...
[FeishuAdapter] WebSocket client started
[FeishuAdapter] ✅ Connected successfully!
Press Ctrl+C to exit

[ws] ws client ready
```

## 配置项说明

| 参数 | 必需 | 说明 |
| ---- | ---- | ---- |
| `appId` | ✓ | 飞书应用 ID |
| `appSecret` | ✓ | 飞书应用密钥 |
| `domain` | ✗ | 域名：`feishu`（默认）或 `lark` |
| `card.permissionTimeout` | ✗ | 权限确认卡片超时时间（秒） |

## 当前已配置信息

- **App ID:** `cli_a918e3194038dcd5`
- **Domain:** `feishu`
- **ACP Executor:** `claude-code`
- **配置文件位置:** `~/workspace/baton.config.json`

## 常用命令

| 命令 | 用途 |
| ---- | ---- |
| `/repo` | 列出并选择仓库 |
| `/current` | 显示当前状态 |
| `/stop` | 停止当前任务 |
| `/reset` | 重置当前会话 |
| `/mode` | 选择 Agent 模式 |
| `/model` | 选择模型 |
| `/help` | 显示帮助 |

## 环境变量配置（可选）

如果不想在配置文件中保存敏感信息，可以使用环境变量：

```bash
export BATON_FEISHU_APP_ID=cli_a918e3194038dcd5
export BATON_FEISHU_APP_SECRET=ftrQd33b8onq4mGUOv0rWfE2MaRCQQls
export BATON_FEISHU_DOMAIN=feishu
```

## 故障排查

### 问题：无法连接飞书

1. 检查网络连接
2. 确认应用权限已正确配置
3. 检查订阅模式是否正确（长连接）
4. 验证 App ID 和 App Secret 是否正确

### 问题：配置文件未加载

- 确认配置文件在当前目录或上级目录（5层内）
- 检查 JSON 格式是否正确
- 确认配置文件的 key 名称是 `feishu`（不是 `feishu`）
