# Endor Labs Triage Scripts

Two Python scripts that add interactive, comment-driven triage to any CI pipeline running [Endor Labs](https://www.endorlabs.com) PR scans.

After a scan runs, `post_triage_comment.py` posts a numbered findings table to the PR. When a developer replies with `/endor fp 1,2`, `handle_triage_command.py` creates the corresponding ignore entries, commits them to the PR branch, and replies with a confirmation.

---

## Scripts

| Script | Run it... | What it does |
|--------|-----------|--------------|
| [`post_triage_comment.py`](post_triage_comment.py) | After your Endor Labs scan step | Posts a numbered findings table to the PR. Skips silently if there are no CI-blocking or CI-warning findings. |
| [`handle_triage_command.py`](handle_triage_command.py) | When a PR comment containing `/endor` is detected | Parses the command, creates `.endorignore.yaml` entries, commits them, and replies with a summary. |

---

## Setup

### 1. Download the scripts into your repository

Copy both scripts into your repo — a `.endor/` folder at the root works well:

```bash
mkdir -p .endor

curl -fsSL https://raw.githubusercontent.com/tsaekao-endor/endor-triage-scripts/main/post_triage_comment.py \
  -o .endor/post_triage_comment.py

curl -fsSL https://raw.githubusercontent.com/tsaekao-endor/endor-triage-scripts/main/handle_triage_command.py \
  -o .endor/handle_triage_command.py
```

Commit them. They live in your repo going forward — no runtime downloads, no external dependencies.

### 2. Add the triage comment step to your scan workflow

After your existing `endorlabs/github-action` scan step, add:

```yaml
- name: Post Endor Labs triage comment
  if: always()
  env:
    ENDOR_NAMESPACE: ${{ vars.ENDOR_NAMESPACE }}
  run: python3 .endor/post_triage_comment.py
```

That's it. `GITHUB_TOKEN`, the PR number, and the repository name are all picked up automatically from the GitHub Actions environment — no extra variables needed.

### 3. Add the triage handler workflow

Create `.github/workflows/endor-triage.yml`:

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
        run: |
          PR_DATA=$(gh pr view ${{ github.event.issue.number }} \
            --repo ${{ github.repository }} --json headRefName)
          echo "branch=$(echo $PR_DATA | jq -r .headRefName)" >> $GITHUB_OUTPUT
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}

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
        run: python3 .endor/handle_triage_command.py
```

Again, everything else — the PR number, comment body, commenter identity, and GitHub token — is read automatically from the GitHub Actions event payload. The only variable you configure is `ENDOR_NAMESPACE`.

### 4. Set `ENDOR_NAMESPACE`

Go to **Settings → Secrets and variables → Actions → Variables** and add:

| Variable | Value |
|----------|-------|
| `ENDOR_NAMESPACE` | Your Endor Labs tenant namespace (e.g. `my-company`) |

This is the same namespace you already pass to the Endor Labs scan step — not new configuration.

---

## Triage commands

Once the findings table is posted, developers reply in the PR:

```
/endor fp 1
/endor accept-risk 2,3
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

## Authentication

### GitHub Actions (recommended)

No extra setup. The scripts automatically use GitHub Actions OIDC keyless authentication when `GITHUB_ACTIONS=true`. Requires `id-token: write` permission on the job.

### Other CI systems

Set one of the following as a CI secret:

```bash
ENDOR_API_KEY=<your-api-key>
# or
ENDOR_KEYID=<key-id>
ENDOR_PRIVATE_KEY=<private-key>
```

For other CI systems you will also need to set `REPO` (`owner/repo`), `PR_NUMBER`, and for the triage handler: `COMMENT_BODY` and `COMMENTER`, since these are not automatically available outside of GitHub Actions.

---

## How it works

```
PR opened → scan runs
└── endorctl scan --pr                  your existing scan step
└── post_triage_comment.py              queries Endor Labs API for findings
    ├── auto-detects repo, PR#, token   from GitHub Actions environment
    ├── auto-detects project UUID       from repo name via Endor API
    └── posts numbered findings table   with hidden UUID map in comment

Developer replies: /endor fp 1,2

issue_comment webhook → handle_triage_command.py
    ├── reads PR#, comment, commenter   from GitHub Actions event payload
    ├── looks up finding UUIDs          from the hidden map in the triage comment
    ├── runs endorctl ignore            for each finding
    ├── commits .endorignore.yaml       to the PR branch
    └── replies with confirmation
```

---

## Requirements

- Python 3.10+
- [`endorctl`](https://docs.endorlabs.com/getting-started/install/) on PATH
- [`gh` CLI](https://cli.github.com/) on PATH, authenticated via `GITHUB_TOKEN`
- `git` configured on the runner (for `handle_triage_command.py`)
