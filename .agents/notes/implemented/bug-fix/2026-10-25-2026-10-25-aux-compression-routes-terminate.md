# Agent Note: 2026 10 25 aux compression routes terminate

Status: implemented

## Context

On 2026-10-25 a long session (20261024_180317_09288d, ~110k tokens) failed pre-API
compression on a hard loop: the aux summary route `amd-radeon/DeepSeek-V4-Flash`
streamed `reasoning_content` continuously for the full 600s compaction ceiling
("still streaming ... last progress 0.0-9.6s ago" x19 heartbeats), then the pinned
retry on `fallback_chain[0]` = `nvidia/z-ai/glm-5.3-flash` re-burned another 600s,
and the turn ended `reason=context_compression_timeout` with NO messages dropped —
so the next turn re-triggered the same fatal compaction. Live probes that day:

- `amd-radeon/DeepSeek-V4-Flash` is healthy ONLY with thinking off:
  non-stream `reasoning_effort:"none"` answers in ~1.8s, stream with
  `reasoning:{"effort":"none"}` completes in ~19s. With thinking on (the
  configured default: no knob was set), the summary phase streams forever.
  Relay auth flaps transiently (one 401 within minutes, healthy again later).
- `nvidia` chat completions black-hole (0 bytes over 60s while /v1/models
  returns instantly) — consistent with the 2026-10-19 log where the aux call
  sat 300s with `time_to_first_progress_ms=null`.
- Hermes' generic fast-lane shape `reasoning:{"enabled":false,"effort":"none"}`
  is rejected by amd-radeon (`param: reasoning.enabled` unsupported), and the
  400-strip retry then falls back to route default = thinking ON, so the
  certified fast lane cannot rescue this relay either.
- Other checked relays unusable that day: dashscope compatible-mode 401 (the
  stored `DASHSCOPE_API_KEY` is an LTAI AK, not an sk- model key), z.ai 401
  expired, HF router 402 out of credits, opencode free 429 quota.
- `google/gemini-2.5-flash` (already a configured provider with key) answered
  clean: non-stream 200/0.9s and stream 200/0.65s with a proper `data: [DONE]`.
  Hermes' own `call_llm(task="compression")` resolved it and round-tripped
  "PING-OK" through `extract_content_or_reasoning` in 1.63s.

## Decision

Pin the compression aux task to a route that provably terminates, and drop the
provably-dead fallback so a stall cannot double-burn the ceiling:

```yaml
auxiliary:
  compression:
    provider: google
    model: gemini-2.5-flash   # no reasoning knob: finishes fast, static-best
    # (fallback_chain removed — nvidia black-holes chat; amd-radeon needs a
    # vendor reasoning knob hermes cannot express without a 400-strip cycle)
```

Operator procedure for re-adding any summary route (recorded in the config
comment): live-probe streaming termination AND thinking-off acceptance BEFORE
it enters `auxiliary.compression`.

## Alternatives considered

### Why not keep amd-radeon + `extra_body: {reasoning: {effort: none}}`?

It verified healthy in a raw probe, but that task-level extra body travels to
EVERY route (task extras are universal), so a fallback to any provider that
rejects a bare `reasoning` object (google does; nvidia untestable today) pays a
400 + auto-strip retry for every summary, and the relay's transient 401 flap
already showed it is not an always-available primary.

### Why not repair the fast lane to accept deepseek-style knobs?

That is a code change to certified-route processors for one relay with flapping
auth and a never-ends-by-default failure mode; wrong trade now that a clean
route exists. If nvidia枠 ever recovers and amd-radeon hardens, the generic
mechanism needs a per-provider reasoning-wire translation table, not an
exception.

### Why not an opencode free model?

The main loop is already 429-free-tier-quota-exhausted for hours per day;
routing aux compression into the same exhausted bucket would guarantee daily
compaction failures.

## Consequences

Compaction now rides gemini-2.5-flash quota/uptime; the summary content
character changes with a different summarizer (acceptable: the structured
summary prompt dominates). No fallback means a gemini outage degrades straight
to the deterministic fallback summary instead of retrying a second cad model —
that is intentional: a bounded degradation beats a 20-minute turn freeze.
The long-lived `gateway run` (pid 427574) and interactive CLI (pid 343894)
retain the stale in-process config until they restart.

## Verification

- Live hang evidence reconciled from `~/.hermes/logs/agent.log` (2
  consecutive 600s waits, then final degrade, session 09288d).
- Post-fix E2E, real path: `call_llm(task="compression")` from the release
  environment against the edited `~/.hermes/config.yaml` returned PING-OK in
  1.63s; raw probes pinned route health the same day (see Context numbers).
- Regression guard after any future route re-addition: a stream PING must
  finish (SSE `data: [DONE]`) within 60s, and a thinking-off variant must be
  accepted (no 400) on that exact base_url before the route is configured.

## Cross-references

- `agent/auxiliary_client.py` — `_compression_fast_lane_controls`,
  `_parameter_rungs` (reasoning-strip retry that unknowingly resumes thinking).
- `agent/context_compressor.py` — `_call_summary_llm` (deliberate no-max_tokens
  summary contract, comment "NEVER add a max_tokens wire cap").
- `~/.hermes/config.yaml` — the applied operator edit (comment block at
  `auxiliary.compression`).
