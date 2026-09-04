---
# .github/workflows/pr-context-package.md
# Compile with:  gh aw compile --strict --actionlint --zizmor --poutine
#
# Companion to ai-review-2job.md — see that file for the full two-job
# rationale. This is the unprivileged half: it must stay a plain
# `pull_request` trigger (never pull_request_target) so it keeps GitHub's
# default fork restriction — read-only GITHUB_TOKEN, no secrets, no OIDC —
# even though it checks out the fork's PR head. Nothing sensitive is
# available here to leak, and this workflow never writes back to the PR.
#
# It only packages data; it never needs the AI engine. The "Skip the AI
# agent" step below always fires a `noop`, so the harness exits before any
# inference cost is incurred (see reference/cost-management#skip-the-agent-
# from-steps-using-noop). `engine:` is still declared because gh-aw requires
# one, but it is never actually invoked.
name: PR Context Package (untrusted, no secrets)

on:
    pull_request:
        types: [ labeled ]
        names: [ ai/review-2job ]         # native label filter, not a hand-rolled check
    roles: [ admin, maintainer, write ] # native allowlist for who may apply the label
    reaction: eyes
    status-comment: false

permissions:
    contents: read

# Full history so `git diff <base_sha> <head_sha>` below has both commits
# locally — the default pull_request checkout is otherwise fetch-depth: 1.
checkout:
    fetch-depth: 0

engine:
    id: claude

steps:
    -   name: Package PR context (source tree + diff + metadata)
        run: |
            set -euo pipefail
            mkdir -p /tmp/pr-context
            printf '%s' "${{ github.event.pull_request.number }}" > /tmp/pr-context/pr_number.txt
            printf '%s' "${{ github.event.pull_request.head.sha }}" > /tmp/pr-context/head_sha.txt
            printf '%s' "${{ github.event.pull_request.base.sha }}" > /tmp/pr-context/base_sha.txt
            git diff "${{ github.event.pull_request.base.sha }}" "${{ github.event.pull_request.head.sha }}" > /tmp/pr-context/pr.diff
            tar -czf /tmp/pr-context/source.tar.gz --exclude=.git .

    -   name: Upload PR context artifact
        uses: actions/upload-artifact@043fb46d1a93c77aae656e7c1c64a875d1fc6a0a # v7.0.1
        with:
            name: pr-context-${{ github.event.pull_request.number }}
            path: /tmp/pr-context
            retention-days: 1

    -   name: Skip the AI agent — this workflow only packages data for ai-review-2job.md
        run: |
            msg="Packaged PR #${{ github.event.pull_request.number }} context (source tree + diff) for the workflow_run consumer; no AI review runs in this workflow."
            printf '{"type":"noop","message":"%s"}\n' "$msg" >> "$GH_AW_SAFE_OUTPUTS"
---

# PR Context Packager

This workflow only runs deterministic steps; the AI agent is always skipped
(see the `noop` step above). This section is unused but required by the
compiler.
