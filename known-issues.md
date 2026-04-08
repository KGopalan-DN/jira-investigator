# Known Issues & Learnings

This file is automatically updated during skill execution. Every time a phase encounters an unexpected behavior, workaround, or failure, it gets recorded here. The skill reads this file at the start of every run so past learnings are never repeated.

**Anyone on the team can add entries manually too — just follow the format below.**

---

## Format

Each entry follows this structure:

```
### <SHORT_TITLE>
- **Phase**: <phase number>
- **Date**: <YYYY-MM-DD>
- **User**: <who encountered it>
- **Symptom**: <what went wrong or was unexpected>
- **Root Cause**: <why it happened>
- **Fix/Workaround**: <what to do instead>
```

---

## Entries

### JIRA description overwritten on ticket creation
- **Phase**: 5
- **Date**: 2025-03-20
- **User**: kgopalan
- **Symptom**: Description field was empty after creating the ticket via POST
- **Root Cause**: JIRA's default project template overwrites the description on initial creation
- **Fix/Workaround**: Always do a two-step process — create with placeholder description, then PUT the full description

### Custom fields unavailable on New Feature issue type
- **Phase**: 5
- **Date**: 2025-03-20
- **User**: kgopalan
- **Symptom**: Setting customfield_10501 (Customer Issue) and customfield_11533 (Customer Severity) failed for NFR tickets
- **Root Cause**: These fields are only on the Bug issue screen, not New Feature
- **Fix/Workaround**: Skip support tab fields when issue type is New Feature. Only set them for Bug.

### Customer Name field expects array of objects
- **Phase**: 5
- **Date**: 2025-03-20
- **User**: kgopalan
- **Symptom**: customfield_11630 (Customer Name) rejected when sent as array of strings
- **Root Cause**: Field schema requires `[{"value": "AT&T - Artemis"}]` not `["AT&T - Artemis"]`
- **Fix/Workaround**: Always use `[{"value": "<name>"}]` format for customfield_11630

### JSON escaping breaks curl with complex Wiki Markup
- **Phase**: 5
- **Date**: 2025-03-20
- **User**: kgopalan
- **Symptom**: curl PUT failed with JSON parse error when description contained special characters
- **Root Cause**: Shell escaping of quotes, braces, and newlines in Wiki Markup corrupts the JSON
- **Fix/Workaround**: Write JSON payload to a temp file, then use `curl -d @filename` instead of inline JSON

### JIRA heading format rejected by user
- **Phase**: 5
- **Date**: 2025-03-20
- **User**: kgopalan
- **Symptom**: User said "remove the h3 things" — `h3.` headings looked bad in JIRA
- **Root Cause**: User preference — bold text (`*Heading*`) looks cleaner than JIRA heading syntax
- **Fix/Workaround**: Always use `*Bold Text*` for section headings, never `h1.`/`h2.`/`h3.`

### {noformat} blocks rejected by user
- **Phase**: 5
- **Date**: 2025-03-20
- **User**: kgopalan
- **Symptom**: User said "i dont like this {noformat}"
- **Root Cause**: User preference — `{code}` blocks look better for CLI output
- **Fix/Workaround**: Use `{code}` blocks for CLI/tabular output, plain text for everything else. Never use `{noformat}`.

### SSH jump host mistaken for router
- **Phase**: 3
- **Date**: 2025-03-20
- **User**: kgopalan
- **Symptom**: Ran `show` commands on a Linux jump host instead of the DNOS router
- **Root Cause**: Assumed the second hop was the router, but it was another jump host
- **Fix/Workaround**: Always clarify the full connection path — how many hops, which is the final DNOS device. Verify by checking the prompt (DNOS shows `hostname#`, Linux shows `$`).

### DNOS CLI command syntax wrong without RST check
- **Phase**: 3
- **Date**: 2025-03-20
- **User**: kgopalan
- **Symptom**: `show config` syntax was wrong, user pointed to RST files
- **Root Cause**: Guessed command syntax instead of checking RST documentation
- **Fix/Workaround**: ALWAYS check RST files at `prod/dnos_monolith/dnos_cli/` before running any command. If RST not found, use `cmd search`/`cmd help` on the router.

