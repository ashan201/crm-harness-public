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

## The jobs

Almost everything runs on GitHub Actions. Only the portal and one LinkedIn
script run somewhere else.

**The daily core, in this order:**

1. **`sync.py`** keeps Google Contacts and Notion's contact facts in
   agreement. No AI, standard library only.
2. **`decision_flow.py`** is the daily AI batch. It is a plain job (fixed
   sequence: gate check, then one Claude call per qualifying contact), not
   an agent, with no tool-calling loop. It never writes Google-owned
   contact facts. It writes review cards to the Ledger. For a few low-risk
   fields (Communities, City, Region, Cadence Tier, AI Snapshot) it writes
   the field and also logs a card you can revert. It runs right after sync.
3. **`portal/`** is a small always-on web dashboard for acting on what the
   other two produced (a FastAPI app with server-rendered templates). It
   reads and writes Notion directly over the plain REST API. Its main
   pages never call an LLM. There is an optional chat feature that does,
   and it is off by default.

**Side jobs:**

| Job | Runs | What it does |
|---|---|---|
| `digest.py` | Weekly, inside the sync workflow | Finds overdue contacts and posts a digest as a GitHub Issue |
| `close_stale_notices.py` | Daily | Closes notice Issues that are 14+ days old |
| `linkedin_sweep.py` | Daily | Reads LinkedIn connection-accepted emails in Gmail and flags new connections in Diagnostics |
| `diagnostics_check.py` | On demand | Writes data-hygiene problems (missing name, duplicates, stuck tier) to the Diagnostics database |
| `execute_duplicate_merges.py` | On demand | Merges only the duplicate groups a human confirmed in the portal |
| `onboarding/`, `troubleshooting/` | Once, on demand | Setup and cleanup passes: import contacts, clear backlogs, repair old incidents. Never scheduled |

### Every couple of months: the LinkedIn catch-up

`linkedin_reconcile.py` is the one job that is deliberately manual. It
works from LinkedIn's own data export, so you request a fresh export, unzip
it, and run the script on your own machine.

It compares the export with the CRM and reports:

- **Job changes.** Company and title differences for people it can match.
- **Name differences.** Where the spellings differ, the fuller name wins.
- **New people.** Anyone you messaged in the last 12 months who is not in
  the CRM yet.

Matching tries the LinkedIn profile link first, then an exact name. It
never merges on a near match. Near matches are listed for a human to decide.
Two different people can share a name, so a link that already matched one
profile blocks a second profile with the same name.

By default it only writes a report. Flags apply each part:
`--apply-company`, `--apply-titles`, `--apply-names`, `--apply-new`. Applied
edits are written as `Pending Push`, so the next sync carries them to
Google. A new person is created in Notion and Google, and gets one
`Approve` card holding a Journal note that Claude drafted (told to leave out
sensitive details). The export stays on your machine. Only the message text
needed for a draft goes to the model. The script keeps no history of old
values.

## How the sync stays safe

Every contact row has a `Sync Status`. It is how you tell the sync what to
do with a row.

| Value | Meaning |
|---|---|
| `Synced` | Normal. Google wins any disagreement. |
| `Pending Push` | Your Notion edit overwrites Google once, then the row goes back to `Synced`. |
| `Pending Pull` | Refresh this row from Google. |
| `Pending Delete` | Delete it everywhere. |
| `Review` | Frozen. The sync ignores the row. |

These rules cause most of the sync bugs when they are broken:

- **Google wins by default.** Notion edits to Google-owned fields revert on
  the next pull unless the row is `Pending Push`.
- **One identity key links a row to a Google contact**
  (`sys_Google Resource Name`). Clear it and the next sync creates a
  duplicate.
- **Deleting a Notion row by hand does not work.** The pull sees a Google
  contact with no row and recreates it. The only delete is `Pending Delete`.
- **Runs queue and do not overlap.**

## Report mode before apply mode

Every flow that writes has a switch with three values: `off`, `report`,
`apply`.

- **`report`** runs everything, including the model calls, and prints what
  it would write. It writes nothing to Notion.
- **`apply`** writes for real.

A new flow starts in `report`. You read its output against real data, and
only then switch it to `apply`. Some flows stay in `report` on purpose. A
wrong write by an AI layer costs more than a missed suggestion.

## Notion schema

Three databases, one Notion integration token shared across all of them.
Each field has exactly one system allowed to write it. That split is the
load-bearing design decision in this project (more on that below). The
tables below show the columns that carry the design. Each database also
has helper columns, many prefixed `sys_`. Read the live schema in Notion for
the full list.

### Master CRM: one row per person

