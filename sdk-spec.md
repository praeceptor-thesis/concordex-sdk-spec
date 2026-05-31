# Concordex SDK Specification

**Version:** see [`VERSION`](./VERSION) — currently `0.5.0`
**Status:** pre-1.0 (MINOR bumps may include breaking wire changes)
**Last updated:** 2026-05-30

This document defines the public surface every Concordex SDK MUST
implement. Every language binding — Python, TypeScript, C#, Java — is a
translation of this surface. Same constructor shape, same methods (under
language-idiomatic naming), same return types, same error hierarchy,
same wire protocol.

A spec tag (`v0.5.0`) corresponds 1:1 to a release tag in every SDK
repository. No SDK ships a version the spec hasn't blessed.

---

## 1. Wire protocol

### 1.1 Base URL

Default: `https://api.concordex.dev`. The constructor MUST accept an
override (used for staging, self-hosted, and local development against
`https://staging.api.praeceptor-thesis.com` or `http://localhost:8080`).

### 1.2 Authentication

Every request MUST carry:

```
Authorization: Bearer ck_<api-key>
```

API keys start with the prefix `ck_`. The SDK MUST reject empty keys or
keys that do not begin with `ck_` at construction time. Treat key
material as a secret in logs (redact or omit).

Do NOT carry the dashboard session cookie. Do NOT set the legacy
`X-Concordex-Key` header — deprecated and slated for removal in v1.0.

### 1.3 Content-Type

All request bodies are JSON-encoded. SDKs MUST set
`Content-Type: application/json` on every POST.

### 1.4 User-Agent

SDKs MUST set:

```
User-Agent: concordex-<language>/<spec-version>
```

Examples: `concordex-python/0.5.0`, `concordex-typescript/0.5.0`,
`concordex-csharp/0.5.0`, `concordex-java/0.5.0`. Optional suffix in the
form `(<runtime-info>)` is permitted.

### 1.5 Timeout

Default: 10000ms (10 seconds). The constructor MUST expose an override
in the language's native duration type (`float` seconds for Python,
`number` ms for TypeScript, `TimeSpan` for C#, `Duration` for Java).

### 1.6 Async mode header

For `/v1/agent-stream/event`, default behavior is synchronous (server
reasons immediately, returns the rich envelope). Setting
`X-Concordex-Async: true` switches to fire-and-forget — server queues,
returns minimal envelope only.

---

## 2. Endpoints

### 2.1 POST /v1/agent-stream/event

Emit one event into the agent stream.

#### Request body

| Field                | Type                | Required | Notes                                                              |
|----------------------|---------------------|----------|--------------------------------------------------------------------|
| `kind`               | enum string         | yes      | `subject_says` \| `tool_call` \| `tool_result` \| `observation`    |
| `agent_subject_id`   | string              | yes      | The conversation anchor — every event grounds against an agent     |
| `payload`            | object              | yes      | May be empty `{}`                                                  |
| `interaction_id`     | string              | no       | Server assigns if omitted; SDK should cache and re-send            |
| `interaction_kind`   | string              | no       | Default `chat_session`                                             |
| `subjects`           | array of subject    | no       | `[{subject_id, role, kind, metadata?}]` — conversation roster     |
| `speaker_subject_id` | string              | no       | The participant who spoke / acted                                  |
| `speaker_role`       | string              | no       | Role label for the speaker                                         |
| `occurred_at`        | string (ISO-8601)   | no       | Defaults to server receive-time                                    |
| `metadata`           | object              | no       | Free-form per-event metadata                                       |

#### Response body — synchronous (default)

```json
{
  "interaction_id": "int_abc",
  "subjects": ["user:ws:bot", "user:ws:cust"],
  "queued": false,
  "frame_id": "frame_xyz",
  "subject_id": "user:ws:bot",
  "outcome": "scored",
  "triage_decision": "deep",
  "tags_fired": ["risk.refund_pressure", "tone.frustrated"],
  "scored_by_canons": ["canon_market_abnormality_v3"],
  "soul_version": 42,
  "ledger_index": 1234,
  "follow_my_data": "/w/ws_xxx/frames/frame_xyz"
}
```

