# Breakthrough

Connects your AI assistant (like Claude, Codex or Cursor) to a living playbook distilled
from your sales calls, so it always has the right context and knows what works best across
your organization, without you having to manage any of it yourself.

Everything is scoped to your own Breakthrough organization. The plugin reads your data and
can add or update contacts when you ask; it never posts, sends, or publishes anything on your
behalf.

## Install

The plugin is distributed through the public GitHub marketplace repository
[`breakthroughsales/plugin`](https://github.com/breakthroughsales/plugin). Adding that
marketplace is normally a per-account step, so each person does it once — a teammate having
it does not put it on your account.

The exceptions are a **ChatGPT workspace**, where an admin can import the marketplace once for
everyone (see below), and a **Cursor team**, where an admin can make it available to the whole
team.

Skills and tools install together wherever the host supports plugins — Claude, ChatGPT/Codex
and Cursor. Claude works either way, desktop or web. ChatGPT has one catch: **chatgpt.com in
a browser cannot add a marketplace**, so use the ChatGPT desktop app, which installs against
your account and so shows up in the browser too. The Codex CLI can also install it, but that
install is local to that machine and does not reach your account or the browser.

### Claude Desktop and claude.ai

1. **Customize** in the left nav, then the **Plugins** tab.
2. **Add** (top right) → **Add marketplace**.
3. **Add from a repository** (not **Browse Anthropic sources**).
4. In **URL**, enter `https://github.com/breakthroughsales/plugin`, then **Sync**.
5. Breakthrough installs from the marketplace on its own, which takes a few seconds. If it
   has not appeared under **Yours** after a minute, open **Discover** and click **Add** on
   the Breakthrough card.
6. In Claude Desktop, quit and reopen the app so the change takes effect.

To check which release you are on, **Add** → **Manage marketplaces** shows the exact commit
it last synced. Compare it against the newest commit on
[`breakthroughsales/plugin`](https://github.com/breakthroughsales/plugin/commits/main).

### Claude Code

```bash
claude plugin marketplace add breakthroughsales/plugin
claude plugin install breakthrough@breakthrough
```

Or interactively: `/plugin marketplace add breakthroughsales/plugin`, then `/plugin`.

### ChatGPT and Codex

**For a whole workspace, an admin does this once.** At **chatgpt.com/admin → Marketplaces →
Add → Import marketplace**, enter
`https://github.com/breakthroughsales/plugin` and leave **Path** and **Branch** empty. Every
member then gets the plugin under the tab named after the workspace, with no setup of their
own. The import re-syncs once a day; the gear on the row has **Sync now** for when you need
a release immediately.

**For one person,** use the **ChatGPT desktop app** — the Codex section of it, which is where
plugins live now. The browser at chatgpt.com can only browse already-installed plugins; it
has no marketplace option.

1. **Plugins** in the left nav.
2. **Add** (top right) → **Add a marketplace**.
3. In **Source**, enter `https://github.com/breakthroughsales/plugin`, leave **Git ref** and
   **Sparse paths** empty, then **Add marketplace**.
4. Open the **Breakthrough** tab, click **Breakthrough**, then **Install plugin**. Adding the
   marketplace only registers it; this step installs the plugin.
5. Quit and reopen the ChatGPT desktop app so the change takes effect.

Or from the Codex CLI, for that machine only — both commands are needed, the first just
registers the marketplace and leaves the plugin uninstalled:

```bash
codex plugin marketplace add breakthroughsales/plugin
codex plugin add breakthrough@breakthrough
```

`codex plugin list` confirms it installed. Interactively, `/plugins` opens the same browser
with marketplace tabs to switch sources.

This route writes to the local `CODEX_HOME` and nothing else. Only OpenAI's own remote
marketplaces install against your ChatGPT account, so a plugin added this way works in that
CLI and will not appear on chatgpt.com — use the desktop app if you want it there.

Codex reads the same marketplace file this repository already publishes, so the skills and
the MCP server install as one unit, the same as on Claude.

**Browser only?** Add the MCP server as a connector instead and you get the tools without the
skills: **Settings** → **Connectors** → **Advanced** → **Developer Mode**, then **Create**
with `https://mcp.breakthroughsales.io/mcp` and **OAuth**. Requires a paid plan (Plus, Pro,
Business, Enterprise or Edu).

### Cursor

1. **Customize** in the left nav, then **Plugins**.
2. **+ Add** → **From GitHub Repository**.
3. In **Repository**, enter `https://github.com/breakthroughsales/plugin`.
4. Set **Scope** to **User** for just you, or **Team** to make it available to your Cursor
   team, then **Import**.
5. Open **Breakthrough** and click **Add to Cursor**.
6. Quit and reopen Cursor so the change takes effect.

Cursor bundles rules, skills, subagents, commands, MCP servers and hooks into one
installable package, so the skills and the MCP server arrive together as they do on Claude
and Codex.

### First run

Whichever host you use, it connects to Breakthrough automatically once enabled; the first
request opens a browser window to sign in. If you have already added Breakthrough as a
custom connector, the plugin reuses that existing connection rather than adding a second
one.

## Skills

Your assistant selects these from your request, so you rarely need to type one. Command
names differ by surface: in Claude Desktop and claude.ai they are unprefixed
(`/draft-email`), while Claude Code namespaces them per plugin
(`/breakthrough:draft-email`).

| Skill | What it does | Backing tools |
| --- | --- | --- |
| `gather-context` | shared prelude: resolve entities, pull evidence and playbook | `whoami`, `resolve_prompt_context`, `contact_profile`, `business_profile`, transcript tools, `sales_playbook` |
| `draft-email` | writes a sales email | via gather-context |
| `draft-linkedin-message` | writes a LinkedIn connection request, InMail, or DM | via gather-context |
| `draft-note` | writes up a call — notes, debrief, MEDDPICC, BANT | via gather-context |
| `answer` | answers questions needing judgment across the data | via gather-context |
| `humanize` | strips the patterns that make writing read as machine-generated; runs automatically on every outbound draft | none |
| `find-contact` | looks up one person | `contact_profile` |
| `find-business` | looks up one company | `business_profile` |
| `research-transcripts` | searches and reads call transcripts | `search_transcripts`, `contact_transcripts_list`, `call_transcript_conversation` |
| `import-contact` | adds a new person | `import_contact` |
| `refresh-contact` | re-pulls an existing person's details | `refresh_contact` |

Because the assistant selects skills by comparing their descriptions, each description carries
literal trigger phrasings and explicit hand-offs to its siblings — a description that only
describes itself competes with every neighbour.

`gather-context` is the exception: it sets `disable-model-invocation: true`. It's a
subroutine the drafting and answering skills run first, not something a user asks for,
and leaving it auto-selectable made it compete with all nine others on every
prospect-related request. You can still run it directly to preload an account before a
working session.

## What it does not do

This brings Breakthrough's **reasoning** to your assistant against the same production
data. It is not a second Breakthrough client. It cannot write into Breakthrough chat threads, or
create and save Breakthrough Documents. Drafts go to your conversation or to local files.

Web search uses the assistant's own built-in search.

**Output is Markdown for every document type**, including emails, LinkedIn messages, and
notes, which the app renders as HTML. See the deviation note in
[`references/output-contracts.md`](references/output-contracts.md).

## Authentication

**Normally** you sign in through the browser on first use. Nothing to configure.

**Without a browser** — for automated or headless use, create an API key on the MCP API
keys page in Breakthrough and register the server yourself:

```bash
claude mcp add --transport http --scope user breakthrough https://mcp.breakthroughsales.io/mcp --header "Authorization: Bearer $BREAKTHROUGH_MCP_API_KEY"
```

Keys are scoped to a single Breakthrough seat and can be revoked from the same page.

## Troubleshooting

**"Repository not found" when adding the marketplace.** Check the owner. It is
`breakthroughsales/plugin`; `breakthrough/plugin` does not exist.

**A teammate has it and you don't see it.** Adding a marketplace is normally per account,
not per company, so add it yourself with the steps above. The exceptions are a ChatGPT
workspace or Cursor team whose admin has added it — there it arrives without you adding the
marketplace yourself, and if it has not, ask the admin rather than adding your own copy, since
two sources registering a plugin of the same name collide.

**"Failed to add marketplace" in Claude Desktop.** Usually stale local state, not a problem
with the repository. Claude Desktop caches the marketplace list and does not notice changes
made on claude.ai until it restarts, so it can refuse an add that collides with a
registration the server no longer has. Quit and reopen Claude Desktop, then look before you
add — the plugin may already be there. Marketplaces are stored per account, not per device,
so adding on either surface is enough for both.

**ChatGPT or Codex shows an older version than the repository.** A ChatGPT workspace can
import this marketplace itself, and that import **syncs once a day** — so the workspace
trails every release by up to 24 hours. A workspace admin can force it:
**chatgpt.com/admin → Marketplaces → the gear on the row → Sync now**.

To tell which copy you have, open `~/.codex/plugins/cache/`. A plugin under
`workspace-directory/` came from your workspace import — it carries a `remote_plugin_id`
in `.codex-remote-plugin-install.json` and a generated `.codex-plugin/` directory that this
repository does not ship. A plugin installed from the marketplace directly is a plain git
clone under `~/.codex/.tmp/marketplaces/`, whose `.codex-marketplace-install.json` names the
source repository and the exact revision. The two are independent: updating one does not
update the other. Both can sit in the caches at once, but they do not both take effect —
they register the same plugin name, so the second one added is the one that appears to do
nothing. Keep one source, which is why a workspace member should ask their admin rather
than adding a direct copy.

**A ChatGPT workspace import stops syncing after the source repository is renamed.** It
fails with `GITHUB_FETCH_FAILED` and the row shows "Needs attention". GitHub's rename
redirect does not save you here: `git` and `raw.githubusercontent.com` follow it, but the
REST API answers `301` and this importer does not follow that. Waiting will not fix it —
the import has to be deleted and re-added against the current name. There is no field to
edit the source in place.

**"Source manifest was not found at any supported path" when importing.** Leave **Path**
and **Branch, tag, or commit** empty. "Repository root" is the label an existing import
displays for an empty path, not a value to type — typing it makes the importer look under a
literal `Repository root/` directory and every manifest path misses.

**`421` from `/mcp` while `/health` returns `200`.** Not an auth failure — the request
is being rejected before it reaches authentication. Report it to the Breakthrough team
rather than regenerating tokens.

**"Couldn't register with Breakthrough's sign-in service."** The client tried to register
itself with our sign-in service, which we do not allow — it should be using the client id the
plugin ships. Plugin 0.20.2 and later declare it, so updating fixes this. To connect without
updating, or when adding the MCP server on its own, open the connector's settings and enter
`breakthrough-mcp` as the **OAuth Client ID**; leave the client secret empty, there is none.

**Every tool returns `{"status": "skipped", "reason": "missing_or_invalid_license_context"}`.**
The session is not authenticated against Breakthrough. Confirm with `whoami` — the
returned `license_id` will be missing.

**Changed a skill and nothing happened.** Skill edits apply immediately. Changes to the
plugin's server or manifest configuration need `/reload-plugins` or a restart.

## Reporting problems

Send the failing request and what you expected to the Breakthrough team. Skill and tool
behaviour is versioned with the plugin, so include the version shown on the plugin's page.
