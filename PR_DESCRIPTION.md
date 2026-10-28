# PR Description: feat(antigravity): add Gemini 4 Argon & 3.8 Flash tiered models and thinking routing

## Summary

This pull request completes model registry definitions for **Gemini 4 Argon** and **Gemini 3.8 Flash** under the Antigravity OAuth provider, and provides configuration best practices to resolve upstream `thought` stripping and long-reasoning SSE timeouts.

### What's Changed

1. **Model Registry (`internal/registry/models/models.json`)**:
   - Added `gemini-4-argon`, `gemini-4-argon-low`, `gemini-4-argon-medium`, `gemini-4-argon-thinking`
   - Added `gemini-3.8-flash`, `gemini-3.8-flash-low`, `gemini-3.8-flash-medium`, `gemini-3.8-flash-tiered`
   - Added Claude思考与分档补全：`claude-opus-4-6`, `claude-sonnet-4-6-thinking`, `claude-opus-5-5-medium`, `claude-sonnet-5-5-medium`
2. **Configuration Examples (`config.example.yaml`)**:
   - Added `payload.default` tiered `thinkingBudget` examples using first-write-wins ordering.
   - Added `model-alias` Antigravity examples mapping `-low` and `-medium` to `-high` endpoints.
3. **Documentation (`docs/antigravity-argon-flash-routing.md`)**:
   - Comprehensive explanation of Google `v1internal` thought stripping on `-low` physical endpoints and the $\ge 16384$ threshold for Argon.
   - Production Nginx buffering and SSE keepalive guidelines.

---

## Technical Context & Root Causes

### 1. Upstream Thought Stripping on `-low` Endpoints
Testing against Google `v1internal` revealed that physical model endpoints named `*-low` or `*-medium` strip the `thought` field even with `includeThoughts: true`. By excluding these raw endpoints and alias-routing them to `*-high` while injecting specific `thinkingBudget` values, clients receive authentic thinking tokens matching each tier.

### 2. Thinking Budget Threshold
The `gemini-4-argon-trial` engine requires `thinkingBudget >= 16384` to emit thinking process tokens. Setting the budget below 16384 results in an empty thinking block.

### 3. Payload Rule Ordering
Because `payload.default` matches with **first-write-wins** semantics, specific tier entries (`-low`, `-medium`) are documented before generic/high entries to prevent configuration shadowing.