#### Response body — async (`X-Concordex-Async: true`)

```json
{
  "interaction_id": "int_abc",
  "subjects": ["user:ws:bot", "user:ws:cust"],
  "queued": true
}
```

#### Fields that MAY be absent

When the server can't determine a value (e.g. no tags fired), it omits
the field rather than returning `null` or an empty array. The SDK MUST
treat missing fields as "unknown", not as an empty result. `EmitResult`
fields backed by optional response keys MUST be exposed as optional /
nullable in the language.

#### Outcome enum

`outcome` ∈ {`scored`, `tagged`, `no_tags_fired`, `rejected`, `error`}.
SDKs SHOULD expose this as an enum in the language.

### 2.2 POST /v1/cb/check

Synchronous circuit-breaker check.

#### Request body

| Field       | Type   | Required | Notes                                       |
|-------------|--------|----------|---------------------------------------------|
| `scope`     | enum   | yes      | `subject` \| `interaction`                  |
| `scope_ref` | string | yes      | The subject_id or interaction_id            |

Exactly one of subject_id / interaction_id must be passed to the SDK
method; the SDK derives `scope` and `scope_ref` from which was set.

#### Response body

```json
{
  "state": "closed",
  "allow": true,
  "warning": false,
  "reason": "no policies fired",
  "fired_policies": [],
  "anchor": null,
  "checked_at": "2026-05-30T12:00:00Z",
  "latency_ms": 12.3,
  "route_latency_ms": 18.7
}
```

`state` ∈ {`closed`, `half_open`, `open`}.
`allow` is `false` only when `state == open`.
`warning` is `true` when `state == half_open`.

`fired_policies` is an array of `{cb_policy_id, name, action}` objects.
`anchor` is an object `{ledger_index, hash}` or `null`.

---

## 3. Error handling

The server returns standard HTTP status codes. Every SDK MUST map them
to a typed exception hierarchy:

| Status | Canonical type      | Meaning                              |
|--------|---------------------|--------------------------------------|
| 400    | `ValidationError`   | malformed payload                    |
| 401    | `AuthError`         | API key missing / invalid / revoked  |
| 403    | `PermissionError`   | key valid but lacks scope            |
| 5xx    | `ServerError`       | transient — safe to retry            |
| other  | `ConcordexError`    | unexpected status                    |

`CBOpenError` (canonical name) is NOT raised from the HTTP layer. It is
raised by the `guard()` context-manager / using-block when
`raiseOnOpen=true` and the check returns `allow=false`.

Every exception MUST expose:

- `message: string`
- `statusCode: number | null`
- `body: object | string | null`

`CBOpenError` additionally exposes:

- `reason: string`
- `firedPolicies: array<{cb_policy_id, name, action}>`
- `anchor: object | null`
- `scopeRef: string`

### 3.1 Network and timeout

Timeouts and underlying network errors MUST be wrapped as `ServerError`
with a descriptive message. The original cause MUST be preserved via the
language's exception-chaining mechanism (`raise … from e` in Python,
`cause` in TypeScript Error, `InnerException` in C#, `initCause()` in
Java).

---

## 4. Client constructor

Each SDK exposes a primary client class. Canonical name in each
language:

| Language    | Class name          |
|-------------|---------------------|
| Python      | `Concordex`         |
| TypeScript  | `Concordex`         |
| C#          | `ConcordexClient`   |
| Java        | `ConcordexClient`   |

C# and Java diverge from the bare `Concordex` name because both
languages reserve unqualified type names for value-bearing entities and
both expect a `Client` / service suffix for HTTP service classes.

### 4.1 Constructor parameters

| Spec name      | Type                 | Required | Default                          |
|----------------|----------------------|----------|----------------------------------|
| `api_key`      | string               | yes      | (none)                           |
| `base_url`     | string               | no       | `https://api.concordex.dev`      |
| `timeout`      | duration             | no       | 10000ms / 10s                    |
| `user_agent`   | string               | no       | `concordex-<lang>/<spec-version>`|

