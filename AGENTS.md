# Kanban sync protocol — for project agents

You are one of several agents, each working on its own project(s). The shared kanban
board is the single source of truth for **what is being worked on, its status, and
where to find it** (GitHub repos, preview/live URLs, PRs).

- Board UI: https://kanban-otwdesign.vercel.app
- Source of truth: Supabase table `kanban_tasks` (+ `kanban_projects` for the project registry)
- The board polls the cloud every 30 s — anything you write shows up automatically. You never edit `index.html` to update status.

## API access

Everything is plain PostgREST over HTTPS. The anon key is public by design (it ships in the board HTML; RLS allows anon read/write on the two kanban tables only).

```bash
KANBAN_URL="https://yxaxmqmkfyhsawbdlloo.supabase.co/rest/v1"
KANBAN_KEY="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6Inl4YXhtcW1rZnloc2F3YmRsbG9vIiwicm9sZSI6ImFub24iLCJpYXQiOjE3Nzg2MjQ2OTEsImV4cCI6MjA5NDIwMDY5MX0.lc_FdyLh-sfXDbuhXUm2aEqGdvb8PcxsAjw2Nqhy5gM"
AUTH=(-H "apikey: $KANBAN_KEY" -H "Authorization: Bearer $KANBAN_KEY")
```

All writes also need `-H "Content-Type: application/json"`.

## Who owns what

Each agent is scoped to one `proj` key (or a small group). **Never touch rows whose `proj` is not yours.**

| `proj` key | Project          | Task ID prefix |
|------------|------------------|----------------|
| `jepeto`   | Jepeto           | `JEP`  |
| `hub`      | PBN Hub          | `HUB`  |
| `omnicity` | Omnicity         | `OMN`  |
| `clpr`     | CLPR             | `CLP`  |
| `voyc`     | VOYC             | `VOY`  |
| `licenser` | Licenser         | `LIC`  |
| `songbyrd` | Songbyrd         | `SBR`  |
| `linkshop` | LinkShop         | `LCS`  |
| `cnvs`     | CNVS Studio      | `CNV`  |
| `wphq`     | WPHQ             | `WPH`  |
| `gloosb`   | Gloo SiteBuilder | `GSB`  |
| `cnvs4`    | CNVS 4 (SDK)     | `CN4`  |
| `chopgui`  | Chop GUI         | `CHP`  |
| `typa`     | Typa             | `TYP`  |

Not on this list? Check the live registry — `curl -s "${AUTH[@]}" "$KANBAN_URL/kanban_projects?select=key,name,prefix,links"` — and match your repo's GitHub URL against `links`. Still nothing? See "Onboarding a brand-new project" below.

## Task schema (`kanban_tasks`)

| column       | type        | meaning |
|--------------|-------------|---------|
| `id`         | text PK     | `PREFIX-NNN`, e.g. `VOY-102` |
| `proj`       | text        | project key (table above) |
| `col`        | text        | column — see state model below |
| `pri`        | text        | `high` \| `med` \| `low` |
| `t`          | text        | title — short, outcome-oriented |
| `s`          | text        | summary — current state of the work, written for Omri. Rewrite it as things progress; it is not an append-only log |
| `scope`      | jsonb array | checklist of strings. Mark finished items by appending ` — DONE` |
| `links`      | jsonb array | task-level links: `[{"kind":"github"\|"vercel"\|"render"\|"live","url":"…","label":"…"}]` — PR URLs, preview deploys, commits |
| `note`       | text        | one-line "latest from the agent" shown highlighted on the card. Keep it short and current; clear it (null) when stale |
| `started_at` | timestamptz | set when work starts |
| `shipped_at` | timestamptz | set when done |
| `updated_by` | text        | your agent slug, e.g. `agent-voyc`. Set it on **every** write |
| `updated_at` | timestamptz | auto-bumped by trigger — never write it |

## State model (`col`)

| value     | board column | when |
|-----------|--------------|------|
| `back`    | Backlog      | known work, not started |
| `wip`     | In progress  | you are actively working it — set `started_at` |
| `review`  | Review       | built & pushed, needs Omri's verification — put preview/PR links in `links` |
| `wait`    | Awaiting you | **blocked on Omri** (decision, credentials, manual install/test). Say exactly what you need in `note` |
| `done`    | Done         | verified/shipped — set `shipped_at` |
| `archive` | Archive      | superseded / obsolete |

Transition rules:
- `→ wip`: set `started_at` (ISO timestamp, only if not already set), `shipped_at: null`.
- `→ done`: set `shipped_at`. Prefer `review` first unless Omri already verified; **don't claim `done` for untested work** — that has burned trust before.
- `→ wait`: the `note` must state the exact unblock needed ("Install v0.1.1 zip on hub.d3v.co.il", "Decide: flip flag or remove legacy path").