### SSH with -J flag unreliable in expect scripts
- **Phase**: 3
- **Date**: 2026-04-02
- **User**: kgopalan
- **Symptom**: Expect script using `ssh -J jump1 jump2` hung after the second password prompt — never reached the shell
- **Root Cause**: ProxyJump (`-J`) tunnels both password prompts through the local SSH client. The expect patterns for the shell prompt (`$`) failed to match because of the long warning banners and system info output between the password and the prompt
- **Fix/Workaround**: Chain SSH hops manually — SSH to hop1, then from the hop1 shell SSH to hop2, then from hop2 SSH to the router. This gives a clear shell prompt after each hop's password.

### Pager (--More--) swallows subsequent commands in expect scripts
- **Phase**: 3
- **Date**: 2026-04-02
- **User**: kgopalan
- **Symptom**: `show config` command was never executed because the pager from `show system aaa-servers tacacs` consumed the input
- **Root Cause**: DNOS CLI pages long output by default. In expect scripts, the `q` or space to advance the pager is not sent, and the next command text gets consumed by the pager
- **Fix/Workaround**: Run `terminal length 0` (or equivalent DNOS command to disable pager) at the start of the session before running any show commands. Alternatively, pipe commands with `| no-more` if supported.

### terminal length 0 does not exist in DNOS CLI
- **Phase**: 3
- **Date**: 2026-04-02
- **User**: kgopalan
- **Symptom**: `terminal length 0` returned "ERROR: Unknown word: 'terminal'"
- **Root Cause**: DNOS CLI does not support `terminal length` command (not Cisco IOS)
- **Fix/Workaround**: Append `| no-more` to each show command to disable paging per-command. Do NOT try `terminal length 0`.

### tmux send-keys is more reliable than expect for interactive sessions
- **Phase**: 3
- **Date**: 2026-04-02
- **User**: kgopalan
- **Symptom**: Expect scripts with `interact` inside tmux kept hanging on hop2 password prompt
- **Root Cause**: Expect pattern matching inside tmux is unreliable — the `*$*` pattern matches banner text prematurely or the interact handoff fails
- **Fix/Workaround**: Use pure `tmux send-keys` for the entire SSH session. Send each command, sleep, check output with `tmux capture-pane`, then send the next command. No expect script needed.

### show file log requires exact container/filename path
- **Phase**: 3
- **Date**: 2026-04-02
- **User**: kgopalan
- **Symptom**: `show file log syslog` and `show file log aaa` both returned "Failed to show file"
- **Root Cause**: Log files are inside container directories (e.g., `management-engine/syslog`). The `show file log` command requires the exact path including the container prefix.
- **Fix/Workaround**: Run `show file log list | no-more` first to see available files and their exact paths, then use the full path. Use `show file log list | include <keyword> | no-more` to search for specific log files.

### System events log is routing_engine/system-events.log
- **Phase**: 3
- **Date**: 2026-04-02
- **User**: kgopalan
- **Symptom**: Could not find the correct log file for historical system events (AAA, TACACS, BFD, BGP state changes, etc.)
- **Root Cause**: The file is `routing_engine/system-events.log` (hyphenated, with `.log` extension). Not `system_events` or `system-events` without extension.
- **Fix/Workaround**: Use `show file log routing_engine/system-events.log | no-more` for full event history. Filter with `| include AAA` or `| include TACACS` for specific subsystems. Key log files for AAA investigation: `routing_engine/system-events.log` (system events), `routing_engine/login` (PAM auth), `routing_engine/sshd_cli_inband` (SSH sessions with full TACACS auth/authz/acct flow).

### Misinterpreted "aaa-local" authentication order
- **Phase**: 3
- **Date**: 2026-04-02
- **User**: kgopalan
- **Symptom**: Incorrectly flagged "aaa-local" as meaning local-only authentication
- **Root Cause**: "aaa-local" means AAA (TACACS/RADIUS) is tried first, then falls back to local if the server is unreachable. It is NOT "use local only."
- **Fix/Workaround**: "aaa-local" = correct order for TACACS with local fallback. Do not flag this as an issue.

### JIRA REST API v2 /search endpoint deprecated
- **Phase**: 1
- **Date**: 2026-04-02
- **User**: kgopalan
- **Symptom**: `curl` to `/rest/api/2/search` returned error: "The requested API has been removed. Please migrate to the /rest/api/3/search/jql API"
- **Root Cause**: Atlassian deprecated the v2 search endpoint
- **Fix/Workaround**: Use MCP `atlassian_jira_search` tool for duplicate detection instead of curl. If MCP unavailable, use `/rest/api/3/search/jql` endpoint.