### 4.2 Thread safety

The client MUST be safe to share across threads / async contexts where
the language permits it:

- Python: thread-safe via `httpx.Client` reuse.
- TypeScript: single-loop-per-context, no cross-thread concern.
- C#: `HttpClient` is thread-safe; the wrapping client must be too.
- Java: pick a thread-safe HTTP client (OkHttp recommended).

### 4.3 Resource lifecycle

The client MUST implement the language's idiomatic resource cleanup:

- Python: `__enter__` / `__exit__` (context manager) + explicit
  `close()`.
- TypeScript: explicit `close()` method.
- C#: `IDisposable.Dispose()` + explicit `Close()`.
- Java: `AutoCloseable.close()` + explicit `close()`.

Closing the client MUST release the underlying HTTP transport. Calling
`close()` more than once MUST be a no-op (not an error).

---

## 5. Methods

This section defines the canonical method surface in language-agnostic
snake_case. See §8 for the per-language idiomatic name map.

### 5.1 `emit_event(kind, ...) → EmitResult`

Low-level event emitter. Every higher-level helper delegates to this.

Parameters (snake_case canonical names; language-idiomatic equivalents
per §8):

- `kind: string` — REQUIRED, must be in `EVENT_KINDS`
- `agent_subject_id: string` — REQUIRED
- `payload: object` — defaults to `{}`
- `interaction_id?: string`
- `interaction_kind?: string` — default `"chat_session"`
- `subjects?: array<subject>`
- `speaker_subject_id?: string`
- `speaker_role?: string`
- `occurred_at?: string` — ISO-8601
- `metadata?: object`

