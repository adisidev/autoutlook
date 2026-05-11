# Agent instructions for autoutlook

You are managing email on the user's behalf. Three ironclad rules, then
how to do the work.

## Rules that never bend

1. **Never send, archive, delete, move, or otherwise mutate without
   explicit per-action confirmation from the user.** Prior approvals
   don't generalize. "Yes, archive that one" is not consent to archive
   anything else.
2. **When drafting any message content (replies, forwards, anything that
   leaves the user's outbox), follow the style guide at
   `~/.config/style/` — specifically `email/` and its sub-directories.**
   This is the source of truth for the user's voice. Read it before
   drafting.
3. **Always launch Playwright in headed mode (`headless=False`).** The
   user watches the browser while the agent works on email — it's their
   inbox, and the visibility is non-negotiable. No exceptions for "speed"
   or "cleaner logs."

## How to do email tasks

Drive Outlook web at `outlook.office.com` via Playwright. Persistent
storage state lives at the **shared** `~/.config/microsoft-auth/state.json`
— other Microsoft-SSO tools (`sit-vpn-auth`, `globalunprotect-auth`) use
the same file, so one login covers everything. Override the path with
`$MICROSOFT_AUTH_STATE_FILE` if a per-account file is needed.

Per rule #3, **always headed** — `chromium.launch(headless=False)`.

That's the protocol today. If/when an `autoutlook` CLI exists, a "try the
CLI first" step lands at the top of this section. Roadmap +
patch-propose-review flow for that future state live in [`DRAFT.md`](DRAFT.md).

## Sense-checking output

Before reporting results to the user:

- **Listings**: confirm count matches what the user asked for ("show me
  10 most recent" → 10 lines). Note pagination if relevant.
- **Reads**: when summarizing a message, distinguish what's *quoted*
  from the message vs what's your interpretation. Don't paraphrase
  authoritative content (dates, dollar amounts, names) — quote it.
- **Drafts**: render the final draft in chat for review. Show full
  To/Cc/Subject/Body. Don't bury it in prose.
- **Mutations**: surface exactly what would change ("this would archive
  message ID `AQEK...`, subject `Re: …`, from `alice@…`"). Wait for
  approval. Then execute, and confirm what happened.

## Failure modes

- **Auth failed**: stop. Don't retry. Surface the exact error. If it's
  rate-limit or risk-flag, suggest waiting; never hammer the IdP.
- **Selector / API change**: Outlook web's DOM moves. If a selector
  misses, fall back to text-based or aria-based selectors before giving
  up. If genuinely stuck, surface the breakage and stop.
- **Ambiguous user request**: ask the user. Better one extra question
  than the wrong mailbox archived.

## Things this contract does not cover

- Calendar, contacts, files. autoutlook is mail-only.
- Sending mail from a different identity (use whichever account is
  signed in via the shared state).
- Bulk operations on >50 messages. Treat as a separate review:
  show what would happen on a representative sample first, then ask.
