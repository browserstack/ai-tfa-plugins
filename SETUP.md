---
name: setup
description: One-time setup for the tfa-rca plugin — confirms the hosted BrowserStack MCP server is connected and that a GitHub route exists, then hands off to /tfa-rca:rca-build. Run this on install, or whenever the plugin reports that it cannot reach BrowserStack. Does NOT configure repos, logs, metrics or CI — the rca-build interview owns those.
---

# Setting up tfa-rca

**Scope.** This gets the two things the plugin cannot start without: the hosted
`bstack` MCP server connected, and a GitHub route existing. Everything else —
which repos, which branches, where the logs are, what runs the services — is settled
by `/tfa-rca:rca-build`'s own first-contact interview, which writes
`.rca-context.json` in the user's project. **Do not ask about any of that here.**
Asking twice reads as not having listened the first time.

Work through the steps in order and report each outcome in one line. Nothing here
writes a file.

## 1. Connect to BrowserStack

The `bstack` server is BrowserStack's hosted MCP endpoint, and it authenticates by
**OAuth** — the client runs the sign-in flow. There are no credentials to set, no
`.env` to fill in, and nothing for you to record.

Check whether the `bstack` tools are available in this session — `fetchBuildInsights`,
`listTestIds`, `tfaRcaTurn`.

- **They are there** — the connection is up. Move to step 2.
- **They are not** — the client has not connected yet. Tell the user to authorise the
  BrowserStack MCP server when their client prompts, or to trigger it (`/mcp` in Claude
  Code). If the server is not listed at all, that is wiring rather than sign-in:
  **[INTEGRATION.md](INTEGRATION.md)** has the fix per client.

**Never ask for a username, an access key, or any token.** This route does not use
them, so a request for one is both useless and a prompt to paste a secret into a
transcript. If the user offers one anyway, say it is not needed and that they should
treat any value they pasted as disclosed — revoke and reissue it.

## 2. Confirm it can actually read — and say what is still unproven

Being connected is not the same as being able to read this account's builds. Every
Observability read needs a build to read, so there is no free call that proves access
on its own. Handle it honestly, in one of two ways:

- **The user has a build id to hand** — use it, and you have real proof:
  `fetchBuildInsights(buildId=<id>)`. Returning the build's name and status means
  the sign-in, the entitlement and the connection all work.
- **They do not** — say plainly that the connection is up and that the first real read
  happens on their first `/tfa-rca:rca-build` run, which fails loudly and immediately
  if authorisation is wrong. Do not call an unrelated tool to manufacture a probe: a
  Test Management read succeeding or failing says nothing reliable about Observability
  access, and reporting it as proof would be worse than reporting nothing.

**Never say "verified" for a check you did not run.** "Connected, not yet exercised"
is the accurate sentence when no build was read, and it is the one to use.

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
