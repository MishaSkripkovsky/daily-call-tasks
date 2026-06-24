# daily-call-tasks

A Claude Code skill that builds a **cited digest of the action items from the calls you personally attended** — by default, yesterday's. It reads your Google Calendar for attended events, pulls the auto-appended **Meeting Resources → Meeting Notes** (and any connected transcript), and uses sonnet sub-agents to extract *your* action items with verbatim citations.

It is **read-only** and **unattended-safe**: it never asks questions, never invents action items, and never modifies Calendar / Drive / transcripts / ClickUp / Slack.

> ClickUp 86ca8brqx · adapted from the read-only [`find-call`](https://github.com/SashaMarchuk/claude-plugins) extraction logic.

## Skills in this repo
This repo ships **three** complementary skills under `.claude/skills/` (they share the extraction logic + the `~/.claude/shared/identity.json` contract):

| Skill | Posture | What it does |
|---|---|---|
| `daily-call-tasks` | read-only, unattended | the cited morning digest (below) |
| `daily-call-tasks-commit` | interactive, write | review/edit the digest → create the chosen items as ClickUp tasks (self only) |
| `morning-brief` | interactive | Geekbot-style standup prep — what you did / on your plate / blockers / open questions → auto-post to Geekbot (see [Morning Brief](#morning-brief-standup-prep)) |

> New here? See **[QUICKSTART.md](QUICKSTART.md)** for install + a ready-to-paste channel message.

## How it's delivered (decided)
The skill **prints** the digest, and that printed digest **is the delivery**: it runs as a daily **cloud routine** (`/schedule`) whose result is a session in your Claude account — you read it each morning in the Claude app (web + mobile). No Slack, no secrets, no extra connector. You then create whatever tickets you want yourself.

## Send to ClickUp (v2 — interactive)
A separate, interactive companion skill **`daily-call-tasks-commit`** turns the digest into ClickUp tasks. The morning digest stays **read-only**; this write step runs only in a Claude session (it refuses to run unattended). Flow:
1. Review the **table** (flat-numbered `1..N`, columns: Task · Priority · Deadline · Description · Status · List); edit by exception — `drop 3`, `edit 4: …`, `desc 4: …`, `prio 4: high`, `due 4: 2026-06-30`, `status 4: backlog`, `list 4: <list>`, `go`, `cancel`.
2. Pick the destination ClickUp list **in the moment** (per item or a batch default).
3. On `go`, each item is CREATEd — or, if a similar task already exists in that list, you're shown a before→after diff and choose `update` / `create new` / `skip`.

Safeguards: writes nothing until you type `go` (`--dry-run` shows the plan and writes nothing); only acts on action items the source attributes to **you** (others → `UNATTRIBUTED`, never auto-committed); updates change **name/description only**, never closed/others' tasks; re-runs don't duplicate (hidden idempotency marker); dismissed items don't re-surface.

```bash
# from the repo root, in a Claude session:
/daily-call-tasks-commit --since=yesterday --dry-run    # preview the create/update plan
/daily-call-tasks-commit --since=yesterday              # review/edit → go → writes
```

Roadmap (optional, later): Slack delivery; PM-oversight of other people's tasks (deliberately a non-goal for now — self only).

## Morning Brief (standup prep)
`morning-brief` is an **interactive** sibling skill that prepares your daily standup from the same calls + your own ClickUp tasks, and (on confirmation) posts it to Geekbot. ClickUp 86cacn12x.

One command assembles five sections: **Done** (attended meetings + ClickUp status changes — `In Progress` = worked on, `Review`/`Done` = sent for review *and to whom*, resolved from the task's assigned comments), **On your plate** (open `In Progress`/`To-Do` tasks + not-yet-ticketed action items pulled inline from yesterday's calls), **Blockers**, **Open questions** (asked, then @-mentioned correctly via a `team.md` roster), and optionally **Emails** (if a Gmail connector is present). It is read-only against ClickUp/Calendar/Drive — the only writes are the self-onboarded identity file and the **confirmed** Geekbot post; it fails closed on any unresolved `@mention`.

```bash
/morning-brief --onboard     # one-time: self-contained identity wizard (writes ~/.claude/shared/identity.json)
/morning-brief --status      # show which dependencies are connected / degraded
/morning-brief --no-post     # compose + print the brief, never post to Geekbot
/morning-brief               # full interactive run → confirm → post
```

Optional dependencies degrade gracefully: a **Geekbot** API key (`GEEKBOT_API_KEY`) enables auto-post; the **Gmail** connector enables the Emails section; a **team.md** roster enables `<@SlackID>` mentions. Running `--onboard` once also satisfies `daily-call-tasks-commit`'s identity requirement. See `.claude/skills/morning-brief/references/` for the snapshot-diff, dedup, and Geekbot details.

## Try it locally (dry-run)
From inside this repo (so the project skill is picked up), pre-approving the read-only tools it needs (a bare `claude -p` would abort at the first tool-approval prompt; **do not** use `--bare` — it disables skill discovery):

```bash
# run from the repo root (so the project skill under ./.claude/skills is discovered):
claude -p "/daily-call-tasks --since=yesterday --dry-run" \
  --allowedTools "Read,mcp__claude_ai_Google_Calendar__list_events,mcp__claude_ai_Google_Drive__read_file_content,mcp__claude_ai_Google_Drive__search_files,Bash(npx @googleworkspace/cli *)"
```
> Reads Docs via the Drive connector's `read_file_content` (no file written). The `npx` CLI is only a local fallback and writes a scratch file under `.tmp/` (gitignored).

Prerequisite: your Google Calendar + Google Drive must be reachable — either a connected Google Calendar/Drive MCP connector, or the `npx @googleworkspace/cli` authenticated locally. The skill auto-detects and falls back. To pin transcript-reading sub-agents to Sonnet locally: `export CLAUDE_CODE_SUBAGENT_MODEL=claude-sonnet-4-6`.

What to verify in the output:
- did it pick the right calls (the ones **you** attended yesterday)?
- are the action items actually **yours**, and do the citations point to real Meeting Notes?
- nothing invented; calls with no notes are listed (not silently dropped).

## Schedule it (per user, one-time)
1. Connect **Google Calendar** + **Google Drive** at `claude.ai/customize/connectors`.
2. Create a daily routine with `/schedule`:
   - **Repo:** `daily-call-tasks` (the routine clones it to get this skill).
   - **Connectors:** Google Calendar + Google Drive.
   - **Model:** select **Sonnet** (so the per-call sub-agents run on Sonnet).
   - **Schedule:** daily, e.g. 08:00.
   - **Prompt:** `Run /daily-call-tasks --since=yesterday and output the digest.`
3. A cloud routine runs on Anthropic's servers even if your laptop is closed. Each run's result is a **session in your Claude account** — open the Claude app (web/mobile) in the morning to read the digest.

> Coverage note: the digest only includes calls whose Meeting Notes / transcript are readable by **your** Google connector. Calls whose notes-bot docs aren't shared with your account are listed as "no accessible notes/transcript" rather than dropped.

## Layout
```
.claude/skills/
  daily-call-tasks/          SKILL.md  references/extraction.md   # read-only digest
  daily-call-tasks-commit/   SKILL.md  references/commit-rules.md # interactive → ClickUp
  morning-brief/             SKILL.md  references/{onboarding,sections}.md  # standup prep + Geekbot
```

## Guarantees (per skill)
- `daily-call-tasks` (the digest) is **read-only** — zero writes to any service; Sonnet sub-agents (citation fidelity); never invents an action item; always emits a result (even "no calls / no notes found"), so a silent failure is distinguishable from a quiet day.
- The two writer siblings act only on explicit confirmation: `daily-call-tasks-commit` writes to ClickUp (self only, never closed/others' tasks, idempotent re-runs); `morning-brief`'s only writes are the self-onboarded identity file and the **confirmed** Geekbot post (fails closed on unresolved mentions).
