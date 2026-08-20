# fix-bugs

A bug-fixing agent that works from your logs. Point it at a log source and a
GitHub repository; it reads each distinct failure the logs show, finds the code
that produced it, fixes the single root cause, proves the fix with a test that
failed before and passes after, and opens a pull request. A person reviews the
pull request. The agent never merges.

It is **abstract**. Nothing in this package names a company, a product, a
codebase, or a log store. Everything specific to *your* setup is configuration
you supply when you install it:

| You configure | What it is |
|---|---|
| `repository.url` / `baseBranch` | the GitHub repository it fixes — private or public — and the branch its pull requests target |
| `repository.tokenKey` | the name of the token that clones, pushes a branch, and opens the PR |
| `repository.pushTo` | optional: a fork to push branches to, when it may not push to the repository itself (the open-source flow) |
| `logSources` | the command(s) that turn your logs into findings — one JSON row per distinct failure |
| `model` | the model its reasoning loop runs on |
| `review.url` | your Human Eyes board — where its questions go for a person |
| `secrets` | the names of the tokens it may use (values live in the harness) |

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

## The log source

The agent does not read logs. It reads **findings** — and a log source is the
command that produces them. This is the one piece you write, and it is small:
query your log store for the recent window, group the failures by whatever
makes two entries "the same bug" in your system (exception + top frame, route +
status, error code), and print a JSON array:

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
  must produce the same `id`. The agent marks every pull request it opens with
  `fix-bugs: <id>` and checks for that marker before it starts, so a finding
  that is already in hand is skipped rather than fixed twice.
- **`samples` must be enough to find the code.** Raw entries, not summaries —
  the message, the frame or route, the inputs the entry reveals. The agent has
  no other bug report.

The harness runs the command itself, from the repository root, with only the
secrets the source declares. The model never runs it and never sees those
secrets. Any language works; a script checked into your repository is the
usual shape.

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
