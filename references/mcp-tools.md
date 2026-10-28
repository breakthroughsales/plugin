# Breakthrough MCP tool reference

Every tool is scoped to the caller's own Breakthrough organization, derived from the
authenticated session. There is no way to query across organizations, and no tool takes
an organization argument.

Every non-success payload carries a `message` written for you, not for a log: what
happened, what to do, and whether retrying can change anything. **Read it and act on
it.** When a message says not to retry, retrying wastes turns on a result that cannot
change — the commonest case being `missing_or_invalid_license_context`, which means the
session is unauthenticated, not that the org has no data. Never report that one to the
user as "no data found."

## Read-only tools

### `whoami(detail="basic" | "full")`

**`detail="basic"`** (default) — `email`, `license_id`, `organization_id`. Reads the
access token only, no database round-trip, so it is free to call at the start of any
task. A populated `license_id` is what every other tool scopes on; if it is missing,
nothing else will work.

**`detail="full"`** — adds `full_name`, `organization_name`, `linkedin_url`,
`linkedin_profile` (markdown: headline, summary, experience, education, location), and
`linkedin_profile_fetched_at`. Use when the task turns on who the user is — writing in
their voice, drawing on their own experience, "what's my background".

`full` reads only **cached** enrichment. It never performs a live LinkedIn fetch and
never writes, which is what keeps the whole tool read-only and free of permission
prompts on the hot path. Branch on `profile_status`:

| `profile_status` | Meaning |
| --- | --- |
| `ok` | `linkedin_profile` populated |
| `no_linkedin_url` | User hasn't connected LinkedIn. Identity valid. Do not retry |
| `not_cached` | URL exists, enrichment hasn't run yet. Identity valid. Do not retry |

### `resolve_prompt_context(user_prompt)`

The entry point for almost every task. Pass the user's message **verbatim** — do not
summarize, clean up, or extract keywords first. The tool does its own signal-token
extraction, and pre-digesting the prompt destroys the signal it keys on.

Returns a context map:

- `status`
- `signal_tokens` — the tokens it keyed on
- `matched_transcripts_by_name` — `[{"pattern": ..., "count": N}, ...]`
- matched contacts and businesses, cross-referenced against each other
- `recommendations` — concrete next tool calls
- `mentions` / `mention_source` — the people, companies, and calls the request names, as
  read by a small model (`mention_source: "model"`); only those are searched. If that read
  fails the tool falls back to probing each word (`"fallback"`).
- `matched_transcripts_by_title` — when the request names a call, the transcripts whose
  title matches that mention (id, name, date, participants), with the query shown so a
  generic mention ("our call") can be judged.
- `clarification_needed` / `ambiguous_names` — set when a name token matches more than
  one person AND the evidence does not separate them. Each candidate carries
  `transcript_count`. Candidates the request rules out are already eliminated before
  the flag is set — another word in the request, a first-name prefix, or shared call
  history when exactly one of them has any. So when the flag is set, ask: nothing
  available distinguishes them.

**Follow `recommendations` unless you have a specific reason not to.** They are generated
from real counts against this org's data, so they encode what actually exists
in scope — something you cannot infer from the prompt text alone.

### `contact_profile(query, limit_candidates=5)`

Resolves a contact from free text. Understands raw email addresses and LinkedIn URLs
embedded in the query, so passing the user's phrasing directly usually works.

### `business_profile(query, limit_candidates=5)`

Same, for businesses. Understands domains and LinkedIn company URLs.

When the company has a sales-methodology scorecard, the response carries a
`sales_methodology` field: methodology, `as_of`, `met` / `partial` / `total`, and a hint.
It is a pointer: the scorecard belongs to the company. `contact_profile` returns the
person's `company` (`id`, `name`) with a yes/no `sales_methodology`; for a person's deal,
pass `company.id` to `sales_methodology`.

### `sales_methodology(business_id=None, query=None, categories=None, detail="summary")`

One company's sales-methodology scorecard (MEDDPICC or the org's own methodology): how the
rep is running the deal against the team's process, built from every recorded sales call.
Each category shows its coverage, what is not yet covered, and the lead of its evidence
(who, what was said, the risk), followed by the overall assessment. Internal and
read-only; not the notes of any one call. How to coach from it:
`${CLAUDE_PLUGIN_ROOT}/references/sales-coaching.md`.

- `business_id` — preferred, from `business_profile`. `query` (name, domain, LinkedIn
  company URL) when the id is unknown.
- `categories` — category names as the scorecard shows them (or ids) to expand in full,
  for a question about one part of the deal ("who is the economic buyer?").
