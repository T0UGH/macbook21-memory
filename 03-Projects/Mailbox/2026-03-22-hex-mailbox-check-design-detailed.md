# Hex 自动检查 Mailbox 完整技术方案

## 目标

Hex 定期检查 mailbox 是否有需要处理的 obligation，有则自动处理。

---

## 一、Hex 调用 Mailbox CLI 的方式

Hex 可以通过两种方式调用 mailbox CLI：

### 方式 1：直接执行 shell 命令（推荐）

Hex 所在的工作目录已经有 mailbox CLI，直接执行：

```bash
mailbox status
mailbox pick-mail
mailbox open <message_id>
```

### 方式 2：通过 MCP 工具

如果有 MCP 工具可用，可以通过 MCP 调用。

---

## 二、Mailbox CLI 命令详解

### 1. 检查状态

```bash
mailbox status
```

**输出格式**：
```json
{
  "total_unread": 5,
  "open_obligations_count": 2,
  "folders": [
    {"name": "INBOX", "unread": 5, "obligations": 2}
  ]
}
```

### 2. 选取一封需要处理的邮件

```bash
mailbox pick-mail
```

**输出格式**：
```json
{
  "ok": true,
  "message_id": "msg_12345",
  "from": "john@example.com",
  "subject": "Meeting tomorrow",
  "summary": "邀请你参加明天下午 3 点的会议",
  "received_at": "2026-03-22T10:30:00Z",
  "requires_reply": true,
  "reason": "meeting_invite"
}
```

### 3. 打开邮件详情

```bash
mailbox open msg_12345
```

**输出格式**：
```json
{
  "ok": true,
  "message_id": "msg_12345",
  "from": "john@example.com",
  "to": "wangguiping@example.com",
  "subject": "Meeting tomorrow",
  "body": "Hi, would you like to meet tomorrow? Let's discuss the project.",
  "snippet": "Hi, would you like to meet tomorrow?",
  "requires_reply": true,
  "reason": "meeting_invite"
}
```

---

## 三、Hex 的行为流程

### 第一次：Hex 需要初始化检查逻辑

Hex 启动时或第一次处理用户消息后，执行：

```bash
# 检查 mailbox 状态
mailbox status
```

如果 `open_obligations_count > 0`，则进入处理流程。

### 完整循环

```
1. 处理用户消息（现有逻辑）
       │
       ▼
2. 检查 mailbox 状态
   $ mailbox status
       │
       ▼
3. 解析输出
   {
     "open_obligations_count": 2,
     ...
   }
       │
       ▼
4. 如果 obligations > 0
       │
       ├── 4.1 选取最紧急的邮件
       │    $ mailbox pick-mail
       │
       ├── 4.2 打开邮件详情
       │    $ mailbox open <message_id>
       │
       ├── 4.3 生成处理建议
       │    - 摘要
       │    - 是否需要回复
       │    - 紧急程度
       │
       └── 4.4 自动处理（完全自动化，不需要用户确认）
            - 分析邮件内容，判断处理方式
            - 自动回复 / 归档 / 忽略
       │
       ▼
5. 等待下一次触发（定时）
```

---

## 四、Hex 具体要加的代码

Hex 需要在自己的行为逻辑里加一个函数：

