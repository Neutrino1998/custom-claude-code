# Claude Code 源码学习笔记

Claude Code (v2.1.87) 构建中间产物的架构分析与学习项目。

## 这是什么

这份代码是 Claude Code **Bun 转译后、bundle 前的中间产物**，不是原始 TypeScript 源码，也不适合做二次开发。

### 构建流程中的位置

```
原始 TS 源码（Anthropic 内部，非公开）
    │  Bun 转译：擦除类型注解，注入 inline source map
    ▼
 转译产物（1,900 个文件，33MB）◄── 就是这份 build-output/
    │  Bun bundle：所有文件 + 第三方依赖 → 单文件
    ▼
 cli.js（13MB）◄── npm install @anthropic-ai/claude-code 得到的
    │  Bun compile：JS + Bun runtime → 原生二进制
    ▼
 Mach-O / ELF 可执行文件 ◄── claude 命令本体
```

### 和原始源码的区别

| | 原始源码 | 这份 build-output（转译产物） | npm 包 |
|---|---|---|---|
| 类型注解 | 完整 | 被擦除 | 被擦除 |
| 模块结构 | 1,900+ 独立文件 | 1,900 独立文件 | 单个 cli.js |
| 第三方依赖 | import 引用 | import 引用 | 已内联 |
| source map | 无 | inline base64（含原始源码） | 无 |
| package.json | 完整依赖声明 | 无 | 仅元信息 |
| tsconfig.json | 有 | 无 | 无 |
| 测试文件 | 有 | 无 | 无 |
| 可直接开发 | 是 | 不可行（无类型系统） | 不可能 |
| 可直接运行 | 需构建 | 需装依赖 | 可以 |

### 为什么不适合二次开发

- **无类型系统**：所有 TypeScript 类型注解已被擦除，改错了只有运行时炸
- **无项目基础设施**：缺少 tsconfig.json、package.json 完整依赖声明、测试文件
- **本质是编译产物**：相当于拿 `.class` 文件做 Java 开发，技术上可行但极其痛苦

### 适合做什么

- 阅读完整代码逻辑，理解 AI Agent 的工程实现
- 研究架构设计模式（工具系统、权限模型、会话管理等）
- 分析提示词工程（系统提示词、工具描述、技能模板）
- 了解 CLI 产品的 UX 设计思路

## 目录结构

```
custom-claude-code/
├── build-output/                  # Claude Code 转译产物
│   ├── main.tsx          # 主入口（4,683 行）
│   ├── services/         # API 层、MCP、分析等服务
│   │   └── api/          # Anthropic API 客户端
│   ├── tools/            # 内置工具（Bash, Read, Edit, Grep 等）
│   ├── commands/         # CLI 命令
│   ├── components/       # Ink (React terminal UI) 组件
│   ├── constants/        # 系统提示词、OAuth 配置等常量
│   ├── utils/            # 工具函数
│   │   └── model/        # 模型配置与选择逻辑
│   ├── skills/           # 技能系统
│   └── hooks/            # React hooks
├── claw-code/                     # 社区重写项目参考（Python / Rust）
├── docs/
│   ├── security-audit.md       # 安全审计报告
│   └── architecture-notes.md   # 架构学习笔记
├── .gitignore
└── README.md
```

## 技术栈

- **运行时**: Bun
- **语言**: TypeScript（转译为 JS，后缀未改）
- **UI**: Ink (React for CLI)
- **API SDK**: @anthropic-ai/sdk

## 学习重点

### 1. 提示词工程

这份代码最有研究价值的部分 —— 所有提示词和工具定义都是明文。

#### 系统提示词

| 文件 | 内容 |
|------|------|
| `build-output/constants/prompts.ts` | 主系统提示词 |
| `build-output/constants/systemPromptSections.ts` | 系统提示词各段落拼装逻辑 |
| `build-output/constants/common.ts` | 通用提示词片段 |

#### 工具 Prompt（每个工具的行为指令）

所有工具的提示词都在 `build-output/tools/*/prompt.ts`：

| 工具 | 文件 |
|------|------|
| Bash | `build-output/tools/BashTool/prompt.ts` |
| 文件读取 | `build-output/tools/FileReadTool/prompt.ts` |
| 文件编辑 | `build-output/tools/FileEditTool/prompt.ts` |
| 文件写入 | `build-output/tools/FileWriteTool/prompt.ts` |
| Grep 搜索 | `build-output/tools/GrepTool/prompt.ts` |
| Glob 匹配 | `build-output/tools/GlobTool/prompt.ts` |
| Agent 子代理 | `build-output/tools/AgentTool/prompt.ts` |
| Web 搜索 | `build-output/tools/WebSearchTool/prompt.ts` |
| Web 抓取 | `build-output/tools/WebFetchTool/prompt.ts` |
| 计划模式 | `build-output/tools/EnterPlanModeTool/prompt.ts` |

#### 技能模板

`build-output/skills/` 下的每个文件定义了一个技能的完整 prompt：

| 技能 | 文件 | 用途 |
|------|------|------|
| claude-api | `claudeApi.ts` | 构建 Claude API 应用的指导 |
| debug | `debug.ts` | 调试辅助 |
| simplify | `simplify.ts` | 代码简化审查 |
| loop | `loop.ts` | 循环执行任务 |
| batch | `batch.ts` | 批量处理 |
| remember | `remember.ts` | 记忆管理 |
| verify | `verify.ts` | 代码验证 |
| scheduleRemoteAgents | `scheduleRemoteAgents.ts` | 远程定时任务 |

### 2. Agent 架构设计

| 关注点 | 关键文件 |
|--------|----------|
| 工具注册与调度 | `build-output/tools/` 下各工具目录 |
| 权限模型 | `build-output/utils/permissions/` |
| 会话与上下文管理 | `build-output/services/` |
| 模型选择与 Provider 抽象 | `build-output/utils/model/` |
| API 客户端工厂 | `build-output/services/api/client.ts` |
| 流式响应处理 | `build-output/services/api/claude.ts` |

### 3. CLI 产品设计

| 关注点 | 关键文件 |
|--------|----------|
| 终端 UI 组件 | `build-output/components/` |
| 命令系统 | `build-output/commands/` |
| 隐私与遥测控制 | `build-output/utils/privacyLevel.ts` |
| OAuth 认证流程 | `build-output/constants/oauth.ts` |
| 后台维护任务 | `build-output/utils/backgroundHousekeeping.ts` |
