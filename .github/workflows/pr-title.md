---
environment: maintainters-only

on:
  pull_request_target:
    types: [labeled]

  # Deterministic gate: decide who "owns" the current title before the agent runs.
  # The title is left alone when the last person who set it is a maintainer
  # (repository write/maintain/admin permission) or a bot (i.e. this workflow),
  # and no contributor renamed it since. With no rename event at all, the title
  # is the one the PR was opened with and the agent is allowed to run.
  steps:
    - name: Check who set the current title
      id: title_owner
      # Only worth computing when the triggering label is the one we react to;
      # the activation job is gated on the label anyway, so skipping here just
      # avoids the API calls.
      if: github.event.label.name == 'status/2-needs-review'
      env:
        GH_TOKEN: ${{ github.token }}
        REPO: ${{ github.repository }}
        PR: ${{ github.event.pull_request.number }}
      run: |
        set -euo pipefail

        last_rename_actor=$(gh api --paginate "repos/$REPO/issues/$PR/timeline?per_page=100" \
          --jq '[.[] | select(.event == "renamed") | .actor.login] | last // ""')

        if [ -z "$last_rename_actor" ]; then
          echo "No rename event: title is the original one, agent may run"
          echo "should_run=true" >> "$GITHUB_OUTPUT"
          exit 0
        fi

        echo "Title was last renamed by: $last_rename_actor"

        case "$last_rename_actor" in
          *"[bot]")
            echo "Last rename was made by a bot (this workflow): skip"
            echo "should_run=false" >> "$GITHUB_OUTPUT"
            exit 0
            ;;
        esac

        permission=$(gh api "repos/$REPO/collaborators/$last_rename_actor/permission" --jq '.permission' 2>/dev/null || echo "unknown")
        echo "Repository permission of $last_rename_actor: $permission"

        if [ "$permission" = "unknown" ]; then
          # Fallback: the maintainers list published in the documentation.
          if gh api "repos/$REPO/contents/docs/content/contributing/maintainers.md" --jq '.content' \
             | base64 -d | grep -qi "github.com/$last_rename_actor)"; then
            permission="write"
          fi
        fi

        case "$permission" in
          admin|maintain|write)
            echo "Last rename was made by a maintainer: skip"
            echo "should_run=false" >> "$GITHUB_OUTPUT"
            ;;
          *)
            echo "Last rename was made by a contributor: agent may run"
            echo "should_run=true" >> "$GITHUB_OUTPUT"
            ;;
        esac




jobs:
  pre-activation:
    outputs:
      should_run: ${{ steps.title_owner.outputs.should_run }}

if: needs.pre_activation.outputs.should_run == 'true' && github.event.label.name == 'status/2-needs-review'

permissions:
  contents: read
  issues: read
  pull-requests: read
  id-token: write

checkout: false

# Cache the pre-fetched diff/metadata/comments on the PR head SHA so a
# re-run on the same commit skips the GitHub API calls.
cache:
  key: pr-prefetch-${{ github.event.pull_request.head.sha }}
  path: /tmp/gh-aw/agent
  restore-keys:
    - pr-prefetch-${{ github.event.pull_request.number }}-

engine:
  id: claude
  auth:
    type: github-oidc
    provider: anthropic
    federation-rule-id: fdrl_013Yw3g9LVvJzRJgQoP5zFnR
    organization-id: 797e3cc2-9e62-4091-a6e2-9bd04249babc
    service-account-id: svac_015rSKKoTmnBF2WbzqpemYW5
    workspace-id: wrkspc_01EZUP6bV8tj87UdRfaC3zKR


# Cost guardrails. A review that needs more than this is a review that should
# be split; fail closed rather than run away.
timeout-minutes: 20
max-turns: 100

imports:
- shared/pr-title.md

source: juliens/ai/workflows/traefik.pr-title.md@5cfe03f5a5cfae996f4f36d72918786cf8c2b1b9
---

Nothing
