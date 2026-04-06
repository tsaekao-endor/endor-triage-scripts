# Endor Labs Triage Scripts

Two Python scripts that bolt onto your existing Endor Labs PR scan to add interactive, comment-driven finding triage.

---

## How to use them

### Step 1 — Download the scripts

Run this once and commit the result:

```bash
mkdir -p .endor

curl -fsSL https://raw.githubusercontent.com/tsaekao-endor/endor-triage-scripts/main/post_triage_comment.py \
  -o .endor/post_triage_comment.py

curl -fsSL https://raw.githubusercontent.com/tsaekao-endor/endor-triage-scripts/main/handle_triage_command.py \
  -o .endor/handle_triage_command.py
```

The scripts live in your repo. No runtime downloads, no external dependencies.

### Step 2 — Append to your scan step

Wherever you run `endorctl scan`, add one line after it:

```bash
endorctl scan --pr --namespace=$ENDOR_NAMESPACE ...

python3 .endor/post_triage_comment.py
```

That's the entire change to your scan pipeline.

### Step 3 — Add a triage handler

Create a separate job (or pipeline) that fires when a PR comment containing `/endor` is detected, checks out the PR branch, and runs:

```bash
python3 .endor/handle_triage_command.py
```

See [Triage handler setup](#triage-handler-setup) below for CI-specific examples.

---

## What you need beyond the scan step

When you run `endorctl scan`, you've already configured:
- ✅ `ENDOR_NAMESPACE` — used by both scripts automatically
- ✅ `endorctl` on PATH — already installed
- ✅ Endor Labs authentication — already set up

The **only new requirement** is a GitHub token so the scripts can post PR comments:

```bash
export GH_TOKEN=<github-token>   # needs: repo scope (or pull-requests:write + contents:write)
```

| CI system | How to provide it |
|-----------|------------------|
| GitHub Actions | `GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}` — already available, no new secret needed |
| Jenkins / GitLab CI / CircleCI | Add a GitHub PAT as a CI secret and pass it as `GH_TOKEN` |

Everything else — the PR number, repository name, commenter identity, and comment body — is auto-detected from the environment. See [Environment variable reference](#environment-variable-reference) if you need to override anything.

---

## Triage commands

Once `post_triage_comment.py` posts the findings table, developers reply in the PR:

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
| `--comment="<text>"` | Note explaining the decision |
| `--expires=YYYY-MM-DD` | Auto-expire the ignore entry on this date |
| `--expire-if-fix` | Auto-expire when a fix becomes available |

---

## Triage handler setup

The triage handler needs to run when a PR comment containing `/endor` is posted. It must check out the PR branch (so it can commit `.endorignore.yaml`) before calling the script.

### GitHub Actions

```yaml
# .github/workflows/endor-triage.yml
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
      id-token: write   # keyless Endor Labs auth

    steps:
      - name: Get PR branch
        id: pr
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          branch=$(gh pr view ${{ github.event.issue.number }} \
            --repo ${{ github.repository }} --json headRefName -q .headRefName)
          echo "branch=$branch" >> $GITHUB_OUTPUT

      - uses: actions/checkout@v4
        with:
          ref: ${{ steps.pr.outputs.branch }}
          token: ${{ secrets.GITHUB_TOKEN }}
          fetch-depth: 0

      - name: Install endorctl
        run: |
          curl -fsSL https://api.endorlabs.com/download/latest/endorctl_linux_amd64 \
            -o endorctl && chmod +x endorctl && sudo mv endorctl /usr/local/bin/endorctl

      - name: Handle triage command
        env:
          ENDOR_NAMESPACE: ${{ vars.ENDOR_NAMESPACE }}
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: python3 .endor/handle_triage_command.py
```

### Jenkins

```groovy
// Triggered by a GitHub webhook on issue_comment events
pipeline {
  agent any
  steps {
    // check out the PR branch first, then:
    sh '''
      ENDOR_NAMESPACE=your-namespace \
      ENDOR_API_KEY=$ENDOR_API_KEY \
      GH_TOKEN=$GITHUB_TOKEN \
      PR_NUMBER=$PR_NUMBER \
      REPO=your-org/your-repo \
      COMMENT_BODY="$COMMENT_BODY" \
      COMMENTER=$COMMENTER \
      python3 .endor/handle_triage_command.py
    '''
  }
}
```

### GitLab CI / CircleCI

Same pattern: check out the PR branch, set the environment variables listed in the reference below, then call `python3 .endor/handle_triage_command.py`.

---

## Environment variable reference

Most values are auto-detected. Only override what you need to.

### `post_triage_comment.py`

| Variable | Required | Default | Notes |
|----------|----------|---------|-------|
| `ENDOR_NAMESPACE` | **Yes** | — | Already set for your scan step |
| `GH_TOKEN` | **Yes** | `GITHUB_TOKEN` | Auto-used in GitHub Actions; set as a secret elsewhere |
| `REPO` | No | `GITHUB_REPOSITORY` | `owner/repo` — auto-detected in GitHub Actions |
| `PR_NUMBER` | No | from event payload / `GITHUB_REF` | Auto-detected in GitHub Actions |
| `ENDOR_PROJECT_UUID` | No | auto-detected from repo name | Set explicitly to skip the API lookup |

### `handle_triage_command.py`

| Variable | Required | Default | Notes |
|----------|----------|---------|-------|
| `ENDOR_NAMESPACE` | **Yes** | — | Already set for your scan step |
| `GH_TOKEN` | **Yes** | `GITHUB_TOKEN` | Auto-used in GitHub Actions; set as a secret elsewhere |
| `REPO` | No | `GITHUB_REPOSITORY` | Auto-detected in GitHub Actions |
| `PR_NUMBER` | No | from event payload | Auto-detected in GitHub Actions |
| `COMMENT_BODY` | No | from event payload | Auto-detected in GitHub Actions; set explicitly elsewhere |
| `COMMENTER` | No | `GITHUB_ACTOR` / event payload | Auto-detected in GitHub Actions |

---

## Authentication against Endor Labs

| Environment | How it works |
|-------------|-------------|
| GitHub Actions | Keyless OIDC — automatic when `GITHUB_ACTIONS=true`. Requires `id-token: write` on the job. No secrets needed. |
| Other CI | Set `ENDOR_API_KEY` **or** `ENDOR_KEYID` + `ENDOR_PRIVATE_KEY` as CI secrets. `endorctl` picks these up automatically. |

---

## Requirements

- Python 3.10+
- `endorctl` on PATH (already installed for your scan step)
- `gh` CLI on PATH (`apt install gh` or [install docs](https://github.com/cli/cli#installation))
- `git` configured on the runner (for `handle_triage_command.py`)
