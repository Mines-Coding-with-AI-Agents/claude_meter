# Agent SDK, Claude Code and subscription limit data

Research date: 2026-11-05.
Local Claude Code version observed: 2.1.318.
Confidence labels: documented (official docs), observed (seen on this machine), inferred (reasoned, not confirmed).

## 1. What the Agent SDK exposes

### Rate limit event (documented)

The TypeScript SDK reference defines a streamed message `SDKRateLimitEvent`.
Source: https://code.claude.com/docs/en/agent-sdk/typescript (section "SDKRateLimitEvent").
It is described as "Emitted when the session encounters a rate limit".
Shape:

```typescript
type SDKRateLimitEvent = {
  type: "rate_limit_event";
  rate_limit_info: {
    status: "allowed" | "allowed_warning" | "rejected";
    resetsAt?: number;
    utilization?: number;
    errorCode?: "credits_required";
    canUserPurchaseCredits?: boolean;
    hasChargeableSavedPaymentMethod?: boolean;
  };
  uuid: UUID;
  session_id: string;
};
```

The `status` field has a warning state (`allowed_warning`), which is the closest thing to a built-in "you are close" signal.
`resetsAt` and `utilization` are optional.
The docs do not say which limit window (5-hour or weekly) the event refers to, and there is no window-type field in the documented shape.
Confidence: documented for the shape, unknown for window semantics.
The `credits_required` fields need Claude Code v2.1.181 or later.

### Result message (documented)

`SDKResultMessage` (type `result`) carries `total_cost_usd`, `usage`, `modelUsage`, `duration_ms`, `num_turns`, `terminal_reason`, and so on.
Source: https://code.claude.com/docs/en/agent-sdk/typescript (section "SDKResultMessage").
`modelUsage` is a map of model name to `ModelUsage` with `inputTokens`, `outputTokens`, `cacheReadInputTokens`, `cacheCreationInputTokens`, `costUSD`, `contextWindow`.
The docs call `costUSD` a client-side estimate, not a billed amount.
`usage` covers the main agent loop only and excludes subagents, so `modelUsage` is preferred.
None of these fields express a share of a subscription limit.
Cost tracking page: https://code.claude.com/docs/en/agent-sdk/cost-tracking.md.

### Slash commands through the SDK (documented)

The SDK reference says that sending `/usage` or `/context` as a prompt returns its output as an `SDKAssistantMessage` (text).
Source: https://code.claude.com/docs/en/agent-sdk/typescript (section "SDKLocalCommandOutputMessage").
This is text meant for humans, not a stable schema.
Confidence: documented that it happens, inferred that it contains weekly figures.

### Limits of the SDK route

The event only fires when a session is actually running and making requests.
A passive tracker would have to run its own query just to learn the state, which spends the user's own quota.
Confidence: inferred.

## 2. Authenticating the SDK with a subscription

### Mechanics (documented)

Claude Code supports Pro, Max and Ultra login via `claude` / `/login`.
`claude setup-token` produces a one-year OAuth token that is exported as `CLAUDE_CODE_OAUTH_TOKEN`.
The token "authenticates with your Claude subscription", can only make model requests, and is not saved anywhere by the command.
Source: https://code.claude.com/docs/en/authentication (sections "Authentication precedence" and "Generate a long-lived token").
The Agent SDK spawns the Claude Code binary, so it inherits the same credential resolution.
The same page notes `apiKeyHelper`, `ANTHROPIC_API_KEY` and `ANTHROPIC_AUTH_TOKEN` apply to the SDK too, and that subscription OAuth is the default when none of those are set.
Credentials live in the macOS Keychain (with a `~/.claude/.credentials.json` fallback).
Source: https://code.claude.com/docs/en/authentication (section "Credential management").
Important: if `ANTHROPIC_API_KEY` is set in the environment, it wins over the subscription, so a tracker must unset it.

### Policy and ToS caveats (documented)

Source: https://code.claude.com/docs/en/legal-and-compliance (section "Authentication and credential use").
OAuth is "intended exclusively for purchasers of Claude Free, Pro, Max, Ultra, Team, and Enterprise subscription plans" and "ordinary use of Claude Code and other native Anthropic applications".
Developers building products on the Agent SDK "should use API key authentication".
Anthropic does not permit third-party developers to offer Claude.ai login in their apps or to route requests through plan credentials on behalf of their users.
Developers may not collect, store or intermediate Claude.ai credentials or session tokens.
The same page says advertised plan limits "assume ordinary, individual usage of Claude Code and the Agent SDK".
Reading for Claude Meter (inferred): a personal tool, run by the plan owner on their own machine, using the unmodified Claude Code binary, is the intended use case.
Anything that reads the OAuth token and replays it against Anthropic endpoints from custom code sits in a grey zone, and distributing such a tool to others would likely violate the "do not collect or intermediate credentials" rule.

## 3. What Claude Code exposes locally

### Status line JSON (documented, best source)

Claude Code pipes JSON to the configured status line command on stdin.
Source: https://code.claude.com/docs/en/statusline.
Fields relevant to limits:

| Field | Meaning |
| :- | :- |
| `rate_limits.five_hour.used_percentage` | 0 to 100, share of the 5-hour window used |
| `rate_limits.five_hour.resets_at` | Unix epoch seconds |
| `rate_limits.seven_day.used_percentage` | 0 to 100, share of the weekly window used |
| `rate_limits.seven_day.resets_at` | Unix epoch seconds |
| `rate_limits.spend_limit.*` | Only behind a Claude apps gateway |

