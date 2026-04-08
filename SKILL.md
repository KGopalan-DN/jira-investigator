---
name: jira-investigator
description: >-
  DriveNets JIRA ticket investigator. Investigates JIRA tickets end-to-end:
  parse ticket, analyze techsupport, collect live DNOS router data, investigate
  DNOS/DNOR/DAP codebase, classify as bug or NFR, and create follow-up R&D
  tickets in any JIRA project. Use ONLY this skill (not any other skill) when
  the user says "investigate <ticket>", "triage <ticket>", or provides a JIRA
  ticket key like ART-1234, CS-1234, SW-1234, JES-1234, WS-1234, AR-1234 for
  investigation or triage. This is the jira-investigator skill located at
  ~/.cursor/skills/jira-investigator/SKILL.md.
---

# JIRA Ticket Investigator

Automated investigation of JIRA tickets with follow-up ticket creation in any project.

**THIS IS THE `jira-investigator` SKILL.** If you are reading this, you MUST follow the phases defined below. Do NOT use any other skill for JIRA ticket investigation. Do NOT mix instructions from other skills.

## Trigger

User provides any JIRA ticket key (e.g., "investigate ART-9684", "triage CS-1234", "investigate SW-253539"). Any mention of investigating, triaging, or analyzing a JIRA ticket should use this skill exclusively.

## Self-Learning

This skill improves with every run. Before starting any investigation, read [known-issues.md](known-issues.md). It contains lessons from past runs — things that failed, workarounds discovered, user preferences, and edge cases. Apply all entries relevant to the current investigation.

**During execution:** if anything unexpected happens — a command fails, JIRA rejects a field, a workaround is needed, syntax was wrong, or the user corrects your approach — append a new entry to `known-issues.md` before moving on. Follow the format defined in that file.

**After completion:** at the end of the run, review whether any new learnings should be recorded. If the run was clean with no surprises, no entry is needed. Then run the feedback sync flow to submit learnings back to the shared repo.

**Feedback sync:** read and follow [feedback-sync.md](feedback-sync.md). This runs automatically at the end of every skill run if `known-issues.md` was modified. It creates a branch, commits the changes, pushes, and opens a PR for the skill owner to review. The user can also trigger it manually by saying "log feedback" or "report issue".

This ensures every team member benefits from every other team member's runs. The skill gets smarter over time regardless of who uses it.

## Workflow

Execute phases sequentially. **Stop after each phase** and present results to the user. Only proceed to the next phase when the user confirms.

### Phase 0 — Bootstrap

Read and follow [phase0-bootstrap.md](phase0-bootstrap.md).

Validates environment: JIRA credentials, repository paths, required tools. Must pass before any other phase runs. If any critical item is missing (credentials), stop and help the user fix it. If tmux is missing, auto-install it via `brew install tmux`. If non-critical items are missing (a specific repo, python-pptx), note it and continue.

### Phase 1 — Parse Ticket

Read and follow [phase1-parse-ticket.md](phase1-parse-ticket.md).

Reads the JIRA ticket, extracts all fields, downloads and parses attachments, reads comments, checks for duplicates and linked tickets across all projects, flags missing data. Presents a structured intake summary.

**Checkpoint**: Present intake summary. User confirms or corrects before proceeding.

### Phase 2 — Techsupport Analysis

Read and follow [phase2-techsupport.md](phase2-techsupport.md).

Downloads techsupport from links found in Phase 1. Extracts and analyzes logs relevant to the reported issue. If no techsupport is available, notes it and moves on.

**Checkpoint**: Present log findings. User confirms or provides additional data.

### Phase 3 — Live Box Data Collection

Read and follow [phase3-live-collection.md](phase3-live-collection.md).

DNOS only. Always ask the user if they want to investigate the box — explain what live data would add and why. User provides router connection details (jump hosts, credentials). Uses `dn-connect-mcp` for SSH — **MUST create profiles with `type: "dnos"`** (inline credentials do NOT work with `ssh_exec`). Profiles enable auto-injected CLI reference guides, DNOS-aware error detection, and persist across sessions. Falls back to tmux if MCP unavailable. Reads MCP CLI resources BEFORE running any command — NEVER guess from Cisco/Juniper. Runs `show` commands (baseline: `show system version`, `system-events.log`, plus issue-specific commands), captures output. Optionally collects a techsupport bundle from the router (`request system tech-support`) and uploads it to FTP — requires explicit user authorization before any `request` command.