### Cursor picks wrong skill when colleague has other skills installed
- **Phase**: 0
- **Date**: 2026-04-02
- **User**: colleague
- **Symptom**: Cursor used a different skill instead of jira-investigator when colleague said "investigate <ticket>"
- **Root Cause**: The colleague had another skill with overlapping trigger words. Cursor's skill resolution picked the wrong one.
- **Fix/Workaround**: Made the SKILL.md description and trigger section more specific — explicitly names the skill, lists all supported project prefixes, and instructs Cursor not to use any other skill for JIRA investigation. If this still happens, the user should prefix with "use jira-investigator skill" or remove/rename conflicting skills.

### tmux must be required, not optional
- **Phase**: 3
- **Date**: 2026-04-02
- **User**: colleague
- **Symptom**: Without tmux, Phase 3 fell back to expect-based batch mode which is unreliable for interactive troubleshooting
- **Root Cause**: Expect scripts are fragile for multi-hop SSH with banners and prompts. tmux send-keys is the only reliable method.
- **Fix/Workaround**: Made tmux a required tool. Phase 0 now auto-installs it via `brew install tmux` if missing. Removed Option B (batch mode) from Phase 3 entirely. **SUPERSEDED by dn-connect-mcp** — see below.

### dn-connect-mcp replaces tmux/expect for SSH
- **Phase**: 3
- **Date**: 2026-04-02
- **User**: kgopalan
- **Symptom**: tmux + expect scripts for multi-hop SSH were fragile — timing issues, prompt matching failures, pager swallowing commands, expect hanging on banners
- **Root Cause**: Shell-level scripting (tmux send-keys, expect pattern matching) is inherently unreliable for complex multi-hop SSH with warning banners and password prompts
- **Fix/Workaround**: `dn-connect-mcp` MCP server handles SSH natively. Key advantages:
  - Multi-hop via nested `jumpHost` objects in `ssh_connect` — no expect scripts
  - `waitForPrompt` with regex `\S+#\s*$` — deterministic prompt detection, no sleep/poll
  - DNOS-aware — returns CLI reference guides with every `ssh_exec` response
  - `ssh_shell_start/send/read/close` for interactive sessions
  - `ssh_exec` for one-shot commands
  - Profiles (`ssh_profile_create`) for saved connection presets
  - tmux is now the legacy fallback, only used if MCP is not available

### CRITICAL: Must use profiles (not inline credentials) for DNOS connections
- **Phase**: 3
- **Date**: 2026-04-06
- **User**: kgopalan
- **Symptom**: `ssh_exec` returned "Invalid command" with exit code 1 when connected via inline credentials (no profile). The command was never sent to the DNOS CLI — it was executed as a bash command on the SSH target.
- **Root Cause**: Without a profile with `type: "dnos"`, the MCP does not know the device is DNOS. It treats it as a generic Linux server. `ssh_exec` uses an exec channel which fails on DNOS. `ssh_shell` still works because it just sends keystrokes, but loses all DNOS intelligence (no CLI guide injection, no capsule resource, no DNOS-aware error detection).
- **Fix/Workaround**: ALWAYS create profiles before connecting:
  1. Create jump host profiles with `type: "jump-host"` and `jumpHostProfile` chaining
  2. Create the router profile with `type: "dnos"` and `jumpHostProfile` referencing the last hop
  3. Connect with `profileName` instead of inline credentials
  4. Check existing profiles first with `ssh_profile_list`
  - Benefits: `ssh_exec` works correctly, every response auto-injects the relevant CLI reference guide, DNOS capsule resource on connect, profiles persist and are reusable
  - Updated Phase 3 Step 3a to mandate profiles instead of inline credentials

