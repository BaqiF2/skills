# 故障排除索引

使用 claude-agent-sdk 开发时的常见问题快速参考。

## 按症状分类

### 技能

#### 技能未找到 / 未加载
**症状**：代理说技能不可用，或技能未触发

**主要文档**：Agent-Skills.md → 故障排除 > 技能未找到

**常见原因**：
1. 缺少 `settingSources` 配置
2. 错误的工作目录 (`cwd`)
3. 技能不在 `allowedTools` 中
4. SKILL.md 语法错误

**诊断步骤**：
```bash
# 1. 验证 settingSources 已配置
# TS: settingSources: ['user', 'project']
# PY: setting_sources=["user", "project"]

# 2. 检查技能目录是否存在
ls .claude/skills/*/SKILL.md      # 项目技能
ls ~/.claude/skills/*/SKILL.md    # 用户技能

# 3. 验证技能工具已允许
# allowedTools 必须包含 "Skill"

# 4. 检查 YAML 语法
# SKILL.md 必须有有效的 YAML 前言
```

**快速修复**：
```typescript
// TypeScript
options: {
  settingSources: ['user', 'project'],  // 添加这个
  allowedTools: ['Skill', /* 其他工具 */]
}
```
```python
# Python
options = {
    "setting_sources": ["user", "project"],  # 添加这个
    "allowed_tools": ["Skill", # 其他工具]
}
```

**相关部分**：
- Agent-Skills.md > 使用 SDK 的技能
- Agent-Skills.md > 技能位置

---

#### 技能未被使用
**症状**：技能存在但 Claude 未调用它

**主要文档**：Agent-Skills.md → 故障排除 > 技能未被使用

**常见原因**：
1. 描述与用户请求不匹配
2. 技能工具未启用
3. 关键字不够具体

**解决方案**：
1. 改进 `description` 字段，添加相关关键字
2. 在提示中显式提及技能名称
3. 确保 `"Skill"` 在 `allowedTools` 中

**示例修复**：
```yaml
# 之前（太模糊）
description: "数据处理助手"

# 之后（具体）
description: "PDF 文档处理专家。用于从 PDF 提取文本、解析 PDF 表格或分析 PDF 结构。"
```

---

### MCP 和自定义工具

#### MCP 工具不可用
**症状**：自定义 MCP 工具不出现或无法调用

**主要文档**：MCP.md → 错误处理、Custom-Tools.md

**常见原因**：
1. 未使用流式输入模式（MCP 需要）
2. MCP 服务器不在 `mcpServers` 选项中
3. 工具不在 `allowedTools` 中
4. 错误的工具名称格式

**诊断步骤**：
```typescript
// 1. 检查您是否对提示使用异步生成器
async function* generateMessages() {  // ✓ 必需
  yield { type: "user", message: { role: "user", content: "..." } };
}

// ❌ 这不适用于 MCP：
query({ prompt: "string prompt", ... })

// 2. 验证 MCP 服务器配置
options: {
  mcpServers: {
    "my-server": customServer  // ✓ 存在
  }
}

// 3. 检查工具名称格式
allowedTools: [
  "mcp__my-server__my-tool"  // 格式：mcp__<server>__<tool>
]
```

**快速修复**：
```typescript
// TypeScript
async function* input() {
  yield { type: "user" as const, message: { role: "user" as const, content: "使用我的工具" } };
}

for await (const msg of query({
  prompt: input(),  // ✓ 异步生成器
  options: {
    mcpServers: { "my-tools": myServer },
    allowedTools: ["mcp__my-tools__my-tool"]
  }
})) { /* ... */ }
```

**相关部分**：
- Custom-Tools.md > 使用自定义工具
- MCP.md > 配置
- Streaming-Input.md

---

#### MCP 服务器连接失败
**症状**：MCP 服务器连接失败，工具不可用

**主要文档**：MCP.md → 错误处理

**常见原因**：
1. stdio 服务器的命令或参数错误
2. HTTP/SSE 服务器的网络问题
3. 认证失败
4. 环境变量未设置

**诊断步骤**：
```typescript
// 检查 init 消息中的 MCP 服务器状态
for await (const msg of query({ ... })) {
  if (msg.type === 'system' && msg.subtype === 'init') {
    console.log('MCP 服务器：', msg.mcp_servers);
    // 检查每个服务器的状态
    msg.mcp_servers.forEach(s => {
      if (s.status !== 'connected') {
        console.error('失败：', s.name, s.error);
      }
    });
  }
}
```

