# Fellow → Linear Claude Routine

A Claude Code Routine that pulls action items out of Fellow meeting notes and files them as Linear issues. This doc walks through the one-time setup on Claude's side, the connectors the routine relies on, and how to trigger it.

## How it works

```
Trigger (API POST, schedule, or manual)
        │
        ▼
Claude Code Routine
        │
        ├── Fellow.ai MCP  ── fetch meeting(s) + action items
        │
        └── Linear MCP     ── create/update issues, assign, label
```

The routine runs on Anthropic's cloud, so neither Claude Desktop nor your laptop needs to be awake for it to fire.

## Prerequisites

| Requirement | Notes |
|---|---|
| Claude plan with Routines enabled | Pro (5 runs/day), Max (15/day), or Team/Enterprise (25/day). Routines are in research preview. |
| Fellow.ai connector | Already configured in this workspace. Provides `get_action_items`, `get_meeting_summary`, `search_meetings`, `get_meeting_transcript`, etc. |
| Linear connector | Already configured. Provides `save_issue`, `list_teams`, `list_issue_labels`, `list_users`, etc. |
| Admin access to the Linear team(s) you want to write to | Required to resolve team IDs, label IDs, and assignee IDs. |

## Setup

### 1. Create the routine

Go to **claude.ai/code/routines → New routine**.

Give it a name like `fellow-to-linear` and paste in a prompt along these lines:

```
You are an engineering architect updating Linear Issues with details from meeting conversation via Fellow. 

You will receive text as input via the API trigger with a linear ticket number which is structured like ZIP-### where `ZIP-` is a static prefix and `###` represents the Issue number. e.g. ZIP-123

You will also receive a Fellow meeting link. e.g. https://fellow.link/KT8Z7C01XRPB

The input text will look something like this ... 

`Update Linear Issue ZIP-123 with the context from Fellow https://fellow.link/KT8Z7C01XRPB`

When this prompt is received, check for a document attached (Linear native) on the Issue named FELLOW_SPEC. If you have this file, you will be updating it. If you do not have this file, create it. Use the full context of the Linear Issue and all available resources within it to enrich the update and ensure context awareness.

Update or Create the FELLOW_SPEC markdown file on the linear Issue to produce an architecture spec which will be passed to engineering for development at a later date. 
```

Tune the filters, team routing, and label rules to match your workflow.

### 2. Attach the connectors

In the routine's **Tools** panel, enable:

- **Fellow.ai** — needed for `get_action_items`, `search_meetings`, `get_meeting_summary`
- **Linear** — needed for `list_teams`, `list_users`, `list_issue_labels`, `save_issue`, `list_issues`
- **Slack** (optional) — if you want the summary posted to a channel

### 3. Add an API trigger

In the routine's **Triggers** panel, click **Add trigger → API**. Claude will generate:

- A **routine ID** of the form `trig_01...`
- A **bearer token** — shown exactly once. Copy it into a secret manager (AWS Secrets Manager, 1Password, Parameter Store).

The full fire URL is:

```
https://api.anthropic.com/v1/claude_code/routines/<routine_id>/fire
```

### 4. (Optional) Add a schedule trigger

If you'd rather run nightly instead of on-demand, add a **Schedule** trigger on the same routine — e.g., daily at 7 AM America/Los_Angeles. Schedule and API triggers coexist on one routine.

## Triggering the routine

### Minimal curl

```bash
curl -X POST https://api.anthropic.com/v1/claude_code/routines/trig_01ABCDEFGHJKLMNOPQRSTUVW/fire \
  -H "Authorization: Bearer $ROUTINE_TOKEN" \
  -H "anthropic-version: 2023-06-01" \
  -H "anthropic-beta: experimental-cc-routine-2026-04-01" \
  -H "Content-Type: application/json" \
  -d '{"text": "Update Linear Issue ZIP-123 with the context from Fellow https://fellow.link/KT8Z7C01XRPB"}'
```

All three headers are required. The `text` field is freeform — it becomes additional context the routine's prompt sees at runtime.

### Successful response

```json
{
  "type": "routine_fire",
  "claude_code_session_id": "session_01HJKLMNOPQRSTUVWXYZ",
  "claude_code_session_url": "https://claude.ai/code/session_01HJKLMNOPQRSTUVWXYZ"
}
```

Open `claude_code_session_url` to watch the run, inspect tool calls, or continue the conversation manually.

## Environment variables

Store these wherever your trigger code runs (Lambda, Fargate, GitHub Actions, laptop, etc.):

| Variable | Value |
|---|---|
| `ROUTINE_URL` | `https://api.anthropic.com/v1/claude_code/routines/<routine_id>/fire` |
| `ROUTINE_TOKEN` | The bearer token shown once when you created the API trigger |

## Monitoring and debugging

- **Run history** — every fire shows up at claude.ai/code/routines under the routine, with the session URL, timestamp, and final status.
- **Quota** — check the **Usage** tab on the routines page. If you hit the daily cap, firing returns an error and the routine does not run.
- **Idempotency** — the `/fire` endpoint has no idempotency key. If your trigger retries on a 5xx, you'll get duplicate sessions. Either dedupe upstream (by meeting ID + a short TTL) or rely on the routine's own "skip if matching title exists" logic.

## Common issues

| Symptom | Likely cause | Fix |
|---|---|---|
| `401 Unauthorized` on fire | Missing or stale bearer token | Re-issue the token in the API trigger panel and update the secret |
| `400 Bad Request` | Missing `anthropic-beta` or `anthropic-version` header | Include both headers on every call |
| Routine runs but skips everything | Fellow.ai connector not scoped to the channel containing the action items | Re-auth the connector or widen its channel access |
| Linear issues land in the wrong team | Team routing rules in the prompt are ambiguous | Tighten the routing block or hardcode a default team |
| Duplicate Linear issues | Upstream retry hit `/fire` twice | Add dedupe logic in the trigger caller; the routine's title-match check catches most dupes but is not a hard guarantee |

## References

- Claude Code Routines docs: https://code.claude.com/docs/en/routines
- Fellow.ai MCP tools: `get_action_items`, `get_meeting_summary`, `search_meetings`, `get_meeting_transcript`, `get_meeting_participants`, `list_channels`, `get_channel_details`
- Linear MCP tools: `save_issue`, `list_teams`, `list_issue_labels`, `list_users`, `list_issues`, `save_comment`
