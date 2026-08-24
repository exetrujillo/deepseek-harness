# Agent Note: Retry OpenRouter in-flight-budget errors like rate limits

Status: implemented

English | [中文](2026-08-24-openrouter-in-flight-budget-retry.zh.md)

## Problem

pi-ai flattens every provider failure to a bare `error.message` string before `dsh-llm-pi-ai`'s stream translation sees it (the existing `XXX(pi-ai upstream)` note in `stream.ts`), so `classifyPiAiError` can only pattern-match text, never status codes or headers. OpenRouter's per-account concurrent-spend cap returns HTTP 402 with a stable `in_flight_budget_exhausted` reason and a `Retry-After` header once too many reserved-but-unsettled requests are open, independent of account balance and the failing request's own `max_tokens`. `classifyPiAiError` had no case for this message, so it fell to the generic `PI_AI_ERROR` code, which `dsh-llm-retry`'s default `retryableCodes` excludes — every occurrence failed the turn immediately instead of waiting out the provider's cooldown.

## Decision

`classifyPiAiError` recognizes the stable `in_flight_budget_exhausted` reason string and the "in-flight requests" prose OpenRouter uses for it, and classifies both as `RATE_LIMIT` — the same code 429 responses use. `RATE_LIMIT` is already in `dsh-llm-retry`'s default `retryableCodes`, so a `normal`-mode retry policy on a pi-ai provider route now backs off and retries this failure automatically instead of surfacing it as a terminal turn error. Because pi-ai does not forward the original response object, the adapter still cannot read the actual `Retry-After` value and falls back to the route's configured local exponential backoff; a deployment hitting this often should raise `retryPolicy.backoff.maxDelayMs` and `maxRetries` on the affected provider route so the local backoff window is likely to clear the provider's real cooldown.

## Alternatives considered

**Switch the provider to `retryPolicy.mode: 'always'`.** Rejected: always mode retries every model-request failure unboundedly, including permanent ones (bad auth, invalid model), masking real misconfiguration instead of only absorbing this one transient condition.

**Parse `Retry-After` from the failed response.** Rejected for now: pi-ai discards the original `Error`/response before it reaches this adapter (the existing upstream note), so there is no header to read without an upstream pi-ai change.

**Leave it classified as `PI_AI_ERROR` and rely on per-deployment `retryableCodes` overrides.** Rejected: the failure is a stable, provider-documented transient condition, not deployment-specific — every pi-ai-routed OpenRouter user hits the same gap.

## Consequences

A pi-ai provider route with a `normal`-mode retry policy now retries `in_flight_budget_exhausted` failures using local backoff instead of failing the turn outright. Deployments that see this often should widen `maxDelayMs`/`maxRetries` to approximate OpenRouter's actual cooldown, since the local backoff is not driven by the provider's real `Retry-After` value. The classification is text-pattern-based like every other case in this function, so a wording change on OpenRouter's side could silently stop matching until pi-ai forwards structured error data.

## Verification

`convert.spec.ts` covers the full OpenRouter 402 in-flight-budget body classifying as `RATE_LIMIT` alongside the existing 429, `insufficient_quota`, 500, timeout, and transport cases in the same table.
