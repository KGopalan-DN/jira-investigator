# Feedback Sync — Automatic Learning Collection and PR Flow

This mechanism ensures every team member's learnings flow back to the shared skill repo for review and merge.

## When This Runs

1. **Auto-capture (end of every skill run)**: After the investigation completes (or fails), check if `known-issues.md` was modified during this run. If yes, trigger the sync flow.
2. **Explicit trigger**: User says "log feedback", "report issue", or "submit learning". Collect details and trigger the sync flow.
3. **User correction**: Whenever the user corrects the skill's approach during execution and a new `known-issues.md` entry is added, mark it for sync.

## Sync Flow

### Step 1: Detect Changes

```bash
cd ~/.cursor/skills/jira-investigator
git diff --name-only
```

If `known-issues.md` (or any phase file) has uncommitted changes, proceed. If clean, skip — nothing to sync.

### Step 2: Identify the User

```bash
git config user.name || whoami
```

Store as `FEEDBACK_USER`.

### Step 3: Create a Feedback Branch

Branch naming: `feedback/<username>-<date>-<short-context>`

```bash
BRANCH_NAME="feedback/$(whoami)-$(date +%Y%m%d)-$(echo '<context>' | tr ' ' '-' | head -c 30)"
git checkout -b "$BRANCH_NAME"
```

Where `<context>` is derived from:
- The ticket being investigated (e.g., `ART-9684`)
- Or "manual-feedback" if triggered explicitly
- Or "session-learnings" if multiple issues were captured

### Step 4: Commit Changes

```bash
git add known-issues.md
git add -A  # include any phase file changes
git commit -m "$(cat <<'EOF'
feedback: <one-line summary of what was learned>

Captured during investigation of <TICKET_KEY> by <FEEDBACK_USER>.
EOF
)"
```

### Step 5: Push and Open PR

```bash
git push -u origin "$BRANCH_NAME"
```

Then open a PR targeting `main`:

```bash
gh pr create \
  --title "Feedback: <one-line summary>" \
  --body "$(cat <<'EOF'
## Context
- **User**: <FEEDBACK_USER>
- **Ticket investigated**: <TICKET_KEY or "manual feedback">
- **Date**: <YYYY-MM-DD>

## Changes
<list of new known-issues.md entries or phase file updates>

## Details
<brief description of what happened and why this learning matters>
EOF
)"
```

### Step 6: Switch Back to Main

```bash
git checkout main
```

### Step 7: Confirm to User

Tell the user:
"Learning submitted as PR #<number> on the shared skill repo. The skill owner will review and merge it so the whole team benefits."

## Explicit Feedback Collection

When the user says "log feedback" or "report issue":

1. Ask: "What went wrong or what did you learn?"
2. Ask: "Which phase was this in?" (0–5, or general)
3. Format it as a `known-issues.md` entry using the standard format
4. Append to `known-issues.md`
5. Run the sync flow above

## Error Handling

- **No git remote**: The skill was installed manually (not cloned). Warn: "Your skill copy is not linked to the shared repo. Run `git remote add origin https://github.com/KGopalan-DN/jira-investigator.git` to enable feedback sync."
- **No `gh` CLI**: Warn: "GitHub CLI not installed. Your feedback was saved locally in `known-issues.md` but could not be submitted as a PR. Install with `brew install gh` and run `gh auth login`."
- **Push rejected (auth)**: Warn: "Could not push to the shared repo. You may not have write access. Ask the skill owner to add you as a collaborator: `gh repo add-collaborator KGopalan-DN/jira-investigator <your-github-username>`"
- **Merge conflicts**: If `known-issues.md` has conflicts on branch creation, pull main first: `git pull origin main`, resolve, then proceed.

## For the Skill Owner (KGopalan)

Review incoming PRs with:

```bash
gh pr list --repo KGopalan-DN/jira-investigator
gh pr diff <number>
gh pr merge <number> --squash
```

After merging, tag a new version:

```bash
git pull origin main
git tag v1.x.0
git push origin v1.x.0
```

All team members will see the "update available" notice on their next run (via Phase 0 auto-update check).
