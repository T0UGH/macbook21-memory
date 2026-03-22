# TODO - Mailbox E2E 测试环境准备

## 背景

目标：准备一个干净的 Docker 环境，部署多个 OpenClaw 都接入 mailbox 系统，然后互相收发消息看看效果。

## 当前状态

- [ ] Docker Desktop 已安装，但需要手动配置
- [ ] 需要你手动运行 sudo 命令激活 docker 命令

## 下一步行动

### 1. 激活 Docker 命令

你回到电脑后需要运行：

```bash
sudo ln -sf /Applications/Docker.app/Contents/Resources/bin/docker /usr/local/bin/docker
```

### 2. 验证 Docker 可用

```bash
docker --version
docker info
```

### 3. 构建测试环境

需要做的：
1. 启动 mailbox-server（Go）
2. 构建 mailbox-cli（Node.js）
3. 可能需要多个 HappyClaw 实例或模拟多 agent 环境

### 4. E2E 测试场景

- 多个 OpenClaw 接入同一个 mailbox-server
- 互相收发消息
- 验证 obligation 追踪
- 验证 Hex 自动检测和处理

## 相关文档

- `/Users/wangguiping/workspace/memory/2026-03-22-hex-mailbox-check-design-detailed.md` - Hex 自动检查 Mailbox 设计方案

## 相关仓库

- `T0UGH/openclaw-mailbox-server` - Go 实现
- `T0UGH/openclaw-mailbox-cli` - Node.js CLI
- `T0UGH/openclaw-mailbox-plugin` - OpenClaw 插件
