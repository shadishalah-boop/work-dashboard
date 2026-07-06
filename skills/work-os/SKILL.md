---
name: work-os
description: Your live "Work OS" — assembles your current operating picture (Top-3, tasks due/overdue, blockers, decisions pending, key metrics, recent wins, inbox asks, next meeting) ON DEMAND from cloud connectors (Granola, Gmail, Slack, Calendar, and — if configured — Looker/Snowflake). Connector-only and self-contained, so it runs anywhere Claude has your connectors, INCLUDING a Claude session launched from Slack. Nothing personal is hardcoded — identity comes from your connectors (and your local dashboard config, if present). Use when the user says "my work os", "work os", "what's on my plate", "my dashboard summary", or "/work-os".
---

# Work OS (live, connector-only)

Assemble the user's current work operating picture **live from cloud connectors** and reply with a concise digest. **No pre-computed cache, no dependence on a running dashboard** — everything is fetched fresh at call time, so this works identically in a laptop Claude Code session and in a Slack-launched cloud (Cowork) session.

This is the companion to the `dashboard` skill: `dashboard` renders the full local React view; `work-os` is the one-message glance you can get anywhere, including from Slack where the laptop and its files don't exist.

## Step 0 — resolve identity (never hardcode a person)

Establish **who "the user" is** and how to classify their work, in this order:

1. **Local config, if reachable.** Try to read `~/.claude/dashboard-config.local` (JSON). If it exists, use it as the source of truth: `user.name/role/timezone`, `org.manager`, `org.seniorStakeholders`, `slack.highSignalChannels`, `mcp.*` server names, and any `projects`/OKR fields for classifying items. This file lives only on the laptop — in a Slack/Cowork session it will be **absent**, which is expected, not an error.
2. **Otherwise, derive identity live from connectors.** With no config file (the normal Slack/Cowork case): resolve the current user from the authenticated connectors — the Slack self-profile (`slack_read_user_profile` / a `from:me` search), and the Gmail account owner (the address that appears as "me" in `to:me` results). Take name, email, and — if exposed — role/title/timezone from there. Do **not** assume a name; use what the connectors return.
3. **Stakeholders & projects.** If config lists a manager, senior stakeholders, or projects/OKRs, use them for ranking and classification. If not, infer the recurring themes (initiatives/projects) from the meeting and email clustering below, and treat the most senior/most-frequent counterparties as the priority stakeholders — don't invent an org chart.

If you genuinely cannot resolve who the user is from either path, say so briefly and ask, rather than guessing.

## Connector resolution (names vary by environment)

For each source, resolve the tool by trying, in order: the config's `mcp.<source>` name, then `mcp__<Server>__<tool>`, then the **`mcp__claude_ai_<Server>__<tool>`** prefixed variant (claude.ai-managed connectors use this), then lowercase legacy. If none resolve, call **`ToolSearch`** (e.g. `query: "granola list meetings"`) and use what it surfaces. If a source is genuinely unavailable (401/403/timeout), **skip it, note it in the output as "(— source unavailable)", and never fabricate** — a missing source degrades the digest, it doesn't fail it.

## What to fetch (parallelize where possible)

1. **Granola — last 7 days of meetings** → the backbone of Top-3, blockers, decisions, and tasks.
   - `list_meetings` (titles/dates/participants/ids only — cheap), then `get_meetings(ids=…)` for the notes of meetings in the last 7 days. (`list_meetings` returns titles only — you MUST follow up with `get_meetings` to read content.)
   - Extract: **action items assigned to / owned by the user** (tasks), **commitments with due dates** (overdue if past, dueSoon if near), **things blocking the user or blocked on others** (blockers), **open decisions the user owes or is driving** (decisions).
2. **Gmail — actionable threads** → inbox asks + Gmail-sourced decisions/tasks.
   - Search `(is:unread OR is:important OR to:me) newer_than:7d -category:promotions -category:social`; read the threads where the user is directly addressed or has an explicit ask awaiting their reply, or where they've waited >2 days on someone. Prioritize threads from the manager and senior stakeholders (from Step 0).
3. **Slack — recent activity** → blockers + today's shipped wins.
   - Search `to:me after:<date-7d-ago>` (mentions/DMs), `from:me after:<yesterday>` (today's shipped), and `incident after:<date-7d-ago>` (filter to the user's `highSignalChannels`, else `#incident*`/`#incidents*`, for live blockers). Dates must be absolute `after:YYYY-MM-DD`.
4. **Calendar — next meeting** → `list_events` for today; the next event after "now" in the user's timezone.
5. **Metrics (optional, config-driven)** → **only if the user has configured metrics.** Read metric definitions from `~/.claude/dashboard-metrics.local.json`, or the config's `metrics.items`. For each, fetch the current value + a prior-period value from its declared source (`snowflake` or `looker`) — for Snowflake "nl" metrics, discover the schema and write read-only SQL yourself; run nothing that mutates data. **If no metrics are configured, or the warehouse connector isn't available, omit the Metrics section entirely — never guess a number and never hardcode a metric.**

## Assemble & reply (concise, Slack-friendly markdown)

Merge/dedupe across sources (an item in both a meeting and an email is one item). Rank Top-3 by: explicit ask from the manager > time-critical/expiring > senior-stakeholder-flagged. Drop empty sections. Target < 3500 chars.

```
🗂️ *Work OS — {weekday, Mon D}*  (live)

🎯 *Top 3*
1. {item} — _{source · why}_
2. …
3. …

⏰ *Due* — {N} overdue · {M} due soon
• *Overdue:* {item}
• {top due-soon items}

🚧 *Blockers*
• {blocker} — _{who/why}_

🧭 *Decisions pending*
• {decision} — _{who}_

📈 *Metrics*   _(only if configured)_
• {label}: {value} {▲/▼ vs prior}

✅ *Recently shipped*
• {win}

📥 *Needs reply*
• {from} — {ask}

📅 *Next:* {title · when}
```

## Notes
- This is the **live** Work OS. It has no access to the laptop dashboard's hand-promoted Top-3, so it **infers** Top-3 from meetings/email instead. That's the intended trade for one-step, no-mirror access from anywhere (including Slack).
- **Read-only across all sources. It never writes anything** — no Snowflake DML/DDL, no emails, no Slack posts, no calendar edits.
- If the user asks for a subset ("just blockers and decisions"), fetch only what those need and reply with just those sections.
- Keep it a glance, not a report. Offer more depth only if asked.
