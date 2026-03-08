# AI Usage Proxy — API Specification

A cached proxy for AI provider usage data. One proxy, multiple providers. External clients (dashboards, CLI tools, extensions) read usage data from a single service without holding provider credentials or hitting upstream APIs directly.

Multiple clients polling provider APIs independently will quickly trigger rate limits. This proxy consolidates all usage requests through cached endpoints — one upstream fetch per provider serves every consumer. Implementations should cache responses and gracefully degrade when upstream endpoints are unavailable.

## Endpoints

```
GET /api/proxy/{provider}/{source}/
```

### Anthropic

Anthropic has two distinct user types with separate upstream data sources:

- **Subscription users** (Claude Max, Claude Pro) authenticate via OAuth. Their usage data — rolling windows, Opus-specific limits, extra credits — comes from an undocumented beta endpoint. This is the `/subscription/` source.
- **API key users** authenticate with API keys. Their usage data comes from Anthropic's official `/v1/usage` endpoint. This is the `/api-key/` source.

These are separate endpoints because the upstream data sources, authentication methods, and response shapes differ.

| Endpoint | Status | Description |
|---|---|---|
| `GET /api/proxy/anthropic/subscription/` | Implemented | Usage data for subscription users (Claude Max, etc.) via the upstream OAuth-authenticated beta endpoint |
| `GET /api/proxy/anthropic/api-key/` | Planned | Usage data for API key users via Anthropic's official `/v1/usage` endpoint |

### Google (planned)

| Endpoint | Status | Description |
|---|---|---|
| `GET /api/proxy/google/api-key/` | Planned | Gemini API usage data |

### OpenAI (planned)

| Endpoint | Status | Description |
|---|---|---|
| `GET /api/proxy/openai/api-key/` | Planned | OpenAI API usage data |
| `GET /api/proxy/openai/subscription/` | Planned | ChatGPT Plus/Pro usage data |

Planned endpoints return `501 Not Implemented`. Response shapes for future providers will be defined when those endpoints are specified.

---

**Authentication:** None. Endpoints are intentionally unauthenticated — external clients shouldn't need credentials for the host application.

**Base URL:** Determined by the host running the proxy.

## Caching Behaviour

Implementations should cache upstream responses to minimise upstream API calls. The recommended TTLs below ensure consumers get reasonably fresh data while protecting against rate limits.

| Scenario | Recommended TTL | Notes |
|---|---|---|
| Successful fetch | 15 minutes | Fresh data from upstream provider |
| Any API error (429, 401, 5xx, timeout) | 30 minutes | Serves last-known-good data with `meta.rate_limited: true` |
| Last-good fallback | 1 hour | Kept separately so stale data survives cache expiry |

## Response — Anthropic Subscription

The response shape below applies to `GET /api/proxy/anthropic/subscription/`. Other providers will define their own response shapes when specified. All responses share the `meta` envelope.

### Success (200)

```json
{
  "five_hour": {
    "utilization": 28.0,
    "resets_at": "2026-03-08T03:00:00.415663+00:00"
  },
  "seven_day": {
    "utilization": 30.0,
    "resets_at": "2026-03-13T03:00:00.415677+00:00"
  },
  "seven_day_opus": {
    "utilization": 0.0,
    "resets_at": "2026-03-13T03:00:00.415677+00:00"
  },
  "extra_usage": {
    "is_enabled": true,
    "utilization": 0.0,
    "used_credits": 0.0,
    "monthly_limit": 7750
  },
  "meta": {
    "source": "anthropic_subscription",
    "rate_limited": false,
    "last_updated": "2026-03-08T00:33:26Z"
  }
}
```

### Rate-limited / stale (200 with flag)

Same shape as above, but `meta.rate_limited` is `true` and `meta.last_updated` shows when data was last successfully fetched (could be minutes or hours ago).

### No credentials configured (503)

```json
{
  "type": "about:blank",
  "title": "Service Unavailable",
  "status": 503,
  "detail": "No Anthropic credentials configured"
}
```

### Upstream failure, no cached data (502)

```json
{
  "type": "about:blank",
  "title": "Bad Gateway",
  "status": 502,
  "detail": "Anthropic API returned 429 and no cached data is available"
}
```

Error responses follow [RFC 9457 Problem Details](https://www.rfc-editor.org/rfc/rfc9457).

## Field Reference

### Usage windows

| Field | Type | Description |
|---|---|---|
| `five_hour` | object | 5-hour rolling usage window |
| `five_hour.utilization` | number | Usage as percentage (0–100) |
| `five_hour.resets_at` | string | ISO-8601 UTC timestamp when the window resets |
| `seven_day` | object | 7-day rolling usage window |
| `seven_day.utilization` | number | Usage as percentage (0–100) |
| `seven_day.resets_at` | string | ISO-8601 UTC timestamp when the window resets |
| `seven_day_opus` | object \| null | Opus-specific 7-day window (same shape as `five_hour`). Null if not applicable. |

### Extra usage

| Field | Type | Description |
|---|---|---|
| `extra_usage` | object \| null | Extra/overuse credits info. Null if not enabled. |
| `extra_usage.is_enabled` | boolean | Whether extra usage billing is active |
| `extra_usage.utilization` | number \| null | Extra usage percentage |
| `extra_usage.used_credits` | number | Dollar amount of credits consumed this month |
| `extra_usage.monthly_limit` | number | Dollar cap for extra usage this month |

### Metadata

| Field | Type | Description |
|---|---|---|
| `meta.source` | string | Data source identifier: `"{provider}_{source}"` (e.g., `"anthropic_subscription"`, `"anthropic_api_key"`, `"google_api_key"`) |
| `meta.rate_limited` | boolean | `true` when serving stale cached data (upstream was unreachable or rate-limited) |
| `meta.last_updated` | string | ISO-8601 UTC timestamp of the last successful upstream fetch |

## Example

```bash
curl https://{your-host}/api/proxy/anthropic/subscription/ | jq .
```

## License

MIT — see [LICENSE](LICENSE).