### CRITICAL: Must read MCP CLI resource before running ANY DNOS command
- **Phase**: 3
- **Date**: 2026-04-06
- **User**: kgopalan
- **Symptom**: Multiple command failures in a single session: `show isis adjacency` (Cisco), `show file list /var/tmp/` (Linux), `show system tech-support` (wrong command for listing files), `show config policy route-map` (wrong config hierarchy). Every failure was caused by guessing from Cisco/Juniper/Linux instead of reading the MCP CLI resource that was right there.
- **Root Cause**: The `dn-connect-mcp` server instructions explicitly say "ALWAYS read the guide before running a command" and "Anti-Pattern: Any command you consider as Cisco/Juniper is wrong." These instructions were ignored. The agent defaulted to Cisco/Juniper/Linux syntax from training data instead of consulting the available DNOS CLI reference guides at `resource://dn-connect/guide/dnos/*`.
- **Fix/Workaround**: This is a HARD GATE, not a suggestion. Before running ANY command on a DNOS router:
  1. Identify the command category (BGP, ISIS, system, file, policy, etc.)
  2. Call `FetchMcpResource` for the relevant guide (e.g., `routing-igp-mpls` for ISIS)
  3. Find the exact command in the "Show Commands" table
  4. Use that exact syntax with `| no-more` appended
  5. If the guide doesn't cover your exact need, use `cmd help` on the router
  6. NEVER type a command from Cisco/Juniper memory
  - Common wrong commands: `show isis adjacency` (→ `show isis neighbors`), `show ip bgp` (→ `show bgp summary`), `show running-config` (→ `show config`), `show version` (→ `show system version`), `show ip route` (→ `show route`), `show file list` (→ `show file <category> list`), `terminal length 0` (→ does not exist, use `| no-more`)
  - Updated SKILL.md Key Rule 4 and Phase 3 Step 2-PRE to enforce this as a hard gate

### Correct command to check existing techsupport files on DNOS
- **Phase**: 3
- **Date**: 2026-04-06
- **User**: kgopalan
- **Symptom**: `show file list`, `show system tech-support status` were wrong commands to check for existing techsupport bundles. `show system tech-support` dumps live data to terminal (huge output).
- **Root Cause**: The techsupport directory is under `show file` hierarchy, not `show system`. Discovery: `show file ?` → `tech-support` → `show file tech-support ?` → `list`.
- **Fix/Workaround**: Use `show file tech-support list | no-more` to check for existing techsupport bundles on the router. Returns a table with Type, Id, File name, Size, and Last modified. Empty table = no files.

### CRITICAL: JIRA ticket description must follow SW-253878 reference format
- **Phase**: 5
- **Date**: 2026-04-08
- **User**: kgopalan
- **Symptom**: User rejected ticket draft 5+ times because formatting was wrong. Used numbered steps ("Step 1", "Step 2"), plain text labels before code blocks, mixed markdown with JIRA markup, and procedural walkthroughs instead of presenting evidence naturally.
- **Root Cause**: The bug template in phase5 was too generic. It listed formatting rules but didn't enforce a concrete reference pattern. The agent kept reverting to its own style.
- **Fix/Workaround**: Use SW-253878 as the canonical reference format:
  1. `*Bold Text*` for ALL section headings AND sub-labels before `{code}` blocks
  2. `{code}` blocks for ALL CLI/router output — no exceptions
  3. `{{inline code}}` for function names, file paths, variable names
  4. Plain text for explanations between code blocks
  5. Brief observation after each code block (e.g., "All 6 PSU sensors alarming. Actual temperatures: 27–40°C.")
  6. NO numbered steps, NO "Step 1/2/3", NO procedural walkthrough
  7. NO `h1.`/`h2.`/`h3.` headings, NO `{noformat}`
  8. Present evidence naturally with bold context labels, then code block, then observation
  9. When presenting draft to user, show ONLY the raw JIRA wiki markup — no markdown wrapping

### CRITICAL: Verify impact through full system flow before stating severity
- **Phase**: 4/5
- **Date**: 2026-04-08
- **User**: kgopalan
- **Symptom**: Claimed "reboot will break all passwords" (severity 1) without checking whether Redis data survives reboots. Config Redis uses AOF persistence and survives normal reboots — the bug only manifests when config Redis is wiped (golden config mode, recovery, data loss).
- **Root Cause**: Jumped to catastrophic conclusion based on one code path (`add_master_key_to_orm` reads from file) without tracing whether that code path actually executes on normal reboot. Did not check `init_redis_if_not_exist()` which skips startup config generation when Redis keys are present.
- **Fix/Workaround**: Before stating impact/severity in a ticket:
  1. Trace the FULL system flow for the failure scenario (boot, reboot, golden config, recovery)
  2. Check Redis persistence model (AOF vs RDB, which Redis instances persist)
  3. Check all entry points to the affected code path and when they execute
  4. Only claim impact for scenarios confirmed by code, not assumptions
  5. Clearly distinguish "normal reboot" vs "golden config mode" vs "factory-default" vs "recovery"