## The protocol — do this every session

### 1. Session start: read your slice

```bash
curl -s "${AUTH[@]}" "$KANBAN_URL/kanban_tasks?proj=eq.voyc&col=not.in.(done,archive)&select=*&order=col.asc,updated_at.desc"
```

Treat this as your work queue and as memory from previous sessions: `wip` = resume, `wait` = check whether Omri unblocked it, `back` = next up. Reconcile against reality (git log, deploy state) — if a card is stale, fix the card.

### 2. Claim before working

```bash
curl -s -X PATCH "${AUTH[@]}" -H "Content-Type: application/json" \
  "$KANBAN_URL/kanban_tasks?id=eq.VOY-102" \
  -d '{"col":"wip","started_at":"2026-07-04T10:00:00Z","shipped_at":null,"note":"Picked up — migrating auth middleware","updated_by":"agent-voyc"}'
```

### 3. While working: keep the card truthful

Update after every meaningful step (commit pushed, deploy up, blocker hit) — not on a timer. A card that says what git says is the goal.

```bash
# progress note + rewritten summary + checklist state
curl -s -X PATCH "${AUTH[@]}" -H "Content-Type: application/json" \
  "$KANBAN_URL/kanban_tasks?id=eq.VOY-102" \
  -d '{
    "s": "Middleware swapped to Supabase JWT. 13 tests green. Render deploy pending.",
    "note": "Commit 8b1a6b0 pushed — awaiting CI",
    "scope": ["Read JWT auth flow — DONE", "Swap middleware — DONE", "Deploy to Render", "Smoke test"],
    "updated_by": "agent-voyc"
  }'
```

Links: `PATCH` replaces the whole array, so **read the current `links` first, merge, write back the full array**:

```bash
curl -s "${AUTH[@]}" "$KANBAN_URL/kanban_tasks?id=eq.VOY-102&select=links"
curl -s -X PATCH "${AUTH[@]}" -H "Content-Type: application/json" \
  "$KANBAN_URL/kanban_tasks?id=eq.VOY-102" \
  -d '{"links":[{"kind":"github","url":"https://github.com/OppositeX/voic-platform/pull/12","label":"PR #12"},{"kind":"vercel","url":"https://voic-git-auth-otwdesign.vercel.app","label":"Preview"}],"updated_by":"agent-voyc"}'
```

### 4. Hand off or finish

```bash
# built, needs Omri's eyes:
-d '{"col":"review","note":"Preview up — please verify login flow","updated_by":"agent-voyc"}'
# blocked on Omri:
-d '{"col":"wait","note":"Need Render env var ENABLE_SUPABASE_AUTH=true, then I can finish","updated_by":"agent-voyc"}'
# verified done:
-d '{"col":"done","shipped_at":"2026-07-04T14:30:00Z","note":null,"updated_by":"agent-voyc"}'
```

### 5. New work you discover → new card

Get the next ID (max numeric suffix for your prefix + 1), then POST:

```bash
curl -s "${AUTH[@]}" "$KANBAN_URL/kanban_tasks?proj=eq.voyc&select=id"   # → next: VOY-103
curl -s -X POST "${AUTH[@]}" -H "Content-Type: application/json" -H "Prefer: return=minimal" \
  "$KANBAN_URL/kanban_tasks" \
  -d '{"id":"VOY-103","proj":"voyc","col":"back","pri":"med","t":"Rotate legacy JWT secret","s":"Found while migrating auth: legacy secret still in Render env.","scope":["Generate new secret","Update Render env","Verify old tokens rejected"],"updated_by":"agent-voyc"}'
```

### 6. Project-level links (`kanban_projects`)

Repo URLs and canonical preview/live URLs live per-project in `kanban_projects.links` (same `{kind,url,label}` shape). When a repo moves or a new permanent deploy exists, update your project's row (read-merge-write, same as task links):

```bash
curl -s -X PATCH "${AUTH[@]}" -H "Content-Type: application/json" \
  "$KANBAN_URL/kanban_projects?key=eq.voyc" \
  -d '{"links":[{"kind":"github","url":"…","label":"GitHub"},{"kind":"vercel","url":"…","label":"Frontend"}]}'
```

Task `links` = ephemeral, task-specific (PRs, preview deploys). Project `links` = durable (repos, production URLs).

Only put a **stable alias** in `links` — e.g. `<project>-otwdesign.vercel.app` — never a
per-deployment hash URL (the kind with a random string in it, like `myproj-a9x8417v4-otwdesign.vercel.app`).
Those rotate on every deploy and go stale immediately.