Validation: throw `ValueError` (Python) / `RangeError` (TS) /
`ArgumentException` (C#) / `IllegalArgumentException` (Java) when
`kind` is not one of `EVENT_KINDS`.

### 5.2 `subject_says(subject_id, text, agent_subject_id, ...) → EmitResult`

Convenience wrapper for `kind = "subject_says"`.

- `subject_id: string` — the speaker, REQUIRED
- `text: string` — REQUIRED
- `agent_subject_id: string` — REQUIRED (conversation anchor)
- `interaction_id?: string`
- `subjects?: array<subject>`
- `payload_extra?: object` — merged into `payload` alongside `{text}`

The SDK passes `speaker_subject_id = subject_id` on the wire.

### 5.3 `tool_call(subject_id, tool, args, ...) → EmitResult`

Convenience wrapper for `kind = "tool_call"`. Use BEFORE the tool runs —
this emits the intent, not the result.

- `subject_id: string` — the agent invoking the tool, REQUIRED
- `tool: string` — REQUIRED
- `args: object` — defaults to `{}`
- `interaction_id?: string`
- `subjects?: array<subject>`

The SDK passes `speaker_role = "agent"` on the wire.

### 5.4 `tool_result(subject_id, tool, result, ...) → EmitResult`

Convenience wrapper for `kind = "tool_result"`. Pair with the prior
`tool_call`.

- `subject_id: string` — REQUIRED
- `tool: string` — REQUIRED
- `result: any` — REQUIRED (the tool's return value; SDK JSON-encodes)
- `interaction_id?: string`
- `subjects?: array<subject>`

### 5.5 `observation(agent_subject_id, subjects, payload, ...) → EmitResult`

Convenience wrapper for `kind = "observation"`. Use for structured
events that don't fit a speech-bubble shape (video keyframes, IoT,
sensor readings).

- `agent_subject_id: string` — REQUIRED
- `subjects: array<subject>` — REQUIRED
- `payload: object` — REQUIRED
- `interaction_id?: string`

### 5.6 `check(subject_id? | interaction_id?) → CheckResult`

Circuit-breaker check. Pass EXACTLY ONE of `subject_id` or
`interaction_id`. Both nil or both set MUST throw a validation error.

### 5.7 `guard(subject_id? | interaction_id?, raise_on_open=false) → context-managed CheckResult`

Resource-scoped wrapper around `check()`. Language idioms:

- Python: `@contextmanager`-style `with cx.guard(...) as g:` yielding
  the `CheckResult`.
- TypeScript: async function returning `{result, [Symbol.dispose]}`
  pair, usable via `using g = cx.guard(...)`. Fallback: callback form
  `cx.guard(opts, async (result) => {...})`.
- C#: `Guard` returns an `IDisposable` wrapper exposing `Result`
  property; usable via `using var g = client.Guard(...);`.
- Java: `Guard` returns an `AutoCloseable` with `getResult()`; usable
  via `try (var g = client.guard(...)) {...}`.

When `raise_on_open = true` and `result.allow == false`, the
language-canonical `CBOpenError` MUST be raised before yielding.

### 5.8 `conversation(participants, agent_subject_id?, kind?, metadata?) → Conversation`

Factory that returns a Conversation handle (see §6).

- `participants: array<participant>` — REQUIRED, non-empty. Each
  participant: `{subject_id, role?, kind?, metadata?}`.
- `agent_subject_id?: string` — derived from participants if omitted;
  see §6.1.
- `kind?: string` — interaction_kind, default `"chat_session"`.
- `metadata?: object`.

### 5.9 `close() → void`

Release the underlying HTTP client. Idempotent.

---

## 6. Conversation handle

A stateful helper for the common single-agent-with-customer case (and
generalizing to multi-agent / multi-subject conversations).

### 6.1 Construction

Conversation is constructed via `client.conversation(...)`. Direct
construction is NOT part of the public API.

`agent_subject_id` is derived from participants when not supplied: first
participant whose `role` (lowercased) is in
`{"agent", "service", "system"}` wins; otherwise the first participant.

`interaction_id` is `null` initially and is captured from the first
`EmitResult.interaction_id` the server returns. Subsequent calls
re-send it so the server stitches events into the same interaction.

### 6.2 Methods on Conversation

| Method                                        | Returns        |
|-----------------------------------------------|----------------|
| `says(subject_id, text, payload_extra?)`      | `EmitResult`   |
| `tool_call(subject_id, tool, args?)`          | `EmitResult`   |
| `tool_result(subject_id, tool, result)`       | `EmitResult`   |
| `observation(payload)`                        | `EmitResult`   |
| `add_subject(subject_id, role?, kind?, metadata?)` | `void`    |
| `check(subject_id)`                           | `CheckResult`  |
| `guard(subject_id, raise_on_open=false)`      | context-managed CheckResult |
| `close()`                                     | `void`         |

Properties:

| Property         | Type             |
|------------------|------------------|
| `interaction_id` | string \| null   |
| `subjects`       | array<subject>   |

`add_subject` is idempotent on `subject_id`: re-adding with the same id
refreshes the role/kind/metadata.

### 6.3 Resource lifecycle

Conversation implements the language's resource idiom (the same one the
client implements). Currently the close path is a no-op; v1.0 will emit
an `end_interaction` event.

---

## 7. Result types

### 7.1 `EmitResult`

Returned by every event-emit method.

| Field             | Type                  | Notes                                          |
|-------------------|-----------------------|------------------------------------------------|
| `interaction_id`  | string                | "" if server omitted                           |
| `subjects`        | array<string>         | subject ids on the resulting frame             |
| `queued`          | boolean               | true in async mode, false in sync              |
| `frame_id`        | string?               | sync mode only                                 |
| `subject_id`      | string?               | sync mode only — the primary subject            |
| `outcome`         | enum?                 | sync mode only — see §2.1                       |
| `triage_decision` | string?               | sync mode only — `deep` \| `shallow`           |
| `tags_fired`      | array<string>?        | sync mode only                                 |
| `scored_by_canons`| array<string>?        | sync mode only — canon ids                      |
| `soul_version`    | integer?              | sync mode only                                 |
| `ledger_index`    | integer?              | sync mode only                                 |
| `follow_my_data`  | string?               | sync mode only — dashboard path to this frame  |
| `raw`             | object                | the full server JSON response                  |

### 7.2 `CheckResult`

Returned by `check()`.

