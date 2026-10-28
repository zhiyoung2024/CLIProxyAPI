# PR Description: feat(antigravity): add Gemini 4 Argon & 3.8 Flash tiered models, thinking routing, and Cloudflare 524 stability

## Summary

This pull request completes model registry definitions for **Gemini 4 Argon** and **Gemini 3.8 Flash** under the Antigravity OAuth provider, and provides configuration best practices to resolve upstream `thought` stripping, Cloudflare Error 524 timeouts, and long-reasoning SSE drops.

### What's Changed

1. **Model Registry (`internal/registry/models/models.json`)**:
   - Added `gemini-4-argon`, `gemini-4-argon-low`, `gemini-4-argon-medium`, `gemini-4-argon-thinking`
   - Added `gemini-3.8-flash`, `gemini-3.8-flash-low`, `gemini-3.8-flash-medium`, `gemini-3.8-flash-tiered`
   - Added Claude model tiers: `claude-opus-4-6`, `claude-sonnet-4-6-thinking`, `claude-opus-5-5-medium`, `claude-sonnet-5-5-medium`
2. **Configuration Examples (`config.example.yaml`)**:
   - Added `streaming.send-keepalive-before-ft: true` to prevent Cloudflare Error 524 by flushing SSE headers and heartbeats before first token.
   - Added `payload.default` tiered `thinkingBudget` examples using first-write-wins ordering, decoupling bare `gemini-3.8-flash` (4000 budget, 4-10s fast turnarounds) from heavy reasoning models (16000 budget).
   - Added `antigravity.connection-pool` recommendations (20s idle timeout & 4 max idle conns) to avoid hanging half-open TCP connections.
   - Added `model-alias` Antigravity examples mapping `-low` and `-medium` to `-high` endpoints.
3. **Documentation (`docs/antigravity-argon-flash-routing.md`)**:
   - Comprehensive explanation of Google `v1internal` thought stripping on `-low` physical endpoints and the $\ge 16384$ threshold for Argon.
   - Full analysis of Cloudflare 120s `origin_response_timeout` (Error 524) and native zero-disconnect stability suite.

---

## Technical Context & Root Causes

### 1. Upstream Thought Stripping on `-low` Endpoints
Testing against Google `v1internal` revealed that physical model endpoints named `*-low` or `*-medium` strip the `thought` field even with `includeThoughts: true`. By excluding these raw endpoints and alias-routing them to `*-high` while injecting specific `thinkingBudget` values, clients receive authentic thinking tokens matching each tier.

### 2. Thinking Budget Threshold & Budget Decoupling
- The `gemini-4-argon-trial` engine requires `thinkingBudget >= 16384` to emit thinking process tokens. Setting the budget below 16384 results in an empty thinking block.
- Bare `gemini-3.8-flash` is mapped to a 4000 thinking budget for swift daily turnarounds (4~10s), while `*-high` and `*-tiered` remain dedicated to full 16000 deep-reasoning workloads.

### 3. Cloudflare Error 524 Prevention
Cloudflare CDN enforces a 120s timeout before the first byte from the origin. By enabling `streaming.send-keepalive-before-ft: true`, CLIProxyAPI commits HTTP 200 SSE headers and sends `: keep-alive\n\n` comments every 15s during upstream model thinking, safely resetting the 120s timer without exposing the origin IP.
