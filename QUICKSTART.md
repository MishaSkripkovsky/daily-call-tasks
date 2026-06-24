# Quickstart — daily-call-tasks suite (3 skills)

A primitive "how to launch it" — paste the block at the bottom into your team channel.

## Install (once)
```text
/plugin marketplace add speedandfunction/snf-automation-plugins
/plugin install daily-call-tasks@snf-automation-plugins
/reload-plugins
```

## Connect + onboard (once)
1. Connect **Google Calendar**, **Google Drive**, and the **ClickUp** connector at `claude.ai/customize/connectors`.
2. Run `/morning-brief --onboard` once — it creates your shared identity file (`~/.claude/shared/identity.json`). This also unblocks `/daily-call-tasks-commit`.

## The three skills
| Command | What you get |
|---|---|
| `/daily-call-tasks` | A plain morning message: per attended call → your action items (with priority/deadline if voiced), cited. Read-only. Schedule it with `/schedule` to get it every morning. |
| `/daily-call-tasks-commit` | Review those items as a **table** (#, task, priority, deadline, description, status), edit by exception (`drop 3`, `prio 4: high`, `status 4: backlog`, `go`), and create them as ClickUp tasks. Writes only on your `go`. |
| `/morning-brief` | Your standup, prepared: what you did (status changes + "sent for review to whom") · what's on your plate (open tasks + not-yet-ticketed call items) · blockers · open questions + Mood. On confirmation, posts to **Geekbot**. Run `/morning-brief --no-post` first to preview. |

## Optional (degrade gracefully if absent)
- `GEEKBOT_API_KEY` (env or `~/.claude/morning-brief/config.json`) → enables the Geekbot auto-post.
- Gmail connector → adds the Emails section to the brief.
- A `~/Work/team.md` roster → correct `@`-mentions in the standup.

---

## Ready-to-paste channel message
> **🗓 Daily Call Tasks + Morning Brief — try it (5 min)**
> A Claude Code plugin suite: a morning digest of *your* action items from yesterday's calls, a one-click "send these to ClickUp", and a Geekbot-style standup prep.
> 1. Paste into Claude: `/plugin marketplace add speedandfunction/snf-automation-plugins` then `/plugin install daily-call-tasks@snf-automation-plugins` then `/reload-plugins`
> 2. Connect Google Calendar + Drive + ClickUp at claude.ai/customize/connectors, then run `/morning-brief --onboard` once.
> 3. Use it: `/daily-call-tasks` · `/daily-call-tasks-commit` · `/morning-brief --no-post`
> Marketplace: https://github.com/speedandfunction/snf-automation-plugins/tree/main/plugins/daily-call-tasks — questions → me.
