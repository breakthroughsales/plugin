---
name: research-transcripts
description: >-
  Searches and reads the user's recorded Breakthrough call transcripts — finds which
  calls touched a topic, lists a contact's calls, or reads one call in full. Answers
  what was actually said, by whom, and when, citing the call it came from.
when_to_use: >-
  Use when the user says "what did they say about pricing", "did we discuss budget",
  "when did we last talk to Acme", "what were their objections", "pull up that call",
  "did they commit to anything", "who was on the call", "did it come up". Do NOT use to
  write the call up as notes or a MEDDPICC; that is answer. Do NOT use for questions needing
  judgment beyond what the calls actually state; that is answer.
---

# Research call transcripts

Wraps `search_transcripts`, `contact_transcripts_list`, and
`call_transcript_conversation`. Full argument details in
`${CLAUDE_PLUGIN_ROOT}/references/mcp-tools.md`.

## Before searching: who is the request about?

When the request names a person or a company ("what did Ben think of the demo",
"pull up that call with Acme"), run `resolve_prompt_context(user_prompt=<verbatim>)`
**first**, before `search_transcripts`. It says which contacts and businesses exist with
that name, how many calls each has, and whether the name is ambiguous. A first name
alone often matches several people, each with their own demo — searching transcripts
for "Ben demo" and reading the top hit picks one of them silently, which is the worst
outcome.

- `clarification_needed: true` → reply with the clarifying question the
  recommendations spell out (the candidates and their calls on record) and stop.
- One matched contact → use its id with `contact_transcripts_list`, then open the
  call you need. Do not fall back to a free-text `search_transcripts` for a person who
  resolved.
- A matched business but no contact ("that call with Acme") →
  `search_transcripts(query=<topic or "">, filter_by="business_names:<name as returned>")`,
  then open the call you need. `business_profile` returns the profile and notes, not
  the call list.
- No match → `search_transcripts` with the topic is the right next step.

## A topic across a window ("product requests from the last week")

The search index holds each call's name, imported summary, participants, businesses,
date and tags — not what was said. A topic like "product requests" is rarely in a title,
so do not search for it. Bound the set instead and read:

1. `search_transcripts(query="", filter_by="call_date:>=<YYYY-MM-DD> && tags:Sales",
   sort_by="call_date:desc", limit=50)` — the prospect calls in the window, newest first.
   The tool returns at most 50; if you get 50 back, the window holds more: query again
   with `call_date:<=` the oldest date you received (`<=`, not `<`: several calls can share
   a timestamp) and drop the ids you already have, until a page comes back short. Say how
   many calls there were in total.
2. Open **each** of them with `format="summary"` (the thematic summary) and read for the
   topic. Do not stop at three because three felt like enough; stop when the list is done.
3. Answer from what the summaries say, citing call and date; go back to `format="full"`
   only for a call an exact quote must come from.

If the window is empty, say so and offer the most recent calls instead of quietly
substituting them.

## After a clarifying question

When you asked "which Ben?" and the user answered with a name, run
`resolve_prompt_context` again on that name before anything else. The map for the
clarified name carries the CALLS ON RECORD step; the first map did not, because the name
was ambiguous. Then `contact_profile` — its `recent_calls` lists their calls, newest
first, with tags — and read the calls the question is about.

## Choosing the entry point

| The question | Tool |
| --- | --- |
| "when did X come up" / topic across calls | `search_transcripts` |
| "what calls have we had with Jane" | `contact_transcripts_list` (needs contact IDs) |
| "what happened on that call" | `call_transcript_conversation` |

`contact_transcripts_list` takes **integer contact IDs**, not names — resolve through
`contact_profile` or `resolve_prompt_context` first. It returns metadata only, no
transcript bodies.

## Searching

`search_transcripts(query, limit, filter_by, sort_by)`:

- `query` — semantic + keyword. Empty browses without relevance ranking.
- `filter_by` — filter grammar, applied **before** ranking. Only five fields exist:
  `name`, `participant_names`, `business_names`, `call_date` (a date as `YYYY-MM-DD`,
  or unix seconds), `tags`. Every response carries `today` and, when you filtered by
  `call_date`, `call_date_window` with the dates actually searched; check it against
  the period the user asked about before reporting that there were no calls.
  Inventing a field name errors out rather than being ignored.
- `tags` is what kind of call it is: `Sales`, `Internal`, `Onboarding`,
  `Instructional`, … A question about prospects or customers means `tags:Sales`; a
  question about "my calls" in a window usually means `tags:!=Internal`. Every result
  carries its `tags`, so say which calls you left out and why.
- `sort_by` — sort grammar. Defaults to `call_date:desc` on an empty query,
  relevance otherwise.

Filter when you know the constraint (a named account, a date range); let relevance rank
when you don't.

## Reading

`call_transcript_conversation(transcript_id, include_structured, format)`:

- **`format="summary"`** when scanning several calls. Pulling three or four at `"full"`
  buries the question in raw dialogue.
- **`format="full"`** for a single deep dive, or when exact wording matters — an
  objection, a commitment, a number.
- `include_structured=True` adds per-sentence timing and speaker structure. Only with
  `format="full"`, and only when you need to attribute lines to speakers.

## Reporting

Cite which call each finding came from — call name and date. "They pushed back on
pricing" is worth little; "on the Feb 12 call, Dana said the $40k figure would need
CFO sign-off" is actionable.

Distinguish what was said from what it implies. And when the search returns nothing, say
the calls don't cover it rather than reasoning from general knowledge about the account —
the entire premise of the question was the user's own call data.
