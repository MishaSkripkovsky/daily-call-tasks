---
name: daily-call-tasks
description: Builds a cited "yesterday's action items" digest from the calls the user personally attended — reads their Google Calendar for yesterday's attended events, pulls the auto-appended "Meeting Resources" Meeting Notes (and any connected transcript), spawns sonnet sub-agents per call to extract THAT USER's action items with verbatim citations, and prints a per-call digest. Designed to run UNATTENDED on a schedule (e.g. a morning cloud routine) — it never asks questions, never invents action items, and never modifies Calendar/Drive/transcripts/ClickUp/Slack. v0 prints the digest only (no delivery, no ticket creation). Use when the user wants "what action items came up in my calls yesterday", a morning recap of their commitments, or to schedule a daily action-item digest.
user-invocable: true
---

# /daily-call-tasks — Daily Action-Item Digest (v0: print-only)

Build a **cited digest of the action items that came up in the calls the user personally attended** in a past window (default: yesterday). Calendar is the index; the notes-bot **"Meeting Resources"** block on each event points to the Meeting Notes (and optionally a transcript). This skill is **read-only** and **unattended-safe**: it NEVER asks a question, NEVER invents an action item, and NEVER writes to Calendar / Drive / transcripts / ClickUp / Slack. v0 **prints** the digest; it does not deliver or create tickets.

> Extraction logic adapted from Sasha Marchuk's read-only `find-call` skill (github.com/SashaMarchuk/claude-plugins), trimmed for an unattended whole-set digest: scoring, interactive disambiguation, and alias-memory are removed because there is no human in the loop and nothing to disambiguate.

## Invocation & flags (parse first)

| Flag | Meaning | Default |
|---|---|---|
| `--since=<when>` | Window to scan: `yesterday`, `today`, `Nd` (last N days), or `YYYY-MM-DD` | `yesterday` |
| `--dry-run` / `--print-only` | Print the digest, deliver/create nothing | **v0: always on** (delivery lands in v1) |
| `--max-subagents=N` | Cap on parallel transcript/notes sub-agents | `5` |

v0 ignores any delivery flag and always prints. (`--deliver=slack` is reserved for v1.)

## Hard rules (NON-NEGOTIABLE — this skill runs unattended)

1. **No questions, ever.** NEVER call AskUserQuestion or block on input — at 8am there is no human. Process the whole set; on ambiguity, include the item and label it, never pause.
2. **Cite everything.** Every action item must anchor to a Meeting Notes Doc URL + section (or a transcript line / meeting id). No citation → it does not go in the digest.
3. **Never invent.** Only emit action items that appear in the notes' `Action Points`/`Action Items` section or are spoken verbatim in a transcript. If a call has none, say so — do not manufacture.
4. **Read-only.** Zero write calls to Calendar / Drive / transcripts / ClickUp / Slack / Gmail. If a follow-up is implied, the digest TELLS the user; it does not act.
5. **Sonnet sub-agents only** for reading notes/transcripts. Never opus, never haiku (citation fidelity).
6. **Never WebFetch a Google URL** — Google URLs need auth WebFetch can't supply. Use the connector or the CLI.
7. **Always emit something** (heartbeat). A green run with no output is indistinguishable from failure — see Step 5 empty-state.

## Step 0 — Resolve "who am I", calendar, providers

- **User identity (optional):** if `~/.claude/shared/identity.json` exists, read `user.name`, `user.email`, `teammates[]`, `trusted_domains[]` (the same file `/clickup` and `/gevent` use; read-only — never write it). Substitute `user.name` wherever this doc says `{user.name}`. If absent, degrade: treat the **calendar account owner / event organizer** as `{user.name}`, and resolve "me" against attendee `self:true` / the account email. Never HALT for a missing identity.
- **Calendar id:** if `~/.claude/gevent/config.json` exists, use `defaults.calendar`; else `primary`.
- **Providers (detect from the session tool list, prefer-then-fallback — get the data):**
  - Calendar: a Google Calendar MCP/connector (`mcp__*Google_Calendar*__list_events`) OR `npx @googleworkspace/cli calendar events list`. In a cloud routine the connector is the available path; locally the CLI may be authed. Try whichever is present; fall back to the other.
  - Docs: a Drive MCP/connector read OR the `npx @googleworkspace/cli drive files export` CLI then `Read` (full params + `--output` rule in `references/extraction.md`).
  - Transcripts (optional): a connected notetaker (e.g. `mcp__sembly-ai__*`). If none connected, run notes-only and say so. Never required.

## Step 1 — Resolve the window

Convert `--since` to `[start, end]` in the user's local timezone, then pad `timeMin`/`timeMax` by ±1 day for UTC/local boundary, and filter post-hoc. Default `yesterday` = the full previous calendar day.

## Step 2 — List ATTENDED events in the window

List calendar events in the padded window (`singleEvents:true`, `orderBy:startTime`). Keep an event only if the user **attended** it:

- the user is the **organizer** (`organizer.self == true`), OR
- the user is an **attendee with `self == true`** AND `responseStatus` ∈ {`accepted`, `tentative`}.

