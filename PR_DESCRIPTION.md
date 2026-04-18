## What does this PR do?

Fixes a class of `HTTP 400: "messages: text content blocks must be non-empty"` errors that occur when Hermes talks to an OpenAI-compatible endpoint that proxies to Anthropic (e.g. `claude-code-router`, `one-api`'s Anthropic channel, any reverse proxy that translates OpenAI `/v1/chat/completions` to Anthropic `/v1/messages`).

Several code paths were producing assistant messages with `content: ""` (empty string) instead of the OpenAI-standard `content: null`. When such a message is translated to Anthropic format, the empty string becomes an empty `{"type": "text", "text": ""}` block, which Anthropic rejects.

The fix takes a defense-in-depth approach:

1. **Source fixes** — avoid producing empty content at three known emission points.
2. **Exit-level safety net** — `_build_api_kwargs` now sanitizes residual empty content on the `chat_completions` path, covering edge cases from session load, context compression, prefill messages, etc.

## Related Issue

Fixes #

## Type of Change

- [x] 🐛 Bug fix (non-breaking change that fixes an issue)
- [ ] ✨ New feature (non-breaking change that adds functionality)
- [ ] 🔒 Security fix
- [ ] 📝 Documentation update
- [ ] ✅ Tests (adding or improving test coverage)
- [ ] ♻️ Refactor (no behavior change)
- [ ] 🎯 New skill (bundled or hub)

## Changes Made

All changes are in `run_agent.py`:

- **New helpers (module-level):**
  - `_needs_empty_text_sanitization(messages)` — read-only pre-scan, so normal traffic pays zero cost.
  - `_sanitize_empty_text_blocks(messages)` — in-place cleanup: empty-string or whitespace-only `content` on `assistant + tool_calls` → `None` (OpenAI-standard); other roles → single-space placeholder; list content filters empty text blocks with the same fallback rules.

- **Source fix — `_build_assistant_message` (~line 7041):** when the model returns no text content, store `None` instead of `""`. Matches OpenAI's spec for `assistant + tool_calls` turns and prevents the empty string from entering conversation history in the first place.

- **Source fix — KV-cache whitespace normalization (~line 8878):** the existing `.strip()` loop silently turned `"   "` into `""`, actively creating the problem it was supposed to prevent. Whitespace-only content now maps to `None` (for `assistant + tool_calls`) or `" "` (other roles).

- **Source fix — Codex Responses placeholder (~line 3915):** the required "following item" after a reasoning-only turn uses `" "` instead of `""` for consistency. This path does not go through `_build_api_kwargs`, so the source fix is the only defense.

- **Exit-level safety net — `_build_api_kwargs` chat_completions path (~line 6787):** runs `_needs_empty_text_sanitization` first; only when empty content is detected does it shallow-copy messages and sanitize. Preserves the existing `kwargs["messages"] is messages` semantics for normal traffic.

## How to Test

**Reproduction (before the fix):**

1. Run any OpenAI→Anthropic proxy locally, e.g. `claude-code-router` on `http://localhost:3001/v1`.
2. Configure Hermes:

   ```yaml
   model:
     default: "claude-opus-4-6"
     provider: "custom"
     base_url: "http://localhost:3001/v1"
   ```

3. Start a conversation that triggers a tool call. As soon as the assistant returns a message with only `tool_calls` (no text), the next turn fails with:

   ```
   HTTP 400: messages: text content blocks must be non-empty
   ```

**Verification (after the fix):**

1. Same setup, same conversation — the tool-calling loop now completes without 400.
2. Unit-level check — the helper produces the correct output for all cases:

   ```python
   from run_agent import _sanitize_empty_text_blocks

   msgs = [{"role": "assistant", "content": "", "tool_calls": [{"id": "1"}]}]
   _sanitize_empty_text_blocks(msgs)
   assert msgs[0]["content"] is None  # OpenAI-standard null
   ```

3. Regression check — `_build_api_kwargs(messages)` returns `kwargs["messages"] is messages` for conversations without empty content (no unnecessary copy).

## Checklist

### Code

- [x] I've read the [Contributing Guide](https://github.com/NousResearch/hermes-agent/blob/main/CONTRIBUTING.md)
- [x] My commit messages follow [Conventional Commits](https://www.conventionalcommits.org/) (`fix(scope):`, `feat(scope):`, etc.)
- [x] I searched for [existing PRs](https://github.com/NousResearch/hermes-agent/pulls) to make sure this isn't a duplicate
- [x] My PR contains **only** changes related to this fix/feature (no unrelated commits)
- [ ] I've run `pytest tests/ -q` and all tests pass
- [ ] I've added tests for my changes (required for bug fixes, strongly encouraged for features)
- [x] I've tested on my platform: macOS 15 (Darwin 25.3.0), Python 3.11

### Documentation & Housekeeping

- [x] I've updated relevant documentation (README, `docs/`, docstrings) — N/A (internal helpers, no public API change)
- [x] I've updated `cli-config.yaml.example` if I added/changed config keys — N/A (no new config)
- [x] I've updated `CONTRIBUTING.md` or `AGENTS.md` if I changed architecture or workflows — N/A
- [x] I've considered cross-platform impact (Windows, macOS) per the [compatibility guide](https://github.com/NousResearch/hermes-agent/blob/main/CONTRIBUTING.md#cross-platform-compatibility) — pure Python string/dict manipulation, no platform-specific code
- [x] I've updated tool descriptions/schemas if I changed tool behavior — N/A

## Screenshots / Logs

**Before:**

```
⚠️  API call failed (attempt 1/3): BadRequestError [HTTP 400]
   🔌 Provider: custom  Model: claude-opus-4-6
   🌐 Endpoint: http://localhost:3001/v1
   📝 Error: HTTP 400: messages: text content blocks must be non-empty
   📋 Details: {'message': 'messages: text content blocks must be non-empty',
                'type': 'invalid_request_error', 'param': '', 'code': None}
⚠️ Non-retryable error (HTTP 400) — trying fallback...
❌ Non-retryable client error (HTTP 400). Aborting.
```

**After:** the tool-calling loop completes normally; no 400 response from the proxy.
