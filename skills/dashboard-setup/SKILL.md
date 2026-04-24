---
name: dashboard-setup
description: Interactive onboarding for the Work Dashboard plugin. Walks the user through creating `~/.claude/dashboard-config.local`, initializes output directories, copies the static bundle, and prints per-MCP-server auth instructions. Run this once after installing the plugin; after that, use `/dashboard` to refresh.
---

# Work Dashboard — interactive setup

First-time setup wizard. Gathers the user's identity, team, OKRs, and pins; writes `~/.claude/dashboard-config.local`; copies the static HTML bundle into the user's chosen output directory; then prints per-MCP-server next steps.

**Design intent:** the user typing in this conversation is a NON-TECHNICAL human. Ask one clear question at a time when the answer branches. Batch obvious related fields into a single message when the user will paste multiple lines. Never dump a raw JSON template and say "fill it in."

## Step 0 — locate the plugin

The plugin root is where this SKILL.md lives minus `skills/dashboard-setup/`. Resolve it once:

```bash
PLUGIN_DIR="$(dirname "$(dirname "$(dirname "$0")")")"  # if invoked as a script
```

In practice, use the Claude Code plugin path. Common locations:
- `~/.claude/plugins/work-dashboard/`
- The path shown when the user ran `/plugin install`.

If unsure, ask the user to run `claude plugin info work-dashboard` and paste back the path. Store as `$PLUGIN_DIR`.

## Step 1 — check for existing config

Read `~/.claude/dashboard-config.local`. If it exists and is valid JSON, ask:

> *"You already have a dashboard config at `~/.claude/dashboard-config.local`. Want to:*
> *1. **Reset** — overwrite with a new setup (the existing file gets backed up to `dashboard-config.local.bak-YYYYMMDD`)*
> *2. **Edit** — I'll open the file and show you what's there so you can tweak specific fields*
> *3. **Cancel** — keep what you have"*

Act on the user's response:
- **Reset**: back up to `~/.claude/dashboard-config.local.bak-$(date +%Y%m%d-%H%M%S)`, then proceed to Step 2.
- **Edit**: display the current JSON, ask which section they want to change, Edit-tool the file, confirm, then skip to Step 8.
- **Cancel**: print "No changes made" and exit the skill.

If no file exists, proceed to Step 2.

## Step 2 — gather identity (one batched question)

Send a single message to the user:

> *"Let's set up your dashboard. Paste answers to these (or skip any with 'none' — you can always edit later):*
>
> *1. First name (what you're called on the dashboard, e.g. 'Alex'):*
> *2. Full name (for formal header, e.g. 'Alex Rivera'):*
> *3. Role title (e.g. 'Head of Marketing'):*
> *4. Work email:*
> *5. Timezone (IANA name, e.g. 'Europe/Madrid' · 'America/New_York' · 'Asia/Singapore'):*
> *6. Company name:*
> *7. Your manager's name + role (e.g. 'Chris Lin, VP Marketing' — or 'none' if you're the manager):*
> *"*

Parse the response. If any field is missing or malformed, ask a targeted follow-up for just that field.

Defaults: `workingHours` = `09:00–18:00 Mon–Fri` unless the user proactively mentions different hours.

## Step 3 — optional team roster

Ask:

> *"Want to add your team roster now? (They show up in the 'Your people' card on the dashboard.) Answer:*
> *- `yes` — I'll ask for their names one block at a time*
> *- `no` — leave it empty, you can add later*
> *- `attention-only` — I'll just ask who needs your attention this week (shown as a red-flagged note)"*

If `yes`, ask:

> *"Paste each teammate as one line: `Name | Role note | status (active or ooo)`. End with a blank line. Example:*
> *`Sam Chen | Sr. PM | active`*
> *`Dev Patel | PM · onboarding | active`*
> *`Priya K. | Designer · OOO till May 5 | ooo`"*

Parse lines into `team.people`.

Ask separately:

> *"In one sentence, who on your team needs the most attention this week? (This is shown at the top of the People card — use HTML `<b>name</b>` if you want to highlight specific people.)"*

Save as `team.attention`.

If user said `no`: set `team.people = []` and `team.attention = ""`.
If `attention-only`: only ask the attention question.

## Step 4 — optional OKRs

Ask:

> *"Want to add your top OKRs now? You can add up to 3 — each with a name, current % done, and trend (on-pace / behind / ahead). Or say 'skip' and add later.*
>
> *Paste each OKR as one line: `name | pct | trend`. Example:*
> *`Launch Q3 campaign | 40 | on-pace`"*

Parse into `dashboard.okrs`. If `skip`, set to empty array.

## Step 5 — pins (links on the right rail)

Ask:

> *"The dashboard shows 6 'pin' cards for quick access to your most-used tools. Defaults:*
> *1. Google Calendar (week view)*
> *2. Gmail inbox*
> *3. Slack workspace — what's your Slack workspace subdomain? (e.g. 'acme' for acme.slack.com, or 'none' to skip)*
> *4. Google Drive*
> *5. Granola (meeting notes)*
> *6. One custom pin of your choice — paste: `label | URL` or 'none' for no 6th pin*
>
> *Reply 'defaults' to accept pins 1-2, 4-5 as-is and only fill in 3 (Slack) + 6 (custom)."*

