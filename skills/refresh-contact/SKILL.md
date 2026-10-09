---
name: refresh-contact
description: >-
  Keeps an EXISTING Breakthrough contact's employer and title right. When the user
  states where someone works now ("Marco works at Kleecks, I just met him", "she's
  advising RudderStack"), records that employer once the user confirms — the fix for
  people LinkedIn has wrong. Otherwise re-pulls their details from LinkedIn when the
  record is stale, and answers whether someone's job is still current ("is Matt still
  at Quindar?", "did he leave?"): Breakthrough holds the user's own record of that
  person. Confirms before changing anything.
  Writing to them or writing up a call is answer; what they said on a call is
  research-transcripts; their record as it stands is find-contact.
when_to_use: >-
  Use for statements of where someone works now ("Mauricio works at Autopistas del
  Café", "he's at Globex now", "he joined Kleecks"); refresh requests ("refresh Jane",
  "update his title"); complaints that a record is wrong ("you have him at his old
  company"); departures ("Sarah left Acme", "FYI they moved on"); and currency
  questions ("is this current?", "where is Sarah now?"). Prefer this over find-contact
  and answer whenever a contact's employer or title is stated, reported
  changed, or questioned — nothing changes without the user's say-so. Do NOT use for
  someone not yet in the system; that is import-contact.
---

# Refresh a contact

Wraps the `refresh_contact` MCP tool, which does two different things:

- **Records an employer the user states.** The user is the source of truth; nothing is
  re-read from LinkedIn, because LinkedIn is usually what was wrong.
- **Re-pulls the record from LinkedIn** when it has gone stale.

## When this applies

The backend routes broadly on purpose, because the tool confirms before changing
anything. All of these belong here:

- **Statements of where someone works now** — "Mauricio works at Autopistas del Café",
  "he joined Kleecks", "she's advising RudderStack". Advisory, fractional and board roles
  count. These go to the correction path below, not a refresh.
- **Explicit commands** — "refresh Jane's contact info", "update this contact's title"
- **Stale-data complaints** — "John's data is wrong", "you have her at her old employer"
- **Employer or title changes, however phrased** — including passive and note-style
  mentions with no refresh verb at all: "Sarah no longer works at Acme", "he's at Globex
  now", "FYI Sarah left Acme"
- **Currency questions** — "is his info up to date?", "do you have more current data?"

## Recording an employer the user states

Call `refresh_contact(query=..., employer="<company>")` — the company as the user said
it, or its website. `employer` is where the person IS:

- "Venkat **left** Splashtop" names where he is NOT — there is no employer; this is a
  plain refresh. If they also say where he went ("left Splashtop for RudderStack"), the
  destination is the `employer`, never the origin.
- A company inside a **question** — "is Matt still at Quindar?" — is something to check,
  never an employer.

Nothing is written until the user confirms. Each status names the one thing still
missing — **ask the user, then call again with their answer. Never fill these in
yourself.**

| Status | Ask the user | Then call again with |
| --- | --- | --- |
| `needs_contact_choice` | which of the listed people they mean | `contact_id` of that candidate |
| `needs_business_choice` | which of the listed companies they mean | `employer_business_id`, or `employer_website` if none is right |
| `needs_new_company_confirmed` | whether the company the tool found is right — or its website, if it found none | `employer_website=<website>` — `employer` stays the company's name |
| `needs_change_kind` | was our record simply wrong, or did they change jobs? (decides whether past calls move) | `employer_change_kind="correction"` or `"job_change"` |
| `needs_since` | when they started | `employer_since="YYYY-MM-DD"` |
| `proposal` | show `message` — it says exactly what will change — and ask whether to apply | `apply_token` from this result, **only after they agree** |
| `no_change_needed` | nothing — the record is already right; say so | — |
| `applied` | nothing — report what changed | — |
| `expired` | the confirmation lapsed; start over | — |

Keep the same `employer` and answers on every follow-up call. `apply_token` is single
use and short lived: never store, reuse, or invent one.

## Re-pulling from LinkedIn

`refresh_contact(linkedin_url=None, email=None, query=None, user_requested=False)`. Pass
the user's message as `query` when you have no identifier — resolution is the tool's job.

`user_requested` is your call, and the tool never infers it from the wording of `query`:

- **true** — the user asked for the refresh outright ("refresh Jane", "re-pull his
  details", "update her record from LinkedIn"), or has just confirmed a refresh you
  offered. The tool starts the refresh.
- **false** — you are acting on a hint: a change report, a complaint, a currency question
  ("Sarah left Acme", "is Matt still at Quindar?"). The tool resolves the contact and
  returns `needs_confirmation` so you can ask first.

Annotated **destructive**; Claude Code prompts before it runs. Don't ask the user to
pre-approve it.

## Handling the result — read this before reporting anything

For a re-pull (no `employer`), five statuses. **Only `started` changed any state.** Every
other status is a safe no-op that enqueued nothing.

| Status | What happened | What you must do |
| --- | --- | --- |
| `started` | Refresh was **enqueued** | Report it's queued. Enrichment is *not* complete — don't present current field values as refreshed. |
| `needs_confirmation` | One contact resolved, but from an **implicit** signal | **Stop and ask the user before refreshing.** Show which contact you matched. Once they confirm, call again with `user_requested=true`. |
| `ambiguous` | Several candidates matched | Present the candidates and ask which. Do not pick one. |
| `no_linkedin_url` | Contact found, but no stored URL to re-enrich from | Report it. Retrying will not help. |
| `no_match` | Nothing resolved | Report it. Offer the import-contact skill if they have a URL or email. |

### Why `needs_confirmation` exists

Breakthrough deliberately separates an explicit refresh *command* from an implicit
*signal*. A passing mention — a note, a complaint, a question about currency — resolves
the contact but stops short of refreshing, so a stray mention never silently triggers
a re-pull.

A statement of a **new employer** — "update Marco's employer, he's now at Kleecks" — is
not a re-pull at all: pass it as `employer` (above). A re-pull only re-reads LinkedIn and
cannot record what the user just told you.

Collapsing these statuses into "done" defeats the guardrail. Report the one you got.

## Reporting

Name the contact you matched — full name, ID, current role and employer — so the user can
catch a wrong match before anything is re-enriched.

For `started`, say the refresh is queued and data will update, not that it is updated.

For a stated employer, report only what the tool says happened: a `proposal` is not a
change until the user agrees and `applied` comes back.
