# Phase 4 — Codebase Investigation

Trace the issue through the codebase, classify as Bug or NFR, and triple-check the analysis.

## Input

- Product and version from Phase 1
- Log evidence and initial theory from Phase 2
- Live data from Phase 3 (if collected)
- Repository path from Phase 0

## Step 0: Verify Repository and Resolve Version Branch

**CRITICAL: You MUST analyze code on the branch matching the customer's EXACT version. Analyzing the wrong branch leads to incorrect conclusions (e.g., reporting a hardcoded key as unfixed when it was already fixed on the customer's version branch).**

**If no repo is available for the identified product:**
- Report: "No local repository available for <product>. Codebase investigation is limited to data from prior phases."
- Skip to Step 6 and present analysis based on available data only.

**If repo is available:**

### Step 0a — Extract exact version from Phase 1

The customer's version from Phase 1 must include the full version string (e.g., `v25.4.13`, not just `v25.4`). If only a major.minor was captured, go back to the ticket and comments to find the exact patch/build version.

### Step 0b — Resolve to the most specific branch

Search for branches from **most specific to least specific**. The first match wins:

```bash
cd <repo_path>

# 1. Try exact version branch (most specific — e.g., dev_v25_4_13)
git branch -r | grep -i "dev_v25_4_13"

# 2. Try release branch pattern
git branch -r | grep -i "release/25.4.13"

# 3. Fall back to major.minor branch (e.g., dev_v25_4)
git branch -r | grep -i "dev_v25_4$"
```

**Branch naming patterns by product:**

| Product | Most Specific | Fallback | Example |
|---------|--------------|----------|---------|
| DNOS | `dev_v<maj>_<min>_<patch>` | `dev_v<maj>_<min>` | `dev_v25_4_13` → `dev_v25_4` |
| DNOR | `release/<version>` | `dev_v<version>` | `release/25.4.13` |
| DAP | `release/<version>` | `main` | `release/3.2.1` |

### Step 0c — Switch to the target branch

Switch the repo to the target branch so all tools (grep, semantic search, file reads) work naturally against the correct codebase.

**Before switching**, preserve the user's working state:

```bash
cd <repo_path>

# 1. Record current branch
ORIGINAL_BRANCH=$(git branch --show-current)

# 2. Check for uncommitted changes
git status --porcelain
```

**If working tree is clean** (no output from `git status --porcelain`):

```bash
# Fetch latest and switch
git fetch origin <target_branch> --quiet
git checkout <target_branch>
git pull origin <target_branch> --quiet
```

**If working tree is dirty** (uncommitted changes):

```bash
# Stash changes, switch, and note the stash
git stash push -m "jira-investigator: auto-stash before branch switch"
git fetch origin <target_branch> --quiet
git checkout <target_branch>
git pull origin <target_branch> --quiet
```

Tell the user: "Switching from `<original_branch>` to `<target_branch>` to analyze version `<version>`. Will switch back after investigation."

### Step 0d — Confirm branch

Verify you're on the right branch:

```bash
git branch --show-current
```

**If the most specific branch doesn't exist**, fall back to the next level and warn:

"Exact branch `dev_v25_4_13` not found. Using `dev_v25_4` instead — note that fixes merged into the patch branch may not appear here."

### Step 0e — Check if fixes were delivered on the target branch

Before deep analysis, do a quick scan for relevant commits that may not exist on other branches:

```bash
git log --oneline --grep="<ticket_key_or_keyword>" | head -10
```

This prevents analyzing stale code when a fix has already been delivered.

### Step 0f — Switch back after investigation (end of Phase 4)

**After Phase 4 is complete**, restore the user's original branch and stash:

```bash
git checkout <ORIGINAL_BRANCH>

# If changes were stashed in Step 0c
git stash pop
```

Always switch back — do not leave the user on the investigation branch.

## Step 1: Identify Code Areas

Based on the issue, identify which parts of the codebase to investigate.

**Mapping issue to code:**

| Issue Area | Where to Look |
|-----------|--------------|
| Alarms | `src/` alarm YAML definitions (`alarm_*.yml`), alarm engine code |
| Hardware monitoring | Platform agents, sensor reading code, threshold logic |
| Protocols (BGP, OSPF, ISIS) | `services/control/quagga/`, protocol-specific packages |
| Configuration | YANG models (`*.yang`), ORM hooks, CLI handlers |
| Security/encryption | `src/py_packages/dn_common/`, crypto utilities |
| gRPC | gRPC server code, certificate handling |
| System services | Service managers, systemd units, process supervisors |
| Interfaces | Interface management code, driver layer |

**Identify YANG models:**

```bash
find <repo_path> -name "*.yang" | xargs grep -l "<feature_keyword>" | head -10
```

YANG models define expected system behavior. If a feature exists in YANG, the system was designed to support it (bug if broken). If it doesn't exist in YANG, it was never designed (NFR).

## Step 2: Code Path Tracing

Trace the code path relevant to the reported issue.

