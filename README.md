# ai-tfa-plugins

Root-cause analysis for a whole BrowserStack build, from inside your coding agent.

Point it at a red build. It reads every failed test, groups them by failure
signature, gathers evidence from the tools you already have — your code, your logs,
your cluster, your metrics, your CI — and works with BrowserStack's analysis agent
to land a root cause per test, naming the pull request that most likely caused it.

Works in **Claude Code**, **Cursor** and **Codex**.

> **The report lands on the Test Observability dashboard, not in your terminal.**
> Your agent prints a short status table and a link. That link is the deliverable.

---

## Before you start

| You need | Why |
|---|---|
| A BrowserStack account with Test Observability | The build, its test logs, and the dashboard the report lands on |
| **GitHub access** — the `gh` CLI signed in, or a GitHub MCP server | Required. Without your code and its merged PRs there is no culprit PR to name, and that is the point of the run |
| Anything else you use — logs, metrics, a cluster, CI | Optional. Each one you skip is recorded and shown in the report as evidence that was not available |

GitHub is the only hard requirement. Everything else is offered, never forced.

**There are no BrowserStack credentials to configure.** The plugin talks to
BrowserStack's hosted MCP server and your client signs you in on first connect. Nothing
is stored in this repo, and there is no `.env` to fill in.

## Install

In Claude Code:

```
/plugin marketplace add browserstack/ai-tfa-plugins
/plugin install tfa-rca@browserstack-ai-tfa
```

The first time it connects, your client walks you through signing in to BrowserStack.
That is the whole credential step. Ask Claude to **run the plugin's setup** if you want
that and your GitHub route checked before you start.

<details>
<summary>Installing from a clone instead (for development)</summary>

```bash
git clone https://github.com/browserstack/ai-tfa-plugins.git
cd ai-tfa-plugins
claude --plugin-dir ./
```

</details>

Everything wires itself on load — the BrowserStack MCP server, the `rca-build`
skill, and the analysis agent are all found by convention.

**Using Cursor or Codex?** See **[INTEGRATION.md](INTEGRATION.md)** for the
per-client setup. The core is identical; only the batching differs.

## Run it

```
/tfa-rca:rca-build <build-id>
```

The build id is the one in your Test Observability URL. A dashboard link works too.

You can hand it more, and anything you supply wins over what it would work out for
itself:

```
/tfa-rca:rca-build <build-id> https://github.com/org/repo/pull/9254 .../pull/7900
```

- **A list of PRs** becomes *the* set of suspects. It is treated as complete — the
  run stops searching for candidates of its own and spends its effort deciding which
  of yours is to blame, reporting each one it rules out and why. Paste the merged-PR
  list straight out of your release thread; good and bad together is exactly right.
- **Anything else you pin** — a CI run, an environment, a branch — overrides what
  the build's metadata says. Pasting a whole regression-bot message works; it reads
  what is in it.

Pinned values apply to this run only and are never written to disk.

## The first run asks you some questions

The first time you run this in a directory, it interviews you — once. It says what
BrowserStack already has and what only you can supply, then asks about your side:
which repos, which branch, where your logs are, what runs your services. It reads
your project first so it only asks what it genuinely cannot see, and it **proves
every answer with a real read** before keeping it.

The result is saved to `.rca-context.json` next to where you ran it. **Commit it** —
a teammate who clones the repo inherits the whole setup and is asked only for their
own credentials.

Every run after that asks **at most one question**, and usually none. On a repeat run
it first shows you the setup it has on file — repos, branches, connectors, when each
was last verified — so you can correct anything or point it at a different
environment before it starts.

## What you get

When every test has an answer, your agent prints a short table — one line per test
with its cluster and confidence — then:

```
Full report on the Test Observability UI: <link>
```

The dashboard report holds the real detail: root cause per test, the evidence behind
it, and linked pull requests on application bugs.

## When it stops and asks

It refuses rather than guessing when guessing would give you a confident wrong
answer. What each one means:

| It says | What happened | What to do |
|---|---|---|
| It starts interviewing you | No setup here yet | Answer — it is once per directory |
| **GitHub could not be verified** | No working `gh` or GitHub MCP server, or it cannot see the repo | Sign in (`gh auth login`) and re-run. This is the one thing that blocks a run |
| **This build does not match your saved setup** | The build's name is not one your `.rca-context.json` describes — often a sibling suite | It offers to add this build to the existing setup, make a new one for it, or use the existing one just this once |
| **Two saved setups both match** | Two profiles claim the same build name | Narrow one of their patterns, or re-run naming the one you want |
| **Your setup file could not be read** | Usually a merge conflict left in `.rca-context.json` | Fix the file. Nothing is overwritten until you do |
| **Run this from your own directory** | You ran it from inside the plugin's own clone | `cd` to your project and re-run — your setup belongs with your code, not ours |
| **This would discard a teammate's setup** | A write would have removed something already verified | Nothing was written. Re-run, or reconcile the file by hand |

Missing logs, metrics or CI never block a run. They are recorded, declared to the
analysis agent as evidence it does not have, and reflected in the confidence of the
result.

## Contributing

`npm test` runs the suite (no build step, no dependencies).
[INTEGRATION.md](INTEGRATION.md) covers the per-client wiring.