- `detail="full"` — every category in full. Several thousand tokens; prefer `categories`.

| Status | Meaning |
|---|---|
| `ok` | `scorecard` text, plus `methodology`, `as_of`, `categories` (the names to expand by); `unknown_categories` if some requested categories did not match |
| `no_scorecard` | None for this company. Do not estimate one |
| `needs_clarification` | `query` matched several companies; ask which, or pass `business_id` |
| `skipped` | `no_match`: no company in this org matches |
| `invalid_input` | No company given, or `detail` was not `summary` or `full`. It reads one company at a time and cannot list scorecards across deals |

### `contact_transcripts_list(contact_ids, limit=50)`

Transcript **metadata** for a set of contact IDs. Takes integer IDs, not names — resolve
via `contact_profile` or `resolve_prompt_context` first. Returns no transcript bodies.

### `search_transcripts(query="", limit=20, filter_by="", sort_by="")`

Semantic + keyword search across the org's call transcripts.
  Set only when the shared name is the request's only handle. When something else in the
  request resolved (a full name, a company, a call found by title), `ambiguous_names`
  still lists the candidates but the recommendation says to proceed with the resolved
  target and not to pick one of them.
- `query` — search text (`"pricing"`, `"SOC2 timeline"`). Leave empty to browse without
  relevance ranking.
- `limit` — 1–50, default 20.
- `filter_by` — filter grammar, applied **before** ranking. Available fields:

  | Field | Type | Matches |
  | --- | --- | --- |
  | `name` | string | the call name |
  | `participant_names` | string[] | any participant |
  | `business_names` | string[] | any business tied to the call |
  | `call_date` | int64 | a date as `YYYY-MM-DD` (a bare date covers the whole day, UTC) or unix seconds |
  | `tags` | string[] | the call's tags: `Sales`, `Internal`, `Onboarding`, `Instructional`, … |

- `sort_by` — sort grammar. Defaults depend on `query`: with an empty query it
  sorts `call_date:desc` (newest first); with a query it sorts by relevance.

Every response carries `today` (UTC date) and, when the filter included `call_date`,
`call_date_window` — the dates actually searched. If the window is not the period the
user asked about, search again with the right dates before reporting no calls.

Only these five fields are filterable. Inventing a field name produces an error, not an
ignored clause. `tags:Sales` keeps to prospect calls; `tags:!=Internal` leaves out your own
team's calls. Every transcript in a list or search result carries its `tags`.

### `call_transcript_conversation(transcript_id, include_structured=False, format="full", part=1)`

Fetches one transcript by integer ID.

- `format="full"` (default) — the tag-wrapped `llm_string`, in parts of 60,000
  characters. The response carries `part`, `parts` and `next_part`; call again with
  `part=next_part` until it is null. Use for a single deep dive, and whenever the request
  is about what was said on one specific call.
- `format="summary"` — thematic summary only. Use when scanning several calls; pulling
  several transcripts at `full` will bury the actual question in raw text.
- `include_structured=True` — adds `transcript_sentences` with per-sentence timing and
  speaker structure. Only meaningful with `format="full"`.

### `sales_playbook(latest_user_message, conversation_history=None, context_hint=None, call_transcript_ids=None, business_ids=None)`

Returns the sections of this org's playbook relevant to the current message, rather than
the whole playbook. `latest_user_message` is the user's current message, passed
**verbatim** — summarizing loses the signal the selector keys on. `conversation_history`
is the prior context, if any; worth passing on later turns, since it is often what
disambiguates a short follow-up like "make it shorter" or "what about for a partner?".

**This is the highest-value tool here.** It is what makes an answer specific to this
company rather than generic sales advice. Call it liberally: retrieval is cheap, it
returns only matching sections, and an empty result costs nothing.

Call it whenever:

- The user asks how to handle, approach, position, pitch, frame, or sell something.
- **You are about to draft anything a prospect or partner will read** — email, LinkedIn
  message, proposal, follow-up, recap. Call it *before* writing, so the draft carries
  this org's positioning and language. Do this even when the user never mentions the
  playbook.
- The user asks what the company does, what makes it different, what its value or
  elevator pitch is, or how to explain it to a particular audience.
- The user asks for discovery questions, talk tracks, next steps, or how to move a deal
  forward.
- The user asks about a competitor, pricing, packaging, or a business case.
- The user asks about partnerships, channel, or co-selling. Some orgs keep a separate
  partnership playbook and this tool selects across all of them, so ask it for
  partner-motion questions exactly as you would for direct-sales ones.

