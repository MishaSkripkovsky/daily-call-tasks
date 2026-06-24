# sections reference — morning-brief

Details for the SKILL.md steps: the "done" detection (Step 2), the "plate" dedup (Step 3), the TeamMD resolver (Step 5/8), and the Geekbot payload (Step 8). Exact tool params are transcribed live from the ClickUp MCP tool schemas, the Geekbot v1 API, and the team.md export.

## ClickUp query params (exact)
- **`clickup_filter_tasks`** — `assignees: string[]` of NUMERIC ids (resolve names/emails/"me" via `clickup_resolve_assignees(["<email>"])` first); `statuses: string[]` (lowercase status names as configured, e.g. `"in progress"`, `"to do"`, `"blocked"`); `include_closed: bool` (MUST be `true` to see closed tasks); `date_closed_from`/`date_closed_to` (pattern `YYYY-MM-DD` or `YYYY-MM-DD HH:MM`); `page` is **0-indexed** (paginate until short page); `order_by ∈ {id,created,updated,due_date}`; scoping `list_ids`/`folder_ids`/`space_ids` (all `string[]`). There is **no** `assignee` singular and **no** `closed` boolean.
- **`clickup_get_task_comments(task_id)`** — returns each comment with `user`, `date`, `resolved` (bool), `assignee` (the person it was sent TO, or `null`), `assigned_by`, `reply_count`. **Verified live** — a real assigned comment on task `86ca8brqx` returned `resolved:true` + `assignee:{id,username,email}` + `assigned_by`. These are NOT declared in the MCP *input* schema, so re-confirm on the first real run and **degrade** to a nameless "sent for review" if a particular comment lacks them. Use for reviewer resolution below.
- **Time-in-status** (`clickup_get_task_time_in_status` / `_bulk_`, ≤100 ids) requires the **"Total time in Status" ClickApp** — OPTIONAL enrichment only; the snapshot-diff below is the ClickApp-independent primitive. Don't put it on the critical path.

## Step 2 — "What was done": UNION of two primitives

### Primitive 1 — closed in the window (exact)
`clickup_filter_tasks(assignees=[me], include_closed=true, date_closed_from=<prior-snapshot date>, date_closed_to=<now>)`. **The `date_closed` window MUST span the SAME gap the snapshot-diff covers** — bound it by the prior-snapshot date (matching the "since `<prior-snapshot date>`" label), NOT the narrow `--since` done-window. Otherwise a task closed during a weekend/gap (e.g. Saturday, on a Monday run) is missing here and Primitive 2's "left the open set" reconciliation mislabels a genuinely-done task as "no longer on your plate". Catches tasks that reached a terminal **Closed** status in the window. NOTE: `date_closed` fires for the terminal `Closed` status — a green **Done** status that isn't the workspace's "closed" type may NOT appear here. That gap is covered by Primitive 2.

