# 架构学习笔记

学习 Claude Code 源码时整理的架构要点，供理解 AI Agent 产品的工程实现参考。

---

## 1. 隐私与遥测机制

Claude Code 提供了分层的隐私控制，这是 CLI 产品设计的一个值得学习的模式。

### 隐私级别

```bash
# 最高级别：禁用所有非必要网络流量
export CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1
```

效果（见 `build-output/utils/privacyLevel.ts`）：
- 禁用所有遥测（Datadog、第一方事件上报）
- 禁用自动更新
- 禁用 GrowthBook 功能开关拉取
- 禁用 release notes 获取
- 禁用模型能力远程拉取
- 隐私级别设为 `essential-traffic`

### 遥测开关的判断逻辑

`build-output/services/analytics/config.ts`：

```typescript
export function isAnalyticsDisabled(): boolean {
  return (
    process.env.NODE_ENV === 'test' ||
    isEnvTruthy(process.env.CLAUDE_CODE_USE_BEDROCK) ||
    isEnvTruthy(process.env.CLAUDE_CODE_USE_VERTEX) ||
    isEnvTruthy(process.env.CLAUDE_CODE_USE_FOUNDRY) ||
    isTelemetryDisabled()
  )
}
```

设计要点：使用 Bedrock/Vertex/Foundry 等企业 provider 时遥测自动禁用，体现了对企业用户隐私需求的尊重。

---

## 2. Provider 抽象层

Claude Code 支持多个 API Provider，抽象设计值得参考。

### Provider 体系

| Provider | 触发方式 | 说明 |
|----------|----------|------|
| Anthropic (直连) | 默认 | 直连 api.anthropic.com |
| AWS Bedrock | `CLAUDE_CODE_USE_BEDROCK=1` | 走 AWS 认证体系 |
| Google Vertex | `CLAUDE_CODE_USE_VERTEX=1` | 走 GCP 认证体系 |
| Azure Foundry | `CLAUDE_CODE_USE_FOUNDRY=1` | 走 Azure 认证体系 |

### 关键文件

| 文件 | 职责 |
|------|------|
| `build-output/services/api/client.ts` | API 客户端工厂，根据 provider 创建不同客户端 |
| `build-output/utils/model/providers.ts` | Provider 选择逻辑（env var 驱动） |
| `build-output/utils/model/configs.ts` | 模型 ID 映射表（不同 provider 下同一模型的 ID 不同） |
| `build-output/utils/model/model.ts` | 模型选择优先级逻辑 |

### 设计模式

- 环境变量驱动 Provider 选择，零代码切换
- `isFirstPartyAnthropicBaseUrl()` 区分直连 vs 代理，影响部分 UX 行为
- 客户端工厂模式统一接口，上层代码无需感知具体 Provider

---

## 3. OAuth 认证流程

### 端点一览

所有 OAuth URL 集中在 `build-output/constants/oauth.ts`：

| 配置项 | 默认值 |
|--------|--------|
| `BASE_API_URL` | `https://api.anthropic.com` |
| `CONSOLE_AUTHORIZE_URL` | `https://platform.claude.com/oauth/authorize` |
| `CLAUDE_AI_AUTHORIZE_URL` | `https://claude.com/cai/oauth/authorize` |
| `TOKEN_URL` | `https://platform.claude.com/v1/oauth/token` |
| `CLIENT_ID` | `9d1c250a-e61b-44d9-88ed-5944d1962f5e` |

设计要点：
- 支持 `ANTHROPIC_API_KEY` 环境变量直接跳过 OAuth
- 支持 `CLAUDE_CODE_CUSTOM_OAUTH_URL` 自定义认证端点
- 认证状态与 API 调用解耦

---

## 4. 后台维护任务

`build-output/utils/backgroundHousekeeping.ts` 在启动时执行的后台任务：

| 任务 | 说明 |
|------|------|
| `initMagicDocs()` | 文档索引 |
| `initSkillImprovement()` | 技能改进 |
| `initExtractMemories()` | 记忆提取 |
| `initAutoDream()` | 后台预处理 |
| `autoUpdateMarketplacesAndPlugins` | 插件自动更新 |
| `ensureDeepLinkProtocolRegistered` | `claude-cli://` URL scheme 注册 |
| `cleanupOldVersions` | 清理旧版本 |
| `cleanupNpmCacheForAnthropicPackages` | 清理 npm 缓存 |

设计要点：所有后台任务非阻塞启动，feature flag 控制开关，隐私级别可一键禁用。

---

## 5. 工具系统设计

每个工具遵循统一的目录结构：

```
tools/
├── BashTool/
│   ├── index.ts       # 工具注册、参数定义、执行逻辑
│   └── prompt.ts      # 工具行为描述（给模型看的 prompt）
├── FileReadTool/
│   ├── index.ts
│   └── prompt.ts
└── ...
```

设计要点：
- 工具定义 = 参数 schema + 执行函数 + prompt 描述，三者同目录管理
- prompt 是工具能力的"说明书"，决定模型何时、如何调用该工具
- 工具可通过配置启用/禁用，支持 `allowedTools` 白名单机制

---

## 6. 非 Claude 模型的兼容性边界

以下功能与 Claude 模型强绑定，如果尝试接入其他模型会遇到兼容问题：

| 功能 | 依赖 | 说明 |
|------|------|------|
| Extended Thinking | Anthropic beta API | Claude 独有能力 |
| Tool Use | Anthropic tool format | 需要 API 格式兼容 |
| Tool Search | Anthropic beta API | Claude 独有能力 |
| Streaming | SSE 格式 | 需确认格式兼容 |
| Context Window | 硬编码在 `configs.ts` | 与模型 ID 绑定 |
| Cost 计算 | 按 Claude 定价 | 与模型 ID 绑定 |
| 模型能力检测 | 按模型 ID 判断 | `utils/model/` 下的能力映射 |

这体现了产品与自家模型深度集成的设计选择 —— 牺牲通用性换取更好的体验。
