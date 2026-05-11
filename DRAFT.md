# Draft: future CLI implementation

> This file captures plans for the `autoutlook` *binary* that doesn't exist
> yet. The agent contract in [`AGENTS.md`](AGENTS.md) is the live half of
> the project; agents currently fall through to the Playwright path on
> every task. This draft becomes relevant when there's enough recurring
> work to justify codifying it.

## When this draft becomes worth executing

Build only when **at least one of these** is true:

- A specific email task has been done 3+ times the same way → it's stable
  enough to encode.
- Token-spend on ad-hoc Playwright drives is becoming noticeable.
- You want to shell-script email operations (`autoutlook unread | …`).
- Microsoft Graph auth gets unblocked (suddenly the whole thing is much
  simpler — see "Auth options" below).

Until then, the agent contract + Playwright + outlook.office.com handles
everything without ceremony.

## Planned commands

Roughly in priority order; only build the ones you'll actually use.

| Command | Purpose | Mutating? |
|---|---|---|
| `auth` | One-time sign-in; saves token / state | no |
| `list [--folder INBOX] [--top N] [--unread]` | List recent messages | no |
| `read <id>` | Fetch full body (text + attachments-meta) | no |
| `search '<query>'` | Outlook search syntax | no |
| `draft --to … --subject … --body …` | New draft → opens for review | no (draft only) |
| `reply <id> --body …` | Threaded reply → opens for review | no (draft only) |
| `send <draft-id>` | Send a previously-saved draft | yes — confirm |
| `archive <id>` | Move to Archive | yes — confirm |
| `delete <id>` | Move to Deleted Items | yes — confirm |
| `move <id> <folder>` | Move to a folder | yes — confirm |
| `mark-read <id>` / `mark-unread <id>` | Toggle read state | yes — confirm |
| `categorize <id> <category>` | Add/remove Outlook category | yes — confirm |
| `doctor` | Smoke-test every command on the user's inbox | no |

Output convention: one record per line, machine-parseable (JSON or
tab-separated). Human-readable rendering is the agent's job.

## Auth options

### A. Microsoft Graph API (currently blocked at author's tenant)

- Public client (Microsoft Graph CLI Tools, `14d82eec-204b-4c2f-b7e8-296a70dab67e`)
  with MSAL device-code flow. Blocked on tenants that disable third-party
  user consent (we hit `AADSTS65001`).
- Self-registered app: blocked on tenants that also disable user-level app
  registration. We tried, no luck.
- Tenant admin consent for the public client: ask IT. Slow / refusal risk.
- Token cache: `~/.config/microsoft-auth/graph-token.json`.

### B. Playwright + Outlook web (always works)

- Same `~/.config/microsoft-auth/state.json` already used by
  `sit-vpn-auth` and `globalunprotect-auth`.
- Per the agent contract rule #3, **always headed (`headless=False`)**.
  No exceptions.
- Brittle wrt UI changes — use semantic selectors (role / aria-label /
  visible text) for everything.

## Implementation order

When/if we build:

1. **Phase 0 — scaffolding.** uv-script in `bin/autoutlook`. argparse with
   subcommands. `doctor` smoke-test runs no commands yet but exits 0.
2. **Phase A — read-only.** `auth`, `list`, `read`, `search`. No mutations
   means low risk; can ship and validate auth + parsing without scary
   side effects.
3. **Phase B — drafting.** `draft` and `reply`. Open the draft, never send.
   Pairs with `automail` conceptually (one tool drafts in Mail.app, the
   other in Outlook).
4. **Phase C — mutations.** `send`, `archive`, `delete`, `move`,
   `mark-read`, `categorize`. Each requires explicit per-action
   confirmation in the agent contract.
5. **Phase D — composition.** Pipe-friendly output, glob-style selectors
   (`--match-subject 'IT ticket *'`), batch helpers.

Don't skip phases. The mutating commands should never land before
read-only is rock solid.

## Test discipline (when there's code)

- Every command ships with at least one test against a fixture mailbox
  (recorded responses, not live).
- `doctor` runs all command smoke-tests against the user's real inbox
  in read-only mode — used to spot Outlook UI regressions early.
- CI nightly runs `doctor` and notifies the user on failure.

## Hand-off from this draft

When you want to start, the canonical first step is: create `bin/autoutlook`,
make it a uv-script with `auth` and `list` commands, validate against your
inbox, commit, push. From there each phase is a separate PR. Update this
file or delete it as needed.