### 7. The handoff manual (`kanban_projects.handoff`) — how sessions survive restarts

Sessions get interrupted, bugged out, or hit context limits. When that happens, Omri needs
to be able to start a **fresh agent session** and have it pick up exactly where the last one
left off — without you being there to explain it. The handoff manual is what makes that
possible, and the board has a button for it: open **Projects** on the board, find your
project's card, click **"📋 Copy resume prompt"**. That copies a complete, ready-to-paste
prompt — the sync protocol + your project's links, wrapped around whatever you wrote in
`handoff` — for Omri to hand to a brand-new agent instantly. No manual assembly, no
digging through old chat logs.

**Your job:** keep `handoff` genuinely useful. Update it:
- before you end a session (always — assume you might not come back)
- right after any big decision, architecture change, or discovery a cold start would need
- the moment you get blocked in a way that isn't obvious from the task card alone

Keep it short — this is "what would I need to know to pick this up cold right now," not a
changelog (that's what `kanban_tasks.s` and git history are for). Good handoffs cover:
- current state in one or two sentences
- key decisions made and *why* (so nobody re-litigates or reverses them by accident)
- gotchas / traps a fresh agent would otherwise rediscover the hard way
- where credentials/config/env vars live — **never the secrets themselves**
- the exact next step

It's a full-text replace (like `links`), not an append — read what's there if you want to
build on it, otherwise just overwrite:

```bash
curl -s -X PATCH "${AUTH[@]}" -H "Content-Type: application/json" \
  "$KANBAN_URL/kanban_projects?key=eq.voyc" \
  -d '{"handoff":"Supabase JWT migration in progress. Middleware swapped, 13 tests green. Render env var ENABLE_SUPABASE_AUTH still needs to flip to true before the legacy path can be removed — that decision is Omri'"'"'s call, not mine. See VOY-101 for the full trail.","handoff_updated_by":"agent-voyc"}'
```

### Onboarding a brand-new project

If your project genuinely isn't registered yet:

```bash
# 1. pick a key (short lowercase slug) and a prefix that doesn't collide with the table above
curl -s -X POST "${AUTH[@]}" -H "Content-Type: application/json" -H "Prefer: return=minimal" \
  "$KANBAN_URL/kanban_projects" \
  -d '{"key":"myproj","name":"My Project","prefix":"MYP","links":[{"kind":"github","url":"https://github.com/OppositeX/my-project","label":"GitHub"}]}'
```
Only `key` and `name` are required — everything else (`color`, `bg`, `logo`, `handoff`) can
be filled in later. Then use `myproj`/`MYP` as your scope everywhere above. This can never
break the board — an unregistered or newly-registered project renders safely either way.

## Hard rules

1. **Only your `proj`.** Read anything, write only your own rows.
2. **Never DELETE.** Move dead cards to `archive`.
3. **Set `updated_by` on every write.**
4. **Never write `updated_at`** — the trigger owns it.
5. **`done` means verified.** Untested work goes to `review` with repro/verify steps in `scope`.
6. **`wait` cards must say what they're waiting for** in `note`.
7. Timestamps are UTC ISO-8601 (`date -u +%Y-%m-%dT%H:%M:%SZ`).
8. If a PATCH/POST fails, retry once; if it still fails, say so in your reply to Omri instead of silently dropping the update.
9. **Update your project's `handoff` before ending a session.** It's the difference between Omri pasting one prompt into a fresh agent and Omri re-explaining everything from scratch.

## Copy-paste block for each project repo's CLAUDE.md

Replace `PROJ`, `PREFIX`, `SLUG`:

```markdown
## Kanban sync (required)

This project is tracked on the shared kanban (proj key: `PROJ`, task prefix `PREFIX`,
agent slug `SLUG`). Follow the protocol in
https://github.com/OppositeX/kanban-otwdesign/blob/main/AGENTS.md

Non-negotiables for every session:
1. START: fetch your open cards (`proj=eq.PROJ`, col not in done/archive) and reconcile
   them with git/deploy reality before doing anything else.
2. CLAIM: move the card you work on to `wip` (+ `started_at`) before coding.
3. DURING: after each push/deploy/blocker, update `s`, `note`, `scope`, and add PR/preview
   URLs to `links`. Set `updated_by: "SLUG"` on every write.
4. END: land the card in `review` (needs Omri), `wait` (blocked on Omri — say on what),
   or `done` (verified, + `shipped_at`). Create cards for follow-up work you found.
Never mark untested work `done`. Never touch cards of other projects.
```
