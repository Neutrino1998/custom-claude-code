# Custom Claude Code

Claude Code (v2.1.87) 的构建中间产物分析与私有化改造研究项目。

## 这是什么

这份代码是 Claude Code **Bun 转译后、bundle 前的中间产物**，不是原始 TypeScript 源码。

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

| | 原始源码 | 这份 src（转译产物） | npm 包 |
|---|---|---|---|
| 类型注解 | 完整 | 被擦除 | 被擦除 |
| 模块结构 | 1,900+ 独立文件 | 1,900 独立文件 | 单个 cli.js |
| 第三方依赖 | import 引用 | import 引用 | 已内联 |
| source map | 无 | inline base64（含原始源码） | 无 |
| package.json | 完整依赖声明 | 无 | 仅元信息 |
| tsconfig.json | 有 | 无 | 无 |
| 测试文件 | 有 | 无 | 无 |
| 可直接开发 | 是 | 困难 | 不可能 |
| 可直接运行 | 需构建 | 需装依赖 | 可以 |

### 能做什么 / 不能做什么

- **能**：阅读完整代码逻辑、研究架构设计、分析提示词工程
- **能但痛苦**：直接修改代码跑起来（没有类型检查，改错了只有运行时炸）
- **不能**：当作正经 TS 项目做二次开发（缺类型系统和项目基础设施）

## 目录结构

```
custom-claude-code/
├── build-output/                  # Claude Code 转译产物
│   ├── main.tsx          # 主入口（4,683 行）
│   ├── services/         # API 层、MCP、分析等服务
│   │   └── api/          # Anthropic API 客户端（改造重点）
│   ├── tools/            # 内置工具（Bash, Read, Edit, Grep 等）
│   ├── commands/         # CLI 命令
│   ├── components/       # Ink (React terminal UI) 组件
│   ├── constants/        # 系统提示词、OAuth 配置等常量
│   ├── utils/            # 工具函数
│   │   └── model/        # 模型配置与选择逻辑
│   ├── skills/           # 技能系统
│   └── hooks/            # React hooks
├── docs/
│   ├── security-audit.md       # 安全审计报告
│   └── privatization-guide.md  # 私有化部署指南
├── .gitignore
└── README.md
```

## 技术栈

- **运行时**: Bun
- **语言**: TypeScript（转译为 JS，后缀未改）
- **UI**: Ink (React for CLI)
- **API SDK**: @anthropic-ai/sdk

## 提示词与工具定义

这份代码最有研究价值的部分 —— 所有提示词和工具定义都是明文：

### 系统提示词

| 文件 | 内容 |
|------|------|
| `build-output/constants/prompts.ts` | 主系统提示词 |
| `build-output/constants/systemPromptSections.ts` | 系统提示词各段落拼装逻辑 |
| `build-output/constants/common.ts` | 通用提示词片段 |

### 工具 Prompt（每个工具的行为指令）

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

### 技能模板

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

## 接入私有模型的关键文件

| 文件 | 说明 |
|------|------|
| `build-output/services/api/client.ts` | API 客户端工厂，支持多 provider 切换 |
| `build-output/services/api/claude.ts` | 核心请求逻辑（流式/非流式） |
| `build-output/utils/model/providers.ts` | Provider 选择（env var 驱动） |
| `build-output/utils/model/configs.ts` | 模型 ID 映射表 |
| `build-output/utils/model/model.ts` | 模型选择优先级逻辑 |

## 改造思路

### 方案 A：新增 Custom Provider

在 `providers.ts` 中新增 `custom` provider，通过环境变量激活：

```bash
export CLAUDE_CODE_USE_CUSTOM=1
export CUSTOM_API_BASE_URL=http://your-private-endpoint
export CUSTOM_API_KEY=your-key
```

在 `client.ts` 中为 `custom` provider 创建指向私有 API 的客户端。

### 方案 B：API 代理（最小改动）

不改源码，在前面架一层 API 代理（如 LiteLLM），将 Anthropic API 格式转译为私有模型格式：

```bash
export ANTHROPIC_BASE_URL=http://localhost:4000  # LiteLLM proxy
export ANTHROPIC_API_KEY=any-key
```

### 注意事项

- 非 Claude 模型可能不支持 extended thinking、tool_search 等 beta 特性
- 模型能力检测、context window、cost 计算等逻辑与 Claude 模型 ID 绑定，需适配
- 建议先用方案 B 验证可行性，再决定是否深度改造

## TODO

- [ ] 确认私有模型的 API 兼容性（tool use / streaming）
- [ ] 选定改造方案（A or B）
- [ ] 搭建本地开发环境（需要 Bun）
- [ ] 验证基本对话功能
- [ ] 适配 tool use 调用
