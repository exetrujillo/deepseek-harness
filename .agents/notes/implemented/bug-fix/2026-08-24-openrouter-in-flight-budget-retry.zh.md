# Agent Note: 把 OpenRouter 的在途预算错误当限流错误重试

Status: implemented

[English](2026-08-24-openrouter-in-flight-budget-retry.md) | 中文

## 问题

在 `dsh-llm-pi-ai` 的流转换代码看到失败之前，pi-ai 已经把每一种提供方失败都压平成一个裸的 `error.message` 字符串（`stream.ts` 中现有的 `XXX(pi-ai upstream)` 注记正说明这一点），因此 `classifyPiAiError` 只能对文本做模式匹配，永远拿不到状态码或响应头。OpenRouter 按账户设置了并发花费上限：一旦已预留但尚未结算的请求过多，它就会返回 HTTP 402，并带有稳定的 `in_flight_budget_exhausted` 原因和 `Retry-After` 响应头，这与账户余额、以及失败请求自身的 `max_tokens` 均无关。`classifyPiAiError` 原本没有匹配这条消息的分支，因此它会落入通用的 `PI_AI_ERROR` 代码；而 `dsh-llm-retry` 默认的 `retryableCodes` 并不包含该代码，因此每一次出现都会让轮次立即失败，而不是等待提供方的冷却期过去。

## 决策

`classifyPiAiError` 现在能识别稳定的 `in_flight_budget_exhausted` 原因字符串，以及 OpenRouter 为同一错误使用的 "in-flight requests" 提示文案，并把两者都归类为 `RATE_LIMIT`，即 429 响应使用的同一个代码。这一判断先于余额耗尽的模式（`isQuotaExceededError`）执行，因此即使某个相邻文案同时匹配了余额耗尽的模式（例如 "credits exhausted by in-flight requests"），也仍然会被读作并发上限，而不是终止性的余额错误：`in_flight_budget_exhausted` 是按账户设置的并发花费上限，与余额无关，一旦先前的请求结算就会自行解除；而 `QUOTA_EXCEEDED_CODE` 在重试循环里是终止性的。`RATE_LIMIT` 本就在 `dsh-llm-retry` 默认的 `retryableCodes` 中，因此 pi-ai 提供方路由上 `normal` 模式的重试策略会自动退避并重试这个失败，而不是把它当作终止轮次的错误直接暴露出来。

pi-ai 不会转发原始的响应对象或响应头（即现有的上游注记所述），但 OpenRouter 的 402 响应体会在 pi-ai 丢弃其余部分之前，把包括 `Retry-After` 在内的响应元数据以 JSON 文本的形式嵌入压平后的 `error.message` 中。`mapStopReason` 的 `error` 分支会从这段文本里匹配 `"Retry-After":"<秒数>"`，并填充 `LlmFailure.providerRetryAfterMs`；`dsh-llm-retry` 已经会原样采信这个值（受 `retryPolicy.maxDelayMs` 上限约束），优先于它自己的本地指数退避。当文本中不含这个模式时——包括来自裸 429 等其他 `RATE_LIMIT` 来源——该路由和之前一样回退到本地退避。

## 考虑过的替代方案

**把提供方切换为 `retryPolicy.mode: 'always'`。** 不采用：always 模式会无限重试每一种模型请求失败，包括永久性失败（错误的身份验证、无效模型），这会掩盖真正的错误配置，而不是只吸收这一种瞬时状况。

**继续归类为 `PI_AI_ERROR`，依赖各部署自行覆盖 `retryableCodes`。** 不采用：这个失败是一种稳定的、由提供方文档记录的瞬时状况，并非某个部署特有的问题，每一个通过 pi-ai 路由 OpenRouter 的用户都会遇到同样的缺口。

**直接匹配裸的 `\b402\b`，而不是 `in_flight` 标记。** 不采用：HTTP 402 是多用途状态码，也包括来自其他提供方的终止性余额拒绝场景；只有 `in_flight_budget_exhausted` 原因（或其提示文案）才能可靠地把瞬时的并发上限读法和终止性读法区分开。

## 后果

配置了 `normal` 模式重试策略的 pi-ai 提供方路由，现在会重试 `in_flight_budget_exhausted` 失败：当压平后的文本携带 `Retry-After` 时按提供方给出的确切冷却时间等待，否则回退到本地退避。这个分类和 `Retry-After` 的提取，都和函数里的其他分支一样基于文本模式匹配，因此如果 OpenRouter 一侧改了错误文案或 JSON 结构，匹配可能会悄悄失效，直到 pi-ai 转发结构化的错误数据为止。

## 验证

`convert.spec.ts` 覆盖了以下用例：完整的 OpenRouter 402 在途预算响应体归类为 `RATE_LIMIT`，与既有的 429、`insufficient_quota`、500、超时和传输类用例并列在同一张表中；一个同时匹配余额耗尽模式的相邻文案仍然解析为 `RATE_LIMIT`（验证顺序）；以及从内嵌 `Retry-After` 值中提取 `providerRetryAfterMs`，并在文本不含该值时确认其缺失。
