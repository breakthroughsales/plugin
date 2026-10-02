# Changelog

All notable changes to this plugin are documented here. Format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/); this project uses
[Semantic Versioning](https://semver.org/).

The plugin is pre-release: versions stay `0.X.Y` — `### Added` / `### Changed` /
`### Breaking` bump X, `### Fixed` / `### Security` bump Y — until the 1.0.0 release is
cut by hand. (1.0.0–1.3.0 were published under those numbers by mistake on
2026-09-03/08 and renumbered to 0.6.0–0.9.0 on 2026-09-08.)

## [Unreleased]

## [0.20.2] - 2026-10-02

### Fixed

- Connecting no longer fails on a fresh install. `.mcp.json` now declares the OAuth client
  id (`breakthrough-mcp`, a public client using PKCE) and the scopes it needs, so the client
  has nothing left to discover. Without it, a client with no credentials falls back to
  dynamic client registration, which our sign-in service refuses by design — the first thing
  a new user did produced "Couldn't register with Breakthrough's sign-in service", with the
  client id published only on the website and only in the section for people connecting the
  MCP server without the plugin.

## [0.20.1] - 2026-09-25

### Fixed

- Install steps checked against each app's live dialogs and made explicit. The README now
  says the marketplace is a GitHub repository, uses its full URL in every app's form (Cursor
  rejects the `owner/repo` shorthand) and names the field each value goes in
  (Claude's **URL**, ChatGPT's **Source**, Cursor's **Repository**). Claude's **Add from a
  repository** choice gets its own step, Cursor gains the missing **Customize** hop, its
  **Scope** and **Import** steps and the final **Add to Cursor**, the ChatGPT workspace-admin path loses a **Plugins** hop
  that isn't there (it is **chatgpt.com/admin → Marketplaces**), and each desktop app now
  says to quit and reopen after installing.

- Cursor now shows the Breakthrough logo. Cursor reads `logo` from the marketplace entry rather
  than the Codex `interface` block, so the entry in `marketplace.json` now carries it too, and
  the validation script allows that one extra field.

- Cursor can sign in. Cursor tried to register itself with our login server, which refuses
  that by design, so its MCP connection failed with "Policy 'Trusted Hosts' rejected request".
  A Cursor-only manifest, `.cursor-plugin/plugin.json`, now tells Cursor to use the existing
  `breakthrough-mcp` client instead; Cursor desktop signs in on `http://localhost:8787/callback`,
  which that client already allows. Claude and Codex don't read that file.

- The ChatGPT desktop steps get back the **Install plugin** step after **Add marketplace**:
  adding a marketplace only registers it, and the earlier rewrite had dropped the install.
  Cursor's **Team** scope is described as making the plugin available to the team (the admin
  chooses whether it is optional, on by default or required) rather than installing it for
  everyone, and the overview and teammate troubleshooting now list a Cursor team alongside a
  ChatGPT workspace.

## [0.20.0] - 2026-09-25

### Added

- The plugin now carries the Breakthrough mark. `assets/icon.png` is the orange watercolour
  triangle the app uses in its own navigation, squared and centred at 512x512, wired up
  through an `interface` block that Codex and the ChatGPT catalogue read. That block carries
  every field Codex's plugin validator requires once an `interface` is present — display and
  developer names, short and long descriptions, category, capabilities and starter prompts —
  so a partial block cannot be rejected where no block at all would have passed. Claude has no icon field at all — its manifest schema
  has none and no official marketplace entry carries one — so on Claude the tile stays blank
  until the plugin is submitted to the Claude directory, which is where those icons come from.

### Fixed

- The install section no longer claims there is "no shared or organization-wide install".
  A ChatGPT workspace admin can import the marketplace once for the whole team, which is now
  documented as the team route, and the per-account statements are scoped rather than
  absolute — a workspace member should ask their admin rather than adding a second copy,
  since two sources registering a plugin of the same name collide.

