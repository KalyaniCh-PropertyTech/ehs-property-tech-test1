# GitHub CI/CD: Dev → Test → Prod (Databricks) — Setup, Deployment & Notifications

This guide documents how to create and operate GitHub Actions workflows to automatically promote code between branches (dev → test → prod) for Databricks projects **using only the GitHub web UI**. It now includes Databricks deployment steps (via Databricks Asset Bundles) and Microsoft Teams notifications.

## 1) Prerequisites

- A GitHub repository with branches: dev, test, prod.
- (Recommended) Separate Databricks workspaces for Dev, Test, and Prod.
- Databricks Asset Bundle present at repo root (databricks.yml with targets: dev/test/prod).

## 2) Required GitHub Settings

- Repository → Settings → Actions → General → Workflow permissions: **Read and write permissions**.
- Tick **Allow GitHub Actions to create and approve pull requests**.
- Repository → Settings → Branches: configure protection for `test` and `prod` as required (reviews/checks).

## 3) Secrets to Create (GitHub → Settings → Secrets and variables → Actions)

**Dev → Test**
- DATABRICKS_HOST_TEST — Test workspace URL (e.g., https://adb-<id>.<region>.azuredatabricks.net)
- DATABRICKS_TOKEN_TEST — Databricks personal access token (or switch to OIDC later)
- TEAMS_WEBHOOK_TEST — Teams channel webhook (Incoming Webhook or Teams Workflow webhook)
**Test → Prod**
- DATABRICKS_HOST_PROD — Prod workspace URL
- DATABRICKS_TOKEN_PROD — Databricks personal access token (or OIDC)
- TEAMS_WEBHOOK_PROD — Teams channel webhook

## 4) Create Workflows in GitHub Web UI

- Go to **Actions** → **New workflow** → **set up a workflow yourself**.
- Create files at:
  - `.github/workflows/dev-to-test.yml`
  - `.github/workflows/test-to-prod.yml`

## 5) Workflow: Promote Dev → Test (with Databricks deploy + Teams)

```yaml
name: Promote Dev to Test

on:
  push:
    branches:
      - dev
  workflow_dispatch:

permissions:
  contents: write
  pull-requests: write

jobs:
  promote:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout (current ref)
        uses: actions/checkout@v4

      # 1) Create or update a PR from dev → test and capture PR number
      - name: Create PR dev → test
        id: create_pr
        uses: repo-sync/pull-request@v2
        with:
          source_branch: "dev"
          destination_branch: "test"
          pr_title: "Auto: Promote Dev to Test"
          pr_body: "Automatically generated PR moving code from dev to test"
          github_token: ${{ secrets.GITHUB_TOKEN }}

      # 2) Label PR for automerge
      - name: Add automerge label
        if: ${{ steps.create_pr.outputs.pr_number != '' }}
        uses: actions-ecosystem/action-add-labels@v1
        with:
          labels: automerge
          number: ${{ steps.create_pr.outputs.pr_number }}
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      # 3) Auto-merge PR when ready (respects branch protections)
      - name: Automerge PR
        if: ${{ steps.create_pr.outputs.pr_number != '' }}
        uses: pascalgn/automerge-action@v0.15.6
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          PULL_REQUEST: ${{ steps.create_pr.outputs.pr_number }}
          MERGE_LABELS: "automerge"
          MERGE_METHOD: merge
          MERGE_COMMIT_MESSAGE_TEMPLATE: "Auto-merged PR #{{PR_NUMBER}} from dev to test"

      # 4) Check out the TARGET branch (test) post-merge
      - name: Checkout test branch (post-merge)
        uses: actions/checkout@v4
        with:
          ref: test

      # 5) Install Databricks CLI (official action)
      - name: Setup Databricks CLI
        uses: databricks/setup-cli@main

      # 6) Configure env for TEST workspace and deploy bundle
      - name: Validate bundle (test)
        run: databricks bundle validate -t test
        env:
          DATABRICKS_HOST: ${{ secrets.DATABRICKS_HOST_TEST }}
          DATABRICKS_TOKEN: ${{ secrets.DATABRICKS_TOKEN_TEST }}

      - name: Deploy bundle to TEST
        run: databricks bundle deploy -t test
        env:
          DATABRICKS_HOST: ${{ secrets.DATABRICKS_HOST_TEST }}
          DATABRICKS_TOKEN: ${{ secrets.DATABRICKS_TOKEN_TEST }}

      # 7) Teams notifications
      - name: Notify Teams (success)
        if: ${{ success() }}
        run: |
          json=$(cat <<'EOF'
          {
            "text": "✅ *Dev → Test* deployment succeeded in **${{ github.repository }}**.\nPR #${{ steps.create_pr.outputs.pr_number }} merged by **${{ github.actor }}**.\nBranch: `test`\nRun: https://github.com/${{ github.repository }}/actions/runs/${{ github.run_id }}"
          }
          EOF
          )
          curl -X POST -H 'Content-Type: application/json' -d "$json" "${{ secrets.TEAMS_WEBHOOK_TEST }}"

      - name: Notify Teams (failure)
        if: ${{ failure() }}
        run: |
          json=$(cat <<'EOF'
          {
            "text": "❌ *Dev → Test* deployment FAILED in **${{ github.repository }}**.\nPR #${{ steps.create_pr.outputs.pr_number }}\nCheck run logs: https://github.com/${{ github.repository }}/actions/runs/${{ github.run_id }}"
          }
          EOF
          )
          curl -X POST -H 'Content-Type: application/json' -d "$json" "${{ secrets.TEAMS_WEBHOOK_TEST }}"
```

## 6) Workflow: Promote Test → Prod (with Databricks deploy + Teams)

```yaml
name: Promote Test to Prod

on:
  push:
    branches:
      - test
  workflow_dispatch:

permissions:
  contents: write
  pull-requests: write

jobs:
  promote:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout (current ref)
        uses: actions/checkout@v4

      # 1) Create or update the PR from test → prod and capture PR number
      - name: Create PR test → prod
        id: create_pr
        uses: repo-sync/pull-request@v2
        with:
          source_branch: "test"
          destination_branch: "prod"
          pr_title: "Auto: Promote Test to Prod"
          pr_body: "Automatically generated PR moving code from test to prod"
          github_token: ${{ secrets.GITHUB_TOKEN }}

      # 2) Label PR for automerge
      - name: Add automerge label
        if: ${{ steps.create_pr.outputs.pr_number != '' }}
        uses: actions-ecosystem/action-add-labels@v1
        with:
          labels: automerge
          number: ${{ steps.create_pr.outputs.pr_number }}
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      # 3) Auto-merge PR when ready
      - name: Automerge PR
        if: ${{ steps.create_pr.outputs.pr_number != '' }}
        uses: pascalgn/automerge-action@v0.15.6
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          PULL_REQUEST: ${{ steps.create_pr.outputs.pr_number }}
          MERGE_LABELS: "automerge"
          MERGE_METHOD: merge
          MERGE_COMMIT_MESSAGE_TEMPLATE: "Auto-merged PR #{{PR_NUMBER}} from test to prod"

      # 4) Check out the TARGET branch (prod) post-merge
      - name: Checkout prod branch (post-merge)
        uses: actions/checkout@v4
        with:
          ref: prod

      # 5) Install Databricks CLI
      - name: Setup Databricks CLI
        uses: databricks/setup-cli@main

      # 6) Validate & deploy bundle to PROD
      - name: Validate bundle (prod)
        run: databricks bundle validate -t prod
        env:
          DATABRICKS_HOST: ${{ secrets.DATABRICKS_HOST_PROD }}
          DATABRICKS_TOKEN: ${{ secrets.DATABRICKS_TOKEN_PROD }}

      - name: Deploy bundle to PROD
        run: databricks bundle deploy -t prod
        env:
          DATABRICKS_HOST: ${{ secrets.DATABRICKS_HOST_PROD }}
          DATABRICKS_TOKEN: ${{ secrets.DATABRICKS_TOKEN_PROD }}

      # 7) Teams notifications
      - name: Notify Teams (success)
        if: ${{ success() }}
        run: |
          json=$(cat <<'EOF'
          {
            "text": "✅ *Test → Prod* deployment succeeded in **${{ github.repository }}**.\nPR #${{ steps.create_pr.outputs.pr_number }} merged by **${{ github.actor }}**.\nBranch: `prod`\nRun: https://github.com/${{ github.repository }}/actions/runs/${{ github.run_id }}"
          }
          EOF
          )
          curl -X POST -H 'Content-Type: application/json' -d "$json" "${{ secrets.TEAMS_WEBHOOK_PROD }}"

      - name: Notify Teams (failure)
        if: ${{ failure() }}
        run: |
          json=$(cat <<'EOF'
          {
            "text": "❌ *Test → Prod* deployment FAILED in **${{ github.repository }}**.\nPR #${{ steps.create_pr.outputs.pr_number }}\nCheck run logs: https://github.com/${{ github.repository }}/actions/runs/${{ github.run_id }}"
          }
          EOF
          )
          curl -X POST -H 'Content-Type: application/json' -d "$json" "${{ secrets.TEAMS_WEBHOOK_PROD }}"
```

## 7) How to Execute the Workflows

- **Manual:** Actions → select workflow → **Run workflow** (choose branch).
- **Automatic:** Push a change to the source branch (dev for Dev→Test, test for Test→Prod).

## 8) Troubleshooting (Based on Real Errors)

- **No 'Run workflow' button**
  - Add `workflow_dispatch:` under `on:` in YAML.
- **PR has no number (not a real PR)**
  - Ensure PR step includes `github_token: ${{ secrets.GITHUB_TOKEN }}`.
- **Automerge error: 'issue_number undefined'**
  - Give PR step an `id:` and use `${{ steps.<id>.outputs.pr_number }}`; pass it as `PULL_REQUEST`.
- **PR created but not merging**
  - Repo Settings → Actions → **Read and write** + **Allow GitHub Actions to create and approve PRs**.
  - Satisfy required checks / approvals in branch protection.
- **Conflicts block merge**
  - Open PR → **Resolve conflicts** in web editor → commit.

## 9) Verifying Promotions

- PR appears with a number (#123) and auto‑closes after checks pass.
- Target branch (test/prod) shows latest commits from source branch.
- Databricks workspace reflects updated jobs/notebooks after deploy.

## 10) Notes & Tips

- Pin the Databricks CLI action to a specific version for stability (e.g., `databricks/setup-cli@v0.288.0`).
- Use production presets/modes in bundle targets for prod deployments.
- If Connectors are disabled, use a Teams **Workflow webhook** instead of classic Incoming Webhook.
- Consider GitHub OIDC → Databricks service principal instead of PAT.

## References

- Databricks CLI setup action (GitHub)
- Databricks Asset Bundles — bundle validate/deploy (Docs)
- GitHub automerge action & GITHUB_TOKEN usage (GitHub Docs/Marketplace)
- Teams Incoming Webhook / Workflows webhook (Microsoft Learn)