### Never ask user to confirm technical findings
- **Phase**: All
- **Date**: 2026-04-08
- **User**: kgopalan
- **Symptom**: Asked "does this match your understanding of how reboots work?" instead of determining the answer from code
- **Root Cause**: Uncertainty about the boot flow led to asking the user for validation instead of investigating further
- **Fix/Workaround**: The user comes for support. Never ask them to confirm your technical analysis. If uncertain, dig deeper into the code. Trace the flow, read the configs, check the entrypoints. Only present findings, not questions about system behavior.

### Batch investigation: multiple related tickets in one run
- **Phase**: 1
- **Date**: 2026-04-08
- **User**: kgopalan
- **Symptom**: User provided 3 related ART tickets (ART-9943, 9944, 9945) for a single investigation
- **Root Cause**: The skill was designed for one ticket at a time
- **Fix/Workaround**: Updated Phase 1 to support batch mode — fetch all tickets in parallel, identify relationships, present combined intake summary. Subsequent phases share context (one codebase investigation covers all related tickets). Updated SKILL.md trigger to accept multiple ticket keys.

### Security/hardening tickets follow a different investigation pattern
- **Phase**: 2/3/4
- **Date**: 2026-04-08
- **User**: kgopalan
- **Symptom**: Three security hardening tickets (master key concerns) had no techsupport, no operational failure, no live data needed. The standard bug investigation flow (techsupport → live collection → code tracing) didn't fit.
- **Root Cause**: The skill assumed all tickets are functional bugs or NFRs. Security hardening tickets focus on architecture review, access control analysis, and evaluating customer-proposed fixes.
- **Fix/Workaround**: Added "Security Hardening" classification to Phase 4. When tickets are security/hardening: skip Phase 2 (no techsupport), skip Phase 3 (no live data), focus Phase 4 on architecture review and evaluating customer's proposed resolutions. Phase 1 now handles PDF/PPTX evidence from customer security reviews.

### External evidence (PDFs, emails, dev discussions) as primary input
- **Phase**: 1
- **Date**: 2026-04-08
- **User**: kgopalan
- **Symptom**: Customer's AT&T security review presentation (PDF) was the primary evidence source, more detailed than the JIRA tickets themselves. Internal dev email chain provided additional context confirming code findings.
- **Root Cause**: Phase 1 only handled JIRA ticket fields and attachments. External documents and communications weren't part of the intake flow.
- **Fix/Workaround**: Updated Phase 1 to accept and incorporate external evidence: PDFs, PPTX, email threads, CLI output, screenshots. Map external findings to specific tickets. Cross-reference dev communications with code analysis.

### Evaluate customer's proposed resolution before drafting SW ticket
- **Phase**: 4
- **Date**: 2026-04-08
- **User**: kgopalan
- **Symptom**: Customer provided specific proposed resolutions for each ticket. Needed to assess validity of each ask independently before drafting R&D tickets.
- **Root Cause**: Phase 4 classified Bug/NFR but didn't include a step for evaluating customer-proposed fixes.
- **Fix/Workaround**: Added Step 5b to Phase 4: evaluate each customer proposed resolution for technical validity, feasibility, side effects, and scope. Present honest assessment — valid, partially valid, or overstated — with code evidence.

### Analyzed wrong branch — missed delivered fix
- **Phase**: 4
- **Date**: 2026-03-24
- **User**: kgopalan
- **Symptom**: Reported hardcoded ENCRYPTION_KEY as unfixed for ART-4288, when in fact the fix had been delivered on `dev_v25_4_13`. The analysis was done on `dev_v25_4` (user's current checkout) instead of the customer's version branch.
- **Root Cause**: Phase 4 did not enforce version-aware branch resolution. It analyzed whatever branch the user happened to be on locally, rather than inspecting the branch matching the customer's version.
- **Fix/Workaround**: Updated Phase 4 Step 0 with mandatory branch resolution:
  1. Extract FULL version from Phase 1 (e.g., `25.4.13`, not just `25.4`)
  2. Search branches from most specific (`dev_v25_4_13`) to least specific (`dev_v25_4`)
  3. `git checkout` to the target branch (stash if dirty, always switch back after)
  4. Always announce which branch is being analyzed
  5. Check for fix-related commits on the target branch before deep analysis
  - Phase 1 also updated to stress extracting exact patch/build version and recording fix_version separately
  - **Updated 2026-03-24**: Changed from `git show` remote inspection to actual `git checkout`. Remote inspection was too clunky — loses grep, semantic search, and normal file tools. Switching branches is safe with auto-stash and always switching back.
