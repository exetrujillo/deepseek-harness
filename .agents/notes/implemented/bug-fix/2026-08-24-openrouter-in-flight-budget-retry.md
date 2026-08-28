# Agent Note: Retry OpenRouter in-flight-budget errors like rate limits

Status: implemented

English | [中文](2026-08-24-openrouter-in-flight-budget-retry.zh.md)

## Problem

pi-ai flattens every provider failure to a bare `error.message` string before `dsh-llm-pi-ai`'s stream translation sees it (the existing `XXX(pi-ai upstream)` note in `stream.ts`), so `classifyPiAiError` can only pattern-match text, never status codes or headers. OpenRouter's per-account concurrent-spend cap returns HTTP 402 with a stable `in_flight_budget_exhausted` reason and a `Retry-After` header once too many reserved-but-unsettled requests are open, independent of account balance and the failing request's own `max_tokens`. `classifyPiAiError` had no case for this message, so it fell to the generic `PI_AI_ERROR` code, which `dsh-llm-retry`'s default `retryableCodes` excludes — every occurrence failed the turn immediately instead of waiting out the provider's cooldown.

## Decision

`classifyPiAiError` recognizes the stable `in_flight_budget_exhausted` reason string and the "in-flight requests" prose OpenRouter uses for it, and classifies both as `RATE_LIMIT` — the same code 429 responses use. This check runs before the quota-exhaustion patterns (`isQuotaExceededError`), so a sibling phrasing that also matches a balance-exhaustion pattern (e.g. "credits exhausted by in-flight requests") still reads as the concurrency cap, not a terminal quota error: `in_flight_budget_exhausted` is a per-account concurrent-spend cap, independent of balance, and clears on its own once earlier requests settle, while `QUOTA_EXCEEDED_CODE` is terminal in the retry loop. `RATE_LIMIT` is already in `dsh-llm-retry`'s default `retryableCodes`, so a `normal`-mode retry policy on a pi-ai provider route backs off and retries this failure automatically instead of surfacing it as a terminal turn error.

pi-ai does not forward the original response object or its headers (the existing upstream note), but OpenRouter's 402 body embeds the response metadata, including `Retry-After`, as JSON text inside the flattened `error.message` before pi-ai discards the rest. `mapStopReason`'s `error` case extracts a `"Retry-After":"<seconds>"` match from that text and populates `LlmFailure.providerRetryAfterMs`, which `dsh-llm-retry` already honors verbatim (bounded by `retryPolicy.maxDelayMs`) ahead of its local exponential backoff. When the pattern is absent — including for other `RATE_LIMIT` sources like a bare 429 — the route falls back to local backoff as before.

## Alternatives considered

**Switch the provider to `retryPolicy.mode: 'always'`.** Rejected: always mode retries every model-request failure unboundedly, including permanent ones (bad auth, invalid model), masking real misconfiguration instead of only absorbing this one transient condition.

**Leave it classified as `PI_AI_ERROR` and rely on per-deployment `retryableCodes` overrides.** Rejected: the failure is a stable, provider-documented transient condition, not deployment-specific — every pi-ai-routed OpenRouter user hits the same gap.

**Match a bare `\b402\b` instead of the `in_flight` marker.** Rejected: HTTP 402 is multi-purpose and includes terminal balance-denial variants from other providers; only the `in_flight_budget_exhausted` reason (or its prose) reliably discriminates the transient concurrency-cap reading from a terminal one.

## Consequences

A pi-ai provider route with a `normal`-mode retry policy now retries `in_flight_budget_exhausted` failures, waiting the provider's exact `Retry-After` cooldown when the flattened text carries one, or local backoff otherwise. The classification and the `Retry-After` extraction are both text-pattern-based, like every other case in this function, so a wording or JSON-shape change on OpenRouter's side could silently stop matching until pi-ai forwards structured error data instead of a flattened string.

## Verification

`convert.spec.ts` covers: the full OpenRouter 402 in-flight-budget body classifying as `RATE_LIMIT` alongside the existing 429, `insufficient_quota`, 500, timeout, and transport cases in the same table; a sibling phrasing that also matches a quota pattern still resolving to `RATE_LIMIT` (ordering); and `providerRetryAfterMs` extraction from an embedded `Retry-After` value, plus its absence when the text carries none.
