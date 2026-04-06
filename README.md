# Endor Labs Triage Scripts

Two Python scripts that add interactive PR comment-driven triage to any CI pipeline that runs [Endor Labs](https://www.endorlabs.com) scans.

After a scan runs, `post_triage_comment.py` posts a numbered findings table to the PR. When a developer comments `/endor fp 1,2` or `/endor accept-risk 3`, `handle_triage_command.py` creates the corresponding ignore entries in `.endorignore.yaml`, commits them to the PR branch, and replies with a confirmation.

> These scripts work with **any CI system** (GitHub Actions, Jenkins, GitLab CI, CircleCI, etc.) as long as the repository is hosted on **GitHub**.

---

## Scripts

| Script | When to run | What it does |
|--------|-------------|--------------|
| [`post_triage_comment.py`](post_triage_comment.py) | After the Endor Labs PR scan step | Posts a numbered findings table to the PR (only when CI-blocking/warning findings exist) |
| [`handle_triage_command.py`](handle_triage_command.py) | When a PR comment containing `/endor` is detected | Parses the command, creates ignore entries, commits `.endorignore.yaml`, replies with a summary |

---

## Prerequisites

Install these tools on the CI runner before calling the scripts:

| Tool | Purpose | Install |
|------|---------|---------|
| Python 3.10+ | Run the scripts | Available by default on most CI runners |
| [`endorctl`](https://docs.endorlabs.com/getting-started/install/) | Interact with Endor Labs API | See install snippet below |
| [`gh` CLI](https://cli.github.com/) | Post PR comments | `apt install gh` or see [install docs](https://github.com/cli/cli#installation) |
| `git` | Commit `.endorignore.yaml` (triage handler only) | Pre-installed on all standard CI runners |

**Install endorctl:**

```bash
curl -fsSL https://api.endorlabs.com/download/latest/endorctl_linux_amd64 -o endorctl
chmod +x endorctl
sudo mv endorctl /usr/local/bin/endorctl
```

---

## Authentication

### GitHub Actions (keyless OIDC — recommended)

No secrets needed. When `GITHUB_ACTIONS=true`, the scripts automatically pass `--enable-github-action-token` to `endorctl`. Requires the job to have `id-token: write` permission.

### API key (all other CI systems)

Create an API key in your Endor Labs tenant and set it as a CI secret:

```bash
export ENDOR_NAMESPACE=your-namespace
export ENDOR_API_KEY=your-api-key
```

### Service account key pair

```bash
export ENDOR_NAMESPACE=your-namespace
export ENDOR_KEYID=your-key-id
export ENDOR_PRIVATE_KEY=your-private-key   # base64-encoded PEM
```

---

## Environment variables

### `post_triage_comment.py`

| Variable | Required | Description |
|----------|----------|-------------|
| `ENDOR_NAMESPACE` | Yes | Endor Labs tenant namespace |
| `PR_NUMBER` | Yes | Pull request number |
| `REPO` | Yes | GitHub repository (`owner/repo`) |
| `GH_TOKEN` | Yes | GitHub token with `pull-requests:write` |
| `ENDOR_PROJECT_UUID` | No | Project UUID — auto-detected from `REPO` if not set |

### `handle_triage_command.py`

| Variable | Required | Description |
|----------|----------|-------------|
| `ENDOR_NAMESPACE` | Yes | Endor Labs tenant namespace |
| `PR_NUMBER` | Yes | Pull request number |
| `REPO` | Yes | GitHub repository (`owner/repo`) |
| `GH_TOKEN` | Yes | GitHub token with `pull-requests:write` and `contents:write` |
| `COMMENT_BODY` | Yes | Full text of the PR comment containing the `/endor` command |
| `COMMENTER` | Yes | GitHub username of the person who posted the command |

---

## Triage commands

Once `post_triage_comment.py` has posted the findings table, developers reply directly in the PR:

```
/endor fp 1,2
/endor accept-risk 3
/endor fp 1 --comment="Not reachable in prod" --expires=2026-12-31 --expire-if-fix
```

| Command | Effect |
|---------|--------|
| `/endor fp <numbers>` | Mark findings as **false positive** |
| `/endor accept-risk <numbers>` | Mark findings as **accepted risk** |

**Optional flags:**

| Flag | Description |
|------|-------------|
| `--comment="<text>"` | Note explaining the triage decision |
| `--expires=YYYY-MM-DD` | Auto-expire the ignore entry on this date |
| `--expire-if-fix` | Auto-expire when a fix becomes available |

---

## Download the scripts

Grab the latest version directly in your CI pipeline — no need to vendor them into your repo:

```bash
curl -fsSL https://raw.githubusercontent.com/tsaekao-endor/endor-triage-scripts/main/post_triage_comment.py \
  -o /tmp/post_triage_comment.py

curl -fsSL https://raw.githubusercontent.com/tsaekao-endor/endor-triage-scripts/main/handle_triage_command.py \
  -o /tmp/handle_triage_command.py
```

---

## CI integration examples

### GitHub Actions

**Scan workflow** — add after your existing `endorlabs/github-action@v1` step:

```yaml
- name: Install endorctl
  run: |
    curl -fsSL https://api.endorlabs.com/download/latest/endorctl_linux_amd64 -o endorctl
    chmod +x endorctl && sudo mv endorctl /usr/local/bin/endorctl

- name: Post triage comment
  if: always()
  env:
    GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    ENDOR_NAMESPACE: ${{ vars.ENDOR_NAMESPACE }}
    PR_NUMBER: ${{ github.event.pull_request.number }}
    REPO: ${{ github.repository }}
  run: |
    curl -fsSL https://raw.githubusercontent.com/tsaekao-endor/endor-triage-scripts/main/post_triage_comment.py \
      -o /tmp/post_triage_comment.py
    python3 /tmp/post_triage_comment.py
```

**Triage handler workflow** — create `.github/workflows/endor-triage.yml`:

```yaml
name: Endor Labs Triage Handler
on:
  issue_comment:
    types: [created]

jobs:
  triage:
    if: |
      github.event.issue.pull_request != null &&
      (contains(github.event.comment.body, '/endor fp') ||
       contains(github.event.comment.body, '/endor accept-risk'))
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write
      contents: write
      id-token: write

    steps:
      - name: Get PR branch
        id: pr
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          PR_DATA=$(gh pr view ${{ github.event.issue.number }} \
            --repo ${{ github.repository }} --json headRefName,headRefOid)
          echo "branch=$(echo $PR_DATA | jq -r .headRefName)" >> $GITHUB_OUTPUT

      - uses: actions/checkout@v4
        with:
          ref: ${{ steps.pr.outputs.branch }}
          token: ${{ secrets.GITHUB_TOKEN }}
          fetch-depth: 0

      - name: Install endorctl
        run: |
          curl -fsSL https://api.endorlabs.com/download/latest/endorctl_linux_amd64 -o endorctl
          chmod +x endorctl && sudo mv endorctl /usr/local/bin/endorctl

      - name: Handle triage command
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          ENDOR_NAMESPACE: ${{ vars.ENDOR_NAMESPACE }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          PR_NUMBER: ${{ github.event.issue.number }}
          REPO: ${{ github.repository }}
          COMMENT_BODY: ${{ github.event.comment.body }}
          COMMENTER: ${{ github.event.comment.user.login }}
        run: |
          curl -fsSL https://raw.githubusercontent.com/tsaekao-endor/endor-triage-scripts/main/handle_triage_command.py \
            -o /tmp/handle_triage_command.py
          python3 /tmp/handle_triage_command.py
```

> **Tip:** If you're using GitHub Actions and want an even simpler setup, see [endor-pr-triage](https://github.com/tsaekao-endor/endor-pr-triage) which wraps the scan-side script as a reusable composite action (`uses: tsaekao-endor/endor-pr-triage@main`).

---

### Jenkins

**Scan job** — add a post-scan shell step (requires `ENDOR_API_KEY` stored as a Jenkins credential):

```groovy
pipeline {
  agent any
  environment {
    ENDOR_NAMESPACE = 'your-namespace'
    ENDOR_API_KEY   = credentials('endor-api-key')
    GH_TOKEN        = credentials('github-token')
    PR_NUMBER       = "${env.CHANGE_ID}"
    REPO            = 'your-org/your-repo'
  }
  stages {
    stage('Endor Labs Scan') {
      steps {
        sh 'endorctl scan --pr'
      }
    }
    stage('Post triage comment') {
      steps {
        sh '''
          curl -fsSL \
            https://raw.githubusercontent.com/tsaekao-endor/endor-triage-scripts/main/post_triage_comment.py \
            -o /tmp/post_triage_comment.py
          python3 /tmp/post_triage_comment.py
        '''
      }
    }
  }
}
```

**Triage handler job** — create a separate job triggered by a GitHub webhook on `issue_comment` events:

```groovy
pipeline {
  agent any
  environment {
    ENDOR_NAMESPACE = 'your-namespace'
    ENDOR_API_KEY   = credentials('endor-api-key')
    GH_TOKEN        = credentials('github-token')
    // These are passed in by the webhook payload — map them in your
    // GitHub webhook plugin or inject them via a parameter trigger
    PR_NUMBER       = "${params.PR_NUMBER}"
    REPO            = "${params.REPO}"
    COMMENT_BODY    = "${params.COMMENT_BODY}"
    COMMENTER       = "${params.COMMENTER}"
  }
  stages {
    stage('Checkout PR branch') {
      steps {
        checkout scm
      }
    }
    stage('Handle triage command') {
      steps {
        sh '''
          curl -fsSL \
            https://raw.githubusercontent.com/tsaekao-endor/endor-triage-scripts/main/handle_triage_command.py \
            -o /tmp/handle_triage_command.py
          python3 /tmp/handle_triage_command.py
        '''
      }
    }
  }
}
```

---

### GitLab CI

**Scan job** — add to your `.gitlab-ci.yml`:

```yaml
endor-scan:
  stage: test
  script:
    - endorctl scan --pr
    - |
      curl -fsSL \
        https://raw.githubusercontent.com/tsaekao-endor/endor-triage-scripts/main/post_triage_comment.py \
        -o /tmp/post_triage_comment.py
      python3 /tmp/post_triage_comment.py
  variables:
    ENDOR_NAMESPACE: "your-namespace"
    # ENDOR_API_KEY: set in GitLab CI/CD settings as a masked variable
    GH_TOKEN: $GITHUB_TOKEN       # GitHub token stored as GitLab CI variable
    PR_NUMBER: $CI_MERGE_REQUEST_IID
    REPO: "your-org/your-repo"
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
```

**Triage handler** — GitLab CI does not natively receive GitHub `issue_comment` webhooks. Options:
- Use a GitLab CI pipeline triggered by a webhook forwarder
- Use a dedicated lightweight service (e.g. a small AWS Lambda or Cloud Run function) that receives the GitHub webhook and triggers a GitLab pipeline with the comment payload as variables

---

### CircleCI

**Scan job** — add to `.circleci/config.yml`:

```yaml
jobs:
  endor-scan:
    docker:
      - image: cimg/python:3.12
    steps:
      - checkout
      - run:
          name: Install endorctl and gh CLI
          command: |
            curl -fsSL https://api.endorlabs.com/download/latest/endorctl_linux_amd64 \
              -o endorctl && chmod +x endorctl && sudo mv endorctl /usr/local/bin/
            # Install gh CLI
            curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg \
              | sudo dd of=/usr/share/keyrings/githubcli-archive-keyring.gpg
            echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] \
              https://cli.github.com/packages stable main" \
              | sudo tee /etc/apt/sources.list.d/github-cli.list
            sudo apt update && sudo apt install gh -y
      - run:
          name: Endor Labs scan
          command: endorctl scan --pr
          environment:
            ENDOR_NAMESPACE: your-namespace
            # ENDOR_API_KEY: set in CircleCI project environment variables
      - run:
          name: Post triage comment
          command: |
            curl -fsSL \
              https://raw.githubusercontent.com/tsaekao-endor/endor-triage-scripts/main/post_triage_comment.py \
              -o /tmp/post_triage_comment.py
            python3 /tmp/post_triage_comment.py
          environment:
            ENDOR_NAMESPACE: your-namespace
            PR_NUMBER: << pipeline.parameters.pr_number >>
            REPO: your-org/your-repo
            # GH_TOKEN: set in CircleCI project environment variables
```

---

## How it works end to end

```
CI scan job runs
└── endorctl scan --pr              # your existing scan step
└── post_triage_comment.py          # downloads + posts the findings table
    ├── Queries Endor Labs API for CI-blocking/warning findings
    ├── Builds a sorted, linked table (Finding # → Endor platform link)
    └── Posts comment with a hidden JSON map of numbers → finding UUIDs

Developer replies: /endor fp 1,2

GitHub webhook fires (issue_comment event)
└── handle_triage_command.py        # downloads + handles the command
    ├── Parses the /endor command and optional flags
    ├── Looks up finding UUIDs from the hidden map in the triage comment
    ├── Calls endorctl ignore for each finding
    ├── Commits .endorignore.yaml to the PR branch
    └── Replies with a confirmation summary
```

---

## Requirements

- Python 3.10+
- `endorctl` on PATH, authenticated (see [Authentication](#authentication))
- `gh` CLI on PATH, authenticated via `GH_TOKEN`
- `git` configured on the runner (for `handle_triage_command.py`)
- GitHub repository (the SCM webhook/comment API is GitHub-specific)