- The plugin shows as "Breakthrough" rather than "breakthrough" in catalogs. `displayName`
  was set in `plugin.json` but not in the `marketplace.json` entry, and a catalog reads the
  marketplace entry — so claude.ai (which reads the installed plugin's own manifest) got it
  right while ChatGPT fell back to the bare `name`.

- Claude install steps corrected against the live UI after reinstalling from the renamed
  repository. The plugin installs from the marketplace on its own with a short lag, so the
  Discover → Add step is now the fallback for when it does not, not a required step. Removed
  the instruction to leave "Sync automatically" on — no such control exists in claude.ai;
  **Add → Manage marketplaces** shows the synced commit instead, which is what actually
  tells you whether you are on the latest release.
- Troubleshooting covers "Failed to add marketplace" in Claude Desktop: it caches the
  marketplace list and does not see changes made on claude.ai until restarted, so it can
  refuse an add that collides with a registration the server no longer has. Restart first,
  and look before adding — marketplaces are per account, not per device.
- Troubleshooting covers ChatGPT/Codex showing an older version than the repository: a
  ChatGPT workspace marketplace import syncs once daily, so the workspace trails a release
  by up to 24 hours until an admin uses Sync now. Also says how to tell a workspace-imported
  copy from a directly installed one, since both can be present under the same plugin name
  and neither updates the other.
- Troubleshooting covers two failures found while re-pointing our own ChatGPT workspace at
  the renamed repository: an import breaks permanently when the source repo is renamed
  (`GITHUB_FETCH_FAILED` — the REST API answers 301 and the importer does not follow it,
  unlike git and raw.githubusercontent.com), and it must be deleted and re-added rather than
  edited; and leaving Path and Branch empty on import, since "Repository root" is a display
  label rather than a value to type.

## [0.19.0] - 2026-09-25

### Changed

- The public repository is now `breakthroughsales/plugin`, renamed from `claude-plugin`.
  One repository serves every assistant: Codex reads the same marketplace file Claude does,
  so the old name was misleading for anyone not on Claude. GitHub redirects the old name,
  and nothing should ever be created at it — that would break the redirect for existing
  installs.
- Description is now assistant-neutral ("your AI assistant, like Claude, Codex or Cursor"),
  since the same package installs on all three from one repository.
- README install steps are the verified click paths, not paraphrases: Claude
  (Customize → Plugins → Add → Add marketplace → Add from a repository → Sync), ChatGPT and
  Codex (desktop app → Plugins → Add → Add a marketplace) and Cursor (Plugins → + Add →
  From GitHub Repository).
- ChatGPT has one surface limit worth stating plainly: chatgpt.com in a browser has no
  marketplace option at all, and the **desktop app is the only route to a plugin that shows
  up in the browser**, because that install is scoped to your ChatGPT account.
- The Codex CLI is documented as a separate, machine-local route. It needs both
  `codex plugin marketplace add` and `codex plugin add` — the first only registers the
  marketplace and leaves the plugin uninstalled — and it writes to the local `CODEX_HOME`,
  so it never reaches your ChatGPT account or chatgpt.com.
- Antigravity dropped — untested, and not a surface we support.

## [0.18.0] - 2026-09-23

### Added

- `refresh_contact` records where the user SAYS someone works, under confirmation. Pass
  the company as `employer` ("Mauricio works at Autopistas del Café", "he's advising
  RudderStack") instead of re-pulling LinkedIn, which is the source that was wrong. The
  tool asks one question at a time — which company, whether our record was wrong or they
  moved, from when — shows exactly what will change, and writes only after the user agrees
  and the `apply_token` it issued comes back. A company the person LEFT, or one named in a
  question, is never an employer. A company not on file is looked up by name, so the user
  only confirms what was found (or gives the website when nothing was); `employer_website`
  carries that web address so `employer` stays the company's name, and `contact_id` picks
  between people who share a name. A company added this way is enriched like one added in
  the app.

### Changed

- `refresh-contact` skill: its description now leads with recording an employer the user
  states, not only re-pulling LinkedIn — claude.ai shows the model a skill's description
  and nothing else, so a statement like "Marco works at Kleecks, I just met him" was going
  to `find-contact` and never reaching the tool. The body no longer says a stated employer
  "returns needs_confirmation"; it walks the confirmation steps instead.

## [0.17.0] - 2026-09-18

### Added

- `sales_methodology`: one company's sales-methodology scorecard (MEDDPICC, or the org's
  own methodology) — how the deal is being run against the team's process. Each category
  shows its coverage, what is not yet covered, and the lead of its evidence (who, what was
  said, the risk); `categories` expands the full evidence for the parts of the deal a
  question is about. `business_profile` flags a company that has one (`sales_methodology`).
  `gather-context` reads it when the request is about the deal, and a new coaching
  reference says how to use it: gaps first with their evidence, the as-of date, the stage
  inferred from the assessment. Drafting and call notes do not use it unless asked.

## [0.16.0] - 2026-09-17

### Changed

- Tool reference: dropped `health()`. The MCP server no longer registers it — it had
  zero calls in 30 days of production and the server already answers `GET /health` for
  liveness, so the tool only cost a slot in every client's tool list. Nothing called it;
  use `whoami()` to check that auth and license context are working.

## [0.15.0] - 2026-09-15

### Changed

- `gather-context` and the tool reference: an ambiguous name is no longer an automatic
  question. The resolver already eliminates candidates the request's other words rule
  out; shared call history is now the last rung of that same narrowing, so a name where
  only one candidate has ever been on a call resolves and the reply states the
  assumption, naming the alternatives and how old that history is. "my call with
  Venkat" matched two people, one never spoken to, and asked anyway. Names where
  several candidates have calls still ask — four of twenty-seven Bens, two of eleven
  Marcos — which is the case the guard was added for.
- `call_transcript_conversation` no longer returns the attendees' business and contact
  descriptions on either format, matching what chat has always done, and `summary` no
  longer returns the thematic summary twice. Measured across nine calls: summary reads
  69% smaller, full reads 39% smaller. Read those descriptions from `business_profile`
  / `contact_profile` when you need them.
- `contact_profile` clarification candidates now carry `transcript_count`, which
  `resolve_prompt_context` has always reported and this tool did not.

## [0.14.0] - 2026-09-14

### Changed

- `research-transcripts` and the tool reference: `search_transcripts` now takes
  `call_date` filters as `YYYY-MM-DD` dates and every response reports `today` plus the
  `call_date_window` actually searched. In Measured's prod logs 5 of 16 date windows
  had been computed a year early and reported as "no calls that week".

## [0.13.0] - 2026-09-11

### Changed

- `refresh-contact` and `find-contact` skill descriptions now say that a contact's own
  Breakthrough record is available for "is X still at Y" and other named-person
  questions. claude.ai shows the model only the `description` field (not
  `when_to_use`) and never the MCP server instructions, so the routing wording has to
  live there.

## [0.12.1] - 2026-09-10

### Fixed

- `import_contact` given only a name that matches a company on record now returns that
  company in `businesses` alongside `invalid_input`, so "add Glean as a company" is
  answered from the record instead of asking for a domain.

## [0.12.0] - 2026-09-10

### Changed

- `refresh_contact` takes `user_requested`. Claude states whether the user asked for the
  refresh outright (or just confirmed one); the tool no longer reads that out of the
  wording of `query`, which MCP clients never sent — every "refresh X" over the plugin
  stalled on a confirmation question. The refresh-contact skill and the server
  instructions say when to set it.

## [0.11.1] - 2026-09-09

### Fixed

- `import_contact` given only a name now checks the org's existing contacts first and
  returns `already_on_record` (or `ambiguous_name` with candidates) instead of asking for
  a URL for someone who is already in Breakthrough. An email-only import is held to the
  name the address carries, so a vendor answer for a different person is rejected rather
  than imported.
- find-contact and refresh-contact now claim "where does X work", "is X still at Y",
  "did X leave" and similar currency questions, and say Breakthrough's record comes
  before a web search. On claude.ai such questions were answered from the web with the
  plugin never consulted.

## [0.11.0] - 2026-09-09

### Added

- `contact_profile` and `business_profile` return `recent_calls`: that person's or
  company's calls, newest first, with each call's `tags`. Resolving an entity now puts its
  calls in front of the model the way the app shows the call list when an entity is pulled
  in. The skills say when to read them (a question about past calls reads the thematic
  summaries; up to 3 calls in full) and to re-run `resolve_prompt_context` on the name the
  user gives after a clarifying question.
- Call `tags` ("Sales", "Internal", "Onboarding", …) on every transcript in
  `search_transcripts` and `contact_transcripts_list` results, and as a `filter_by` field
  (`tags:Sales`, `tags:!=Internal`). Calls recorded before this release show their tags
  once the search index has been refreshed. The research-transcripts skill
  now handles "a topic across a window" by bounding the set (date + tags) and reading each
  call's thematic summary, because the search index holds names and imported summaries,
  not what was said.

### Fixed

- `resolve_prompt_context` probes a multi-word person's surname on its own as well as
  the whole name, so a nickname or a misspelt first name ("Rob Jackson" for Robert
  Jackson) still finds the person. A surname that came from a named person is not an
  incidental word hit, and a named first name that starts the candidate's first name
  ("Rob" for Robert) counts as evidence, so one Robert among eight Jacksons resolves
  while two Mark Jacksons still get a question.
- When a resolved contact has calls on record, the contact recommendation no longer
  calls the entity notes "the primary source" — the notes are context; the call is the
  source for what was said, and the **CALLS ON RECORD** step carries the open-the-call
  instruction. In 3 of 82 harness runs the old wording stopped the model at the notes.
- Recommendations and skills call the `<PastCallNotes>` block what it is: the entity notes
  (what we extracted about a person or company from every past call), not a summary of any
  call. The transcript ceiling is stated as a rule: full transcripts up to 3 calls, thematic
  summaries above that. The app's ceiling is 5; the plugin's is lower because everything it
  opens stays in Claude's context.
- The research-transcripts skill resolves the request through `resolve_prompt_context`
  before searching when a person or company is named, and asks when the name is
  ambiguous. Searching transcripts for "Ben demo" and reading the top hit picked one of
  two Bens with a demo on record silently.

## [0.10.2] - 2026-09-08

### Changed

- Marketplace listing and manifest description rewritten: the plugin connects Claude to
  a living playbook distilled from your sales calls, so it has the right context and
  knows what works across your organization. Listing copy only; no behavior change,
  hence a patch version rather than the minor bump a `### Changed` entry normally takes.

## [0.10.1] - 2026-09-08

### Fixed

- A shared first name blocks with a clarifying question only when it is the request's
  only handle. "Email Gianluca Peretti … tie it to Alex's directive" proceeds with
  Gianluca; the map still lists the Alexes and says not to pick one. A call the request
  names is searched by title (`matched_transcripts_by_title`) and recommended directly.
- `resolve_prompt_context` now has a small model read the request for the people and
  companies it names, and searches only those (`mentions`, `mention_source`). Before, every
  word was probed against the index, so "write a short email thanking Ioanna" surfaced a
  Tim Short on the word "short". The per-word probe remains as a fallback.
- `resolve_prompt_context` no longer guesses from wording whether a prompt refers to a
  past call; it reports the resolved person's calls on record (**CALLS ON RECORD**) and
  leaves that judgment to the model. A company or surname qualifier ("Ben at
  Splashtop") resolves a shared first name through an org-scoped lookup, and the
  calls-on-record counts include calls imported by colleagues.

## [0.10.0] - 2026-09-08

### Changed

- `resolve_prompt_context` now flags a first name that matches several people
  (`clarification_needed`, `ambiguous_names` with each candidate's calls on record)
  and its first recommendation says ask, do not pick. The gather-context skill stops
  and asks on that flag instead of guessing the person with the most calls.
- `call_transcript_conversation` returns long transcripts in parts of 60,000
  characters (`part`, `parts`, `next_part`, `total_chars`) and no longer duplicates the
  raw text beside the tag-wrapped copy unless `include_structured` is set. Single
  results of 94-163KB overflowed the client's tool-result cap and the call went unread.
- `resolve_prompt_context` sets `call_reference` when the prompt talks about a call
  ("our call", "my last call", "the demo") and recommends the exact transcript(s) to
  open; a full name in the prompt ("Ben Beal") is not flagged as ambiguous just because
  the first name alone matches several people.
- gather-context, draft-email, and draft-linkedin-message open the call when the
  request is about it (a thank-you, a follow-up to the demo) and read every part
  before writing; the stored call notes are a digest, not the transcript.

## [0.9.2] - 2026-09-08

### Fixed

- The marketplace manifest now declares the plugin version, so claude.ai and Claude
  Code can see that a new release exists. Without it, a marketplace added on claude.ai
  stayed on the version from the day it was added.

## [0.9.1] - 2026-09-08

### Fixed

- The publisher shown in claude.ai and Claude Code is now "Breakthrough" rather than
  "Breakthrough Engineering" (marketplace owner and plugin author).

## [0.9.0] - 2026-09-08

### Changed

- `answer` now fires on any question about how this company sells — pricing,
  discovery questions, positioning, objections, competitors — even when no person,
  company, or call is named. Before, its description only claimed questions "about a
  prospect, account, deal, or call", so "what's our pricing?" or "give me 3 discovery
  questions for a CFO" fired no skill at all and were answered from general sales
  knowledge instead of the org's playbook (measured: 5 of 27 turns on org 7, all of
  this shape). The skill now resolves people/companies/calls first when the question
  names one — so the resolved call's sales/partnership tag reaches `sales_playbook`,
  which matters on orgs with two playbooks — and otherwise calls `sales_playbook`
  directly on every question, the way the native chat does. Exactly one playbook
  call per question either way.


## [0.8.0] - 2026-09-06

### Added

- `sales_playbook` takes three optional arguments for saying what the conversation is
  about: `context_hint`, `call_transcript_ids` and `business_ids`. They matter on orgs
  that run more than one playbook — a sales playbook and a partnership playbook — where
  a question like "what should I cover on this call?" does not say which motion applies.
  Without them such a question could match nothing and come back
  `skipped: no_rendered_content`, leaving the answer with no playbook behind it and no
  sign anything was missing. Orgs with a single playbook ignore all three.
- The result can now carry a `clarification` field: the motion was not determined, the
  sections returned are the ones that hold under either motion, so use them and then ask
  the user that question rather than picking a motion silently.

## [0.7.0] - 2026-09-03

### Changed

- Expanded the guidance for `sales_playbook` so it is reached in the cases where it
  helps. It is now described as covering the org's playbook family, including a
  separate partnership playbook where one exists, and skills are told to call it
  before drafting anything a prospect or partner will read rather than only when a
  question is explicitly about methodology.

### Fixed

- `gather-context` told Claude to skip retrieval entirely for a request with no
  contact, company, or call in it. That skipped the playbook too, which is exactly
  what such a request needs — the playbook is org-level guidance and needs no
  entity. Entity lookups are still skipped; the playbook step no longer is.

## [0.6.0] - 2026-09-03

### Breaking

- Renamed the bundled MCP server from `api` to `breakthrough`. It is now identifiable in
  the connector list and in tool names. Any saved tool-permission rules referring to the
  old name need to be re-approved once.

### Changed

- Rewrote the install instructions. They now cover the two paths that actually exist —
  adding the marketplace in Claude Desktop / claude.ai, and the Claude Code CLI — and
  state that a marketplace is added per account rather than shared across a company.

### Fixed

- Added the `humanize` skill to the skill table. It shipped in 0.5.0 but was never listed.
- Documented that skill command names are unprefixed on Claude Desktop and claude.ai, and
  namespaced (`/breakthrough:<skill>`) only in Claude Code.
- Corrected the 0.2.0 entry below, which named a tool that was never released under that
  name.
- Pointed the manifest's repository link at this repository instead of a private one, and
  declared the license.

## [0.5.0] - 2026-08-12

### Added

- Humanize skill, run automatically on outbound drafts.

## [0.4.1] - 2026-08-12

### Changed

- Removed internal implementation details from plugin-facing docs.

## [0.4.0] - 2026-08-12

### Changed

- Applied agent-tool guidance to the MCP surface.

## [0.3.1] - 2026-08-12

### Fixed

- Split skill triggers into `when_to_use` for clearer selection.

## [0.3.0] - 2026-08-12

### Changed

- Rewrote skill descriptions for clearer selection.

## [0.2.0] - 2026-08-10

### Added

- The caller's own LinkedIn background, as `whoami(detail="full")`. (This entry
  originally announced a separate `my_profile` tool; no such tool shipped.)

## [0.1.0] - 2026-08-12

### Added

- Initial release of the Breakthrough Claude plugin.
