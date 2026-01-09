# 快速代码模式参考

常见 claude-agent-sdk 模式的快速参考。所有示例均来自官方文档。

## 基本设置

### TypeScript - 简单查询
```typescript
import { query } from '@anthropic-ai/claude-agent-sdk';

for await (const message of query({
  prompt: "分析这个代码库",
  options: {
    allowedTools: ['Read', 'Grep', 'Glob']
  }
})) {
  if (message.type === 'result') console.log(message.result);
}
```

### Python - 简单查询
```python
from claude_agent_sdk import query

async for message in query(
    prompt="分析这个代码库",
    options={"allowed_tools": ["Read", "Grep", "Glob"]}
):
    if hasattr(message, 'result'):
        print(message.result)
```

---

## 自定义工具

### 模式：API 集成工具

**目的**：创建调用外部 API 的工具
**文档**：Custom-Tools.md

**TypeScript**：
```typescript
import { createSdkMcpServer, tool } from "@anthropic-ai/claude-agent-sdk";
import { z } from "zod";

const weatherServer = createSdkMcpServer({
  name: "weather-api",
  version: "1.0.0",
  tools: [
    tool(
      "get_weather",
      "获取位置的当前温度",
      {
        latitude: z.number(),
        longitude: z.number()
      },
      async (args) => {
        const response = await fetch(
          `https://api.open-meteo.com/v1/forecast?latitude=${args.latitude}&longitude=${args.longitude}&current=temperature_2m`
        );
        const data = await response.json();
        return {
          content: [{
            type: "text",
            text: `温度：${data.current.temperature_2m}°F`
          }]
        };
      }
    )
  ]
});

// 与流式输入一起使用（MCP 需要）
async function* generateMessages() {
  yield {
    type: "user" as const,
    message: { role: "user" as const, content: "SF 的天气怎么样？" }
  };
}

for await (const message of query({
  prompt: generateMessages(),
  options: {
    mcpServers: { "weather-api": weatherServer },
    allowedTools: ["mcp__weather-api__get_weather"]
  }
})) { /* ... */ }
```

**Python**：
```python
from claude_agent_sdk import create_sdk_mcp_server, tool
import aiohttp

@tool("get_weather", "获取当前温度", {"latitude": float, "longitude": float})
async def get_weather(args):
    async with aiohttp.ClientSession() as session:
        async with session.get(
            f"https://api.open-meteo.com/v1/forecast?latitude={args['latitude']}&longitude={args['longitude']}&current=temperature_2m"
        ) as response:
            data = await response.json()
    return {"content": [{"type": "text", "text": f"温度：{data['current']['temperature_2m']}°F"}]}

server = create_sdk_mcp_server("weather-api", "1.0.0", [get_weather])
```

---

## 权限

### 模式：文件保护钩子

**目的**：阻止修改敏感文件
**文档**：Control-Execute-With-Hooks.md

**TypeScript**：
```typescript
import { query, HookCallback } from "@anthropic-ai/claude-agent-sdk";

const protectEnvFiles: HookCallback = async (input, toolUseID) => {
  const filePath = input.tool_input?.file_path as string;
  if (filePath?.endsWith('.env')) {
    return {
      hookSpecificOutput: {
        hookEventName: input.hook_event_name,
        permissionDecision: 'deny',
        permissionDecisionReason: '无法修改 .env 文件'
      }
    };
  }
  return {};
};

for await (const message of query({
  prompt: "更新配置",
  options: {
    hooks: {
      PreToolUse: [{ matcher: 'Write|Edit', hooks: [protectEnvFiles] }]
    }
  }
})) { /* ... */ }
```

**Python**：
```python
from claude_agent_sdk import query, HookMatcher

async def protect_env_files(input_data, tool_use_id, context):
    file_path = input_data['tool_input'].get('file_path', '')
    if file_path.endswith('.env'):
        return {
            'hookSpecificOutput': {
                'hookEventName': input_data['hook_event_name'],
                'permissionDecision': 'deny',
                'permissionDecisionReason': '无法修改 .env 文件'
            }
        }
    return {}