An un-answered invite (`responseStatus == needsAction`) or a `declined` one does **NOT** count as attended — do not extract the user's action items from a call they didn't attend. (Matching `self == true` is mandatory: without it, every non-declined attendee on a shared calendar would pass and you'd mis-attribute other people's calls.) Then **re-narrow to the true `[start,end]` window post-hoc** (drop the padded ±1 day). Drop `eventType` ∈ {`workingLocation`, `focusTime`, `outOfOffice`}. This whole-set filter replaces find-call's relevance scoring — there is no query to rank against.

## Step 3 — Per event: pull Meeting Notes (and transcript if available)

For each attended event, parse the description (HTML — match links, do NOT parse as a tree). **Capture every doc/drive link TOGETHER WITH its adjacent anchor text** (`Meeting Notes`, `Transcription`, `This Call`, `Project Calls`, `Video`, `Parent Folder`) — the label is the ONLY way to tell the Meeting Notes doc apart from a transcript or video doc; a bare ID can't. Regexes for the ids:
- Doc: `https://docs\.google\.com/document/d/([A-Za-z0-9_-]+)`
- Drive file: `https://drive\.google\.com/file/d/([A-Za-z0-9_-]+)`
- Drive folder: `https://drive\.google\.com/drive/folders/([A-Za-z0-9_-]+)`

Strip query strings (`?usp=…`, `?tab=…`) before use. Then:
- Route ONLY the **`Meeting Notes`-labeled** Doc to export → sub-agent (Step 4). **Skip the `Video`-labeled link** (binary, not transcribed). If a `Transcription`-labeled doc exists, pass it as the optional transcript (not as the notes).
- If a notetaker (Sembly) is connected → also fetch that meeting's structured output (decisions/tasks) by date+title fuzzy match, in parallel.
- If **neither** notes nor transcript exists for the event → record it as `no notes/transcript` (Step 5 lists it so the user knows it was skipped, not silently dropped).

## Step 4 — Extract this user's action items (sonnet sub-agent per call)

For each event that has notes/transcript, spawn a **sonnet** sub-agent (cap `--max-subagents`, default 5; if more events qualify, process the most recent N and list the rest as `not deep-read` — recency is a v0 heuristic, not a ranking). **Pin the model to sonnet deterministically**, don't rely on this prose alone: in a routine, select **Sonnet** in the routine's model selector (sub-agents inherit it); locally, set `CLAUDE_CODE_SUBAGENT_MODEL=claude-sonnet-4-6`. Sub-agent prompt:

```
You are reading ONE call's notes/transcript. Source(s) — read ONLY these:
- Meeting Notes: <Doc URL> (look for the `Action Points` section keyed to {user.name})
- Transcript (optional): <path / meeting id>
Answer, for {user.name} ONLY:
1. Action items / commitments assigned to or owned by {user.name}. Quote the source line verbatim and cite the Doc URL + section (or transcript line). 
2. If the notes' Action Points list items for {user.name}, return those verbatim; do not paraphrase a commitment without a quote.
RULES: Cite every item. If the source has no action item for {user.name}, return exactly "NONE FOR USER". NEVER invent or infer an item that isn't stated. Output ≤ 800 tokens.
```

## Step 5 — Compose & PRINT the digest (v0)

Lead with a one-line header, then one block per attended call **that had action items**. Drop calls with `NONE FOR USER` from the main list but COUNT them. Always include the coverage footer (heartbeat).

```
🗓 Action items from your calls — <window label> (<N> attended call(s))

## <Date HH:MM> — <Event Title>
- <verbatim action item for {user.name}>  ([Notes](<doc url>) → <section>)
- <…>

## <Date HH:MM> — <Event Title>
- <…>

—
Scanned <N> attended call(s): <X> with action items, <Y> with notes but none for you, <Z> with no notes/transcript<, W not deep-read (cap)>.
```

**Empty-state (MANDATORY — never silent):**
- 0 attended calls → `No calls attended <window> — nothing to extract.`
- attended calls but 0 notes anywhere → `Found <N> call(s) but none had Meeting Notes/transcript yet — nothing to extract. (Notes bots often attach within a few hours.)`
- attended calls with notes but 0 action items for the user → `Scanned <N> call(s) with notes — no action items for you.`

The skill **prints** the digest — and that IS the delivery. It is meant to run as a daily **cloud routine** (`/schedule`); the routine's result is a session in the user's Claude account that they read each morning (web/mobile). No Slack, no ClickUp, no secrets. (Scheduling is set up per-user via `/schedule`, see README — a plugin can't self-schedule.)

## Failure handling (never throws away the run)
- Calendar provider unavailable on BOTH paths → print `Could not read calendar (no working provider).` and the coverage footer; do not crash.
- A Doc export 403 → skip that doc, note it in the footer, continue with the rest.
- HTML description, no Meeting Resources block (common for 1-on-1s) → treat as `no notes` (Step 3), continue.

## Optional future enhancements (not built; not needed for the chosen delivery)
- Slack delivery via the first-party Slack MCP connector, if a push-to-channel surface is later wanted (the chosen delivery is the cloud-routine session output, which needs no Slack and no secret).
- Optional confirm → ClickUp ticket creation (hand off to `/clickup`; never write tickets from here).

See `references/extraction.md` for the regexes, the attended predicate, and notes-section parsing details.
