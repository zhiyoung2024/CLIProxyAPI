# Antigravity Gemini 4 Argon & 3.8 Flash Tiered Thinking Routing Guide

This document provides upstream maintainers and users with deep technical insight into integrating **Gemini 4 Argon** and **Gemini 3.8 Flash** via the Antigravity OAuth channel, resolving thinking-content stripping, and ensuring zero-disconnect production stability.

---

## 1. Upstream Behaviors & Empirical Findings

During high-concurrency production testing of Google's internal API (`v1internal`) through the Antigravity channel, several critical behaviors were observed:

### Issue A: Upstream Physical Endpoint Thought Stripping
When invoking physical models with low/medium naming conventions directly:
- `gemini-4-argon-low`
- `gemini-4-argon-medium`
- `gemini-3.8-flash-low`

The upstream Google backend silently strips the `thought` (reasoning content) block from the SSE stream, even when the request explicitly specifies:
```json
{
  "generationConfig": {
    "thinkingConfig": {
      "includeThoughts": true,
      "thinkingBudget": 16384
    }
  }
}
```
However, when sending the exact same payload to the full `-high` physical endpoint (e.g., `gemini-4-argon-high` or `gemini-3.8-flash-high`), the backend streams thoughts normally.

### Issue B: Gemini 4 Argon Thinking Budget Threshold
The `gemini-4-argon-trial` engine requires an explicit threshold:
- If `thinkingBudget < 16384`, the internal engine treats the request as a non-thinking or suppressed request and returns empty thoughts.
- Setting `thinkingBudget >= 16384` triggers full internal chain-of-thought emission.

---

## 2. Solution: Model Exclusion + Forked Alias Rerouting

To expose standard `-low`, `-medium`, and bare model names to API clients while guaranteeing 100% thought emission, we combine:
1. `oauth-excluded-models`: Suppresses the raw stripped physical endpoints.
2. `oauth-model-alias` (`fork: true`): Exposes client-facing models that route internally to `-high`.
3. `payload.default`: Dynamically injects tiered thinking budgets.

### Configuration

```yaml
# 1. Block physical endpoints that strip thoughts
oauth-excluded-models:
  antigravity:
    - gemini-4-argon-low
    - gemini-4-argon-medium
    - gemini-3.8-flash-low

# 2. Reroute client models to -high endpoints
oauth-model-alias:
  antigravity:
    - alias: gemini-4-argon
      fork: true
      name: gemini-4-argon-high
    - alias: gemini-4-argon-low
      fork: true
      name: gemini-4-argon-high
    - alias: gemini-4-argon-medium
      fork: true
      name: gemini-4-argon-high
    - alias: gemini-4-argon-thinking
      fork: true
      name: gemini-4-argon-high
    - alias: gemini-3.8-flash
      fork: true
      name: gemini-3.8-flash-high
    - alias: gemini-3.8-flash-low
      fork: true
      name: gemini-3.8-flash-high
    - alias: gemini-3.8-flash-tiered
      fork: true
      name: gemini-3.8-flash-high
```

---

## 3. First-Write-Wins Rule Ordering for Tiered Budgets

In CLIProxyAPI, `payload.default` uses a **first-write-wins** matching strategy. If a generic or catch-all model rule precedes a specific sub-model rule, the specific rule will be shadowed and ignored.

Therefore, the configuration must order specific tiers before general models:

```yaml
payload:
  default:
    # 1. Low tier (16384 - reaches threshold)
    - models:
        - name: gemini-4-argon-low
          protocol: antigravity
      params:
        generationConfig.thinkingConfig.includeThoughts: true
        generationConfig.thinkingConfig.thinkingBudget: 16384

    # 2. Medium tier (24576)
    - models:
        - name: gemini-4-argon-medium
          protocol: antigravity
      params:
        generationConfig.thinkingConfig.includeThoughts: true
        generationConfig.thinkingConfig.thinkingBudget: 24576

    # 3. High / Default tier (32768)
    - models:
        - name: gemini-4-argon-high
          protocol: antigravity
        - name: gemini-4-argon
          protocol: antigravity
        - name: gemini-4-argon-thinking
          protocol: antigravity
      params:
        generationConfig.thinkingConfig.includeThoughts: true
        generationConfig.thinkingConfig.thinkingBudget: 32768

    # 4. Flash Low (1000)
    - models:
        - name: gemini-3.8-flash-low
          protocol: antigravity
      params:
        generationConfig.thinkingConfig.includeThoughts: true
        generationConfig.thinkingConfig.thinkingBudget: 1000

    # 5. Flash Medium (4000)
    - models:
        - name: gemini-3.8-flash-medium
          protocol: antigravity
      params:
        generationConfig.thinkingConfig.includeThoughts: true
        generationConfig.thinkingConfig.thinkingBudget: 4000

    # 6. Flash High / Default (16000)
    - models:
        - name: gemini-3.8-flash-high
          protocol: antigravity
        - name: gemini-3.8-flash
          protocol: antigravity
        - name: gemini-3.8-flash-tiered
          protocol: antigravity
      params:
        generationConfig.thinkingConfig.includeThoughts: true
        generationConfig.thinkingConfig.thinkingBudget: 16000
```

---

## 4. Reverse Proxy & Keep-Alive Requirements

Deep thinking models can ponder for $30 \sim 120$ seconds before streaming tokens. To prevent TCP connection drops, HTTP 499, and client `Connection error`:

1. **CLIProxyAPI Keep-Alives**:
   ```yaml
   streaming:
     keepalive-seconds: 15
     bootstrap-retries: 2
   nonstream-keepalive-interval: 15
   ```
2. **Nginx Reverse Proxy Directives**:
   ```nginx
   proxy_http_version 1.1;
   proxy_set_header Connection "";
   proxy_buffering off;
   proxy_cache off;
   chunked_transfer_encoding on;
   proxy_read_timeout 600s;
   proxy_send_timeout 600s;
   ```