### Primitive 2 — snapshot-diff (transitions, ClickApp-free)
State file: `~/.claude/morning-brief/snapshot-<YYYY-MM-DD>.json` (date key in the user TZ):
```json
{ "generated_at": "<ISO8601>", "tz": "<IANA>", "me": "<clickup id>",
  "tasks": { "<task_id>": { "status": "in progress", "name": "…", "url": "…" } } }
```
- **Capture set:** the user's NON-closed tasks (`clickup_filter_tasks(assignees=[me], include_closed=false)`, paginated). Write it at the END of every full run (Step 8).
- **Diff:** load the most recent snapshot whose date is **< today** (NOT today's own fresh write — so a same-day re-run still diffs against the prior day). For each task:
  - in both, status changed → classify the NEW status: `→ in progress` = "worked on, not finished"; `→ review` = "sent for review"; `→ done`-family = "done".
  - in prior snapshot but GONE from today's open set → it was closed/done or reassigned away. **Reconcile against Primitive 1:** if its id is in the closed-in-window result → "done/closed"; else it left the open set for another reason (reassigned, deleted) → list as "no longer on your plate" (do not claim done).
  - new in today's set but not prior → that's a *new* task (belongs to "plate", not "done").
- **Window label:** name the section "since `<prior-snapshot date>`" — the prior snapshot may be older than yesterday after a weekend/skip; never hardcode "yesterday".
- **First run / no prior snapshot:** print "baseline captured — status changes will show from the next run", still emit Primitive 1.
- **Multi-hop within a day** (To-Do→In Progress→Review same day) → only endpoints are seen. Documented limitation; acceptable for a standup.
- **Retention:** keep the last ~14 snapshots; prune older.

### Reviewer resolution — "sent for review TO WHOM"
For each task classified `→ review`/`→ done`: `clickup_get_task_comments(task_id)` → pick the most recent comment with `resolved == false` AND `assignee != null`. `assignee` = the person it was sent to. Render the name from `assignee.username`; resolve a Slack mention via the TeamMD resolver (by `assignee.id` = ClickUp id, or `assignee.email`). No such comment → "sent for review" with no name (NEVER guess a reviewer).

## Step 3 — "On your plate" dedup: marker-first → Jaccard
Inline call items (from the imported `../daily-call-tasks/references/extraction.md` extraction) are deduped against the user's OPEN tasks so only genuinely un-ticketed items get the "⟂ not yet in ClickUp" flag.
1. **Marker-first.** Items previously ticketed via `daily-call-tasks-commit` carry a hidden marker in the ClickUp task description:
   ```
   <!-- dca:<workspace_id>:<list_id>:<assignee_id>:<source_doc_id>:<action-key> -->
   ```
   where `action-key` hashes a STABLE locator (`source_doc_id` + Action-Points heading + line ordinal), not the prose. Enumerate the user's open tasks (`clickup_filter_tasks(include_closed=false)`), READ each description, and if a candidate's `<source_doc_id>:<action-key>` matches a marker on an open user-owned task → **already ticketed, drop it** (don't flag). (`clickup_filter_tasks` can't filter description bodies — match by reading fetched descriptions, or `clickup_search` the marker string then intersect.)
2. **Jaccard fallback** (pre-existing human tasks, no marker): casefold + NFKC title tokens, drop stopwords, `|A∩B| / |A∪B| ≥ 0.70` against the same open set → treat as already-on-plate, drop from the "not yet ticketed" flag (still may appear as a normal open task). 
3. Closed/done tasks and tasks not assigned to the user are NEVER matched → a resembling call item stays flagged as new.
This mirrors `daily-call-tasks-commit`'s dedup exactly (`../daily-call-tasks-commit/references/commit-rules.md`) so the two skills agree on what's "already ticketed".

## Step 5/8 — TeamMD resolver
File: resolve from the FIRST that exists — config `teammd_path` → `~/Work/team.md` (capital W) → a synced `team.md` under `~/.claude/**` or a `Speed and Function` folder (Andy's TeamMD skill's synced copy — prefer it when present, it stays current across subprojects). Two GFM pipe-table sections — `## Full Members` and `## External / Multi-Channel Guests` — **union both**. Columns, in order:
```
Full Name | Slack Username | Display Name | Slack ID | Work Email | ClickUp ID
```
Parsing: split on `|`, trim; a literal em-dash `—` (U+2014) = null. Skip the header + `|---|` separator rows.
Join keys (reliability order): **Slack ID** (present on every row — the value to emit), **Work Email** (unique, some null), **ClickUp ID** (unique, ~half null), **Name** (collision-prone — never resolve on first-name alone).
Resolvers:
- `clickup_id_to_record(id) → {name, slack_id, email}` — for reviewer resolution (Step 2) from a comment's `assignee.id`.
- `email_to_slack_id(email) → slack_id` — for reviewer/`assignee.email`.
- `name_to_slack_id(name) → [candidates]` — returns a LIST; **>1 hit ⇒ ambiguous, do NOT auto-pick** (ask or keep plain name).
**Mention format:** emit `<@SlackID>` (angle bracket + `@` + the raw `U…` Slack id, e.g. `<@U01ABCDEF>`) — never the username or display name. Known ambiguities to respect: duplicate display names exist (e.g. two "Lana Mamukova", two "Misha") — first-name-only resolution is forbidden. **Fail closed (Hard Rule 4):** an unresolved/ambiguous name in text destined for a Geekbot post → keep the plain name, WARN, and never emit a broken `@`.

## Step 8 — Geekbot post
API base `https://api.geekbot.com/v1` (trailing slashes matter). Auth header **`Authorization: <RAW_API_KEY>`** (NO `Bearer`/`Token` prefix — a prefix 401s) + `Content-Type: application/json`. Key is MEMBER-scoped (per-user) and requires a paid plan; read it from env `GEEKBOT_API_KEY` or `~/.claude/morning-brief/config.json` (`geekbot.api_key`) — NEVER hardcode.
1. **Discover:** `GET /v1/standups/` → array; pick the configured `geekbot.standup_id` (or, if one standup, use it). Each standup has integer `id` and `questions[]` with integer `id` + `text`.
2. **Map** our sections to that standup's questions by matching question text (best-effort): "what did you do / yesterday" → Done; "what will you do / today / plan" → On-your-plate; "blockers / blocked" → Blockers; a **"mood / how are you / how do you feel"** question → the user's **Mood** (Step 5); a catch-all / "anything else" → Open questions. If the standup's questions don't match, show the mapping and let the user confirm/edit before posting.
3. **Render the exact payload and PREVIEW it** (Hard Rule 4):
   ```json
   { "standup_id": <int>,
     "answers": { "<question_id>": { "text": "<section text>" }, "<question_id>": { "text": "…" } } }
   ```
   `answers` keys = question ids as STRINGS; ALL of the standup's questions must be present (empty-but-required → send a short "—").
4. **Confirm** via `AskUserQuestion` (Post / Skip). On Post → `POST /v1/reports/` with the header above. Handle `401` (bad/free-plan key) and `429` (back off) by reporting + keeping the paste-ready block; never retry-loop. Read-back optional via `GET /v1/reports/?standup_id=&user_id=`.
