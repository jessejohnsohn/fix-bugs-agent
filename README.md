# fix-bugs

A bug-fixing agent that works from wherever your bugs are reported. Point it
at a data source — your product's report-a-bug table, a log query, an
endpoint, a bucket — and a GitHub repository; it reads each distinct finding,
finds the code behind it, fixes the single root cause, and opens a pull
request. It never builds the project or runs its tests — its container has `git` and `gh`,
not your toolchain — the pull request's own checks do that, and a person
reviews the pull request. The agent never merges.

It is **abstract**. Nothing in this package names a company, a product, a
codebase, or a data store. Everything specific to *your* setup is configuration
you supply when you install it:

| You configure | What it is |
|---|---|
| `repository.url` / `baseBranch` | the GitHub repository it fixes — private or public — and the branch its pull requests target |
| `repository.tokenKey` | the name of the token that clones, pushes a branch, and opens the PR |
| `repository.pushTo` | optional: a fork to push branches to, when it may not push to the repository itself (the open-source flow) |
| `dataSources` | where the findings come from — a database table, a Mongo collection, an HTTP endpoint, an R2 bucket, or a command of your own |
| `model` | the model its reasoning loop runs on |
| `secrets` | the names of the tokens it may use (values live in the harness) |

Where its **questions** go is not configuration of the agent at all. The
package lists the Human Eyes plugin under `permissions.plugins`; you install
that plugin on your harness once (board URL + token, from
[humaneyes.work](https://humaneyes.work)) and grant it to this agent. Until it
is granted the agent is not ready to run, and setup says so.

## What it is

- **Runtime:** none of its own. It is a *declarative* agent that runs on the
  harness's `work-loop` engine — no code to deploy, nothing to trust but a
  prompt, a policy, and your config.
- **Prompt:** [`prompt/agent.md`](prompt/agent.md) — the whole of the agent's
  behavior, readable in a few minutes.
- **Policy:** [`policy.json`](policy.json) — the Human Eyes policy. Its spine:
  the agent may commit, push its own branch, and open a pull request without
  asking, because the pull request *is* the review; everything else that leaves
  the working tree parks for a person, and **nothing irreversible and public
  happens on its say-so**.
- **Manifest:** [`agent-package.json`](agent-package.json) — the shape the
  harness installs against, and the configuration schema above.

## Data sources

The agent does not read logs or tables. It reads **findings** — rows of
`{ id, title, ... }` — and a data source is what yields them. Five kinds, all
declared in configuration:

- **`database`** — postgres, mysql, sqlite, or Cloudflare D1, plus one query.
  If your product has a "report a bug" feature writing rows into a table, this
  is the whole integration — no script, no webhook, no code in your repository:

  ```json
  {
    "kind": "database",
    "id": "bug-reports",
    "description": "Open rows from the product's report-a-bug table, newest first.",
    "driver": "postgres",
    "urlKey": "BUGS_DB_URL",
    "query": "SELECT id::text AS id, summary AS title, details, reporter, created_at FROM bug_reports WHERE status = 'open' ORDER BY created_at DESC"
  }
  ```

  The query's columns must yield `id` and `title` (alias in the SQL, or set
  `"map": { "id": "report_id", "title": "summary" }`); every other column rides
  along as evidence the agent reads. Sources are **read-only** — the agent
  never writes back, marks rows, or changes state in your database.

- **`mongodb`** — a collection and a filter, same contract.
- **`http`** — GET a URL that answers with the JSON row array (bearer-token
  secret, optional `rowsPath` when the array is nested).
- **`r2`** — an object holding the array, or a prefix where each object is one
  row.
- **`command`** — the original shape, for anything else: a command run from the
  repository root that prints the array. An entry with just `command` and no
  `kind` still means this. The usual shape is a small script in your repository
  that queries your log store for the recent window and groups the failures by
  whatever makes two entries "the same bug" (exception + top frame, route +
  status, error code):

```json
[
  {
    "id": "TypeError-cannot-read-null-checkout.ts:212",
    "title": "TypeError: Cannot read properties of null (reading 'total') at checkout.ts:212",
    "count": 137,
    "firstSeen": "2026-08-19T02:14:00Z",
    "lastSeen": "2026-08-20T11:52:00Z",
    "samples": [
      "2026-08-20T11:52:00Z ERROR req=7f3a TypeError: Cannot read properties of null (reading 'total')\n    at priceCart (src/checkout.ts:212:31)\n    at POST /api/cart ..."
    ]
  }
]
```

Two rules that matter:

- **`id` must be stable.** The same bug, seen on Tuesday and again on Thursday,
  must produce the same `id`. Every pull request the agent opens is marked
  `fix-bugs: <id>` and lands on a branch derived from that id, and **the harness
  checks both before a run starts** — one `gh` call covering every finding at
  once, so a finding already in hand never opens a path at all. The model is
  never asked, because there is no judgement in the question and a model turn
  re-sends the whole conversation to ask it.
- **The row must be enough to find the code.** Raw log samples, the
  reporter's own words, the message, the frame or route — not summaries. The
  agent has no other bug report.

The harness reads every source itself — a command from the repository root,
the declared kinds through its own runner — with only the secrets the source
names. The model never runs them and never sees those secrets.

## Private or public repository

The agent cannot tell the difference and does not behave differently. What
changes is what the token must be allowed to do:

- **A repository you control** (private or public): a token that can read it,
  push branches to it, and open pull requests. Fine-grained token, scoped to
  that repository, `contents: read+write` and `pull_requests: write`.
- **A public repository you do not control:** fork it, set `pushTo` to the fork,
  and give the token write access to the fork plus pull-request access to the
  upstream. The agent pushes `fix/<id>` to the fork and opens the PR against
  the upstream from there.

In both cases, **protect the base branch.** The agent's prompt and policy forbid
pushing to it and forbid force pushes; branch protection is what makes that
true regardless of what a model decides. The credential never sits inside the
checkout: it is configured for `git` and `gh` outside the working tree, and the
policy parks every network command that is not `git` or `gh`.

## Install

This is an Agent Store package. Install it into any compatible harness by its
repository and a pinned version; supply the configuration above. The harness
reads this repository at a pinned commit, resolves the prompt and policy, and
records an immutable version. It never runs any code from this repository; a
declarative agent has none.

[`examples/installation.json`](examples/installation.json) is a complete
configuration with placeholder values.

## The one boundary that is not configurable

The agent opens pull requests. It does **not** merge them, push to the base
branch, deploy, publish, or take any irreversible outward-facing action — ever,
by any configuration. That is a change to the policy, made by a person. If your
setup needs the agent to do more, you change `policy.json` deliberately and in
the open; you do not grant it as an approval at runtime.
