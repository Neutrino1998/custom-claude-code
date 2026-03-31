# 私有化部署指南

将 Claude Code 改造为接入私有模型时，需要处理以下几个方面。

---

## 1. 禁用遥测和非必要网络流量

### 最高级别：禁用所有非必要网络流量

```bash
export CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1
```

效果（见 `src/utils/privacyLevel.ts`）：
- 禁用所有遥测（Datadog、第一方事件上报）
- 禁用自动更新
- 禁用 GrowthBook 功能开关拉取
- 禁用 release notes 获取
- 禁用模型能力远程拉取
- 隐私级别设为 `essential-traffic`

### 仅禁用遥测

```bash
export DISABLE_TELEMETRY=1
```

效果：禁用 Datadog 和第一方分析事件，但保留其他网络功能。

### 遥测实现细节

遥测开关的判断逻辑在 `src/services/analytics/config.ts`：

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

注意：使用 Bedrock/Vertex/Foundry provider 时，遥测会**自动禁用**。

---

## 2. 禁用深度链接注册

Claude Code 会在启动时自动注册 `claude-cli://` URL scheme 到操作系统。

### 通过设置禁用

在 `~/.claude/settings.json` 中：

```json
{
  "disableDeepLinkRegistration": "disable"
}
```

### 通过代码禁用

相关文件：`src/utils/deepLink/registerProtocol.ts`

该功能还受 feature flag `tengu_lodestone_enabled` 控制（见 `src/utils/backgroundHousekeeping.ts:39`），
如果 GrowthBook 被禁用（通过上面的 `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC`），feature flag 拉取不到，
此功能也会自动失效。

---

## 3. API 端点改造

### 方案 A：环境变量替换（最小改动）

```bash
# 指向你的 API 代理（需兼容 Anthropic Messages API 格式）
export ANTHROPIC_BASE_URL=http://your-private-api:8080

# API 密钥
export ANTHROPIC_API_KEY=your-private-key
```

注意：`src/utils/model/providers.ts` 中的 `isFirstPartyAnthropicBaseUrl()` 会检查 URL 是否指向
`api.anthropic.com`。当你设置了自定义 URL 后，该函数会返回 `false`，这会影响部分功能（如某些
UI 提示、OAuth 流程等），但核心 API 调用不受影响。

### 方案 B：使用 Bedrock/Vertex Provider

如果你的私有部署走的是 AWS Bedrock 或 Google Vertex：

```bash
# AWS Bedrock
export CLAUDE_CODE_USE_BEDROCK=1
export AWS_REGION=us-east-1
# + 标准 AWS 认证配置

# Google Vertex
export CLAUDE_CODE_USE_VERTEX=1
export ANTHROPIC_VERTEX_PROJECT_ID=your-project
export CLOUD_ML_REGION=us-east5
```

这是官方已支持的路径，遥测也会自动关闭。

### 方案 C：新增 Custom Provider（深度改造）

需要修改的文件：

| 文件 | 改动 |
|------|------|
| `src/utils/model/providers.ts` | 新增 `'custom'` provider 类型和 env var 判断 |
| `src/services/api/client.ts` | 为 `custom` provider 创建 API 客户端实例 |
| `src/utils/model/configs.ts` | 添加各模型在 custom provider 下的 ID 映射 |
| `src/services/analytics/config.ts` | 在 `isAnalyticsDisabled()` 中加入 custom provider 判断 |

---

## 4. OAuth 认证改造

如果不使用 Anthropic 账号体系，需要处理 OAuth 流程。

### 跳过 OAuth，直接用 API Key

```bash
export ANTHROPIC_API_KEY=your-key
```

设置了此环境变量后，Claude Code 会跳过 OAuth 登录流程，直接使用该 key。

### OAuth 端点一览

所有 OAuth URL 集中在 `src/constants/oauth.ts`：

| 配置项 | 默认值 |
|--------|--------|
| `BASE_API_URL` | `https://api.anthropic.com` |
| `CONSOLE_AUTHORIZE_URL` | `https://platform.claude.com/oauth/authorize` |
| `CLAUDE_AI_AUTHORIZE_URL` | `https://claude.com/cai/oauth/authorize` |
| `TOKEN_URL` | `https://platform.claude.com/v1/oauth/token` |
| `CLIENT_ID` | `9d1c250a-e61b-44d9-88ed-5944d1962f5e` |

如果你需要自建 OAuth，可以通过 `CLAUDE_CODE_CUSTOM_OAUTH_URL` 环境变量指向自己的认证服务。

---

## 5. 后台维护任务

`src/utils/backgroundHousekeeping.ts` 在启动时会执行以下后台任务：

| 任务 | 说明 | 私有化处理 |
|------|------|------------|
| `initMagicDocs()` | 文档索引 | 可保留 |
| `initSkillImprovement()` | 技能改进 | 可保留 |
| `initExtractMemories()` | 记忆提取 | 可保留 |
| `initAutoDream()` | 后台预处理 | 可保留 |
| `autoUpdateMarketplacesAndPlugins` | 插件自动更新 | 内网无外部 registry 可访问，建议禁用或指向内部源 |
| `ensureDeepLinkProtocolRegistered` | URL scheme 注册 | 建议禁用（见上文第 2 节） |
| `cleanupOldVersions` | 清理旧版本 | 可保留 |
| `cleanupNpmCacheForAnthropicPackages` | 清理 npm 缓存 | 仅 ant 用户触发，可忽略 |

---

## 6. 推荐的内网部署环境变量模板

```bash
# === 核心 API 配置 ===
export ANTHROPIC_BASE_URL=http://your-private-api:8080
export ANTHROPIC_API_KEY=your-private-key

# === 隐私与安全 ===
# 禁用所有非必要网络流量（遥测、自动更新、feature flags 等）
export CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1

# === 可选：模型覆盖 ===
# export ANTHROPIC_MODEL=your-model-id

# === 可选：如果走 Bedrock/Vertex ===
# export CLAUDE_CODE_USE_BEDROCK=1
# export AWS_REGION=us-east-1
```

加上 settings.json：

```json
{
  "disableDeepLinkRegistration": "disable"
}
```

---

## 7. 非 Claude 模型的兼容性风险

如果你接入的不是 Claude 模型，以下功能可能不兼容：

| 功能 | 依赖 | 风险 |
|------|------|------|
| Extended Thinking | Anthropic beta API | 非 Claude 模型不支持，需禁用 |
| Tool Use | Anthropic tool format | 需要 API 代理做格式转换 |
| Tool Search | Anthropic beta API | 非 Claude 模型不支持 |
| Streaming | SSE 格式 | 需确认私有模型的 streaming 格式兼容 |
| Context Window | 硬编码在 `configs.ts` | 需按实际模型调整 |
| Cost 计算 | 按 Claude 定价 | 需调整或禁用 |
| 模型能力检测 | 按模型 ID 判断 | 需要修改 `utils/model/` 下的能力映射 |
