# Custom Claude Code

基于 Claude Code 源码的私有化改造项目。

## 目录结构

```
custom-claude-code/
├── src/                # Claude Code 原始源码
│   ├── main.tsx        # 主入口
│   ├── services/       # API 层、MCP、分析等服务
│   │   └── api/        # Anthropic API 客户端（改造重点）
│   ├── tools/          # 内置工具（Bash, Read, Edit, Grep 等）
│   ├── commands/       # CLI 命令
│   ├── components/     # Ink (React terminal UI) 组件
│   ├── utils/          # 工具函数
│   │   └── model/      # 模型配置与选择逻辑
│   ├── skills/         # 技能系统
│   └── hooks/          # React hooks
├── .gitignore
└── README.md
```

## 技术栈

- **运行时**: Bun
- **语言**: TypeScript
- **UI**: Ink (React for CLI)
- **API SDK**: @anthropic-ai/sdk

## 接入私有模型的关键文件

| 文件 | 说明 |
|------|------|
| `src/services/api/client.ts` | API 客户端工厂，支持多 provider 切换 |
| `src/services/api/claude.ts` | 核心请求逻辑（流式/非流式） |
| `src/utils/model/providers.ts` | Provider 选择（env var 驱动） |
| `src/utils/model/configs.ts` | 模型 ID 映射表 |
| `src/utils/model/model.ts` | 模型选择优先级逻辑 |

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
