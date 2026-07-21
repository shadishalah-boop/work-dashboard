---
name: dashboard
description: Refresh the Work Dashboard with live data. Fetches calendar, granola, gmail, slack, drive, wellness (and, when configured, custom Looker/Snowflake metrics) in THIS interactive session — because claude.ai-managed MCP connectors aren't available to a headless subprocess — then merges into data-override.jsx + drive-index.jsx. Invoke when the user says "/dashboard", "refresh dashboard", "update dashboard", or "pull fresh data".
---

# Work Dashboard — refresh skill

Orchestrator. Runs entirely in the **interactive session** (claude.ai-managed
connectors only exist here): a prep call, an in-session fan-out to the data agents +
inline Slack, then a merge into `data-override.jsx` + `drive-index.jsx` for the local
React-in-browser dashboard. All per-user identity and paths come from
`~/.claude/dashboard-config.local` (created by the `dashboard-setup` skill).

## Architecture (v0.19 — direct-tool inline, no sub-agents, no ad-hoc shell)

```
(interactive session — connectors live here, and ONLY here)
  bash prep.sh            → dates/window/cache + pre-delete + MCP server names
  ONE parallel tool block, all in THIS session:
    ├─ mcp__…__list_events        → Write calendar.json + calendar-week.json
    ├─ mcp__…__search_threads(+get_thread) → Write gmail.json
    ├─ mcp__…__list_meetings(+get_meetings) → Write granola.json
    ├─ mcp__…__list_recent_files  → Write drive-raw.json
    └─ 4× slack_search_public_and_private   → Write slack.json
  (wellness: Read calendar-week.json → Write wellness.json — no MCP)
  python3 drive-transform.py ; python3 build-overrides.py
        └─ data-override.jsx + drive-index.jsx  (+ cache-bust the HTML)
```

> ## ⛔ PERMISSIONLESS & FAST — the two hard rules (read before anything else)
>
> A prior build spawned sub-agents (which CAN'T see the claude.ai connectors here, so
> they fail and get redone) and improvised shell to fake the work. Both are banned.
>
> **RULE 1 — the ONLY things this skill may run in Bash are the three bundled scripts:**
> `prep.sh`, `slack-prep.sh`, and `python3 …/drive-transform.py` / `python3 …/build-overrides.py`.
> **NEVER** emit a heredoc (`<<EOF`), `cat >`/`cat <<`, `python3 -c`, `echo >`, a `bash`
> function definition, `chmod`, or any `/tmp/*.sh` scratch file. Those are flagged as
> un-analyzable/obfuscated and can NEVER be allowlisted, so every one forces a manual
> prompt — the exact thing we're eliminating. All dates/paths already come out of
> `prep.sh`; you never need to compute them in a shell. **All JSON files are written with
> the `Write` tool** (allowlisted for `~/.claude/dashboard-data/**`), never via shell redirect.
>
> **RULE 2 — fetch every source INLINE via direct MCP tool calls. Do NOT use the
> `Agent`/sub-agent tool.** In this environment the connectors are visible only to the
> main session; a sub-agent finds no tool, fails, and doubles the work. You, the main
> assistant, call the MCP tools directly. This is both permissionless (the read-only
> connector tools are allowlisted) and the fast path (one parallel round-trip, ~15–30s).
>
> Everything the refresh touches — MCP read tools, the Write tool, `prep.sh`, `python3`
> — is already in `~/.claude/settings.json`. If you follow both rules, a refresh runs
> start-to-finish with **zero prompts** and completes in **under a minute**.

> The old headless `refresh-headless.sh` / `headless-prompt.md` path and the
> `wait-and-merge.sh` poller are **deprecated** — the poller waited up to 5 min for
> files that inline writing has already put on disk. Merge with the two direct
> `python3` calls in Step 4 instead.

## How to refresh — runs in THIS Claude Code session

> **Why in-session?** claude.ai-managed MCP connectors (named like
> `claude_ai_Google_Calendar`, `claude_ai_Slack`) live in the Claude Code session
> context, not in a raw `claude -p` subprocess spawned by launchd/cron. So the refresh
> runs here, in the session. **To automate it**, a *Claude Code scheduled task* that
> runs `/work-os:dashboard` works great — it executes in an authenticated session, so
> the connectors are available (run `allowlist.sh` first so it doesn't stop on a
> prompt). The deprecated `refresh-headless.sh` (raw `claude -p`) is the thing that may
> not carry the connectors — prefer the in-session flow below / a scheduled task.
>
> **First-run prompts:** the first refresh asks you to approve each connector tool
> once — choose **"don't ask again"** and subsequent refreshes are prompt-free. To
> skip the click-through entirely, run
> `bash ${CLAUDE_PLUGIN_ROOT}/skills/dashboard/allowlist.sh` once (it writes the
> read-only allow-rules to `~/.claude/settings.json`; effective next session).

