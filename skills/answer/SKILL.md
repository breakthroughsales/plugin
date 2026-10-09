---
name: answer
description: >-
  Answers any sales question for this user's company: how we sell, what we charge, how
  we position, what to ask a buyer, how to handle an objection, how we compare to a
  competitor, what to do next on a deal — plus open-ended questions about a prospect,
  account, or call. Answers come from the company's own playbook and sales data
  (contacts, companies, call transcripts), which only this skill can reach. The
  fallback for any Breakthrough request that isn't a record lookup, including writing
  an email, a message, or call notes.
when_to_use: >-
  Use for ANY question about selling that the user's company would answer differently
  from a generic one — even when no person, company, or call is named: "what's our
  pricing", "give me 3 discovery questions for a CFO", "how do I handle the pricing
  objection", "how do we position against X", "what's our elevator pitch", "what should
  I do next here", "help me prep for this call", "should we chase this". The company
  keeps a playbook for exactly these; answering them from general sales knowledge is
  the failure this skill exists to prevent. Do NOT use to pull up one person or company
  record; that is find-contact or find-business. Do NOT use for "what did they say on
  the call"; that is research-transcripts. Also use when the user wants something
  written from their calls or records — "write a follow-up email to Jane after our
  call", "draft a LinkedIn message to him", "write up yesterday's call as a MEDDPICC":
  load the context here, then write it the way you normally would.
---

# Answer a question

The default for questions, general chat, brainstorming, and explanations.

## Steps

**1. Entities — only when named.** When the question touches a person, company, or
call the user knows, follow `${CLAUDE_PLUGIN_ROOT}/skills/gather-context/SKILL.md` in
full. Its last step calls `sales_playbook` with the ids it just resolved — that is the
playbook step for this path, so do not call `sales_playbook` again afterwards. Order
matters: on an org with more than one playbook, the resolved call's sales/partnership
tag is the strongest signal for which playbook applies, and it is only available
after resolution. Skip this step entirely for questions that reference none of the
user's data.

**2. Playbook — when nothing was named.** If step 1 did not run, call
`sales_playbook` directly with the user's message verbatim as `latest_user_message`
and the conversation so far as `conversation_history`. Do this for every such
question: the native chat runs this retrieval on every turn, and the tool decides for
itself when nothing applies (it returns `skipped`, which costs nothing). "What's our
pricing?" and "give me 3 discovery questions for a CFO" are playbook questions even
though they mention no one.

Either way, exactly one `sales_playbook` call per question.

For questions about the user themselves — their background, experience, or how to
position who they are — call `whoami(detail="full")` rather than the default
`whoami()`. The default returns bare scope; `detail="full"` adds their LinkedIn
headline, summary, experience, and education. It reads cached data only, so it is
cheap and never prompts.

**2. Answer to the contract** in
`${CLAUDE_PLUGIN_ROOT}/references/output-contracts.md`:

- No hard length cap, but the conciseness rules bind. Conversational questions get
  conversational answers — typically 1–5 sentences.
- Answer, then stop. No recap of the question, no "additional considerations" section,
  no "let me know if you need anything else."
- State each point once.
- Markdown.

## Writing something

When the user asks for an email, a LinkedIn message, or call notes, gather the context as
above, then write it in your usual format. If it is about a call (notes on it, a
follow-up to it), read that call with `call_transcript_conversation` first; listing the
call or reading `<PastCallNotes>` is not reading it. Every specific in the draft should
come from what you retrieved; leave a placeholder for anything you don't have, such as
the user's availability. Nothing is sent: the user sends it.

## Web search

Use Claude Code's `WebSearch` / `WebFetch` under the rules in
`${CLAUDE_PLUGIN_ROOT}/references/context-assembly.md`. The one that gets violated most:
**do not web-search a question about a call the user had.** Pull the transcript. If the
transcript isn't in scope, say so and offer to find it — substituting a web search for
the user's own call data is the specific failure the system prompt calls out.

## Grounding

Distinguish what the data shows from what you're inferring. "They raised a Series B in
March" and "they're probably budget-constrained this quarter" are different claims and
the user is making decisions on both.

If the question can't be answered from what's in scope, say what's missing and what would
answer it.