**解决方案**：
- **stdio**：验证命令存在且可执行
- **HTTP/SSE**：检查 URL 和网络连接
- **Auth**：验证环境变量已设置
- **Headers**：检查认证头

**相关部分**：
- MCP.md > 传输类型
- MCP.md > 认证

---

### 权限

#### 工具权限被拒绝
**症状**：工具执行被阻止，权限被拒绝错误

**主要文档**：Handling-Permissions.md

**常见原因**：
1. 工具不在 `allowedTools` 中
2. 被权限规则阻止（拒绝规则）
3. `canUseTool` 回调拒绝
4. 钩子返回 'deny'

**诊断流程**：
```
1. 检查 allowedTools
   ├─ 工具在数组中吗？
   ├─ 工具名称正确吗？
   └─ 检查拼写错误

2. 检查权限规则（settings.json）
   ├─ 有匹配的拒绝规则吗？
   └─ 权限模式已设置吗？

3. 检查 canUseTool 回调
   └─ 它是否返回 'deny'？

4. 检查钩子
   └─ PreToolUse 钩子拒绝了吗？
```

**快速修复**：
```typescript
// 1. 将工具添加到 allowedTools
allowedTools: ['Read', 'Write', 'Bash', 'YourTool']

// 2. 检查权限模式
permissionMode: 'default',  // 不是 'plan' 模式

// 3. 验证 canUseTool 允许它
canUseTool: async (toolName, input) => {
  console.log('检查：', toolName);  // 调试
  return { behavior: 'allow' };
}
```

**相关部分**：
- Handling-Permissions.md > 权限流程图
- Handling-Permissions.md > canUseTool
- Control-Execute-With-Hooks.md > PreToolUse

---

#### 权限回调未被调用
**症状**：`canUseTool` 回调从未触发

**主要文档**：Handling-Permissions.md → 权限流程图

**理解**：
`canUseTool` 在以下情况下作为**后备**被调用：
- 没有钩子允许/拒绝
- 没有权限规则匹配
- 不在 `bypassPermissions` 模式

**如果未调用，是因为**：
- 钩子已经允许/拒绝工具
- 权限规则已经处理它
- 在 `bypassPermissions` 模式

**诊断**：
```typescript
canUseTool: async (toolName, input) => {
  console.log(`为以下工具调用 canUseTool：${toolName}`);  // 添加日志
  return { behavior: 'allow' };
}
```

如果没有日志，检查：
1. 钩子是否处理权限？
2. 权限规则是否匹配？
3. 权限模式是否设置为 `bypassPermissions`？

**相关部分**：
- Handling-Permissions.md > 权限流程图
- Control-Execute-With-Hooks.md

---

### 子代理

#### 子代理未被调用
**症状**：Claude 直接处理任务而不是委托给子代理

**主要文档**：Subagents.md → 故障排除 > Claude 不委托

**常见原因**：
1. `Task` 工具不在 `allowedTools` 中
2. 子代理描述与任务不匹配
3. 用户未显式请求子代理

**解决方案**：

**1. 确保启用任务工具：**
```typescript
allowedTools: ['Read', 'Write', 'Task']  // 子代理需要 Task
```

**2. 改进子代理描述：**
```typescript
agents: {
  'code-reviewer': {
    description: '专家代码审查员。用于安全审计、样式检查和最佳实践审查。',  // ✓ 具体
    // 不要是：'帮助处理代码'  // ❌ 太模糊
  }
}
```

**3. 显式请求子代理：**
```typescript
prompt: "使用代码审查代理来分析此代码"
```

**相关部分**：
- Subagents.md > 创建子代理
- Subagents.md > 调用子代理

#### 子代理无法使用工具
**症状**：子代理报告它无法使用某些工具

**常见原因**：工具不在子代理的 `tools` 数组中

**解决方案**：
```typescript
agents: {
  'my-agent': {
    description: '...',
    prompt: '...',
    tools: ['Read', 'Grep', 'Bash'],  // ✓ 显式列出工具
    // 如果省略，从父级继承所有工具
  }
}
```

**注意**：子代理不能生成其他子代理。不要在子代理工具中包含 `'Task'`。

**相关部分**：
- Subagents.md > 工具限制

---

### 结构化输出