### Step 1 — prep (one bash call)

```
Bash(command: "bash ${CLAUDE_PLUGIN_ROOT}/skills/dashboard/prep.sh", description: "Dashboard prep")
```

Capture `TODAY`, `TOMORROW`, `NOW`, `WINDOW_DAYS`, `SINCE_*`, `START_TS`, `TZNAME`,
`DATA_DIR`, `DASH_DIR`, `RUN_AGENTS`, the per-source server names `MCP_CALENDAR`
/ `MCP_GMAIL` / `MCP_DRIVE` / `MCP_GRANOLA` (these come from config and are typically
the `claude_ai_`-prefixed names), `MCP_ZOOM` (optional Zoom notes source merged into the
granola agent), plus `MCP_LOOKER` / `MCP_SNOWFLAKE` / `HAS_METRICS` / `METRICS_DEFS` for
the custom Metrics card. prep.sh is plain bash (no MCP) and allowlistable.

### Step 2 — refresh Slack YOURSELF, in this session (do NOT spawn a sub-agent)

Slack must be fetched by **you, the main interactive assistant** — NOT via the
`Agent`/sub-agent tool. Two reasons, both confirmed in the field:
- **Sub-agents are sandboxed** to the bare `mcp__Slack__*` tool names and CANNOT
  reach this session's managed connector, which is often exposed under a prefix like
  `mcp__claude_ai_Slack__…`. A spawned `dashboard-slack` agent therefore finds no
  Slack tool and fails. The MAIN session CAN reach the prefixed connector.
- Slack's `slack_search_public_and_private` needs **user consent**, which only an
  interactive session can grant (you may see a one-time consent prompt — expected).

Do this yourself, inline:

1. Get paths/dates: `Bash(command: "bash ${CLAUDE_PLUGIN_ROOT}/skills/dashboard/slack-prep.sh", description: "Slack prep")`.
   Capture `DATA_DIR`, `SINCE_WINDOW`, `SINCE_1D`, `SINCE_30D`, `MCP_SLACK`, `WORKSPACE`, `TZNAME`.
2. **Resolve the Slack search tool in THIS session**, in order:
   `mcp__<MCP_SLACK>__slack_search_public_and_private` (MCP_SLACK from config may be
   `claude_ai_Slack`) → `mcp__claude_ai_Slack__slack_search_public_and_private` →
   `mcp__Slack__slack_search_public_and_private` → else `ToolSearch` with
   `query: "slack search messages"`. Use whatever resolves.
3. Run the 4 searches (absolute dates). **Keep responses small**: pass `limit: 20`
   and `include_context: false` on every search. **Do NOT pass `response_format:
   "concise"`** — the concise format strips permalinks (the schema's required
   `permalink` field), forcing a second round of detailed re-fetches that costs more
   than the original "savings". The default detailed format already fits comfortably
   under the response cap at `limit: 20`.
   `to:me after:<SINCE_WINDOW>` · `from:me after:<SINCE_1D>` (questions = those
   containing `?`) · `from:me after:<SINCE_1D>` (shipped) · `incident after:<SINCE_WINDOW>`.
   **Retry-once-then-fast-fail:** if the FIRST search hits a *transient* network error
   (5xx/connection reset/429) retry it once; if it hits a *deterministic* failure (auth/
   permission/timeout) or the retry also fails, the connector is down — do NOT run the other
   three, Write `slack.json` with `sourceOk:false`, and move on (step 5).