| Property | Owner | Notes |
|---|---|---|
| Name, Email, All Emails, Phone, All Phones, Company, Title, Address, Birthday, Labels | Google | The sync overwrites Notion with Google's value on every pull. `All Emails` and `All Phones` hold the extras. |
| `sys_Google Resource Name` | Sync | The identity key. See "How the sync stays safe". |
| Sync Status | You and the sync | How a row asks to be pushed, pulled, deleted, or frozen. |
| LinkedIn URL, LinkedIn Profiles | Notion | The primary profile link, and every link found for the person. |
| Journal | Notion | Free-form relationship notes. The sync never writes it. |
| Cadence Tier | Notion | How often the person should come up: `Tier 1 (30 Days)`, `Tier 2 (90 Days)`, `Semi-Annual (180 Days)`, `Tier 3 (365 Days)`, `Tier 4 (As-Needed)`, `Organic - No Reminder`, or `Rolodex Only`. |
| Last Contacted, Communities, City, Region | Notion | Relationship fields. City and Region come from Google's address when it has one, or from `decision_flow.py` when it does not. |
| `sys_Cadence Days (Auto)`, Next Follow-up | Derived | The number of days behind the tier, and the date the next check-in is due. Never set by hand. |
| AI Snapshot | `decision_flow.py` | An optional cached summary, refreshed when the Journal changes. |
| Ledger, Diagnostics | Relations | Link a person to their cards and data-hygiene rows. |

### Ledger: one row per AI-generated suggestion

Written by `decision_flow.py` and a few other flows. Read and resolved by
the portal.

| Property | Notes |
|---|---|
| Entry | Title |
| Contact | Relation to Master CRM. A combined card can hold several contacts. |
| Card Type | `Approve`, `Acknowledge`, or `Answer`. See "Card types". |
| Status | Where the card is: open, done, snoozed, dismissed, answered, and so on. |
| Field Affected, Source | Which contact field the card is about, and which flow produced it. |
| Suggested Text | The model's draft, or your own text for a manual reminder. |
| Old Value | The prior field value, so a write can be reverted. |
| User Response | What you said or did, logged for context. |
| Snooze Until, Resolved | Dates for resurfacing and closing. |

### Diagnostics: one row per data-hygiene issue

This is a separate concern from the Ledger. It asks whether a contact's data
is well-formed (missing name, likely duplicate) rather than what to do about
the relationship. It has its own lifecycle. A row goes from flagged, to fix
requested, to fixed (or fix failed, or dismissed). It does not share the
Ledger's statuses.

| Property | Notes |
|---|---|
| Entry, Contact, Detail | What is wrong and for whom |
| Issue Type | For example missing name, duplicate, possible identity match |
| Source | Which check found it |
| Fix Type | `Deterministic`, `Judgment`, or `Manual`. The portal's batch actions fork on this. |
| Status | The lifecycle above |
| Ledger Card | Link to a card, when a fix also needs an approvable write |
| Detected, Resolved | Dates |

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

- **Suggested Interactions**: cadence-due, has Journal history. An Approve card with a drafted next step.
- **Questions**: cadence-due, empty Journal. An Answer card asking why you care about this person.
- **Communities** and **Geography**: infer tags and location from Journal and labels. An Acknowledge card.
- **Cadence Tier**: catches the "no reminder" contacts where the evidence suggests they should have one. An Acknowledge card.
- **Milestones**: a recent Journal entry looks like a real milestone (new job, promotion). An Approve card, independent of cadence.
- **Cross-Mention**: someone else's Journal entry mentions this contact. An Approve card that can attach to several people at once.
- **External Signals** (optional): folds in calendar, email, and briefing-doc context as a read-only extra source. It feeds the processors above instead of being its own card type.
- **Email Reply**: when you reply to an email from a known contact, sets Last Contacted to the reply date and logs a revertable Acknowledge card. It also drafts a short Journal note as an Approve card.

Two jobs in the same file sit outside the card system. **Snapshot**
regenerates the AI Snapshot when a contact's Journal changes. **Resurface**
reopens snoozed cards when their date arrives. Resurface is date bookkeeping
only and uses no model.

## Design decisions worth knowing before you fork this

- **Split authority, not shared authority.** Every field has exactly one
  system allowed to write it (see the Owner column in the
  schema tables above). Fights over whose edit wins are solved
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
- **A human approves the AI's work.** The model drafts and the person
  decides (Approve), or the model writes a low-risk field and tells the
  person, who can revert it (Acknowledge). Every card keeps the old value.
  Nothing the AI does is a one-way door.
- **The daily sync and the portal's main pages use no AI.** The model runs
  in a few clearly separated places: the daily batch (`decision_flow.py`),
  the optional portal chat, and a few one-time or manual scripts (the
  onboarding passes and the LinkedIn catch-up). The always-on web app stays
  simple and cheap. The costed part runs once a day on GitHub's
  infrastructure, not on every page load.
- **Report before apply.** New flows run in `report` mode first. See
  "Report mode before apply mode".

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
  and logged with real token/dollar totals. The optional portal chat and a
  few one-time scripts call it too.

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
