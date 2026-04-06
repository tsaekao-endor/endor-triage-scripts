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

---

## Environment variables

Both scripts auto-detect as much as possible from the environment. The only variables you need to set explicitly are:

| Variable | Script | Notes |
|----------|--------|-------|
| `GH_TOKEN` | both | GitHub token with `pull-requests:write` and `contents:write`. In GitHub Actions, use `${{ secrets.GITHUB_TOKEN }}`. |
| `COMMENT_BODY` | `handle_triage_command.py` only | The text of the `/endor` comment. In GitHub Actions, use `${{ github.event.comment.body }}`. |

Everything else (`ENDOR_NAMESPACE`, `REPO`, `PR_NUMBER`, `COMMENTER`) is picked up automatically from the environment when running in GitHub Actions. Set them explicitly if you're running in a different CI system.