async for message in query(
    prompt="更新配置",
    options={
        'hooks': {
            'PreToolUse': [HookMatcher(matcher='Write|Edit', hooks=[protect_env_files])]
        }
    }
): # ...
```

---

## 子代理

### 模式：并行代码审查

**目的**：并发运行多个审查任务
**文档**：Subagents.md

**TypeScript**：
```typescript
import { query } from '@anthropic-ai/claude-agent-sdk';

for await (const message of query({
  prompt: "审查此 PR 的问题",
  options: {
    allowedTools: ['Read', 'Grep', 'Glob', 'Task'],
    agents: {
      'security-reviewer': {
        description: '安全代码审查员',
        prompt: '识别安全漏洞和最佳实践违规。',
        tools: ['Read', 'Grep', 'Glob'],
        model: 'sonnet'
      },
      'style-checker': {
        description: '代码样式和格式检查器',
        prompt: '检查代码样式、格式和命名约定。',
        tools: ['Read', 'Grep', 'Glob']
      },
      'test-analyzer': {
        description: '测试覆盖率分析器',
        prompt: '分析测试覆盖率并建议缺失的测试。',
        tools: ['Read', 'Grep', 'Glob', 'Bash']
      }
    }
  }
})) {
  if ('result' in message) console.log(message.result);
}
```

**Python**：
```python
from claude_agent_sdk import query, AgentDefinition

async for message in query(
    prompt="审查此 PR 的问题",
    options={
        'allowed_tools': ['Read', 'Grep', 'Glob', 'Task'],
        'agents': {
            'security-reviewer': AgentDefinition(
                description='安全代码审查员',
                prompt='识别安全漏洞和最佳实践违规。',
                tools=['Read', 'Grep', 'Glob'],
                model='sonnet'
            ),
            'style-checker': AgentDefinition(
                description='代码样式和格式检查器',
                prompt='检查代码样式、格式和命名约定。',
                tools=['Read', 'Grep', 'Glob']
            ),
            'test-analyzer': AgentDefinition(
                description='测试覆盖率分析器',
                prompt='分析测试覆盖率并建议缺失的测试。',
                tools=['Read', 'Grep', 'Glob', 'Bash']
            )
        }
    }
):
    if hasattr(message, 'result'):
        print(message.result)
```

---

## 结构化输出

### 模式：TODO 提取与 Git 归因

**目的**：提取包含作者信息的 TODO
**文档**：Structured-outputs.md

**TypeScript**：
```typescript
import { query } from '@anthropic-ai/claude-agent-sdk';

const todoSchema = {
  type: 'object',
  properties: {
    todos: {
      type: 'array',
      items: {
        type: 'object',
        properties: {
          text: { type: 'string' },
          file: { type: 'string' },
          line: { type: 'number' },
          author: { type: 'string' },
          date: { type: 'string' }
        },
        required: ['text', 'file', 'line']
      }
    },
    total_count: { type: 'number' }
  },
  required: ['todos', 'total_count']
};

for await (const message of query({
  prompt: '查找 src/ 中的所有 TODO 注释并识别谁添加了它们',
  options: {
    outputFormat: {
      type: 'json_schema',
      schema: todoSchema
    }
  }
})) {
  if (message.type === 'result' && message.structured_output) {
    console.log(`找到 ${message.structured_output.total_count} 个 TODO`);
    message.structured_output.todos.forEach(todo => {
      console.log(`${todo.file}:${todo.line} - ${todo.text}`);
    });
  }
}
```

**Python**：
```python
from claude_agent_sdk import query

todo_schema = {
    "type": "object",
    "properties": {
        "todos": {
            "type": "array",
            "items": {
                "type": "object",
                "properties": {
                    "text": {"type": "string"},
                    "file": {"type": "string"},
                    "line": {"type": "number"},
                    "author": {"type": "string"},
                    "date": {"type": "string"}
                },
                "required": ["text", "file", "line"]
            }
        },
        "total_count": {"type": "number"}
    },
    "required": ["todos", "total_count"]
}

async for message in query(
    prompt="查找 src/ 中的所有 TODO 注释并识别谁添加了它们",
    options={"output_format": {"type": "json_schema", "schema": todo_schema}}
):
    if hasattr(message, 'structured_output'):
        print(f"找到 {message.structured_output['total_count']} 个 TODO")