| Field              | Type            | Notes                                            |
|--------------------|-----------------|--------------------------------------------------|
| `state`            | enum            | `closed` \| `half_open` \| `open`                |
| `allow`            | boolean         | `false` only when state is `open`                |
| `warning`          | boolean         | `true` when state is `half_open`                 |
| `reason`           | string          |                                                  |
| `fired_policies`   | array<object>   | `[{cb_policy_id, name, action}]`                |
| `anchor`           | object \| null  | `{ledger_index, hash}` or null                  |
| `checked_at`       | string          | ISO-8601, may be ""                              |
| `latency_ms`       | number          | server-side cb.check() latency                   |
| `route_latency_ms` | number          | server-side route handler latency                |
| `raw`              | object          | the full server JSON response                    |

### 7.3 Immutability

Result types MUST be immutable in the language (Python `@dataclass(frozen=True)`,
TypeScript `readonly`, C# `record`, Java `record`).

---

## 8. Naming map

The canonical surface above uses snake_case. Each language's SDK
translates as follows. **No creative liberty** — these are the names
your SDK MUST expose.

### 8.1 Top-level

| Canonical            | Python             | TypeScript         | C#                  | Java               |
|----------------------|--------------------|--------------------|---------------------|--------------------|
| `Concordex`          | `Concordex`        | `Concordex`        | `ConcordexClient`   | `ConcordexClient`  |
| `Conversation`       | `Conversation`     | `Conversation`     | `Conversation`      | `Conversation`     |
| `EmitResult`         | `EmitResult`       | `EmitResult`       | `EmitResult` (record) | `EmitResult` (record) |
| `CheckResult`        | `CheckResult`      | `CheckResult`      | `CheckResult` (record) | `CheckResult` (record) |
| `EVENT_KINDS`        | `EVENT_KINDS`      | `EVENT_KINDS`      | `EventKinds.All`    | `EventKinds.ALL`   |

### 8.2 Methods

| Canonical          | Python             | TypeScript          | C#                  | Java                |
|--------------------|--------------------|---------------------|---------------------|---------------------|
| `emit_event`       | `emit_event`       | `emitEvent`         | `EmitEvent`         | `emitEvent`         |
| `subject_says`     | `subject_says`     | `subjectSays`       | `SubjectSays`       | `subjectSays`       |
| `tool_call`        | `tool_call`        | `toolCall`          | `ToolCall`          | `toolCall`          |
| `tool_result`      | `tool_result`      | `toolResult`        | `ToolResult`        | `toolResult`        |
| `observation`      | `observation`      | `observation`       | `Observation`       | `observation`       |
| `check`            | `check`            | `check`             | `Check`             | `check`             |
| `guard`            | `guard`            | `guard`             | `Guard`             | `guard`             |
| `conversation`     | `conversation`     | `conversation`      | `Conversation` (method) | `conversation`  |
| `close`            | `close`            | `close`             | `Close` / `Dispose` | `close`             |
| `add_subject`      | `add_subject`      | `addSubject`        | `AddSubject`        | `addSubject`        |

### 8.3 Constructor parameters

| Canonical    | Python       | TypeScript   | C#           | Java         |
|--------------|--------------|--------------|--------------|--------------|
| `api_key`    | `api_key`    | `apiKey`     | `apiKey`     | `apiKey`     |
| `base_url`   | `base_url`   | `baseUrl`    | `baseUrl`    | `baseUrl`    |
| `timeout`    | `timeout`    | `timeout`    | `timeout`    | `timeout`    |
| `user_agent` | `user_agent` | `userAgent`  | `userAgent`  | `userAgent`  |

### 8.4 Result fields

| Canonical            | Python              | TypeScript            | C# (record property) | Java (record component) |
|----------------------|---------------------|-----------------------|----------------------|-------------------------|
| `interaction_id`     | `interaction_id`    | `interactionId`       | `InteractionId`      | `interactionId`         |
| `subject_id`         | `subject_id`        | `subjectId`           | `SubjectId`          | `subjectId`             |
| `frame_id`           | `frame_id`          | `frameId`             | `FrameId`            | `frameId`               |
| `triage_decision`    | `triage_decision`   | `triageDecision`      | `TriageDecision`     | `triageDecision`        |
| `tags_fired`         | `tags_fired`        | `tagsFired`           | `TagsFired`          | `tagsFired`             |
| `scored_by_canons`   | `scored_by_canons`  | `scoredByCanons`      | `ScoredByCanons`     | `scoredByCanons`        |
| `soul_version`       | `soul_version`      | `soulVersion`         | `SoulVersion`        | `soulVersion`           |
| `ledger_index`       | `ledger_index`      | `ledgerIndex`         | `LedgerIndex`        | `ledgerIndex`           |
| `follow_my_data`     | `follow_my_data`    | `followMyData`        | `FollowMyData`       | `followMyData`          |
| `fired_policies`     | `fired_policies`    | `firedPolicies`       | `FiredPolicies`      | `firedPolicies`         |
| `route_latency_ms`   | `route_latency_ms`  | `routeLatencyMs`      | `RouteLatencyMs`     | `routeLatencyMs`        |

### 8.5 Exceptions

| Canonical            | Python              | TypeScript            | C#                                       | Java                                    |
|----------------------|---------------------|-----------------------|------------------------------------------|-----------------------------------------|
| `ConcordexError`     | `ConcordexError`    | `ConcordexError`      | `ConcordexException`                     | `ConcordexException`                    |
| `AuthError`          | `AuthError`         | `AuthError`           | `ConcordexAuthException`                 | `ConcordexAuthException`                |
| `PermissionError`    | `PermissionError`   | `PermissionError`     | `ConcordexPermissionException`           | `ConcordexPermissionException`          |
| `ValidationError`    | `ValidationError`   | `ValidationError`     | `ConcordexValidationException`           | `ConcordexValidationException`          |
| `ServerError`        | `ServerError`       | `ServerError`         | `ConcordexServerException`               | `ConcordexServerException`              |
| `CBOpenError`        | `CBOpenError`       | `CBOpenError`         | `CircuitBreakerOpenException`            | `CircuitBreakerOpenException`           |

C# and Java rename to the `…Exception` convention idiomatic to their
ecosystems. TypeScript follows JS convention with `…Error`. Python keeps
its `…Error` convention.

### 8.6 EVENT_KINDS

The constant array `["subject_says", "tool_call", "tool_result", "observation"]`
MUST be exposed under the language's naming:

- Python: `EVENT_KINDS` (tuple).
- TypeScript: `EVENT_KINDS` (readonly tuple).
- C#: `EventKinds.All` (static IReadOnlyList) plus `EventKinds.SubjectSays`, etc.
- Java: `EventKinds.ALL` (immutable List) plus `EventKinds.SUBJECT_SAYS`, etc.

The on-wire `kind` string is ALWAYS the snake_case form regardless of
how the SDK exposes the enum.

---

## 9. Webhook signature verification

Concordex outbound webhooks are signed with HMAC-SHA256 using the
subscription's secret. Every SDK MUST expose a helper:

```
verify_webhook_signature(payload, signature_header, secret, tolerance_seconds=300) → bool
```

The verifier MUST:

1. Parse `t=<unix>,v1=<hex>` from the header.
2. Reject (return false / raise) if `t` is missing, non-numeric, or older
   than `tolerance_seconds`.
3. Compute `hmac_sha256(secret, f"{t}.{payload}")` and constant-time
   compare against `v1`.
4. Return `true` only on match.

Reference vectors are in
[`contract-tests/signature-vectors.json`](./contract-tests/signature-vectors.json).
Every SDK's contract test harness MUST run all vectors.

Canonical helper name in each language:

| Language    | Function name                          |
|-------------|----------------------------------------|
| Python      | `verify_webhook_signature` (free fn)   |
| TypeScript  | `verifyWebhookSignature` (free fn)     |
| C#          | `WebhookSignature.Verify` (static)     |
| Java        | `WebhookSignature.verify` (static)     |

---

## 10. Contract tests

Every SDK repository MUST contain a `spec-conformance.yml` GitHub
Actions workflow that:

1. Checks out a pinned tag of this repo.
2. Runs the contract test corpus from `contract-tests/` against the SDK
   build.
3. Reports red/green to the release coordination workflow here (via
   GitHub status check `spec-conformance/<spec-version>`).

The contract test corpus contains:

- [`golden-envelopes.json`](./contract-tests/golden-envelopes.json) —
  fixed inputs → expected wire bodies. Serialization must round-trip
  byte-for-byte (after JSON normalization: sort keys, no trailing
  whitespace).
- [`signature-vectors.json`](./contract-tests/signature-vectors.json) —
  HMAC verification vectors.
- [`error-mapping.json`](./contract-tests/error-mapping.json) — HTTP
  status → exception type the SDK must raise. The harness drives the
  SDK with a fake transport that returns each status and asserts the
  expected exception type.

See [`contract-tests/runner-spec.md`](./contract-tests/runner-spec.md)
for how the harness MUST be wired in each language.

---

## 11. Versioning

The `VERSION` file in this repo is the source of truth. Every SDK
release tag matches this exactly.

Pre-1.0 (current): MINOR bumps MAY include breaking wire changes; PATCH
bumps MUST remain backwards-compatible at both API and wire level.

Post-1.0: MAJOR bumps for breaking changes only, with at least one
MINOR-version deprecation cycle. The deprecation cycle MUST be
documented in `CHANGELOG.md` of the spec repo before a MAJOR is cut.

### 11.1 Spec version pinning

Each SDK pins to a spec version in its language-native manifest:

- Python: `pyproject.toml` → `[tool.concordex] spec-version = "0.5.0"`
- TypeScript: `package.json` → `"concordex": {"specVersion": "0.5.0"}`
- C#: `Directory.Build.props` → `<ConcordexSpecVersion>0.5.0</ConcordexSpecVersion>`
- Java: `pom.xml` → `<concordex.spec.version>0.5.0</concordex.spec.version>`

The SDK's CI MUST fail-loud if the pinned spec version doesn't match
the version of the spec repo it checks out.

### 11.2 Coordinated release

The `promote.yml` workflow in this repo:

1. Verifies all four SDK repos have a release branch tagged
   `release/v<version>`.
2. Verifies each repo's `spec-conformance` status check is green.
3. Dispatches `publish.yml` to each SDK repo (parallel).
4. Polls each registry (PyPI, npm, NuGet, Maven Central) for the
   artifact.
5. Marks the spec release `promoted` when all four confirm.
6. If any publish fails, runs `yank.yml` to pull artifacts that already
   landed.

---

## 12. Quick-start example (snake_case canonical)

```text
cx = Concordex(api_key="ck_…")

cx.subject_says(
    agent_subject_id="user:ws:bot",
    subject_id="user:ws:cust",
    text="I want a refund.",
)

g = cx.check(subject_id="user:ws:bot")
if not g.allow:
    return refuse(g.reason)
```

Each SDK's README MUST show this example transliterated into the
language's idiomatic form.

---

## Appendix A — Subject and participant shapes

```json
{
  "subject_id": "user:ws_xxx:bot",
  "role":       "agent",
  "kind":       "agent",
  "metadata":   {}
}
```

`role` is free-form (`agent`, `customer`, `observer`, `detected_person`,
`counterparty`, `mentioned`, `system`, `service`, `other`, …).
`kind` describes the substrate (`agent`, `human`, `sensor`, `service`,
`other`).

Both default to `"other"` when omitted.

## Appendix B — Wire compatibility commitments

For a spec MINOR bump, server changes that constitute a breaking change:

- Removing or renaming a request field.
- Removing or renaming a response field that was previously documented
  as always-present.
- Changing an enum value.
- Tightening a previously permissive validation.

Server changes that do NOT constitute a breaking change:

- Adding a new optional response field.
- Adding a new enum value (SDKs MUST tolerate unknown values by exposing
  the raw string).
- Adding a new endpoint.
- Loosening a validation.
