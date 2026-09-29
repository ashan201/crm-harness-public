# personal-crm-sync

A personal CRM built on top of Google Contacts and Notion, not a database of
its own. Google stays the source of truth for who people are; Notion is
where relationships actually get managed; a small AI batch layer turns raw
signal (notes, labels, time since last contact) into a short queue of
reviewable suggestions instead of requiring anyone to re-read their whole
contact list.

Everything here is meant to be forkable: point it at your own Google account
and your own Notion workspace and it runs against your own data. This file
is self-contained — it doesn't assume you have any other file from this
project open.

## Why Notion and Google Contacts, specifically

Neither one is load-bearing to the *pattern* — they just happen to be the
two tools that filled two generic roles cheaply, without building a custom
UI or a custom contacts database from scratch. If you don't have or want
either, you can replace them, as long as the replacement fills the same
role:

- **Google Contacts' role: an existing, already-populated identity source.**
  Something that already knows who these people are (name, email, phone,
  company) and exposes it over an API, so this project doesn't have to be
  the system of record for basic facts. The only two things the
  architecture actually needs from it: a stable per-contact id to key off
  of, and read (ideally read/write) API access to those core fields.
  Apple Contacts (via CardDAV), a CSV export, or another CRM's API export
  could fill this role instead — you'd rewrite `sync.py`'s API calls, but
  the rest of the system (Ledger, cards, portal) wouldn't change.

- **Notion's role: a structured store with a human-readable view for
  free.** The project needs somewhere to (a) hold structured rows with a
  real API — the Master CRM, Ledger, and Diagnostics "databases" described
  below — and (b) let a human browse and hand-edit that same data without
  anyone building a CRUD UI for it. Notion happens to give you both at
  once. Airtable does too. A plain Postgres/SQLite database would give you
  (a) but not (b) — you'd need to build the portal into a full admin UI
  instead of the current thin layer that reads/writes Notion and lets
  Notion's own interface handle bulk browsing and hand-editing.

So the real shape of this project, independent of any specific tool, is:

```
 [ identity/facts source ]  --sync-->  [ structured store, human-browsable ]
                                                    │
                                                    ▼
                              [ batch job that reads signal, writes suggestions ]
                                                    │
                                                    ▼
                                       [ thin app that resolves suggestions ]
```

Google Contacts and Notion are just today's answer for the first two boxes.

## Architecture

```
 Google Contacts                                    Notion
 (name, email, phone,                        "Master CRM" database
  company, address...)                      (contact facts + relationship
        │        ▲                            fields: Journal, Cadence
        │        │                            Tier, Communities, ...)
        │  People │ API                              │        ▲
        ▼        │                                    ▼        │
 ┌─────────────────────┐   push/pull, daily   ┌──────────────────────┐
 │      sync.py         │◀────────────────────▶│   (same database)    │
 │  GitHub Actions       │                      └──────────┬───────────┘
 └─────────────────────┘                                  │
                                                            │ reads Journal/
                                                            │ Labels/signal
                                                            ▼
                                              ┌──────────────────────────┐
                                              │   decision_flow.py        │
                                              │   GitHub Actions, daily,  │
                                              │   after sync. One Claude  │
                                              │   call per candidate.     │
                                              └──────────┬───────────────┘
                                                          │ writes review
                                                          │ cards
                                                          ▼
                                              ┌──────────────────────────┐
                                              │   "Ledger" database       │
                                              │   (one row per suggestion)│
                                              └──────────┬───────────────┘
                                                          │ read/write, plain
                                                          │ REST API
                                                          ▼
                                              ┌──────────────────────────┐
                                              │   portal/  (FastAPI app)  │
                                              │   Home queue, Rolodex,    │
                                              │   contact detail, Overdue │
                                              │   no LLM in this app's    │
                                              │   own code path           │
                                              └──────────────────────────┘
```

Three separate jobs, three separate concerns:

1. **`sync.py`** keeps Google Contacts and Notion's contact facts in
   agreement. No AI, standard library only. Runs on GitHub Actions.
2. **`decision_flow.py`** is the only place an LLM runs. It's a plain batch
   job (fixed sequence: gate check, then one Claude call per qualifying
   contact) — not an agent, no tool-calling loop. It never writes contact
   facts; it only writes to the Ledger. Also runs on GitHub Actions, right
   after sync.
3. **`portal/`** is a small always-on web dashboard for acting on what the
   other two produced (`main.py`, `cards.py`, and friends — a FastAPI app
   with server-rendered templates). It reads and writes Notion directly
   over the plain REST API and never calls an LLM in its own request path
   (there's an optional, separately-gated chat feature that does call an
   LLM, off by default).

