# onboarding reference — morning-brief

Self-contained identity onboarding for `~/.claude/shared/identity.json`. It writes the SAME cross-plugin schema `/clickup`, `/gevent`, `/find-call`, and `daily-call-tasks-commit` read — so running it once here also satisfies the commit skill's identity gate. **It does NOT depend on `/clickup` being installed** (that plugin's `/clickup:onboard` is the old, dead pointer this replaces).

> The file is a shared cross-plugin contract. We are a third writer of a file two other live plugins mutate — so the write MUST honor the schema gate (`schemaVersion: 2`), the `flock`, atomic replace, and unknown-key preservation, or the next `/clickup`/`/gevent` run quarantines it to `identity.json.corrupt-<epoch>`.

## When this runs
Step 0 of `SKILL.md` calls this when `identity.json` is absent, `onboarding_complete != true`, or the running identity is ambiguous. `/morning-brief --onboard` runs ONLY this and stops. It is interactive (read-back confirm); never run it headless.

## Canonical schema (what we WRITE)
Top level: `schemaVersion` (bare integer `2` — missing/string/float ⇒ instant quarantine), `schemaVersion_bumped_at`, `schemaVersionHistory[]`, `onboarding_complete: true`, `updated_at` (ISO8601 UTC), `trusted_domains[]` (TOP LEVEL, not under `user`), `user{}`, `teammates[]`.
- `user{}` = EXACTLY `name`, `email`, `external_ids{}` (reserved keys clickup/google/slack/jira; `{}` if none). **No `latin_alias` on `user{}`** — it lives only on `teammates[]`.
- `teammates[]` entry = `first_name`, `latin_alias` (ASCII, required, the resolver key), `full_name`, `email` (upsert key), `external_ids{}`, `active` (bool), `sources[]`, `last_validated_at`.
- **Self-record:** upsert a `teammates[]` entry for the user themself (same email as `user.email`, `latin_alias` = their Latin first name) so `daily-call-tasks-commit`'s `Action Points → {latin_alias}` attribution resolves. The gate passes without it (on `user.name`+`user.email` presence); the self-record only improves attribution recall.

Minimal valid file (self-record uses the same email as `user.email`):
```json
{ "schemaVersion": 2, "onboarding_complete": true, "updated_at": "<ISO8601>",
  "user": { "name": "<full name>", "email": "<work email>", "external_ids": {} },
  "trusted_domains": ["<own-domain>"],
  "teammates": [ { "first_name": "<first>", "latin_alias": "<ascii first>",
    "email": "<work email>", "active": true, "sources": [], "last_validated_at": null } ] }
```

## Wizard steps (mirror the plugin `onboard-identity` so output is byte-compatible)
1. **State check.** Exists + `onboarding_complete` + explicit re-run → proceed with a "refreshing roster; existing teammates preserved" warning. Exists but incomplete → resume. Absent → fresh.
2. **Ask identity** (`AskUserQuestion`, one round): full name + work email. Seed in-memory skeleton `{schemaVersion:2, user:{name,email,external_ids:{}}, trusted_domains:["<email-domain>"], teammates:[], onboarding_complete:false, updated_at:<now>}`.
3. **Cross-source read-back confirm (DO NOT skip).** Probe + echo, then one confirm:
   ```
   Identified as:
     Full name:  <as typed>
     Work email: <as typed>
     ClickUp:    <clickup_resolve_assignees(email) → name + id, or "not found">
     Google:     <primary calendar id (= authed email), or "not authed">
   Is this you?  [Yes, continue / Pick different ClickUp record / Fix email]
   ```
   Capture `user.external_ids.clickup`. Initialize top-level `trusted_domains` with the user's own `@domain`. Do NOT proceed without this confirm — it is exactly the "ambiguous identity" the commit gate refuses on.
4. **Discover teammates** (parallel, merge by email): `clickup_get_workspace_members` (source `clickup-workspace`), assignees on the user's open tasks (`clickup-tasks`), calendar attendees last 14 days (`google-calendar`). Tag each with its `sources` value; UNION on re-discovery. (Optional for v1 — a `[]` roster still passes the gate; the self-record is the one entry that matters for attribution.)
5. **Alias confirm** for non-Latin `first_name` (read-back). Misha's name is Latin → `latin_alias = first_name`, no-op.
6. **Write atomically** via the helper below.

## The write helper (atomic + flock + preserve-unknown-keys)
Run this via `Bash python3 - <<'PY' … PY`. It is stdlib-only and matches the plugins' `atomic_update` contract: hold `flock` on `~/.claude/shared/identity.json.lock`, read-modify-write, round-trip unknown keys, upsert teammates by email, atomic `os.replace`.

