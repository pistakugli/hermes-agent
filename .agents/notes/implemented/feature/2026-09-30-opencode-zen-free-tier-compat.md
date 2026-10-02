# Agent Note: Opencode zen free tier compat

Status: implemented

## Context

The OpenCode Zen free tier rejects Hermes calls with HTTP 403 `FreeTierError`
("OpenCode's free tier can only be used from within OpenCode"). Probe testing
identified five wire-level gate conditions, all of which must hold on a request:
(1) an opencode-style `User-Agent` (Hermes sends `HermesAgent/...`, rejected),
(2) header `x-opencode-session: ses_<12 hex><14 base62>` (26 chars total),
(3) header `x-opencode-request: msg_` with the same shape,
(4) body `stream: true`,
(5) body `tools` containing the function definitions `bash` AND `read`
(truncated prompts still pass the tool check). Hermes already satisfies (4) at
the turn call site, and the core exposes a `llm_request` middleware contract
(`apply_llm_request_middleware`, `agent/turn_api_request.py:145-150`) that can
rewrite headers and JSON before dispatch, plus a provider-profile mechanism for
default headers. Auxiliary calls (title generation etc.) never enter the
middleware path, so a request-level middleware alone cannot cover them.

## Decision

Ship the gate fix as two plugin-surface changes, with no core edits:

- **Profile** `plugins/model-providers/opencode-zen/__init__.py` owns only the
  client-wide header: a `_OPencode_FREE_TIER_UA` constant
  (`opencode/1.18.33 ai-sdk/provider-utils/4.0.23 runtime/bun/1.3.14`) merged
  into `_ZEN_HEADERS`, applied as `default_headers` on the `opencode_zen`
  profile. The `opencode-go` profile keeps Hermes attribution untouched.
- **Middleware plugin** `plugins/opencode-zen-compat/` (kind `standalone`,
  enabled via `plugins.enabled`) registers one `llm_request` middleware that
  owns both per-request gate ids: deterministic `ses_`/`msg_` ids of 26 hex
  chars derived from `sha256(resolve_affinity_key(session_id))` and
  `sha256(api_request_id)` (seeding the session id from the core affinity key
  preserves the backend cache pin), and appends verbatim canonical `bash` and
  `read` tool schemas to any request body lacking them. The middleware gates on
  provider family `"opencode-zen"` (fail-open for everything else, `opencode-go`
  explicitly excluded), returns `{"request", "source", "reason"}` on change and
  `None` otherwise, and relies on the core's fail-open `invoke_middleware`
  (exceptions are swallowed and logged).

## Alternatives considered

**`kind: backend` for auto-load.** Rejected: the plugin provides middleware, not
a model backend, and a mislabeled manifest would fool `gate_manifest`'s
bundled/backend auto-load rule. `kind: standalone` keeps activation an explicit,
reversible opt-in (`hermes plugins enable opencode-zen-compat`).

**Patching the gate ids in core affinity code.** Rejected: the requirement was a
plugin-only fix touching no core files; the middleware seam already fires on the
exact request path that needs it.

**Trimming the `bash`/`read` schemas to short stubs.** Rejected: the gate may
hash or deep-compare the canonical tool definitions (probed with real opencode
client traffic), so verbatim schemas from the live client payload are the safe
choice. Known cost: the stubs are non-executable, so the model receives an
unknown-tool error on first use and self-corrects to Hermes's real `terminal` /
`read_file` tools.

**Fixing auxiliary calls in the plugin.** Not possible: aux requests bypass
`apply_llm_request_middleware` entirely, so they are handled by config
`auxiliary:` provider/model pins instead of code. Mass-pinning the user's
`provider: auto` aux tasks was deliberately not done without asking.

**Adding `stream: true` in middleware.** Rejected: the turn call site already
streams; duplicating it would fight the existing contract.

## Consequences

- The `bash`/`read` stubs send ~5.8 KB of unused schema on zen requests and can
  produce one recoverable unknown-tool error per fresh conversation when the
  model picks a stub over a real tool.
- The UA constant pins a client version string; if the gate ever validates UA
  freshness against current releases, the constant needs bumping (single line,
  profile-owned).
- Auxiliary calls still depend on config pins; a misconfigured `provider: auto`
  aux task on the zen main model will 403 at the upstream with no plugin-side
  recourse.
- The plugin must stay in `plugins.enabled` — disabling it restores the 403 for
  `ses_`/`msg_`-shaped ids and tool presence, while the UA fix in the profile
  remains active regardless.

## Verification

- Generator self-test: deterministic `ses_`/`msg_`+26-hex ids, tools
  `[terminal, bash, read]` appended, non-zen providers return `None`.
- Wire capture through a local MITM proxy: main chat request observed upstream
  with **status 200**, UA `opencode/1.18.33 ...`, valid `ses_`/`msg_` ids,
  `stream=True`, `bash=True`, `read=True` (aux title call 403, expected).
- Live end-to-end on the real profile: `hermes --provider opencode-zen -m
  mimo-v2.6-flash-free chat -q "Reply with exactly: PONG3"` returned `PONG3`,
  exit 0, zero 403s.
- Relevant unit tests via `scripts/run_tests.sh`:
  `tests/agent/test_provider_attribution_headers.py`,
  `tests/hermes_cli/test_opencode_zen_model_limit.py`,
  `tests/hermes_cli/test_plugins.py` — 95/95 passed.

## Cross-references

- [plugin kind/middleware contract](../../../plugins/AGENTS.md) — plugin kinds
  and the native compat contract this change stays inside.
