# Design

## The problem

LLM agents are excellent at unstructured tasks (drafting nuanced replies,
summarizing a thread, deciding which messages matter today). They're
terrible at routine ones at scale (paging through an inbox, archiving
five threads matching a pattern, marking all newsletters as read on
Friday). Routine work via an LLM is slow, expensive, and nondeterministic.

The opposite problem is also true: traditional CLIs handle routine work
beautifully but can't deal with novel asks ("find that thread from last
week where Alice said something about the deployment").

## The split

| Concern | Best handled by |
|---|---|
| Routine, structured operations (list, archive, mark-read, label) | Deterministic CLI |
| Novel queries, content interpretation, drafting | LLM |
| Authorization of mutating actions | Human (always) |

The CLI handles the 80% well-known case. The LLM handles the 20% novel
case. When the LLM encounters a case it thinks should be in the 80%,
it proposes a patch.

## The evolution loop

```
┌─────────────────────────────────────────────────────────────────┐
│  user gives agent an email task                                  │
└────────────────────────────────┬─────────────────────────────────┘
                                 │
                  ┌──────────────▼──────────────┐
                  │  agent: "do I have a CLI    │
                  │  command for this?"          │
                  └──────┬───────────────┬───────┘
                         │ yes           │ no
                         ▼               ▼
                  ┌─────────────┐  ┌────────────────────────────┐
                  │ run command │  │ use Playwright / Graph     │
                  │ parse output│  │ directly to complete task  │
                  └──────┬──────┘  └────────────┬───────────────┘
                         │                       │
                         │                       ▼
                         │         ┌─────────────────────────────┐
                         │         │ propose: "should this be    │
                         │         │ a CLI command? if so, here's│
                         │         │ a diff for review"          │
                         │         └─────────────┬───────────────┘
                         │                       │
                         │                       ▼
                         │         ┌─────────────────────────────┐
                         │         │ human reviews diff, tests   │
                         │         │ pass, merge to main         │
                         │         └─────────────┬───────────────┘
                         │                       │
                         ▼                       ▼
                  ┌─────────────────────────────────────────────┐
                  │  sense-check the result                      │
                  │   - mutating? require explicit approval     │
                  │   - drafting? read ~/.config/style/         │
                  └─────────────────────────────────────────────┘
```

## Token economics

For an inbox listing task, ballpark per-operation:

| Approach | Tokens (rough) |
|---|---|
| LLM drives Playwright through every step | 5–10 k input + 2–3 k output |
| LLM calls `autoutlook list` and parses JSON | ~300 input + ~200 output |
| LLM calls CLI but command doesn't exist; proposes a patch | ~3–5 k input + ~2 k output (one-time) |
| Same LLM call for the same task on the next run | ~300 input + ~200 output (CLI now covers it) |

So patching has up-front cost that amortizes over future runs. If you do
a task once and never again, the patch was a waste; if you do it weekly,
the savings pile up fast. Choose what to encode in CLI based on observed
frequency, not anticipated.

## Risks and mitigations

### LLM modifies its own tools (sycophantic drift)

When an LLM patches the CLI, it tends to:

- Add the new path but accidentally break an existing one.
- Generate plausible-looking code that doesn't actually work.
- Match the surface request literally rather than its intent.

**Mitigations**:

- **Never auto-apply.** Every patch is a PR (or a local diff) reviewed
  by a human. The agent's job is to propose; merging is human.
- **Tests are mandatory** for every CLI command. Patches that don't
  include a test for the new behavior are rejected by convention.
- **Regression tests** for previously-working commands run on every PR.
- **Git is the source of truth.** Rollback is one revert away.

### Sprawl

If every novel case becomes a CLI command, the surface area explodes.

**Mitigations**:

- **Frequency threshold**: only patch when a case has been seen *twice*.
  First time → handle via LLM, no patch. Second time → propose patch.
- **Periodic prune**: a quarterly review removes commands not used in
  N months.
- **Composition over special-case**: prefer a few primitives that
  compose into many tasks over one bespoke command per task.

### Selector / API rot

Outlook's web UI changes; Graph API endpoints shift. CLI breaks silently.

**Mitigations**:

- **Smoke tests** that hit each command's path on a real inbox, run
  via CI nightly.
- **Semantic selectors** in any Playwright code (`role`, `aria-label`,
  visible text) — these are tied to accessibility and more stable than
  CSS classes.
- **Health command**: `autoutlook doctor` runs each command's smoke
  test on demand to localize a regression.

### Auth state ages out

Microsoft tokens expire. Refresh works silently most of the time;
sometimes the refresh token itself ages out (typically 90 days inactive).

**Mitigations**:

- **Single shared state** at `~/.config/microsoft-auth/state.json`
  (Playwright) or `~/.config/microsoft-auth/graph-token.json` (Graph),
  reused by all SSO-dependent tools so one re-auth covers everything.
- **Clear error message** when the token's gone: surface the path to
  delete and re-run.

## Why not just use Microsoft Graph directly with no CLI

You could. The CLI exists for three reasons:

1. **Predictable structured output.** Graph returns paginated JSON with
   nested objects; the CLI normalizes to one-message-per-line plus
   relevant fields you actually want.
2. **Composability.** `autoutlook list | grep IT | autoutlook archive`
   beats reasoning your way through Graph for every task.
3. **Surface for evolution.** Without a CLI to grow, there's nowhere
   to codify "what worked." Every session restarts from zero.

## Why not just use Outlook's own filters / rules / Power Automate

You can, and you should for stable, well-defined automations. autoutlook
is for the case where the rule changes from week to week, or where the
selection criterion is fuzzy enough that only an LLM can apply it.

## Non-goals

- Becoming a full Outlook client.
- Handling calendar, contacts, files, or anything outside the mail surface.
- Supporting personal Microsoft accounts (consumer outlook.com) — Graph
  works fine there but auth quirks differ; left for later.
