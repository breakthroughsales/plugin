---
name: find-contact
description: >-
  Looks up one person in Breakthrough by name, email address, or LinkedIn URL and
  returns their stored record — role, employer, linked company. Answers "is this person
  already in the system" and pins down which person the user means before acting. For
  a factual question about who a named person is, what they do, or where they work,
  this returns the user's own record of them.
  An open-ended "what do we know about" or "how should I approach" them is answer; so
  is writing them an email or LinkedIn message, or notes on a call with them; what they
  said on a call is research-transcripts.
when_to_use: >-
  Use when the user says "who is this", "do we have Jane", "look up jane@acme.com",
  "what's her title", "is he in the system", "which Jane do we know", "pull up her
  record", "where does Matt work". Also use for a factual question about someone the
  user sells to or works with (a contact, prospect or customer): Breakthrough holds the
  user's own record of them. Do NOT use to add
  someone new; that is import-contact. Do NOT use to update a record that has gone
  stale; that is refresh-contact. Do NOT use for what was said
  on calls with them; that is research-transcripts. Do NOT use for questions needing
  judgment about the person rather than their record; that is answer.
---

# Find a contact

Wraps the `contact_profile` MCP tool.

## Usage

Pass the user's phrasing as `query`. The tool extracts embedded email addresses and
LinkedIn URLs itself, so `"do we have anything on jane@acme.com"` works as well as
`"jane@acme.com"` — no need to strip the sentence down first.

`limit_candidates` defaults to 5. Raise it when a common name returns an ambiguous set.

When the request is vague about who is meant, call `resolve_prompt_context` first — it
cross-references contacts against businesses and returns concrete recommendations, which
disambiguates "the CTO at that fintech" better than a name search can.

## Reporting

Report what resolved: full name, contact ID, role, employer, LinkedIn URL.

**If several candidates come back, present them and ask.** Do not pick the top hit and
proceed — the downstream action is usually an email to a real person, and picking wrong
means writing to the wrong one.

**If nothing resolves, say so plainly.** "No contact matching that in Breakthrough" is
the first thing to say. If the user asked what *they* have on file, stop there. If they
asked a factual question about the person (where someone works, what their title is),
the web may still answer it: do so, but label it as not from Breakthrough, never as if
it came from the CRM. Either way, offer the import path (the import-contact skill) if a
LinkedIn URL or email is available.