Build the `dashboard.pins` array. Each pin needs: `id`, `label`, `sub`, `letter` (one-char), `bg` (color var), `href`.

Use these bg colors rotating: `var(--teal-100)`, `var(--pink-100)`, `var(--red-100)`, `var(--blue-100)`, `var(--yellow-100)`, `var(--grey-100)`.

## Step 6 — output directories (just confirm defaults)

Ask:

> *"Last question: where should I put the dashboard files?*
>
> *Defaults (press enter / reply 'ok' to accept):*
> *- Static bundle: `~/Documents/Claude/work-dashboard/`*
> *- Agent cache: `~/.claude/dashboard-data/`*
>
> *Or give me your own absolute paths."*

## Step 7 — write config + create dirs + copy bundle

Once all fields are gathered, build the full config object. Schema (copy exactly — fields in this order):

```json
{
  "_README": "Private config for <firstName>'s work-dashboard plugin. Never bundled. Never committed.",
  "user": {
    "name": "...", "fullName": "...", "role": "...", "email": "...",
    "timezone": "...",
    "workingHours": { "start": "09:00", "end": "18:00", "days": ["Mon","Tue","Wed","Thu","Fri"] }
  },
  "org": {
    "company": "...",
    "manager": { "name": "...", "role": "..." },
    "seniorStakeholders": [],
    "team": { "attention": "...", "people": [...] }
  },
  "slack": { "workspace": "...", "userId": "", "highSignalChannels": [] },
  "dashboard": {
    "workstreams": [],
    "classificationKeywords": [],
    "okrs": [...],
    "pins": [...],
    "weather": { "city": "..." },
    "focusTarget": 4
  },
  "output": { "dashboardDir": "...", "dataCacheDir": "..." }
}
```

Actions:
1. Write `~/.claude/dashboard-config.local` with the JSON (pretty-printed, 2-space indent).
2. `mkdir -p` the `dashboardDir` and `dataCacheDir`.
3. Copy the plugin's static bundle: `cp -R "$PLUGIN_DIR/public/." "$dashboardDir/"`.
4. Create `~/.claude/dashboard-filters.local` if it doesn't exist with the content of `$PLUGIN_DIR/templates/dashboard-filters.local.example`.

## Step 8 — MCP server status

Print a per-server status table. The plugin bundles `.mcp.json` at its root so the 5 servers auto-register, but each needs per-user auth:

```
Your MCP servers (5 of them — all needed for the full dashboard):

  ┌──────────┬────────────────────────────────────────────────────────────┐
  │ Server   │ Next step                                                  │
  ├──────────┼────────────────────────────────────────────────────────────┤
  │ calendar │ Needs Google Cloud OAuth credentials.json.                 │
  │          │ Docs: https://github.com/nspady/google-calendar-mcp#setup  │
  │ gmail    │ Auto-OAuth. On first `/dashboard`, a browser tab opens.    │
  │ slack    │ Needs SLACK_BOT_TOKEN (xoxb-…) + SLACK_TEAM_ID.            │
  │          │ Create at: https://api.slack.com/apps                       │
  │ drive    │ Auto-OAuth. Same flow as Gmail, first run only.            │
  │ granola  │ Just needs the Granola desktop app running locally.        │
  └──────────┴────────────────────────────────────────────────────────────┘

If a server fails at dashboard refresh, its data section will show "source unavailable" — the rest of the dashboard still renders. You can set up MCPs incrementally.
```

If the user wants guided setup for a specific MCP, offer to open its README in the browser and paste a sample `.mcp.json` snippet inline.

## Step 9 — final message

Print exactly:

```
Setup complete.

Your config: ~/.claude/dashboard-config.local
Dashboard output: <dashboardDir>/Work Dashboard.html
Open it once now so the tab stays ready.

Run /dashboard anytime to refresh with live data from the 5 MCPs.
Edit ~/.claude/dashboard-config.local to update your team / OKRs / pins later.

— welcome to your Work Dashboard, <firstName>.
```

## Rules

- **One question at a time when there's a branch.** Batch only obvious related fields.
- **Never paste a raw JSON block at the user and ask them to edit it.** That defeats the point of this skill.
- **Always back up** an existing config before overwriting. Never silent-destroy user data.
- **Validate IANA timezone** against `Intl.DateTimeFormat().resolvedOptions().timeZone`-style names. If the user types a vague "CET" or "Pacific time", offer the canonical name (e.g. "Europe/Madrid", "America/Los_Angeles").
- **Timezone default**: if the user says "use my system timezone", run `date +%Z` for display and check `/etc/localtime` for the IANA name.
- **Don't call the 6 agents from this skill.** Setup only. The user explicitly runs `/dashboard` after.
- **If the user aborts mid-setup**, discard any partial state — don't write a half-filled config.
- **Currency / language**: the dashboard is English-only today; don't offer localization options.

## Why a separate skill

Keeps the refresh skill (`dashboard`) simple and repeatable — it just merges JSON. All the once-per-user UX lives here.
