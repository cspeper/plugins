---
name: "sentry"
description: "Use when the user asks to inspect Sentry issues or events, summarize recent production errors, or pull basic Sentry health data. Prefer connected Sentry MCP tools; use the bundled read-only API script with `SENTRY_AUTH_TOKEN` only as a fallback."
---


# Sentry (Read-only Observability)

## Routing and authentication

1. Check the active tool catalog for Sentry provider tools before inspecting local credentials or running the bundled script.
   - When Sentry tools are available, use them directly. Do not check `SENTRY_AUTH_TOKEN` first.
   - If connection status is uncertain, make the narrowest relevant read-only call. For example, find organizations when an organization slug is not known.
   - Treat the provider tool's authentication or connection result as authoritative. A missing local environment variable says nothing about the connected plugin.
2. Use the bundled script only when Sentry provider tools are unavailable, the provider route cannot authenticate, or the user explicitly requests the local script/API path.
   - Set `SENTRY_AUTH_TOKEN` as an environment variable with read-only scopes such as `project:read`, `event:read`, and `org:read`.
   - Optional defaults: `SENTRY_ORG`, `SENTRY_PROJECT`, `SENTRY_BASE_URL`.
   - Never ask the user to paste the full token in chat. Ask them to set it locally and confirm when ready.
3. Always query Sentry directly. Do not substitute heuristics or cached results.

If the fallback is required and the token is missing, give the user these steps:

1. Create a Sentry auth token: https://sentry.io/settings/account/api/auth-tokens/
2. Create a token with read-only scopes such as `project:read`, `event:read`, and `org:read`.
3. Set `SENTRY_AUTH_TOKEN` as an environment variable in their system.
4. Offer to guide them through setting the environment variable for their OS/shell if needed.

Do not say that Sentry is unavailable, unconfigured, or unauthenticated based only on a missing local token.

## Core tasks (connected tools)

- Use the Sentry issue-search tool for grouped issue lists.
- Use the Sentry event-search tool for individual events, logs, counts, aggregations, or time series.
- Use the Sentry resource tool for a specific issue, event, trace, replay, monitor, or other supported Sentry URL.
- Use the organization and project discovery tools only when the required slug is not already known.
- When no direct tool fits, search the Sentry tool catalog before deciding that the operation is unavailable.
- Keep this workflow read-only. Do not use mutation tools.

## Fallback: bundled script

Use `scripts/sentry_api.py` for deterministic API calls when the connected-tool route is unavailable. It handles pagination and retries once on transient errors.

### Bundled script path

```bash
export SENTRY_API="plugins/sentry/skills/sentry/scripts/sentry_api.py"
```

If you are running from an installed plugin copy instead of this repo checkout, use the same
`skills/sentry/scripts/sentry_api.py` path inside the installed plugin directory.

### 1) List issues (ordered by most recent)

```bash
python3 "$SENTRY_API" \
  --org {your-org} \
  --project {your-project} \
  list-issues \
  --environment prod \
  --time-range 24h \
  --limit 20 \
  --query "is:unresolved"
```

### 2) Resolve an issue short ID to issue ID

```bash
python3 "$SENTRY_API" \
  --org {your-org} \
  --project {your-project} \
  list-issues \
  --query "ABC-123" \
  --limit 1
```

Use the returned `id` for issue detail or events.

### 3) Issue detail

```bash
python3 "$SENTRY_API" \
  --org {your-org} \
  issue-detail \
  1234567890
```

### 4) Issue events

```bash
python3 "$SENTRY_API" \
  --org {your-org} \
  issue-events \
  1234567890 \
  --environment prod \
  --time-range 24h \
  --limit 20
```

### 5) Event detail (no stack traces by default)

```bash
python3 "$SENTRY_API" \
  --org {your-org} \
  --project {your-project} \
  event-detail \
  abcdef1234567890
```

## Fallback API requirements

Always use these endpoints (GET only):

- List issues: `/api/0/projects/{org_slug}/{project_slug}/issues/`
- Issue detail: `/api/0/organizations/{org_slug}/issues/{issue_id}/`
- Events for issue: `/api/0/organizations/{org_slug}/issues/{issue_id}/events/`
- Event detail: `/api/0/projects/{org_slug}/{project_slug}/events/{event_id}/`

## Inputs and defaults

- `org_slug`: default to `{your-org}` (required for issue detail, issue events, and event detail).
- `project_slug`: default to `{your-project}` (required for list issues and event detail).
- `time_range`: default `24h` (pass as `statsPeriod` for list issues and issue events).
- `environment`: default `prod` (used by list issues and issue events).
- `limit`: default 20, max 50 (paginate until limit reached).
- `search_query`: optional `query` parameter.
- `issue_short_id`: resolve via list-issues query first.

## Output formatting rules

- Issue list: show title, short_id, status, first_seen, last_seen, count, environments, top_tags; order by most recent.
- Event detail: include culprit, timestamp, environment, release, url.
- If no results, state explicitly.
- Redact PII in output (emails, IPs). Do not print raw stack traces.
- Never echo auth tokens.

## Golden test inputs

- Org: `{your-org}`
- Project: `{your-project}`
- Issue short ID: `{ABC-123}`

Example prompt: “List the top 10 open issues for prod in the last 24h.”
Expected: ordered list with titles, short IDs, counts, last seen.

Routing expectations:

- Connected Sentry tools plus no `SENTRY_AUTH_TOKEN`: use the connected tools without mentioning the token.
- No connected Sentry tools plus a configured token: use the bundled script.
- Neither route available: explain that connected tools are unavailable, then provide the local-token setup steps.