```python
import json, os, fcntl, tempfile, time
from datetime import datetime, timezone

HOME = os.path.expanduser("~")
DIR  = os.path.join(HOME, ".claude", "shared")
PATH = os.path.join(DIR, "identity.json")
LOCK = PATH + ".lock"                      # exact contract path: identity.json.lock (no leading dot)
os.makedirs(DIR, exist_ok=True)
now = datetime.now(timezone.utc).strftime("%Y-%m-%dT%H:%M:%SZ")

# --- values gathered by the wizard (Steps 2-5) ---
USER = {"name": "<full name>", "email": "<work email>",
        "external_ids": {}}   # set {"clickup": "<id>"} ONLY if the wizard resolved one; {} otherwise — never a placeholder string
DOMAIN = USER["email"].split("@")[-1]
SELF_TEAMMATE = {
    "first_name": "<first>", "latin_alias": "<ascii first>", "full_name": USER["name"],
    "email": USER["email"], "external_ids": USER.get("external_ids", {}),
    "active": True, "sources": ["clickup-workspace", "google-calendar"], "last_validated_at": now,
}
DISCOVERED = []  # optional list of teammate dicts from Step 4

with open(LOCK, "w") as lk:
    fcntl.flock(lk, fcntl.LOCK_EX)         # cross-plugin mutual exclusion; kernel frees on death
    base = {}
    if os.path.exists(PATH):
        try:
            with open(PATH) as f: base = json.load(f)
        except Exception:
            os.replace(PATH, PATH + ".corrupt-%d" % int(time.time()))  # unparseable → quarantine + reskeleton
            base = {}
    # Quarantine a corrupt-TYPED existing file BEFORE writing — match the cross-plugin gate:
    # an existing non-empty file whose schemaVersion is missing, or not a BARE int (string "2", float, null, bool)
    # must be moved aside, NEVER silently coerced (silent coercion is the downgrade vector /clickup + /gevent close).
    sv = base.get("schemaVersion")
    if base and ("schemaVersion" not in base or isinstance(sv, bool) or not isinstance(sv, int)):
        os.replace(PATH, PATH + ".corrupt-%d" % int(time.time())); base = {}; sv = None
    # refuse to downgrade a file a NEWER plugin owns (read-only fallback)
    elif isinstance(sv, int) and sv > 2:
        raise SystemExit("identity.json schemaVersion > 2 — a newer plugin owns it; not writing.")
    base.setdefault("schemaVersionHistory", [])
    if base.get("schemaVersion") != 2:
        base["schemaVersionHistory"].append({"from": base.get("schemaVersion", 1), "to": 2, "at": now})
        base.setdefault("schemaVersion_bumped_at", now)
    base["schemaVersion"] = 2              # BARE INT — the single hard gate
    base["onboarding_complete"] = True
    base["updated_at"] = now
    # user: merge known keys, preserve any unknown ones already present
    # user: scalar fields replace; external_ids DEEP-MERGE so a prior google/slack/jira is never clobbered
    base.setdefault("user", {})
    base["user"]["name"] = USER["name"]; base["user"]["email"] = USER["email"]
    base["user"]["external_ids"] = {**base["user"].get("external_ids", {}), **USER.get("external_ids", {})}
    base["trusted_domains"] = sorted(set(base.get("trusted_domains", []) + [DOMAIN]))
    # teammates: upsert by email; PRESERVE email-less entries (round-trip, never drop); DEEP-MERGE external_ids; UNION sources
    keyed   = [t for t in base.get("teammates", []) if t.get("email")]
    unkeyed = [t for t in base.get("teammates", []) if not t.get("email")]    # legacy/manual entries the plugins own
    by_email = {t["email"]: t for t in keyed}
    for incoming in [SELF_TEAMMATE, *DISCOVERED]:
        e = incoming["email"]; cur = by_email.get(e, {})
        merged_eids = {**cur.get("external_ids", {}), **incoming.get("external_ids", {})}
        cur_sources = sorted(set(cur.get("sources", [])) | set(incoming.get("sources", [])))
        cur.update(incoming); cur["external_ids"] = merged_eids; cur["sources"] = cur_sources
        by_email[e] = cur
    base["teammates"] = unkeyed + list(by_email.values())
    # atomic replace (tmp in same dir + fsync + os.replace)
    fd, tmp = tempfile.mkstemp(dir=DIR, prefix=".identity.", suffix=".tmp")
    with os.fdopen(fd, "w") as f:
        json.dump(base, f, ensure_ascii=False, indent=2); f.flush(); os.fsync(f.fileno())
    os.replace(tmp, PATH)
print("identity.json written:", PATH)
```

## After writing
Echo the confirmed `user.email` back. The commit skill's Step 0.2 also echoes `user.email` and asks the user to confirm before any write — the two read-backs are aligned (both pivot on `user.email`). If onboarding was interrupted, `onboarding_complete` stays `false`; the next run resumes.