**Checkpoint**: Present collected data. User authorizes techsupport collection (if needed). User confirms before codebase investigation.

### Phase 4 — Codebase Investigation

Read and follow [phase4-codebase.md](phase4-codebase.md).

Traces code paths, checks YANG models, alarm definitions, git history. Triple-check validation. Classifies as Bug or NFR. If no relevant repo is available, works with data from prior phases only and flags it.

**Checkpoint**: Propose analysis to the user — present theory, evidence, classification, and reasoning. Ask for their input. User confirms Bug/NFR and approves before proceeding.

### Phase 5 — Create R&D Ticket

Read and follow [phase5-create-ticket.md](phase5-create-ticket.md).

Asks the user which JIRA project to create the follow-up ticket in. Formats JIRA Wiki Markup description, creates the ticket via REST API, links to source ticket, optionally uploads techsupport to FTP.

**Checkpoint**: Present draft ticket and confirm target project. User approves before creation.

## Special Cases

### CVE / Security Tickets
If the ticket is about CVEs or security vulnerabilities (e.g., JFrog scan results):
- Phase 1 downloads and parses CSV/PPTX attachments
- Phase 4 focuses on package versions in the codebase rather than code path tracing
- Response drafting focuses on risk posture and remediation timeline

### NFR Tickets
If classified as NFR:
- Phase 4 verifies the feature doesn't exist in YANG models
- Phase 5 uses the NFR template instead of the bug template
- Phase 5 captures customer requirements from comments (including internal DN stakeholders)

### Cross-Project
The source ticket can be from any JIRA project (ART, CS, SW, WS, AR, or any other). The follow-up ticket can be created in any project the user has access to. The skill does not hardcode project mappings — it asks the user.

### Feedback Sync

Read and follow [feedback-sync.md](feedback-sync.md).

Runs automatically after every investigation if `known-issues.md` was modified. Also triggered when user says "log feedback", "report issue", or "submit learning". Creates a PR on the shared skill repo for the skill owner to review and merge.

## Key Rules

1. **User authorization for all write/destructive actions** — any action that modifies external systems, could be risky, or could have unintended consequences must get an explicit "Yes" from the user before executing. Never assume permission. Specifically:
   - Creating a JIRA ticket → user must approve the draft and target project
   - Updating a JIRA ticket (description, fields) → user must approve the content
   - Linking JIRA tickets → user must confirm
   - Posting comments on JIRA tickets → user must approve the comment text
   - Running `request` commands on routers (`request system tech-support`, `request file upload`) → user must authorize each one
   - Uploading files to FTP server → user must confirm destination
   - SSHing into a live router → user must provide credentials and confirm
   - Checking out a different git branch in the local repo → allowed for version-specific investigation (auto-stash if dirty, always switch back after)
   - Any action not listed here that changes state → ask first
2. **Read operations are safe** — fetching JIRA ticket data, reading local files, searching code, running `show` commands on routers, downloading techsupport — these do not require additional authorization beyond the initial phase confirmation
3. **Strictly show commands only** on live routers — never enter config mode. Exception: `request system tech-support` and `request file upload` are allowed **only with explicit user authorization** (never use `inclusive` flag)
4. **MANDATORY: Read the MCP CLI resource BEFORE running ANY command on a DNOS router.** DNOS is NOT Cisco/Juniper — do NOT guess syntax. The `dn-connect-mcp` provides CLI guides at `resource://dn-connect/guide/dnos/*`. Read the relevant guide (e.g., `routing-igp-mpls` for ISIS/OSPF, `routing-bgp` for BGP, `core` for system/AAA, `operations` for tech-support). If MCP is unavailable, check RST files. Then `cmd help`/`cmd search` on the router. NEVER run a command from Cisco/Juniper memory.
5. **JIRA Wiki Markup** for ticket descriptions — bold headings (`*Heading*`), `{code}` blocks for CLI/tabular output only, plain text for explanations
6. **Triple-check** codebase analysis against logs, code, and alternative theories before presenting
7. **Never fabricate** — if analysis is inconclusive, say so
8. **Customer vs R&D scope** — don't leak internal scope details to customer-facing tickets
9. **Project-agnostic** — don't assume source or target JIRA projects, ask the user
10. **Self-learning** — read `known-issues.md` before every run, append new learnings during execution whenever something unexpected happens or a workaround is used
