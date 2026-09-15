---
# Linked-issue triage step for traefik/traefik. The decision itself (data
# fetch, safe outputs, prompt) is shared/base.pr-triage-link.md; this file
# holds only what gh-aw does not accept from an import: trigger, permissions,
# engine, budgets.
#
# Installed copy: .github/workflows/pr-triage-link.md in the target repository,
# next to .github/workflows/shared/base.pr-triage-link.md. Never edit those
# copies. To roll out a change to either file, run from the target repository:
#   gh aw add traefik/ai/traefik.pr-triage-link -n pr-triage-link --force
#   gh aw compile --strict --actionlint
# (zizmor and poutine run in traefik/ai CI on the same lock content; in a
# target repository, without the zizmor configuration kept there, they fail.)
# Not `gh aw update`: it refreshes this file but leaves shared/ as it was.
name: "PR triage: linked issue"
description: "When a pull request enters triage, promotes it to status/2-needs-review if it closes a confirmed bug with the Fixes keyword. Read-only agent; writes only one pinned label transition through a safe output."
emoji: "🔗"

# pull_request_target, not pull_request: under a plain pull_request trigger a
# fork PR gets neither secrets nor an OIDC id-token, so the Anthropic OIDC
# exchange fails. pull_request_target runs in the base repository context, so
# it works for forks too, and most Traefik pull requests come from forks. The
# compiler refuses this trigger with a checkout (see `checkout: false` below):
# the PR head is never cloned, which is what makes the trigger safe.
#
# `status/0-needs-triage` is applied by traefiker when the pull request is
# opened, so this runs once per pull request, right at the start of triage.
# Re-running it on a pull request that has already been promoted is a no-op:
# `required-labels: [status/0-needs-triage]` in the base refuses the
# transition once the label is gone.
on:
  pull_request_target:
    types: [labeled]
  roles: [admin, maintainer, write]   # exact-match allowlist of who may apply the label
  status-comment: false               # one less write to the PR timeline

# pull_request_target has no label-name filter; gate on the exact label.
if: github.event.label.name == 'status/0-needs-triage'

checkout: false

# The agent job is read-only. The single label transition happens in a
# separate safe-output job.
permissions:
  contents: read
  issues: read
  pull-requests: read
  id-token: write

# Keyless Anthropic auth (Workload Identity Federation). No ANTHROPIC_API_KEY
# secret exists in any repository; the federation rule decides which GitHub
# repositories may exchange their OIDC token. These IDs are identifiers, not
# credentials. Do not add PATs, custom github-token overrides, or MCP servers.
engine:
  id: claude
  auth:
    type: github-oidc
    provider: anthropic
    federation-rule-id: fdrl_013Yw3g9LVvJzRJgQoP5zFnR
    organization-id: 797e3cc2-9e62-4091-a6e2-9bd04249babc
    service-account-id: svac_015rSKKoTmnBF2WbzqpemYW5
    workspace-id: wrkspc_01EZUP6bV8tj87UdRfaC3zKR

# Cost guardrails. This runs on every pull request, and the whole task is
# reading two small files and calling one tool; a run that needs more than
# this has gone wrong. Fail closed rather than run away.
timeout-minutes: 5
max-turns: 10

imports:
  - shared/base.pr-triage-link.md
source: youkoulayley/ai/workflows/traefik.pr-triage-link.md@f1cd1e8d80ca183620b35981d0496a657fad0d1d
---

# Repository

`traefik/traefik` is public: pull requests usually come from forks, and the
issues they link are reported by users. `kind/bug/confirmed` is applied by a
maintainer during issue triage, which is why it is the only label that
promotes a pull request out of `status/0-needs-triage`.
