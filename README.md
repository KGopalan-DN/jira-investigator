# JIRA Ticket Investigator — Cursor Skill

Automated investigation of JIRA tickets with R&D ticket creation.

## What It Does

Given a JIRA ticket key (any project), this skill:

1. Parses the ticket — fields, comments, attachments, linked tickets
2. Downloads and analyzes techsupport logs
3. Optionally collects live data from DNOS routers via SSH
4. Optionally generates and downloads techsupport from the router
5. Investigates the codebase — code path tracing, YANG models, git history
6. Classifies the issue as Bug or NFR
7. Creates an R&D ticket in the project you choose, links it, and uploads techsupport

Each phase stops for your review before proceeding. Every write/destructive action requires your explicit approval.

The skill learns from every run — mistakes and workarounds are recorded in `known-issues.md` and applied automatically in future runs by anyone on the team.

## One-Time Setup

Run this once. Everything after is automatic.

### Step 1: Install the skill

Open a terminal and run:

```bash
mkdir -p ~/.cursor/skills && cd ~/.cursor/skills && tar -xzf /path/to/jira-investigator.tar.gz
```

Replace `/path/to/jira-investigator.tar.gz` with wherever you saved the file you received.

Verify:

```bash
ls ~/.cursor/skills/jira-investigator/SKILL.md
```

If you see the file, installation is complete.

### Step 2: JIRA API token

You need an Atlassian API token. If you already have MCP configured with JIRA access, skip this step — the skill auto-detects your credentials from `~/.cursor/mcp.json`.

If you don't have one:

1. Go to https://id.atlassian.com/manage-profile/security/api-tokens
2. Click **Create API token**
3. Name it anything (e.g., "Cursor JIRA")
4. Copy the token

You have two options to store it:

**Option A — Add to MCP config (recommended, one-time):**

If you already have `~/.cursor/mcp.json`, add the headers to your MCP server entry:

```json
{
  "mcpServers": {
    "dn-mcp-server": {
      "url": "http://ai-server:8000/mcp",
      "headers": {
        "X-Email-User": "your.email@drivenets.com",
        "X-Atlassian-Token": "your-token-here"
      }
    }
  }
}
```

**Option B — Let the skill prompt you:**

Do nothing. On first run, the skill will ask for your email and token. You'll need to provide them each session.

### Step 3: Install optional tools

Optional tools (most functionality works without them):

```bash
pip3 install python-pptx
```

| Tool | Required? | What it enables | Without it |
|------|-----------|----------------|------------|
| `dn-connect-mcp` | Recommended | SSH to routers via MCP (Phase 3) — handles multi-hop, DNOS-aware, CLI guide injection | Falls back to tmux + expect |
| `tmux` | Only if no MCP | Legacy SSH method for Phase 3 | Auto-installed via `brew install tmux` if needed |
| `python-pptx` | No | Parse PPTX attachments (CVE reports) | PPTX files skipped |
| `sshpass` | No | Simpler FTP/SCP operations | Falls back to expect scripts |

**Important note on `dn-connect-mcp`:** When using this MCP for router access, you MUST create SSH profiles with `type: "dnos"`. Without a profile, `ssh_exec` will silently fail. The skill handles profile creation automatically during Phase 3, but you can also pre-create profiles to save time on repeat investigations. Profiles persist across sessions — create once, reuse forever.

### That's it.

No other configuration needed. The skill auto-detects:
- Your code repositories (searches `~/cheetah-1`, `~/dnos`, `~/dnor`, `~/dap`, etc.)
- Your OS tools (`curl`, `python3`, `ssh`, `expect` — all pre-installed on macOS)
- MCP server availability (uses it if available, falls back to curl)

If anything is missing or in an unexpected location, the skill asks you on the first run.

## Usage

Open Cursor in any of your code repositories, and type any JIRA ticket key:

```
investigate SW-253561
investigate ART-9684
triage CS-1234
investigate WS-4567
triage AR-890
```

Any JIRA project works — ART, CS, SW, WS, AR, or any other project you have access to. The skill takes over from there.

## Phase Flow

```
Phase 0: Bootstrap (auto — validates your environment)
    ↓
Phase 1: Parse Ticket → checkpoint (you review the intake summary)
    ↓
Phase 2: Techsupport Analysis → checkpoint (you review log findings)
    ↓
Phase 3: Live Box Collection → checkpoint (optional, you provide router access)
    ↓
Phase 4: Codebase Investigation → checkpoint (you confirm Bug/NFR classification)
    ↓
Phase 5: Create R&D Ticket → checkpoint (you approve before creation)
    ↓
Done
```

You are in control at every checkpoint. Nothing gets created, uploaded, or modified without your approval.

## What Requires Your Approval

| Action | When |
|--------|------|
| Creating a JIRA ticket | Phase 5 — you approve the draft first |
| Linking / commenting on JIRA tickets | Phase 5 — you confirm each action |
| Running `request` commands on routers | Phase 3 — you authorize before execution |
| Uploading techsupport to FTP | Phase 3 / Phase 5 — you confirm destination |
| SSHing into a live router | Phase 3 — you provide credentials |
| Checking out a git branch | Phase 4 — you confirm the branch |

Everything else (reading JIRA data, searching code, running `show` commands, downloading files) is read-only and safe.

## Self-Learning

The file `known-issues.md` tracks lessons from every run. When something unexpected happens — a JIRA field doesn't work, a command syntax is wrong, a workaround is needed — it gets recorded automatically.

On every subsequent run (by you or anyone on the team), the skill reads this file first and avoids repeating known mistakes.

You can also add entries manually if you discover something outside of a skill run.

## Troubleshooting

**"JIRA credentials invalid"** — regenerate your API token at https://id.atlassian.com/manage-profile/security/api-tokens and update `~/.cursor/mcp.json`

**"Repository not found"** — the skill will ask you for the path. Provide the full path to your local clone (e.g., `~/my-repos/cheetah`)

**"tmux not installed"** — `brew install tmux`. Without it, Phase 3 uses batch expect scripts (less interactive but functional)

**Phase 3 SSH fails** — verify you can SSH to the jump host manually first. Provide the full hop-by-hop path when the skill asks. If using `dn-connect-mcp`, make sure profiles are created with `type: "dnos"` — inline credentials do NOT work with `ssh_exec`.

**"show command returns ERROR: Unknown word"** — DNOS is not Cisco/Juniper. The skill should read the MCP CLI resources before running commands, but if it guesses wrong, use `cmd search <keyword>` on the router to find the correct syntax.

## Files

| File | Purpose |
|------|---------|
| `SKILL.md` | Orchestrator — entry point, phase flow, key rules |
| `phase0-bootstrap.md` | Environment validation |
| `phase1-parse-ticket.md` | Ticket parsing |
| `phase2-techsupport.md` | Techsupport download and analysis |
| `phase3-live-collection.md` | Live router data + techsupport collection |
| `phase4-codebase.md` | Codebase investigation and classification |
| `phase5-create-ticket.md` | R&D ticket creation |
| `known-issues.md` | Self-learning — updated every run |
| `README.md` | This file |
