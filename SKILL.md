---
name: ai-usage-proxy
description: Implement, audit, or consume the AI Usage Proxy API. Use when asked to build a usage proxy endpoint, add an AI usage display, check provider usage data, or integrate with the usage proxy spec.
license: MIT
compatibility: ">=1.0.0"
metadata:
  author: jamesprnich
  version: "1.0"
  description: Works with any agent that can read and write code.
---

# AI Usage Proxy

The AI Usage Proxy is a cached proxy for AI provider usage data. One proxy, multiple providers. Currently supports **Anthropic subscription users** (Claude Max, OAuth). Planned: Anthropic API key, Google Gemini, OpenAI.

The proxy caches and normalises data from upstream provider endpoints, consolidating all usage requests through cached endpoints to avoid rate limits. This skill helps implement, audit, or consume the proxy as defined by the [AI Usage Proxy Specification](https://github.com/jamesprnich/claude-usage-proxy-spec).

> **Subscription vs API key:** Anthropic has two distinct user types. *Subscription* users (Claude Max, Claude Pro) use OAuth and have rolling usage windows (5-hour, 7-day), Opus-specific limits, and extra credits. *API key* users use the official `/v1/usage` endpoint with different data. These are separate proxy endpoints (`/subscription/` vs `/api-key/`) because the upstream sources, auth methods, and response shapes differ. This spec currently defines the subscription endpoint only.

## Actions

Determine which action the user wants. If unclear, ask.

### Implement

Build the proxy endpoint in an existing backend application.

1. Check if a usage proxy endpoint already exists in the project. If it does, switch to **Audit**.
2. Identify the backend framework (Django, FastAPI, Express, etc.).
3. Create the endpoint at `GET /api/proxy/anthropic/subscription/`.
4. The endpoint must:
   - Accept no authentication
   - Fetch usage data from the Anthropic API (the upstream source)
   - Cache successful responses (recommended TTL: 15 minutes)
   - On upstream failure, serve last-known-good cached data with `meta.rate_limited: true`
   - Return error responses following RFC 9457 Problem Details
5. Return the response in the shape defined in [Response Shape](#response-shape) below.

### Consume

Build a client that reads from an existing proxy.

1. Ask the user for the proxy base URL.
2. Identify the target (CLI tool, dashboard widget, VS Code extension, etc.).
3. Implement a client that:
   - Calls `GET {baseUrl}/api/proxy/anthropic/subscription/`
   - Handles all response cases: fresh data, stale data (`meta.rate_limited: true`), 502, 503
   - Displays utilization percentages and reset times appropriately
4. The client should check `meta.rate_limited` and indicate to the user when data is stale.

### Audit

Verify an existing implementation against the spec.

1. Read the proxy endpoint code.
2. Check compliance against the rules below:
   - Endpoint path matches `GET /api/proxy/anthropic/subscription/`
   - Response shape matches [Response Shape](#response-shape)
   - Nullable fields (`seven_day_opus`, `extra_usage`, `extra_usage.utilization`) return `null` when not applicable, not omitted
   - Error responses follow RFC 9457 (`type`, `title`, `status`, `detail`)
   - `meta.rate_limited` is `true` when serving cached data after an upstream failure
   - `meta.last_updated` reflects the actual upstream fetch time, not the cache-serve time
3. Report findings as a checklist. If issues found, offer to fix.

## Response Shape

### Success (200)

| Field | Type | Required | Description |
|---|---|---|---|
| `five_hour` | object | Yes | 5-hour rolling usage window |
| `five_hour.utilization` | number | Yes | Usage as percentage (0–100) |
| `five_hour.resets_at` | string | Yes | ISO-8601 UTC timestamp when the window resets |
| `seven_day` | object | Yes | 7-day rolling usage window |
| `seven_day.utilization` | number | Yes | Usage as percentage (0–100) |
| `seven_day.resets_at` | string | Yes | ISO-8601 UTC timestamp when the window resets |
| `seven_day_opus` | object \| null | Yes | Opus-specific 7-day window (same shape as `five_hour`). Null if not applicable. |
| `extra_usage` | object \| null | Yes | Extra/overuse credits info. Null if not enabled. |
| `extra_usage.is_enabled` | boolean | Yes | Whether extra usage billing is active |
| `extra_usage.utilization` | number \| null | Yes | Extra usage percentage |
| `extra_usage.used_credits` | number | Yes | Dollar amount of credits consumed this month |
| `extra_usage.monthly_limit` | number | Yes | Dollar cap for extra usage this month |
| `meta` | object | Yes | Response metadata |
| `meta.source` | string | Yes | Data source identifier: `"{provider}_{source}"` (e.g., `"anthropic_subscription"`) |
| `meta.rate_limited` | boolean | Yes | `true` when serving stale cached data |
| `meta.last_updated` | string | Yes | ISO-8601 UTC timestamp of the last successful upstream fetch |

### Error responses (502, 503)

Follow RFC 9457 Problem Details:

| Field | Type | Required | Description |
|---|---|---|---|
| `type` | string | Yes | URI identifying the problem type (use `"about:blank"` for standard HTTP errors) |
| `title` | string | Yes | Short, human-readable summary (e.g., "Bad Gateway", "Service Unavailable") |
| `status` | integer | Yes | HTTP status code |
| `detail` | string | No | Human-readable explanation specific to this occurrence |

## Caching Rules

Implementations should cache upstream responses. Recommended TTLs:

| Scenario | TTL | Behaviour |
|---|---|---|
| Successful upstream fetch | 15 minutes | Serve fresh data, `meta.rate_limited: false` |
| Upstream error (429, 401, 5xx, timeout) | 30 minutes | Serve last-known-good data, `meta.rate_limited: true` |
| Last-good fallback | 1 hour | Separate cache entry so stale data survives primary cache expiry |
| No cached data and upstream fails | — | Return 502 error |
| No credentials configured | — | Return 503 error |

## Guidelines

- The endpoint must be unauthenticated — external clients should not need host application credentials.
- Nullable fields must be present in the response as `null`, never omitted.
- `meta.last_updated` must reflect when data was actually fetched from the upstream provider, not when the cache was served.
- Utilization values are percentages (0–100), not decimals (0–1).
- All timestamps must be ISO-8601 UTC.
- Error responses must use RFC 9457 Problem Details, not bare `{ "error": "..." }`.
