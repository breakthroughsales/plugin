---
name: import-contact
description: >-
  Adds a NEW person to Breakthrough from a LinkedIn profile URL or an email address,
  then starts enrichment to fill in their role, employer, and background. Requires a
  URL or email — a name alone is not enough to import from.
when_to_use: >-
  Use when the user supplies a LinkedIn URL or email address and says "import", "add",
  "pull in", "get this person into the system", "track this guy", "add her to
  Breakthrough". Do NOT use for someone already in the system whose details have gone
  stale; that is refresh-contact. Do NOT use to check whether they already exist; that
  is find-contact.
---

# Import a contact

Wraps the `import_contact` MCP tool.

## Before calling

**Check whether they already exist.** Run the find-contact skill first. Importing
someone already in the CRM enqueues enrichment work for nothing and can create a
duplicate the user then has to clean up.

**You need a LinkedIn URL or an email address.** `query` alone is a weak signal for
import — the tool imports from an identifier, not from a name. If the user has neither,
say so rather than importing something approximate.

## Calling

`import_contact(linkedin_url=None, email=None, query=None)` — pass whichever identifier
you have.

This tool is annotated **destructive** and Claude Code will prompt before it runs. That
prompt is correct; do not tell the user to pre-approve it.

## What it does

- Queues background enrichment against external data sources
- Waits for the contact to be created, for up to **~90 seconds**, and returns then
- The fit result keeps generating after it returns: `contact_without_fit_generating`
  means it is still running; `contact_profile` shows it a minute or two later

A pause is expected. Don't retry a call that seems slow — you'll queue a second import.

## Reporting

Report the returned status and the contact if one was created.

Enrichment continuing after the tool returns is normal. Say the import started and that
data will fill in, rather than presenting a partially-enriched record as final.

If the result is `{"status": "skipped", "reason": "missing_or_invalid_license_context"}`,
nothing was imported — that's an auth failure. Tell the user to authenticate.