```

---

## 会话管理

### 模式：恢复对话

**目的**：继续之前的对话
**文档**：Session-Management.md

**TypeScript**：
```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

// 第一次对话 - 捕获会话 ID
let sessionId: string;

for await (const message of query({
  prompt: "开始分析代码库"
})) {
  if (message.type === 'system' && message.subtype === 'init') {
    sessionId = message.session_id;
    console.log(`会话：${sessionId}`);
  }
}

// 稍后 - 恢复会话
for await (const message of query({
  prompt: "继续分析",
  options: { resume: sessionId }
})) { /* ... */ }
```

**Python**：
```python
from claude_agent_sdk import query

# 第一次对话
session_id = None
async for message in query(prompt="开始分析代码库"):
    if hasattr(message, 'subtype') and message.subtype == 'init':
        session_id = message.data.get('session_id')

# 稍后 - 恢复
async for message in query(
    prompt="继续分析",
    options={"resume": session_id}
): # ...
```

---

## 成本跟踪

### 模式：监控 API 使用

**目的**：跟踪令牌使用和成本
**文档**：Tracking-Costs.md

**TypeScript**：
```typescript
import { query } from '@anthropic-ai/claude-agent-sdk';

let totalTokens = 0;

for await (const message of query({
  prompt: "复杂分析任务",
  options: {
    onUsage: (usage) => {
      totalTokens += usage.input_tokens + usage.output_tokens;
      console.log(`使用的令牌：${usage.input_tokens + usage.output_tokens}`);
      console.log(`总计：${totalTokens}`);
    }
  }
})) { /* ... */ }
```

**Python**：
```python
from claude_agent_sdk import query

total_tokens = 0

def track_usage(usage):
    global total_tokens
    tokens = usage['input_tokens'] + usage['output_tokens']
    total_tokens += tokens
    print(f"使用的令牌：{tokens}，总计：{total_tokens}")

async for message in query(
    prompt="复杂分析任务",
    options={"on_usage": track_usage}
): # ...
```

---

## 常见配置选项

### TypeScript 选项摘要
```typescript
{
  allowedTools: ['Read', 'Write', 'Bash', 'Grep', 'Glob'],
  permissionMode: 'default' | 'acceptEdits' | 'bypassPermissions',
  canUseTool: async (toolName, input) => ({ behavior: 'allow' }),
  hooks: { PreToolUse: [...], PostToolUse: [...] },
  agents: { 'agent-name': AgentDefinition },
  mcpServers: { 'server-name': McpServer },
  outputFormat: { type: 'json_schema', schema: {...} },
  settingSources: ['user', 'project'],
  onUsage: (usage) => { /* track */ },
  model: 'claude-sonnet-4-5',
  cwd: '/path/to/project'
}
```

### Python 选项摘要
```python
{
    "allowed_tools": ["Read", "Write", "Bash", "Grep", "Glob"],
    "permission_mode": "default" | "acceptEdits" | "bypassPermissions",
    "can_use_tool": lambda tool, input: {"behavior": "allow"},
    "hooks": {"PreToolUse": [...], "PostToolUse": [...]},
    "agents": {"agent-name": AgentDefinition(...)},
    "mcp_servers": {"server-name": mcp_server},
    "output_format": {"type": "json_schema", "schema": {...}},
    "setting_sources": ["user", "project"],
    "on_usage": lambda usage: None,
    "model": "claude-sonnet-4-5",
    "cwd": "/path/to/project"
}
```

---

## 关键差异：TypeScript 与 Python

| 功能 | TypeScript | Python |
|:-----|:-----------|:-------|
| **选项** | camelCase | snake_case |
| **异步迭代** | `for await (const x of ...)` | `async for x in ...` |
| **类型模式** | Zod (`z.string()`) | 类型提示或 Pydantic |
| **导入** | `import { query }` | `from ... import query` |
| **工具装饰器** | 不适用（使用 `tool()`） | `@tool(...)` |
| **MCP 流式传输** | 需要异步生成器 | 需要异步生成器 |

---

*所有模式均来自官方 claude-agent-sdk 文档*
