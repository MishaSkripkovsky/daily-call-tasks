# Extraction reference — daily-call-tasks

Logic adapted (and trimmed) from Sasha Marchuk's read-only `find-call` skill. This skill targets an **unattended whole-set digest**, so it KEEPS the proven extraction primitives and DROPS everything interactive.

## Kept from find-call
- Calendar-as-index discovery of the user's events.
- "Meeting Resources" block parsing → Meeting Notes Doc export.
- Optional notetaker (Sembly) augment in parallel.
- Sonnet sub-agent per call to read notes/transcript and return cited action items.
- Anti-slop rules: cite everything; never invent; read-only; sonnet-only; never WebFetch Google URLs.

## Dropped (deliberately — no human in the loop / nothing to rank)
- Relevance **scoring** (Step 3 in find-call) — there is no query to score against; we take the whole attended set.
- **Disambiguation / AskUserQuestion** (find-call Step 4) — unattended runs cannot answer prompts.
- **Alias memory** (find-call Step 8) — stateless.
- `config_io.py` / `~/.claude/find-call/config.json` preference file — not needed; provider is auto-detected.

## Attended predicate (Step 2)
Keep event IFF:
- `organizer.self == true`, OR
- there is an `attendees[]` entry with `self == true` AND `responseStatus` ∈ {`accepted`, `tentative`}.

`self == true` is mandatory (else you mis-attribute other people's calls). `needsAction` (un-answered invite) and `declined` do NOT count as attended.

Exclude:
- `eventType` ∈ {`workingLocation`, `focusTime`, `outOfOffice`}.
- Re-narrow to the true `[start,end]` after the ±1-day boundary padding.

## Meeting Resources regexes (Step 3)
Description is HTML — match links, do not parse as a tree. Strip query strings before use.
- Doc id:        `https://docs\.google\.com/document/d/([A-Za-z0-9_-]+)`
- Drive file id: `https://drive\.google\.com/file/d/([A-Za-z0-9_-]+)`
- Drive folder:  `https://drive\.google\.com/drive/folders/([A-Za-z0-9_-]+)`

**Capture each link together with its adjacent anchor text and select by that label** — bare ids alone cannot tell the Meeting Notes doc from the transcript/video doc. The notes-bot block typically labels links `Transcription` (sometimes `This Call` / `Project Calls`), `Meeting Notes`, `Video`, `Parent Folder`. Route only the **`Meeting Notes`**-labeled doc to extraction; pass a `Transcription`-labeled doc as the optional transcript; **skip `Video`** (binary; not transcribed here).

## Meeting Notes sections
Typical Markdown/Doc structure: `Topic:`, `Date:`, `Short Summary`, `Key Discussion Points`, `Action Points` (often per attendee), `Meeting Resources`. For action-item extraction the `Action Points` section keyed to `{user.name}` is the highest-signal source; quote it verbatim and cite the Doc URL + that section heading.

## Provider fallback (Step 0)
Per source, try the present provider first, fall back to the other; only fail a source if every provider fails.
- Calendar: `mcp__*Google_Calendar*__list_events`  ⇄  `npx @googleworkspace/cli calendar events list`
- Docs:     Drive MCP/connector read              ⇄  `npx @googleworkspace/cli drive files export --params '{"fileId":"<id>","mimeType":"text/plain"}' --output ./.tmp/daily-call-tasks/<id>.txt` then `Read`
- Transcripts (optional): `mcp__sembly-ai__*` or any connected notetaker; skip silently if none.

In a **cloud routine** the claude.ai connectors are the available, pre-authed path (the local `npx` CLI is typically NOT authed there). Locally the CLI may be the faster path. Detect, don't assume.