When in doubt, call it. The common failure is not calling it and producing advice that
could have been written about any company.

**Carry over what you already know.** An org can run more than one playbook — typically a
sales playbook for selling directly to a buyer and a partnership playbook for working
through a partner — and a question like "what should I cover on this call?" does not say
which motion is in play. Three optional arguments settle it:

- `context_hint` — a short phrase describing the conversation when the question alone
  does not ("partner conversation with a CTV platform", "direct sales discovery call").
- `call_transcript_ids` — ids of the calls the question is about, if you already resolved
  them via `resolve_prompt_context` or `search_transcripts`. Their names and
  sales/partnership labels are a stronger signal than a freeform hint, and this is the
  most reliable way to land on the right playbook.
- `business_ids` — ids of the businesses the question is about, used the same way.

Without any of these, a vague question on a multi-playbook org can match nothing and come
back `skipped: no_rendered_content`. If that happens, retry ONCE with whatever of the
three you have, and do not retry again after that. Orgs with a single playbook ignore all
three.

The result may also carry a `clarification` field: the motion was not determined, the
sections returned are the ones that hold either way, so use them and then ask the user
that question rather than picking a motion silently.

Not for facts about a specific person or company (`contact_profile` /
`business_profile`), or for what was said on a call (`search_transcripts`).

## Mutating tools

Both are annotated `destructive`, so Claude Code prompts before calling them. Never
instruct the user to pre-approve these.

### `import_contact(linkedin_url=None, email=None, query=None)`

Imports a contact from a LinkedIn profile URL or an email address.

Given only a name (`query="Ben Beal"`), it first checks the org's existing contacts:
`already_on_record` (with `contact_id`) means the person is already in Breakthrough and
nothing was imported; use `contact_profile` or `refresh_contact` instead. `ambiguous_name`
lists `candidates`; ask which one. `invalid_input` means no URL or email was found and
nobody on record matches: ask the user for one, never guess a profile. When the query
names a company that is on record, `invalid_input` also carries `businesses` (id, name,
domain): if the user meant the company, it is already in Breakthrough (use
`business_profile`); if they meant a person at it, ask for that person's URL or email.

An email-only import is held to the name the address carries: a vendor answer whose
name contradicts `john.smith@…` is rejected rather than imported.

Side effects: queues background enrichment against external data sources and waits for
the new contact for up to ~90s. A slow return is normal. It returns once the contact
exists; `contact_without_fit_generating` means the fit result is still running, and
`contact_profile` shows it when done.

### `refresh_contact(linkedin_url=None, email=None, query=None, user_requested=False, employer=None, employer_business_id=None, employer_change_kind=None, employer_since=None, apply_token=None, employer_website=None, contact_id=None)`

Re-enriches an existing contact from its stored LinkedIn URL. See
`skills/refresh-contact/SKILL.md` — the five return statuses mean materially different
things, and only one of them changed any state.

`user_requested=true` means the user asked for the refresh outright or has just confirmed
one; the tool starts it. `false` (the default) means you are acting on a hint; the tool
resolves the contact and returns `needs_confirmation` so you can ask. The tool does not
read intent out of `query`.

**When the user TELLS you where someone works** — "Mauricio works at Autopistas del
Café", "she's at Odinsa now", "he's advising RudderStack" — pass that company as
`employer`. A LinkedIn re-pull cannot help: the provider is what was wrong. `employer`
is where the person IS. "Venkat left Splashtop" names where he is NOT and carries no
employer at all (if they also say where he went, that destination is the `employer`); a
company inside a question — "is Matt still at Quindar?" — is something to check, never an
employer.

Nothing is written until the user confirms. Each status names the one thing still
missing; ask the user and call again with the answer, never filling it in yourself:
`needs_contact_choice` → pass the chosen candidate's `id` as `contact_id`;
`needs_business_choice` → pass the chosen `employer_business_id`, or the website as
`employer_website` if none is right; `needs_new_company_confirmed` → ask whether the
company the tool found is right (or, if it found none, for the website), then pass the
confirmed website as `employer_website`, keeping `employer` as the company's name; `needs_change_kind` → pass `employer_change_kind` as
`correction` (our record was wrong) or `job_change` (they moved), which decides whether
past calls move; `needs_since` → pass `employer_since` as YYYY-MM-DD; `proposal` → show the
`message` and, only after the user agrees, call with the `apply_token` it returned (single
use, short-lived — never store or invent one); `no_change_needed` → the record is already
right, say so.
