# 安全审计报告

审计时间：2026-03-31
审计范围：`src/` 目录全部 1,900 个文件

## 结论：未发现恶意代码

### 扫描维度

| 检查项 | 结果 |
|--------|------|
| 硬编码密钥/凭证 | 未发现（仅有 Datadog 公开 client token 和 GrowthBook 公开 SDK key） |
| 反向 shell / 后门 | 未发现 |
| 数据外泄 | 未发现 |
| 加密货币挖矿 | 未发现 |
| 代码混淆 | 未发现 |
| 键盘记录 | 未发现 |
| 供应链风险 | 未发现 |
| 原型链污染 | 未发现 |
| 异常文件写入 | 未发现 |

### 已知的外部通信端点

所有网络请求均指向合法服务：

| 端点 | 用途 | 文件位置 |
|------|------|----------|
| `api.anthropic.com` | Anthropic API | `constants/oauth.ts`, `services/api/` |
| `platform.claude.com` | OAuth 认证 | `constants/oauth.ts` |
| `claude.ai` | Claude AI 平台 | `constants/product.ts` |
| `http-intake.logs.us5.datadoghq.com` | Datadog 遥测 | `services/analytics/datadog.ts` |
| `1.1.1.1` | 网络连通性检测（仅 HEAD） | `utils/env.ts` |
| `cognitiveservices.azure.com` | Azure 认证（Foundry provider） | `services/api/client.ts` |
| `googleapis.com` | Google Cloud 认证（Vertex provider） | `services/api/client.ts` |

### 嵌入的公开 Token（非 secret）

| Token | 类型 | 文件 |
|-------|------|------|
| `pubbbf48e6d78dae54bceaa4acf463299bf` | Datadog client token（只写权限） | `services/analytics/datadog.ts` |
| `sdk-yZQvlplybuXjYh6L` | GrowthBook SDK key (dev) | `constants/keys.ts` |
| `sdk-xRVcrliHIlrg4og4` | GrowthBook SDK key (prod, ant) | `constants/keys.ts` |
| `sdk-zAZezfDKGoZuXXKe` | GrowthBook SDK key (prod, external) | `constants/keys.ts` |
| `9d1c250a-e61b-44d9-88ed-5944d1962f5e` | OAuth Client ID（公开的） | `constants/oauth.ts` |

这些都是设计上嵌入客户端的公开 token，不构成安全风险。
