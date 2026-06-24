# commit-rules — daily-call-tasks-commit

Details for the create/update path. Extraction itself is in `../../daily-call-tasks/references/extraction.md` (unchanged).

## Attribution anchor (Step 2)
An item is **auto-committable** only if the source explicitly assigns it to the user:
- it appears under an `Action Points` / `Action Items` section keyed to `{user.name}` (or `latin_alias`), OR
- the transcript has the user (by name) stating/accepting it, or someone assigning it to them by name.
Otherwise → `UNATTRIBUTED`: shown in a separate block, default unselected, never auto-committed. (Being an attendee ≠ owning the item. A verbatim citation proves the text exists, not that it is the user's.)

## Hidden idempotency marker (Step 6)
Every CREATEd task gets, at the end of its description:
```
<!-- dca:<workspace_id>:<list_id>:<assignee_id>:<source_doc_id>:<action-key> -->
```
- `workspace_id:list_id:assignee_id` SCOPE the marker so it can never match a different user's or a different list's task (e.g. a recurring item a teammate committed with the same source). Match only when all components equal the current run.
- `source_doc_id` = the Google Doc id of the Meeting Notes/Transcription.
- `action-key` = hash of a STABLE source locator — `source_doc_id` + Action-Points section heading + the item's line ordinal — NOT the volatile extracted prose (LLM wording drifts run-to-run → a prose hash both duplicates on drift and collides when two different items normalize alike). Caveat: if the notes bot **re-orders** the Action-Points list between runs the ordinal shifts → at worst a benign duplicate-create, caught by the Jaccard fallback (Step 5).
- **Retrieval:** `clickup_filter_tasks(include_closed=false)` enumerates the list; then READ each candidate's description for the marker substring — field filters do not see description bodies. Match is over OPEN, user-owned tasks only, exact on all components.

## Dedup decision table (Step 5, per item, in the CHOSEN list)
| Signal | Decision |
|---|---|
| Marker exact-match on an OPEN task | already committed → **skip** (offer update only if the user reworded it this session) |
| No marker, Jaccard ≥0.70 vs an OPEN task | **candidate** → show before→after diff, ask per item: `update` / `create new` / `skip` (default = create new) |
| Match is a CLOSED/done task | ignore the match → **create new** |
| No marker, no Jaccard hit | **create** |
Jaccard rule = casefold+NFKC token sets, drop the `/clickup` stopword list, `|A∩B|/|A∪B|`, threshold 0.70 (copied from the clickup plugin so behavior matches). `clickup_filter_tasks` is called with `include_closed=false`.

## UPDATE safety (Step 6)
- Changes **name and/or description ONLY**. Never status, assignee, tags, priority, dates, comments, custom fields.
- **Append**, don't replace: add the new citation/quote to the existing description; never delete human-written content. If the existing description has no marker, add one.
- Requires a per-item confirm that displays the exact before→after for both fields.
- Never updates a closed task. Snapshot the old name+description in the run report before applying (so a wrong update is recoverable by hand).

## Dismiss ledger (Step 3)
`~/.claude/daily-call-tasks/dismissed.json`:
```json
{ "schemaVersion": 1, "dismissed": [ { "fp": "<source_doc_id>:<action-hash>", "text": "<short>", "at": "<ISO>" } ] }
```
- On `drop`, offer to remember; on yes, append the fingerprint **immediately** (independent of whether the batch later commits or is cancelled — so a dropped item can't reappear because the run was aborted). Subsequent runs skip matching candidates silently (don't re-show).
- Same fingerprint scheme as the marker, so dismissed and committed items both key off `source_doc_id:action-hash`.
- Atomic write (tmp + replace). Gitignored (lives under `~/.claude/`, not the repo).

## Destination list resolution (Step 5)
- The user picks per item (`list 4: <name>`) or a batch default (`list all: <name>`).
- Resolve the name against the workspace hierarchy (`clickup_get_workspace_hierarchy` / `clickup_get_list`); on ambiguity, show candidates and ask. Never guess a list.
- Dedup is scoped to the resolved destination list.

## Task shape (Step 6 CREATE)
- Title: `[Call: <event name> <date>] <verb-first action>` (≤ ~100 chars; regenerate shorter rather than truncate).
- Description: the cited block — `> <verbatim quote>` + source `<call name>, <date>` + `Notes: <Doc URL>` + (if voiced) the short context description — then the hidden marker.
- Assignee: the user.
- **Status / Priority / Deadline — set from the reviewed table:** `status` = To-Do or Backlog (default via the heuristic below, user-overridable); `priority` = urgent/high/normal/low if voiced; `due_date` if voiced. Validate before the call (status ∈ the list's real To-Do/Backlog status names via `clickup_get_list`+`expand_statuses`; priority ∈ the ClickUp enum; deadline a real `YYYY-MM-DD`); a value that fails validation drops to blank with a note, never guessed.

## Status heuristic (To-Do vs Backlog)
Default from the per-item fields in `../../daily-call-tasks/references/extraction.md`: a **near-term deadline** (this week, user TZ) OR **urgent/high** priority → **To-Do** ("do this week"); a far/blank deadline OR low priority → **Backlog** ("later / unknown deadline"). On conflict, **any positive To-Do signal wins → To-Do**. The user overrides with `status N: to-do|backlog` at review. **UPDATE never sets status/priority/deadline** — those are CREATE-only (see UPDATE safety: name/description only).

## Interactivity guard (Step 0)
Detect unattended/no-TTY (no human). If unattended → refuse and stop (the routine must never reach a write). This is the mechanical backstop that keeps the scheduled digest read-only.