#### 结构化输出验证失败
**症状**：错误 `error_max_structured_output_retries` 或无效 JSON

**主要文档**：Structured-outputs.md → 错误处理

**常见原因**：
1. 模式太复杂
2. `additionalProperties` 未设置为 `false`
3. 代理无法从可用数据产生所需格式

**解决方案**：

**1. 简化模式：**
```typescript
// ❌ 太复杂
{ type: 'object', properties: { /* 20+ 嵌套字段 */ } }

// ✓ 从简单开始
{ type: 'object', properties: { summary: { type: 'string' } } }
```

**2. 设置 additionalProperties：**
```typescript
{
  type: 'object',
  properties: { /* ... */ },
  additionalProperties: false  // ✓ 必需
}
```

**3. 检查数据可用性：**
确保代理有工具收集所需数据。

**相关部分**：
- Structured-outputs.md > 支持的 JSON 模式功能
- Structured-outputs.md > 错误处理

---

### 会话管理

#### 会话恢复失败
**症状**：无法恢复之前的对话，找不到会话

**主要文档**：Session-Management.md → 恢复会话

**常见原因**：
1. 未捕获会话 ID
2. 会话过期
3. 错误的会话 ID

**解决方案**：
```typescript
// 从 init 消息捕获会话 ID
let sessionId: string;

for await (const msg of query({ prompt: "..." })) {
  if (msg.type === 'system' && msg.subtype === 'init') {
    sessionId = msg.session_id;  // ✓ 保存这个
    console.log('会话：', sessionId);
  }
}

// 稍后，用正确的 ID 恢复
for await (const msg of query({
  prompt: "继续",
  options: { resume: sessionId }  // ✓ 使用保存的 ID
})) { /* ... */ }
```

**相关部分**：
- Session-Management.md > 获取会话 ID
- Session-Management.md > 恢复会话

---

## 按错误消息分类

### "技能未找到"
→ 参见[技能未找到](#技能未找到--未加载)

### "工具未允许"
→ 参见[工具权限被拒绝](#工具权限被拒绝)

### "MCP 服务器连接失败"
→ 参见[MCP 服务器连接失败](#mcp-服务器连接失败)

### "error_max_structured_output_retries"
→ 参见[结构化输出验证失败](#结构化输出验证失败)

### "会话未找到"
→ 参见[会话恢复失败](#会话恢复失败)

---

## 一般调试检查表

当某些东西不工作时：

### 1. 检查配置
- [ ] 功能工具是否在 `allowedTools` 中？
- [ ] 是否设置了必需选项（例如，技能的 `settingSources`）？
- [ ] `cwd` 指向正确的目录了吗？
- [ ] 环境变量设置了吗？

### 2. 检查控制台输出
- [ ] 查看系统消息（type === 'system'）
- [ ] 检查 init 消息中的服务器状态
- [ ] 在工具结果中查找错误消息

### 3. 启用日志记录
```typescript
for await (const msg of query({ ... })) {
  console.log('消息类型：', msg.type, msg.subtype);
  if (msg.type === 'system') {
    console.log('系统消息：', JSON.stringify(msg, null, 2));
  }
}
```

### 4. 简化
- [ ] 暂时移除钩子和权限
- [ ] 用最小配置测试
- [ ] 隔离您正在测试的功能

### 5. 查阅文档
- [ ] 完全阅读相关文档部分
- [ ] 检查文档中的代码示例
- [ ] 寻找注释和警告

---

## 快速参考：在哪里查找

| 问题区域 | 主要文档 | 要搜索的关键字 |
|:---------|:---------|:--------------|
| 技能 | Agent-Skills.md | settingSources, SKILL.md, description |
| 自定义工具 | Custom-Tools.md, MCP.md | createSdkMcpServer, tool, streaming |
| 权限 | Handling-Permissions.md | allowedTools, canUseTool, permissionMode |
| 钩子 | Control-Execute-With-Hooks.md | PreToolUse, PostToolUse, matcher |
| 子代理 | Subagents.md | agents, Task, description |
| 输出 | Structured-outputs.md | outputFormat, json_schema, validation |
| 会话 | Session-Management.md | resume, session_id |
| 流式传输 | Streaming-Input.md | async generator, AsyncIterable |
| 成本 | Tracking-Costs.md | onUsage, tokens |
| 部署 | Securely-deploying-AI-agents.md | security, production |

---

*有关详细解决方案，请始终查阅完整文档*
