# autoutlook

A self-evolving CLI for managing email in Outlook, paired with an agent
contract that lets an LLM drive it and propose its own extensions when
the deterministic path falls short.

> **Current state**: docs only. No binary. The agent contract in
> [`AGENTS.md`](AGENTS.md) is the live half of the project — agents follow
> it and fall through to driving Outlook web directly via Playwright. That
> turns out to be enough for most use, so the CLI is deliberately deferred.
> When/if it becomes worth building, the roadmap is in [`DRAFT.md`](DRAFT.md).

## Architecture (the 80/20 self-evolving idea)

Three layers, each handling what it's best at:

1. **CLI** — deterministic, fast, cheap. Handles known operations
   (`list`, `read`, `archive`, `reply`, …). Each operation is a small,
   testable shell/Python function. Output is structured (JSON or
   newline-delimited) for easy chaining.
2. **LLM with Playwright or Graph SDK** — adaptive, slower, expensive in
   tokens. Used only when the CLI doesn't know how to do something
   (a new label scheme, a weird search, an inbox in another mailbox).
3. **Human sense-check** — every mutating action (send, archive, delete,
   move, mark-read) requires explicit user approval *for that specific
   action*. Prior approvals never generalize. Drafting respects
   `~/.config/style/` (see `~/.claude/CLAUDE.md`).

**The evolution loop**:

```
agent attempts task
  ├── CLI has a command for it    → run it → sense-check output → done
  └── CLI doesn't                  → agent uses Playwright/Graph directly
                                       ↓
                                  proposes a patch to add a CLI command
                                       ↓
                                  human reviews diff + tests → merge → done
```

Over time the CLI grows to cover whatever you actually do. Untouched
edge cases stay in the LLM path; that's fine.

See [`DESIGN.md`](DESIGN.md) for a deeper treatment of the tradeoffs,
risks (sycophantic drift, sprawl), and token economics.
See [`AGENTS.md`](AGENTS.md) for the contract an LLM should follow when
invoked for an email task.

## Status

| Component | State |
|---|---|
| Agent contract | ✅ [`AGENTS.md`](AGENTS.md) — live, used by agents today via the Playwright fallback |
| Design | ✅ [`DESIGN.md`](DESIGN.md) |
| Future CLI roadmap | 🟡 [`DRAFT.md`](DRAFT.md) — planned commands, auth options, phase order; pick up when there's enough recurring work to justify it |
| Binary | ❌ intentionally not built. Agent + Playwright + `outlook.office.com` covers everything for now. |
| Tests | n/a until the binary exists |

**Auth status** (relevant when the CLI eventually wants Graph API): blocked
at the author's tenant — user consent to the public Graph CLI client
denied (`AADSTS65001`), and user-level app registration also disabled.
Less locked-down tenants won't hit this. Until then, the Playwright path
sidesteps the problem entirely.

## Install

There's nothing to install yet. Agents read `AGENTS.md` and act on it —
that's the whole product right now.

When/if you want the binary: see [`DRAFT.md`](DRAFT.md) for the planned
implementation roadmap.

## License

MIT — see [`LICENSE`](LICENSE).