**Start from the symptom** (error message, alarm, behavior) and trace backwards:

1. **Find the symptom in code:**

```bash
grep -rn "<error_message_or_alarm_name>" <repo_path>/src/ --include="*.rs" --include="*.py" --include="*.go" --include="*.c" --include="*.cpp" --include="*.yang" --include="*.yml" | head -20
```

2. **Trace the logic:**
   - Read the function that produces the symptom
   - Follow the call chain upwards (what calls this function?)
   - Follow the data chain (where do input values come from?)

3. **Check alarm definitions (if alarm-related):**

```bash
find <repo_path> -name "alarm_*.yml" | xargs grep -l "<alarm_keyword>" | head -5
```

Read the YAML to understand:
- Condition that triggers the alarm
- Predicate logic (thresholds, comparisons)
- Which agent/service reports the alarm

4. **Check YANG models for expected behavior:**

```bash
grep -rn "<feature_keyword>" <repo_path> --include="*.yang" | head -10
```

Read the relevant YANG leaf/container to understand:
- What the system was designed to do
- What parameters/values are expected
- Any constraints or enumerations

## Step 3: Git History Analysis

Check if the behavior is a regression or long-standing issue.

**Recent changes to the affected code:**

```bash
git log --oneline -20 -- <affected_file_or_directory>
```

**Blame the specific lines:**

```bash
git blame <file> -L <start>,<end>
```

**Check if a specific commit introduced the issue:**

```bash
git log --all --oneline --grep="<keyword>" | head -10
```

**Cross-branch comparison:**

```bash
# Check if the same code exists in other version branches
git diff <other_branch> -- <file>
```

This reveals:
- When the problematic code was introduced
- Who made the change and why (commit message)
- Whether the issue exists in other versions (cross-branch impact)

## Step 4: Triple-Check Validation

Before presenting findings, validate the analysis against three independent sources:

### Check 1 — Code vs. Logs

Does the code path you traced explain the log evidence from Phase 2?
- Match specific error messages in code to what appeared in logs
- Verify that the conditions in code would be triggered by the system state in logs
- Check timestamps: could the code path execute in the observed timeframe?

### Check 2 — Code vs. Alternative Theories

Consider at least one alternative explanation:
- Could a different code path produce the same symptom?
- Could the issue be a configuration error rather than a code bug?
- Could it be an infrastructure/hardware issue rather than software?
- Could it be expected behavior that the customer misunderstands?

### Check 3 — Code vs. Design Intent

Check YANG models and documentation to verify design intent:
- Is the current behavior what was intended?
- Did the code deviate from what the YANG model specifies?
- Was this behavior explicitly designed or is it an unintended side effect?

**If any check fails or is inconclusive, note it explicitly.** Do not present the analysis as certain if it isn't.

## Step 5: Classification

Based on the investigation, classify the issue:

### Bug

The feature exists in the YANG model / system design, but the implementation is broken.

Evidence:
- YANG model defines the behavior
- Code deviates from the YANG specification
- Regression introduced by a specific commit
- Error in logic (wrong comparison, missing condition, race condition)

### NFR (New Feature Request)

The feature was never designed or implemented.

Evidence:
- No YANG model support for the requested behavior
- No code path exists for the requested functionality
- Customer is requesting behavior beyond current design scope

### CVE / Security

The issue is a known vulnerability in a dependency or system component.

Evidence:
- CVE ID matches a known vulnerability database entry
- Affected package version is present in the codebase
- Security scan (JFrog, etc.) confirms the vulnerability

### Inconclusive

Not enough data to classify with confidence.

- State what is known and what is missing
- Recommend specific additional data to collect

## Step 6: Propose Analysis to User

Don't just dump findings — present a clear theory and proposal. Walk the user through your reasoning and ask for their input.

**Step 6a — Present your theory:**

"Here's what I found and my proposed analysis:

**Theory:** <one-sentence summary of what you think is happening>

**Evidence:**
1. <code evidence — file:line, what it shows>
2. <log evidence — how logs confirm the code path>
3. <alternative theories considered and why they were ruled out>

**Code Path:**
<trace description — which files, functions, and logic are involved>

**Root Cause:** <concise explanation, or "inconclusive" with what's missing>

**Classification:** I'm proposing this as a **<Bug / NFR / CVE>** because <reasoning>.

<If Bug:> This appears to be a regression introduced by <commit/change>. Suggested fix: <brief description>.
<If NFR:> The YANG model has no support for this. The customer is requesting: <requirements>.
<If Inconclusive:> I don't have enough data to be confident. Specifically, I'm missing <what's needed>.

**Cross-Branch Impact:** <which other versions are affected>

**Confidence:** <High / Medium / Low> — <why>

Does this analysis look right to you? Anything you'd change or add before I draft the R&D ticket?"

**Step 6b — Incorporate user feedback:**

If the user corrects or adds to the analysis, update accordingly. Do not proceed until the user confirms the classification and analysis.

**Wait for explicit user confirmation before proceeding to Phase 5.**
