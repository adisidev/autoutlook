# autoutlook

A self-evolving CLI for managing email in Outlook, paired with an agent
contract that lets an LLM drive it and propose its own extensions when
the deterministic path falls short.

> **Current state**: design + agent contract only. No working binary yet.
> Implementation is blocked on Microsoft Graph auth at the author's tenant
> (admin consent required for the public Graph CLI client; user-level app
> registration also disabled). See [Status](#status) below. If your tenant
> doesn't have those restrictions, the path forward is much shorter.

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
| Design | ✅ this README + `DESIGN.md` + `AGENTS.md` |
| Auth path | ⚠️ blocked at author's tenant: user consent to the public Graph CLI client denied (`AADSTS65001`), and user-level app registration disabled. Less locked-down tenants won't hit this. |
| `bin/autoutlook` | 🟡 stub. Prints "not implemented yet" until auth is unblocked. |
| Tests | ❌ none yet |

**Possible unblocks for Graph API**:

- Open a ticket asking the tenant admin for tenant-wide consent for
  `14d82eec-204b-4c2f-b7e8-296a70dab67e` (Microsoft Graph CLI Tools),
  delegated scopes: `Mail.Read`, `Mail.ReadWrite`, `Mail.Send`,
  `User.Read`. Risk: refused or slow.
- Use a different OAuth client ID that's already pre-consented in the
  tenant (unknown which, if any).

**Fallback while Graph is blocked**: agents should drive Outlook web on
`outlook.office.com` via Playwright with persistent storage state, per
the protocols in `AGENTS.md`. Same sense-check rules apply.

## Install (when there's something to install)

```sh
git clone https://github.com/adisidev/autoutlook.git ~/src/autoutlook
ln -sf ~/src/autoutlook/bin/autoutlook ~/.local/bin/autoutlook
```

Right now this just gets you the stub binary.

## License

MIT — see [`LICENSE`](LICENSE).
