# daily-call-tasks

A Claude Code skill that builds a **cited digest of the action items from the calls you personally attended** — by default, yesterday's. It reads your Google Calendar for attended events, pulls the auto-appended **Meeting Resources → Meeting Notes** (and any connected transcript), and uses sonnet sub-agents to extract *your* action items with verbatim citations.

It is **read-only** and **unattended-safe**: it never asks questions, never invents action items, and never modifies Calendar / Drive / transcripts / ClickUp / Slack.

> ClickUp 86ca8brqx · adapted from the read-only [`find-call`](https://github.com/SashaMarchuk/claude-plugins) extraction logic.

## How it's delivered (decided)
The skill **prints** the digest, and that printed digest **is the delivery**: it runs as a daily **cloud routine** (`/schedule`) whose result is a session in your Claude account — you read it each morning in the Claude app (web + mobile). No Slack, no secrets, no extra connector. You then create whatever tickets you want yourself.

Roadmap (optional, later):
- Slack delivery (first-party Slack connector) if a push-to-channel surface is wanted.
- Optional confirm → ClickUp ticket creation.

## Try it locally (dry-run)
From inside this repo (so the project skill is picked up), pre-approving the read-only tools it needs (a bare `claude -p` would abort at the first tool-approval prompt; **do not** use `--bare` — it disables skill discovery):

```bash
cd daily-call-tasks
claude -p "/daily-call-tasks --since=yesterday --dry-run" \
  --allowedTools "Read,Bash(npx @googleworkspace/cli *),mcp__claude_ai_Google_Calendar__list_events,mcp__claude_ai_Google_Drive__read_file_content,mcp__claude_ai_Google_Drive__search_files"
```

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
.claude/skills/daily-call-tasks/
  SKILL.md                 # the skill (extraction + digest)
  references/extraction.md  # regexes, attended predicate, notes parsing
```

## Guarantees
- Read-only — zero writes to any service.
- Sonnet sub-agents for reading notes/transcripts (citation fidelity); never invents action items.
- Always emits a result (even "no calls / no notes found"), so a silent failure is distinguishable from a quiet day.