3b. **Resolve the user's Slack avatar + workspace.** **Cached for 30 days** (v0.14.4)
   — avatars and workspace slugs almost never change, so we skip the
   `slack_search_users` call on most refreshes.

   - **If `SLACK_PROFILE_FRESH=yes`** (the orchestrator's prep flag from
     `SLACK_PROFILE_FILE` mtime < 30 days):
       1. **Read** `<SLACK_PROFILE_FILE>` — a JSON `{ "userAvatar": "...", "workspace": "..." }`.
       2. Use those values directly. **Skip the `slack_search_users` call entirely.**
       3. Move on to step 4. (Saves ~2–3k tokens per refresh and one tool call.)

   - **If `SLACK_PROFILE_FRESH=no`** (no file yet, or > 30 days old):
       1. Call `mcp__<MCP_SLACK>__slack_search_users` with the config `user.email`
          (then `user.name`). This reliably returns the full profile with image URLs.
          (`slack_read_user_profile` often omits image fields — fall back to it only
          if step 1 returned no image.)
       2. From whatever it returns, pull the FIRST present of these image fields,
          checking **both** the top level and a nested `profile` object: `image_512`
          → `image_192` → `image_72` → `image_1024` → `image_original` → `image_48`.
          Accept any `https://…` value.
       3. Derive `workspace` from the result's permalink hostname (e.g.
          `preply.slack.com` → `"preply"`), the config (`slack.workspace`), or
          `"slack"` as last resort.
       4. **Write `<SLACK_PROFILE_FILE>`** with `{ "userAvatar": "...", "workspace":
          "...", "generatedAt": "..." }` so future refreshes hit the cache.
       5. Use those values for the rest of this run. If you truly couldn't find any
          image after both tools, set `"userAvatar": ""` — the tab falls back to the
          default icon (but **still** write the profile file so we don't retry every
          refresh; only the next ≥30-day cycle will re-try).

   **To force a re-fetch** (e.g. you changed your Slack photo): delete
   `~/.claude/dashboard-slack-profile.json` and run `/dashboard` — the next refresh
   sees no file, fetches fresh, and writes it.
4. Build `slack.json` following the schema + scope/classification rules in
   `${CLAUDE_PLUGIN_ROOT}/agents/dashboard-slack.md` (Read it for the exact schema —
   apply the scope filter: DMs + channels you posted in + `#incident-*`; include the
   `userAvatar` from 3b). **Write** it to `<DATA_DIR>/slack.json`.
5. If no Slack tool resolves at all, Write `slack.json` with `"sourceOk": false` and
   continue — the rest of the dashboard renders fine. Never block the refresh on Slack.

> The headless/button refresh (`refresh-headless.sh`) fetches Slack too — under
> `--permission-mode bypassPermissions` (see `headless-prompt.md` STEP 1b). It's
> time-boxed: if the Slack call stalls it's skipped and the last good `slack.json` is
> kept, so Slack never blocks the run. The whole refresh is capped by `serve.py`
> (`REFRESH_TIMEOUT`), so a stuck refresh always resolves and reports a result.

### Step 3 — fetch every remaining source INLINE, in ONE parallel tool block

Do NOT spawn sub-agents (see RULE 2). You call the MCP tools yourself. For each source
in `RUN_AGENTS` except wellness, resolve the tool as `mcp__<MCP_*>__<tool>` (the server
names came from prep; fall back to `mcp__claude_ai_<Server>__…` then `ToolSearch`), and
issue **all of these calls together in a single tool-use block** so they run
concurrently. When a source's data comes back, **Write** its JSON with the Write tool
(never a shell redirect). If a source needs a follow-up call (gmail `get_thread`,
granola `get_meetings`), make those in the next block, then Write. The per-source specs
(schema, incremental rules, caps) live in `${CLAUDE_PLUGIN_ROOT}/agents/dashboard-<name>.md`
— Read the one you're unsure about; do NOT paste its logic into a shell script.

Fetch calls to issue in the parallel block (substitute the captured prep values):