## Notion schema

Three databases, one Notion integration token shared across all of them.
Each has exactly one system that's allowed to write to any given field —
that split is the load-bearing design decision in this project (more on
that below).

### Master CRM — one row per person

Google-authoritative (sync.py overwrites Notion with Google's value every
pull; changing these in Notion doesn't stick unless you flag the row):

| Property | Type | Notes |
|---|---|---|
| Name | Title | |
| Email, All Emails | Email, Text | `Email` is the primary; `All Emails` holds the rest |
| Phone, All Phones | Phone, Text | same pattern as email |
| Company | Text | Google's `organizations[0].name` |
| Title | Text | Google's `organizations[0].title` |
| Address | Text | Google's formatted address |
| Birthday | Date | |
| Labels | Multi-select | Google contact-group memberships |
| Google Extras | Text | Google's free-form `userDefined` key/value pairs |
| `sys_Google Resource Name` | Text | The identity key — see "Design decisions" |
| Sync Status | Select: `Synced` / `Pending Push` / `Pending Delete` | How a row asks to be pushed or deleted, see below |
| `sys_Last Synced` | Date | |

Notion-authoritative (sync.py never writes these — this is where the actual
relationship-management happens):

| Property | Type | Notes |
|---|---|---|
| Journal | Text | Freeform relationship notes/log |
| Cadence Tier | Select: `Tier 1 (30 Days)` / `Tier 2 (90 Days)` / `Semi-Annual (180 Days)` / `Tier 3 (365 Days)` / `Tier 4 (As-Needed)` / `Organic - No Reminder` / `Rolodex Only` | How often this person should come up for a check-in |
| Last Contacted | Date | |
| Communities | Multi-select | Freeform relationship tags (e.g. shared context, groups) |
| City, Region | Text | Filled either deterministically from Google's address or, failing that, inferred by `decision_flow.py` |

System-computed, derived rather than hand-set:

| Property | Type | Notes |
|---|---|---|
| `sys_Cadence Days (Auto)` | Number | Numeric twin of Cadence Tier (30/90/180/365, or empty for the two no-reminder tiers) |
| Next Follow-up | Date (formula) | Built from Last Contacted + `sys_Cadence Days (Auto)` |
| `sys_Duplicate Group` | Text | Set by the duplicate-detection pass |
| AI Snapshot, `sys_Snapshot Updated` | Text, Date | An optional cached summary, refreshed periodically |

### Ledger — one row per AI-generated suggestion

Written by `decision_flow.py`, read and resolved by the portal.

| Property | Type | Notes |
|---|---|---|
| Entry | Title | |
| Contact | Relation → Master CRM | Can hold more than one contact for a combined card |
| Card Type | Select: `Approve` / `Acknowledge` / `Answer` | See "Card types" below |
| Status | Select: `Open` / `Done` / `Will do` / `Not now` / `Dismissed` / `Answered` / `Declined` / `Looks Good` / `Reverted` | |
| Field Affected | Select | Which Master CRM field this card is about, if any (`Journal`, `Cadence`, `Community`, `Geography`, ...) |
| Source | Select | Which processor generated it (`Journal`, `Milestone`, `Cross-Mention`, `Manual`, ...) — `Manual` also covers user-set ad-hoc reminders |
| Suggested Text | Text | The model's draft, or the user's own text for a manual reminder |
| Old Value | Text | The prior field value, so a write can be reverted |
| User Response | Text | What the user actually said/did, logged back for context |
| Snooze Until | Date | When a snoozed or manually-scheduled card resurfaces |
| Resolved | Date | |

### Diagnostics — one row per data-hygiene issue

A separate concern from the Ledger: "is this contact's data well-formed"
(malformed email, likely duplicate) rather than relationship intelligence.
Written by `diagnostics_check.py` and related scripts, read by the portal.

| Property | Type | Notes |
|---|---|---|
| Entry | Title | |
| Contact | Relation → Master CRM | |
| Issue Type | Select | e.g. malformed email, likely duplicate |
| Detail | Text | |
| Fix Type | Select | What kind of fix this issue takes; the portal's batch actions fork on this |
| Status | Select | Same open/resolved shape as the Ledger |
| Ledger Card | Relation → Ledger | When a diagnostics fix also needs an approvable write |
| Detected | Date | |

## Card types (the Ledger's core idea)

Three card types, corresponding to three different trust levels for AI
output:

| Card type | Pattern | Meaning |
|---|---|---|
| **Approve** | model drafts, you decide | e.g. "suggest reaching out — here's a draft" |
| **Acknowledge** | model writes, then tells you | e.g. auto-tags Communities from context, logs an FYI card you can revert |
| **Answer** | model asks you directly | used when there isn't enough signal to guess — asks *why you care about this person*, never a schema-shaped question like "what tier is this" |

Every card can be approved, snoozed (`Will do` / `Not now`), or dismissed.
Nothing the AI layer does is a one-way door.

### The processors (`decision_flow.py`)

Each is gated independently (mostly: is this contact "due," per Cadence
Tier) and claims a contact for at most one open card per run:

- **Suggested Interactions** — cadence-due, has Journal history → Approve card with a drafted next step.
- **Questions** — cadence-due, empty Journal → Answer card asking why you care about this person.
- **Communities** / **Geography** — infers tags/location from Journal + labels → Acknowledge card.
- **Cadence Tier** — catches the minority of "no reminder" contacts where the evidence suggests they should actually have one → Acknowledge card.
- **Milestones** — a recent Journal entry looks like a real milestone (new job, promotion, etc.) → Approve card, independent of cadence.
- **Cross-Mention** — someone else's Journal entry mentions this contact → Approve card, attachable to multiple people at once.
- **External Signals** (optional) — folds in calendar/email/briefing-doc context as an opt-in, read-only extra source, feeding the same processors above rather than being its own card type.

## Design decisions worth knowing before you fork this

- **Split authority, not shared authority.** Every field has exactly one
  system allowed to write it (see the Google-authoritative /
  Notion-authoritative split above). Fights over whose edit wins are solved
  at the schema level, not at merge time.
- **One identity key ties a Notion row to a Google contact**
  (`sys_Google Resource Name`). Losing or clearing it un-links the row —
  the next sync treats the Google contact as new and creates a duplicate
  row, rather than updating the original.
- **Nothing hard-deletes.** Both the sync and the AI layer treat "remove
  this" as a status flag (`Sync Status = Pending Delete`) that another pass
  acts on, never a direct delete. Deleting a Notion row by hand doesn't
  work — the next pull just recreates it from Google, since the identity
  key is still live on the Google side.
- **The AI layer is a batch job, not an agent.** Fixed sequence, one call
  per candidate, budget-capped per run. Cheaper to build and run, and much
  easier to reason about when something misfires than an open-ended agent
  loop would be.
- **The portal has no AI in its main code path.** It only resolves what the
  batch job already produced, so the always-on web app stays simple and
  cheap to run; the metered/costed part runs once a day, on GitHub's
  infrastructure, not on every page load.

## Stack

- Python, standard library only, for `sync.py` and the batch processors —
  no dependencies to install to run the unattended jobs.
- FastAPI + Jinja templates for `portal/` — no frontend build step.
- Plain REST calls to both the Google People API and the Notion API — no
  SDKs, to keep the dependency surface small on scripts that run
  unattended.
- GitHub Actions for scheduling (sync and decision-flow run daily, one
  after the other).
- Claude, one call per candidate in `decision_flow.py`, cost-capped per run
  and logged with real token/dollar totals.

## Configuration

**Repository secrets** (Settings → Secrets and variables → Actions), used
by the GitHub Actions workflows in `.github/workflows/`:

| Secret | Used by | Value |
|---|---|---|
| `GOOGLE_CREDENTIALS_JSON` | `sync.py` | Full contents of a Google OAuth credential file (`client_id`, `client_secret`, `refresh_token`) for the Google account being synced |
| `NOTION_TOKEN` | everything | A Notion internal integration token, shared across all three databases |
| `NOTION_DATABASE_ID` | `sync.py`, `decision_flow.py`, `portal/` | The Master CRM database's id |
| `ANTHROPIC_API_KEY` | `decision_flow.py` | Anthropic API key — one call per qualifying contact per run |
| `GITHUB_TOKEN` | optional | Lets a run close its own stale issue notices; GitHub provides this automatically inside Actions, nothing to set |

None of these are committed; `.gitignore` blocks them.

`decision_flow.py` also needs a Ledger database id and a Diagnostics
database id — separate from the Master CRM database above, matching the
schemas described earlier in this file. In this codebase they're set as
plain constants near the top of `decision_flow.py`
(`LEDGER_DATABASE_ID`, `DIAGNOSTICS_DATABASE_ID`) rather than secrets,
since a Notion page id alone isn't enough to access anything without the
integration token — set them to your own two databases' ids before running
it.

**Running the portal locally:**

```bash
cd portal
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt
cp .env.example .env   # fill in NOTION_TOKEN, NOTION_DATABASE_ID (ANTHROPIC_API_KEY is optional)
.venv/bin/python run.py
```

Then open `http://127.0.0.1:8420/`.
