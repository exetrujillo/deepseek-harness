# Agent Note: 把 OpenRouter 的在途预算错误当限流错误重试

Status: implemented

[English](2026-08-24-openrouter-in-flight-budget-retry.md) | 中文

## 问题

在 `dsh-llm-pi-ai` 的流转换代码看到失败之前，pi-ai 已经把每一种提供方失败都压平成一个裸的 `error.message` 字符串（`stream.ts` 中现有的 `XXX(pi-ai upstream)` 注记正说明这一点），因此 `classifyPiAiError` 只能对文本做模式匹配，永远拿不到状态码或响应头。OpenRouter 按账户设置了并发花费上限：一旦已预留但尚未结算的请求过多，它就会返回 HTTP 402，并带有稳定的 `in_flight_budget_exhausted` 原因和 `Retry-After` 响应头，这与账户余额、以及失败请求自身的 `max_tokens` 均无关。`classifyPiAiError` 原本没有匹配这条消息的分支，因此它会落入通用的 `PI_AI_ERROR` 代码；而 `dsh-llm-retry` 默认的 `retryableCodes` 并不包含该代码，因此每一次出现都会让轮次立即失败，而不是等待提供方的冷却期过去。

## 决策

`classifyPiAiError` 现在能识别稳定的 `in_flight_budget_exhausted` 原因字符串，以及 OpenRouter 为同一错误使用的 "in-flight requests" 提示文案，并把两者都归类为 `RATE_LIMIT`，即 429 响应使用的同一个代码。`RATE_LIMIT` 本就在 `dsh-llm-retry` 默认的 `retryableCodes` 中，因此 pi-ai 提供方路由上 `normal` 模式的重试策略现在会自动退避并重试这个失败，而不是把它当作终止轮次的错误直接暴露出来。由于 pi-ai 不会转发原始的响应对象，适配器仍然读不到真正的 `Retry-After` 值，只能回退到该路由配置的本地指数退避；如果某个部署经常遇到这个问题，应该调高受影响提供方路由上的 `retryPolicy.backoff.maxDelayMs` 和 `maxRetries`，让本地退避窗口更有机会覆盖提供方真正的冷却时间。

## 考虑过的替代方案

**把提供方切换为 `retryPolicy.mode: 'always'`。** 不采用：always 模式会无限重试每一种模型请求失败，包括永久性失败（错误的身份验证、无效模型），这会掩盖真正的错误配置，而不是只吸收这一种瞬时状况。

**从失败响应中解析 `Retry-After`。** 目前不采用：pi-ai 在到达这个适配器之前就丢弃了原始的 `Error`／响应（即现有的上游注记所述），因此在 pi-ai 上游做出改动之前，没有响应头可读。

**继续归类为 `PI_AI_ERROR`，依赖各部署自行覆盖 `retryableCodes`。** 不采用：这个失败是一种稳定的、由提供方文档记录的瞬时状况，并非某个部署特有的问题，每一个通过 pi-ai 路由 OpenRouter 的用户都会遇到同样的缺口。

## 后果

配置了 `normal` 模式重试策略的 pi-ai 提供方路由，现在会用本地退避重试 `in_flight_budget_exhausted` 失败，而不是直接让轮次失败。经常遇到这个问题的部署应该调宽 `maxDelayMs`／`maxRetries`，以逼近 OpenRouter 实际的冷却时间，因为本地退避并不是由提供方真正的 `Retry-After` 值驱动的。这个分类和函数里的其他分支一样，都基于文本模式匹配，因此如果 OpenRouter 一侧改了错误文案，匹配可能会悄悄失效，直到 pi-ai 转发结构化的错误数据为止。

## 验证

`convert.spec.ts` 在同一张表中覆盖了完整的 OpenRouter 402 在途预算响应体归类为 `RATE_LIMIT` 的用例，与既有的 429、`insufficient_quota`、500、超时和传输类用例并列。
