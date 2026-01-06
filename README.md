# Personal Skills Repository

[中文版本](README-zh.md) | English Version

This repository serves as a personal collection of specialized skills for AI agent development. Each skill is designed to provide expert assistance in specific domains, complete with comprehensive documentation, code examples, and best practices.

## Skills Overview

### 1. Agent SDK Development Assistant (`agent-sdk-dev`)

A comprehensive expert assistant for developing with Claude Agent SDK, providing guidance on:

#### Core Capabilities
- **Custom Tools & MCP Servers** - Create and integrate custom tools using MCP (Model Context Protocol)
- **Security & Permissions** - Set up permission controls, security hooks, and access management
- **Subagents & Parallel Processing** - Implement subagents for parallel task execution and context isolation
- **Session & I/O Management** - Handle session management, streaming input/output, and structured outputs
- **Agent Skills & Commands** - Configure Agent Skills and slash commands for enhanced functionality
- **File Checkpointing & Prompts** - Customize system prompts and implement file checkpointing
- **Cost Tracking & Monitoring** - Track API costs, monitor usage, and optimize performance
- **Deployment & Production** - Host agents securely and deploy to production environments

#### Documentation Structure

The skill includes access to 20+ comprehensive documentation files covering:

**Core Concepts:**
- Agent SDK TypeScript API Reference
- Custom Tools Development
- MCP Server Integration
- Agent Skills System
- Structured Outputs
- Subagents & Parallel Execution

**Control & Security:**
- Permission Management
- Hook System for Interception
- Secure Deployment Practices

**Session & I/O:**
- Session Management
- Streaming Input/Output
- File Checkpointing
- Slash Commands

**Configuration:**
- System Prompt Customization
- Plugin System

**Operations:**
- Cost Tracking
- Todo Lists & Progress Monitoring
- Agent Hosting

#### Language Support
- **TypeScript** - Complete SDK support with type-safe schemas (Zod)
- **Python** - Full SDK implementation with type hints and Pydantic

#### Key Features
- Auto-loads relevant documentation based on user queries
- Provides working code examples from official docs
- Explains SDK features with context and best practices
- Guides through implementation workflows step-by-step
- Troubleshoots common issues with diagnostic steps
- Supports both TypeScript and Python implementations

#### Reference Materials
- **doc-map.json** - Quick mapping from development scenarios to documentation
- **quick-patterns.md** - Common code patterns for both TypeScript and Python
- **troubleshooting-index.md** - Symptom-based troubleshooting guide

## Repository Structure

```
skills/
├── agent-sdk-dev/                 # English version
│   ├── SKILL.md                   # Skill definition and guidelines
│   ├── claude-agent-sdk-doc/      # SDK documentation (20+ files)
│   └── references/                # Quick reference materials
│
├── agent-sdk-dev-zh/              # Chinese version
│   ├── SKILL.md                   # 技能定义和指南
│   ├── claude-agent-sdk-doc/      # SDK 文档（20+ 文件）
│   └── references/                # 快速参考材料
│
└── README.md                      # This file
```

## Usage

Each skill in this repository can be used as a specialized assistant for its respective domain. The Agent SDK Development Assistant automatically:

1. **Understands** your query and identifies the relevant SDK features needed
2. **Searches** the bundled documentation for accurate information
3. **Provides** concrete code examples in TypeScript or Python
4. **Explains** implementation steps and best practices
5. **Troubleshoots** common issues with diagnostic guidance

## Documentation Coverage

The SDK documentation spans over 400KB of content, including:
- Complete API references
- Step-by-step tutorials
- Integration patterns
- Security best practices
- Production deployment guides
- Performance optimization tips
- Comprehensive troubleshooting sections

## Best Practices

The skills emphasize:
- Always referencing official documentation
- Providing complete, runnable code examples
- Including proper error handling
- Supporting both TypeScript and Python
- Following security best practices
- Optimizing for performance and cost

---

**Note:** This repository is a personal skill collection designed to enhance productivity and provide expert-level assistance in specialized domains.