- **calendar** → `mcp__<MCP_CALENDAR>__list_events` for the whole work-week in ONE call (`WEEK_START` 00:00 → `WEEK_END` 00:00, `timeZone=<TZNAME>`). Then Write **both** `<DATA_DIR>/calendar.json` (today's events) and `<DATA_DIR>/calendar-week.json` (the full week).
- **gmail** → `mcp__<MCP_GMAIL>__search_threads` with `after:<SINCE_WINDOW>`; for the actionable hits, `get_thread`; merge into the existing `gmail.json` (dedupe by thread id, drop >14d). Write `<DATA_DIR>/gmail.json`.
- **granola** → `mcp__<MCP_GRANOLA>__list_meetings`; if any meeting started after `<SINCE_ISO>`, `get_meetings` for those new IDs only and merge; else re-Write the existing file unchanged. (Zoom via `<MCP_ZOOM>` optional — skip silently if unresolved.) Write `<DATA_DIR>/granola.json`.
- **drive** → `mcp__<MCP_DRIVE>__list_recent_files` (last 14 days). Write the raw response to `<DATA_DIR>/drive-raw.json`.
- **metrics** (only if `HAS_METRICS=yes`) → read defs from `<METRICS_DEFS>`; query `mcp__<MCP_SNOWFLAKE>__sql_exec` / `mcp__<MCP_LOOKER>__query` per `agents/dashboard-metrics.md`. Write `<DATA_DIR>/metrics.json`.

**Wellness** (if in `RUN_AGENTS`): after calendar's JSON is written, **Read**
`<DATA_DIR>/calendar-week.json` yourself, classify focus vs meeting hours, write a short
personalized `weeklyMessage`, and **Write** `<DATA_DIR>/wellness.json`. No MCP call, no
sub-agent — it's a Read + a Write.

**If a source errors** (tool not found / connector down): Write its JSON with
`"sourceOk": false` and move on. One dead source never blocks the others or the merge.

### Step 4 — merge (two direct python3 calls — allowlisted, no polling)

Every JSON is already on disk from Step 3, so there's nothing to wait for. Run the merge
as **two separate, single-command Bash calls** — each is a plain `python3 <script>` that
the `Bash(python3 *)` allow-rule matches exactly (no `&&`/pipes/heredocs → no prompt):

```
# only if drive was in RUN_AGENTS:
Bash(command: "python3 ${CLAUDE_PLUGIN_ROOT}/skills/dashboard/drive-transform.py", description: "Drive transform")
# always:
Bash(command: "python3 ${CLAUDE_PLUGIN_ROOT}/skills/dashboard/build-overrides.py", description: "Merge dashboard data")
```

`build-overrides.py` merges
every source JSON + the config static blocks into `data-override.jsx` / `drive-index.jsx`,
bumps their cache versions, and prints the confirmation line. **Relay that line.** Do NOT
use `wait-and-merge.sh` — its poll loop is what added minutes to the old flow.

**Incremental window (v0.14).** The lookback cutoff (`SINCE_ISO`/`SINCE_EPOCH`/
`SINCE_WINDOW`) is the **exact moment of the last refresh** — agents fetch only what
arrived since then and merge into their prior JSON, because the history before that
can't have changed. Fallbacks: a fresh install backfills **14 days**; a gap of **>7
days** caps catch-up at 7. gmail + granola refresh incrementally (their prior JSON is
kept, not deleted, so they can merge); granola skips the expensive `get_meetings`
entirely when no meeting is newer than `SINCE`. **Per-agent TTL cache:** gmail always
runs; calendar 30m; granola 2h; wellness 4h; drive 8h reuse cached JSON, so a quick
re-refresh may run fewer agents.

If any agent fails, its JSON gets `sourceOk:false`, the merge falls back to empty
arrays for its fields, and the confirmation line appends `· failed: <agents>`.

## Where per-user content lives

All identity/team/OKRs/pins/paths come from `~/.claude/dashboard-config.local`
(see `templates/dashboard-config.local.example` and the `dashboard-setup` skill).
`build-overrides.py` reads it on every refresh — **edit the config file, never
`data-override.jsx`** (it's overwritten each run). The shipped repo contains only
generic placeholders.

## Requirements

- MCP connectors for calendar / gmail / slack / drive / granola, available **in this
  interactive session**. **No servers are bundled** — the agents use whatever
  connectors the user has. Names are read from the config `mcp` section (set by
  setup; commonly `claude_ai_Google_Calendar` etc.), with `claude_ai_`/bare/legacy
  variants in agent frontmatter and a ToolSearch fallback.
- (Only the deprecated headless path needs the `claude` CLI on `PATH`; the in-session
  refresh does not.)

## Rules & gotchas

- **Direct-tool inline, never sub-agents, never ad-hoc shell** (RULES 1 & 2 up top).
  All sources are fetched by the main session via MCP tool calls and written with the
  Write tool; the only Bash is `prep.sh` / `slack-prep.sh` / `python3 …drive-transform.py`
  / `python3 …build-overrides.py`. No heredocs, no `cat >`, no `python3 -c`, no bash
  functions. This is what keeps the refresh prompt-free and under a minute.
- **Should be fully prompt-free** once `allowlist.sh` has run (read-only connector
  tools + Write(`dashboard-data`/`dashboard-os`) + `python3`/`prep.sh` are allowlisted).
  If you ever see a permission prompt, it means an ad-hoc shell command was emitted —
  that's a RULE 1 violation, not a missing allow-rule; fix the command, don't ask the
  user to approve it.
- **Static blocks live in `~/.claude/dashboard-config.local`**, read by
  `build-overrides.py`. Update the config to change roster / OKRs / pins / greeting.
- **Never bump the other cache params.** The merge bumps only `data-override.jsx`
  and `drive-index.jsx`.
- **`nextMeeting` is computed at load time** from `SEED.calendar`, so the countdown
  stays accurate even if the tab is opened hours later.
- **Never render HTML, never open the browser** — the dashboard is its own HTML file;
  this skill only feeds it data. The user reloads the tab (or the server auto-reloads).
