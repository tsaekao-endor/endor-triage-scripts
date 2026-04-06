# Endor Labs Triage Scripts

Two Python scripts that add interactive, comment-driven finding triage to your existing Endor Labs PR scan.

---

## Usage

Download the scripts and place them anywhere in your repo, then call them right after your `endorctl scan` step:

```bash
# Your existing scan step
endorctl scan \
  --pr \
  --pr-incremental \
  --enable-pr-comments \
  --namespace=$ENDOR_NAMESPACE

# Append these two lines
python3 post_triage_comment.py
```

`post_triage_comment.py` posts a numbered findings table to the PR. Developers can then reply with `/endor fp 1,2` or `/endor accept-risk 3` to triage findings without leaving the PR.

When a `/endor` comment is detected, run `handle_triage_command.py` in a separate job or pipeline step (after checking out the PR branch):

```bash
python3 handle_triage_command.py
```

This creates the corresponding ignore entries in `.endorignore.yaml`, commits them to the PR branch, and replies with a confirmation.

> **Important:** `handle_triage_command.py` does not run automatically. It needs a trigger — something that watches for PR comments and calls the script when an `/endor` command is detected. In GitHub Actions, this means adding a separate workflow file to your repo that listens for `issue_comment` events. A ready-to-use template is available at [endor-pr-triage/templates/endor-triage.yml](https://github.com/tsaekao-endor/endor-pr-triage/blob/main/templates/endor-triage.yml) — copy it into `.github/workflows/` and you're set.

---

## Environment variables

### GitHub Actions

When running in GitHub Actions, the scripts auto-detect everything from the environment. The only variables you need to set explicitly are:

| Variable | Script | Notes |
|----------|--------|-------|
| `GH_TOKEN` | both | GitHub token to post PR comments. Use `${{ secrets.GITHUB_TOKEN }}`. |
| `COMMENT_BODY` | `handle_triage_command.py` only | The text of the `/endor` comment. Use `${{ github.event.comment.body }}`. |

### Non-GitHub Actions CI (Jenkins, GitLab CI, CircleCI, homegrown, etc.)

If your CI runner is not GitHub Actions, none of the GitHub-specific environment variables are present. You must set all of the following before calling the scripts:

**`post_triage_comment.py`**

```bash
export ENDOR_NAMESPACE=your-namespace        # same value you pass to endorctl scan
export ENDOR_API_KEY=your-endor-api-key      # Endor Labs auth (instead of OIDC)
export GH_TOKEN=your-github-pat              # GitHub PAT with pull-requests:write
export REPO=owner/repo                       # GitHub repository
export PR_NUMBER=123                         # pull request number
```

**`handle_triage_command.py`**

```bash
export ENDOR_NAMESPACE=your-namespace
export ENDOR_API_KEY=your-endor-api-key
export GH_TOKEN=your-github-pat              # GitHub PAT with pull-requests:write and contents:write
export REPO=owner/repo
export PR_NUMBER=123
export COMMENTER=github-username             # GitHub username of whoever posted the comment
export COMMENT_BODY="$WEBHOOK_COMMENT_BODY"  # the full text of the PR comment, extracted from
                                             # the GitHub issue_comment webhook payload
                                             # (e.g. "/endor fp 1,2" or "/endor accept-risk 3")
```

`COMMENT_BODY` is whatever the developer actually typed in the PR. Your webhook listener receives this in the GitHub `issue_comment` event payload at `comment.body` and passes it through to the script.

You will also need to install the `gh` CLI on your runner and ensure `git` is configured with credentials to push to the repository.