Caveats stated in the docs:

- `rate_limits` appears only for claude.ai Pro, Max and Ultra subscribers.
- It appears only after the first API response in a session.
- Each window may be independently absent, and a window is dropped once its `resets_at` passes.

Other fields include `context_window.*`, model, cost and session data.
Observed: this machine already runs a custom `~/.claude/statusline.sh` that renders `5H:<pct>% W:<pct>%`, which confirms the fields are present in practice on this account.
Confidence: documented and observed.

Design consequence: a status line script (or a wrapper that tees the stdin JSON to a file) can persist weekly utilization on every refresh, with no network call and no credential access.

### /usage and /status (documented)

`/usage` shows plan usage in the interactive UI, and `/status` shows login method, organization, email and whether the login is expired.
Source: https://code.claude.com/docs/en/authentication (section "Renew an expiring login") for `/status`.
The `/usage` output has no documented machine-readable form.
Confidence: documented for existence, unstable as a parsing target.

### Hooks (inferred)

Hook inputs carry session id, transcript path, cwd and event name.
I did not verify any rate limit field in hook payloads.
Confidence: inferred (not confirmed in this pass).
Hooks docs: https://code.claude.com/docs/en/hooks.

### OpenTelemetry (documented)

Metrics include `claude_code.token.usage` (split by type: input, output, cacheRead, cacheCreation, and model) and `claude_code.cost.usage` (USD).
Source: https://docs.anthropic.com/en/docs/claude-code/monitoring-usage (search result summary; the page was not fetched in full).
Cost is computed client-side and is approximate, and OTel support is labelled beta.
These are consumption counters, not limit utilization, so they cannot give "percent of weekly cap".
Confidence: documented, though not re-verified on the live page.

### Local transcripts (observed)

Files: `~/.claude/projects/<project-slug>/<session>.jsonl`.
Each assistant line has `message.usage` with `input_tokens`, `cache_creation_input_tokens`, `cache_read_input_tokens`, `output_tokens`, `service_tier`, `speed`, plus `message.model`, `timestamp`, `requestId`, `sessionId`.
Observed on a real transcript line with Claude Code 2.1.318.
I grepped for `rate_limit` in transcripts and found nothing usable, so no limit percentages are stored there.
Also present: `~/.claude/stats-cache.json` (aggregate stats, contents not inspected).
Transcripts allow computing token burn rate (useful for a projection) but not the denominator, because Anthropic does not publish the plan's token cap.
Confidence: observed.

## 4. Undocumented endpoints

### OAuth usage endpoint (undocumented, unstable)

Community tools (ccusage packaging notes and gists) report that Claude Code's own client calls `GET https://api.anthropic.com/api/oauth/usage` with `Authorization: Bearer <oauth access token>` and header `anthropic-beta: oauth-2025-04-20`.
Reported response: `five_hour` and `seven_day` objects with `utilization` (percent) and `resets_at` (timestamp), plus per-model windows such as `seven_day_sonnet` and an `extra_usage` object.
Related endpoints reported: `/api/oauth/profile`, `/api/oauth/account/settings`.
Sources (third party, not Anthropic): https://pypi.org/project/ccusage/0.2.0/ and https://gist.github.com/monperrus/3ac4b303a84946bbeaf2b1123ee99491.
One source says the flat `seven_day_*` keys may now be null with data moved to a structured limits array, and another gist (June 2026) claims the weekly reset behaves differently than a plain 7 days.
Both claims are single-source and unverified.
Anthropic does not document this endpoint.
I did not call it, since that requires reading the OAuth token from the Keychain, which this task forbade.
Confidence: undocumented, third-party reports only, schema may have drifted.
Using it also means a custom program handles the OAuth access token, which runs against the spirit of the credential policy in section 2.

## What this means for Claude Meter

Verdict: yes, weekly limit data is obtainable without an API key, and the best method needs no credential handling at all.

Recommended primary method: capture the status line JSON.

- Install a status line command (or wrap the existing `~/.claude/statusline.sh`) that appends `rate_limits.seven_day.used_percentage` and `.resets_at` (and the 5-hour pair) with a timestamp to a local file such as a JSONL or SQLite store.
- This is documented, stable, uses the subscription login the user already has, calls no network endpoint, and never touches tokens.
- Weakness: data only refreshes while Claude Code is running and has completed at least one API response, so the tracker shows the last known value and its age.
- The meter then computes burn rate from the stored series (and optionally from transcript token counts) and warns when projected usage would hit 100% before `resets_at`.

Secondary signal: SDK `rate_limit_event` with `status: "allowed_warning"` is useful if the meter ever runs its own SDK sessions, but window type is undocumented and it requires spending quota, so it is not a good polling source.

Not recommended as primary: the undocumented `/api/oauth/usage` endpoint.
It would allow polling while Claude Code is idle, but it is unstable, unofficial, and requires reading and replaying the OAuth token.
If ever used, treat it as an optional, clearly flagged fallback, personal use only.

Not sufficient on their own: transcripts, OTel and `modelUsage` give consumption but not the plan cap, so they cannot produce a percent of weekly limit.

Open items to verify next: exact hook payload fields, whether `/usage` text via SDK includes weekly figures, and the update cadence of the status line `rate_limits` values.
