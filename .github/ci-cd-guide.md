GitHub CI/CD: Dev → Test → Prod (Databricks) — Setup & Operations Guide
This guide documents how to create and operate GitHub Actions workflows to automatically promote code between branches (dev → test → prod) for Databricks projects using only the GitHub web UI. It includes required settings, how to run workflows, and fixes for the exact errors encountered during setup.

1) Prerequisites

A GitHub repository with branches: dev, test, prod.
Branch protection rules for test and prod as per your governance (optional but recommended).
(Optional) Databricks Repos connected to your GitHub repo.


2) Required GitHub Settings
Repository → Settings → Actions → General

Workflow permissions → Read and write permissions
✅ Allow GitHub Actions to create and approve pull requests

Repository → Settings → Branches (optional)

Configure branch protection for test and prod (required reviews, required status checks, etc.)


3) Create the Workflows in GitHub Web UI

Go to Actions → New workflow → set up a workflow yourself.
Create the files below and paste the corresponding YAML.


.github/workflows/dev-to-test.yml
.github/workflows/test-to-prod.yml


4) Workflow: Promote Dev → Test (dev-to-test.yml)
YAMLname: Promote Dev to Teston:  push:    branches:      - dev  workflow_dispatch:permissions:  contents: write  pull-requests: writejobs:  promote:    runs-on: ubuntu-latest    steps:      - name: Checkout        uses: actions/checkout@v4      # 1) Create or update a PR from dev → test and capture its number      - name: Create PR dev → test        id: create_pr        uses: repo-sync/pull-request@v2        with:          source_branch: "dev"          destination_branch: "test"          pr_title: "Auto: Promote Dev to Test"          pr_body: "Automatically generated PR moving code from dev to test"          github_token: ${{ secrets.GITHUB_TOKEN }}      # 2) Add an 'automerge' label to the PR we just created      - name: Add automerge label        if: ${{ steps.create_pr.outputs.pr_number != '' }}        uses: actions-ecosystem/action-add-labels@v1        with:          labels: automerge          number: ${{ steps.create_pr.outputs.pr_number }}        env:          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}      # 3) Auto-merge the PR using its explicit number      - name: Automerge PR        if: ${{ steps.create_pr.outputs.pr_number != '' }}        uses: pascalgn/automerge-action@v0.15.6        env:          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}          PULL_REQUEST: ${{ steps.create_pr.outputs.pr_number }}          MERGE_LABELS: "automerge"          MERGE_METHOD: merge          MERGE_COMMIT_MESSAGE_TEMPLATE: "Auto-merged PR #{{PR_NUMBER}} from dev to test"Show more lines

5) Workflow: Promote Test → Prod (test-to-prod.yml)
YAMLname: Promote Test to Prodon:  push:    branches:      - test  workflow_dispatch:permissions:  contents: write  pull-requests: writejobs:  promote:    runs-on: ubuntu-latest    steps:      - name: Checkout        uses: actions/checkout@v4      # 1) Create or update the PR from test → prod and capture PR number      - name: Create PR test → prod        id: create_pr        uses: repo-sync/pull-request@v2        with:          source_branch: "test"          destination_branch: "prod"          pr_title: "Auto: Promote Test to Prod"          pr_body: "Automatically generated PR moving code from test to prod"          github_token: ${{ secrets.GITHUB_TOKEN }}      # 2) Add automerge label to this PR      - name: Add automerge label        if: ${{ steps.create_pr.outputs.pr_number != '' }}        uses: actions-ecosystem/action-add-labels@v1        with:          labels: automerge          number: ${{ steps.create_pr.outputs.pr_number }}        env:          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}      # 3) Auto-merge this PR using PR number      - name: Automerge PR        if: ${{ steps.create_pr.outputs.pr_number != '' }}        uses: pascalgn/automerge-action@v0.15.6        env:          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}          PULL_REQUEST: ${{ steps.create_pr.outputs.pr_number }}          MERGE_LABELS: "automerge"          MERGE_METHOD: merge          MERGE_COMMIT_MESSAGE_TEMPLATE: "Auto-merged PR #{{PR_NUMBER}} from test to prod"Show more lines

6) How to Execute the Workflows


Manual run:
Actions → select the workflow → Run workflow → choose the branch (dev or test).


Automatic run:
Push a small change to the source branch:

dev triggers Dev → Test
test triggers Test → Prod




7) Troubleshooting & Fixes (the exact ones we applied)


No “Run workflow” button
Add workflow_dispatch: to the on: section.


PR shows no number (not a real PR)
Ensure the PR creation step includes github_token: ${{ secrets.GITHUB_TOKEN }}.


Automerge fails with Empty value for parameter 'issue_number'

Give the create‑PR step an id: create_pr
Use ${{ steps.create_pr.outputs.pr_number }} in subsequent steps
Pass it to automerge via PULL_REQUEST env var.



PR created but not merging

Repo Settings → Actions → Read and write permissions
Enable Allow GitHub Actions to create and approve pull requests
Satisfy branch protections (approvals / required checks)



Conflicts block merge
Open the PR → Resolve conflicts (web editor) → Commit.
The automerge step will finish after resolution.



8) Verifying Promotions

Pull Requests: You should see a PR like #15 Auto: Promote Dev to Test that closes automatically.
Target branch commits: Latest Dev commits appear on test; latest Test commits appear on prod.
Databricks: In Repos, switch to the target branch and Pull to see updated notebooks/jobs.


9) Optional Enhancements

Use MERGE_METHOD: squash for a fresh commit each merge (clean history & updated timestamps).
Add a Databricks deployment step after merge
(e.g., databricks bundle deploy -t test or -t prod).
Send Teams/Email notifications on successful promotion.
