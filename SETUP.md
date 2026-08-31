---
name: setup
description: One-time setup for the tfa-rca plugin — proves the bundled BrowserStack MCP server can authenticate and that a GitHub route exists, then hands off to /tfa-rca:rca-build. Run this on install, or whenever the plugin reports that it cannot reach BrowserStack. Does NOT configure repos, logs, metrics or CI — the rca-build interview owns those.
---

# Setting up tfa-rca

**Scope.** This gets the two things the plugin cannot start without: the bundled
`bstack` MCP server authenticating, and a GitHub route existing. Everything else —
which repos, which branches, where the logs are, what runs the services — is settled
by `/tfa-rca:rca-build`'s own first-contact interview, which writes
`.rca-context.json` in the user's project. **Do not ask about any of that here.**
Asking twice reads as not having listened the first time.

Work through the steps in order and report each outcome in one line. Nothing here
writes a file.

## 1. Credentials for the bundled MCP server

The `bstack` server needs `BROWSERSTACK_USERNAME` and `BROWSERSTACK_ACCESS_KEY`.
Check whether they are already present in the environment.

If either is missing, tell the user exactly this and stop — do not proceed to step 2:

> Add your BrowserStack credentials, then reload the plugin. Copy `.env.example` to
> `.env` and fill in `BROWSERSTACK_USERNAME` and `BROWSERSTACK_ACCESS_KEY` — both are
> on your [account settings](https://www.browserstack.com/accounts/profile/details)
> page. Exporting them in your shell works too.

**Never write a credential value anywhere, and never echo one back.** If the user
pastes a key into the conversation, say that the value is now in the transcript and
should be revoked and reissued, then ask them to put the new one in `.env` and tell
you only that it is set. You record that a variable is set; you never record what is
in it.

`O11Y_TFA_RCA_BASE_URL` is optional and almost always unset — only a customer on a
non-default tenant needs it. Do not ask for it.

## 2. Confirm the server loaded — and say what is still unproven

Two different things can be wrong, and they have different fixes, so separate them.

**Did the server load?** Check whether the `bstack` tools — `fetchBuildInsights`,
`listTestIds`, `tfaRcaTurn` — are available in this session. If they are not, the MCP
server did not start, which is client wiring rather than credentials:
**[INTEGRATION.md](INTEGRATION.md)** has the fix per client. Point at the section for
the client actually in use.

**Do the credentials work?** Every Observability read on this server needs a build to
read, so there is no free call that proves authentication on its own. Handle it
honestly, in one of two ways:

- **The user has a build id to hand** — use it, and you have real proof:
  `fetchBuildInsights(buildId=<id>)`. Returning the build's name and status means
  credentials, entitlement and connectivity all work. An auth error means the values
  are being read and rejected, so rotating or re-copying the access key is the fix,
  not re-exporting them.
- **They do not** — say plainly that the variables are set and that the first real
  authenticated read happens on their first `/tfa-rca:rca-build` run, which fails
  loudly and immediately if the credentials are wrong. Do not call an unrelated tool
  to manufacture a probe: a Test Management read succeeding or failing says nothing
  reliable about Observability access, and reporting it as proof would be worse than
  reporting nothing.

**Never say "verified" for a check you did not run.** "Variables are set, not yet
exercised" is the accurate sentence when no build was read, and it is the one to use.

## 3. A GitHub route — the one hard requirement

The run's deliverable is naming the pull request that caused a failure, so the plugin
refuses to start without a way to read code and merged PRs. Either is enough:

- the `gh` CLI, signed in for the org that owns the repos (`gh auth status`), or
- a GitHub MCP server available in this session.

If neither is present, say so and give `gh auth login` as the shortest path. Do not
offer to work around it — there is no useful RCA without it, and saying otherwise
wastes the user's time.

**This is a route check only.** Whether that route can actually see *their* repos and
branches is proven later, per repo, by the interview in step 4. Do not claim GitHub is
verified here.

## 4. Hand off

Once steps 1–3 pass, stop and tell the user:

> Setup is done. Run `/tfa-rca:rca-build <build-id>` from your project directory —
> the first run there asks about your repos and services once, verifies each answer,
> and saves the result to `.rca-context.json` so later runs ask nothing.

**Run it from their project, not from this plugin's clone.** The context file belongs
beside their code so a teammate inherits it; started from here it would land in the
plugin's own checkout, which the plugin refuses.

Optional capabilities — application logs, metrics, a cluster, CI — are offered by that
interview and are all skippable. Each one skipped is recorded and shown in the report
as evidence that was not available. Do not pre-empt those questions here.