```typescript
/**
 * 检查 mailbox 是否有需要处理的邮件
 */
async function checkMailbox() {
  try {
    // 1. 检查状态
    const statusOutput = await execShell('mailbox status');
    const status = JSON.parse(statusOutput);
    
    if (!status.open_obligations_count || status.open_obligations_count === 0) {
      return; // 没有需要处理的邮件
    }
    
    // 2. 有 obligation，选取最紧急的邮件
    const pickOutput = await execShell('mailbox pick-mail');
    const mail = JSON.parse(pickOutput);
    
    if (!mail.ok) {
      return;
    }
    
    // 3. 打开详情
    const openOutput = await execShell(`mailbox open ${mail.message_id}`);
    const fullMail = JSON.parse(openOutput);
    
    // 4. 生成处理建议
    const suggestion = generateSuggestion(fullMail);
    
    // 5. 通知用户
    await sendMessageToUser(
      `📧 新邮件\n` +
      `From: ${fullMail.from}\n` +
      `Subject: ${fullMail.subject}\n` +
      `\n${suggestion.summary}\n` +
      `\n需要回复: ${fullMail.requires_reply ? '是' : '否'}`
    );
    
    // 6. 可选：生成回复草稿
    if (fullMail.requires_reply) {
      const draft = await generateReplyDraft(fullMail);
      await sendMessageToUser(`📝 回复草稿:\n${draft}`);
    }
    
  } catch (error) {
    console.error('检查 mailbox 失败:', error);
  }
}

/**
 * 执行 shell 命令的辅助函数
 */
async function execShell(command: string): Promise<string> {
  // Hex 可以通过 MCP 工具或直接执行 shell
  // 具体实现取决于 Hex 的工具能力
}

/**
 * 生成邮件摘要
 */
function generateSuggestion(mail): string {
  return {
    summary: mail.snippet || mail.body?.slice(0, 200),
    urgent: mail.reason === 'meeting_invite' || mail.reason === 'deadline',
    requiresReply: mail.requires_reply
  };
}
```

---

## 五、触发时机

### 触发方式：定时检查（推荐）

类似 OpenClaw 的 heartbeat 机制，Hex 定期主动检查 mailbox：

```typescript
// 每 5 分钟检查一次
setInterval(async () => {
  await checkMailbox();
}, 5 * 60 * 1000);
```

**为什么推荐定时检查**：
- 不依赖用户发消息
- 用户不发消息时也能处理邮件
- 更像"后台服务"的主动模式
- 和 OpenClaw heartbeat 机制类似

### 备选：每次处理完用户消息后检查

Hex 处理完用户消息后，顺手检查 mailbox：

```typescript
async function handleUserMessage(message: string) {
  // 原有逻辑：处理用户消息
  const response = await processMessage(message);
  
  // 新增：检查 mailbox
  await checkMailbox();
  
  return response;
}
```

**这个方案的缺点**：
- 只有用户主动发消息时才检查
- 用户不发消息时邮件会堆积

---

## 六、需要 Hex 实现的具体步骤

### Step 1: 实现 execShell 辅助函数

Hex 需要能执行 shell 命令并获取输出。

### Step 2: 实现 checkMailbox 函数

按照上面的流程实现检查逻辑。

### Step 3: 在处理完用户消息后调用

在现有处理流程的最后加上 `await checkMailbox()`。

### Step 4: 测试

1. 手动跑 `mailbox status` 确认 CLI 可用
2. 制造一些 obligation 测试
3. 验证 Hex 能正确检测和处理

---

## 七、已知限制

1. **Hex 需要有执行 shell 命令的能力**（通过 MCP 工具或直接执行）
2. **Mailbox CLI 需要在 Hex 的工作目录下可用**
3. **处理方式需要进一步确认**（直接回复 / 生成草稿 / 只通知）

---

## 八、待确认问题

1. **处理方式**：完全自动化，不需要用户确认。Hex 分析邮件后自动决定处理方式（回复/归档/忽略/通知）。
2. **定时间隔**：5 分钟是否合适？（可调整）
3. **CLI 路径**：mailbox CLI 是否已经在 Hex 工作目录下可用？

## 九、自动化处理决策

Hex 根据邮件内容自动决定处理方式：

| 邮件类型 | 处理方式 |
|---------|----------|
| 需要回复（requires_reply=true） | 自动生成回复并发送 |
| 会议邀请 | 自动回复确认 |
| Newsletter/通知 | 归档 |
| 重要邮件 | 通知用户 + 处理 |
| 其他 | 归档或忽略 |
