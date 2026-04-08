# Phase 0 — Bootstrap

Validate the environment before investigation begins. Run this once at the start of every investigation.

## Step 1: JIRA Credentials

Extract Atlassian email and API token from the user's MCP config.

1. Read `~/.cursor/mcp.json`
2. Look for an MCP server entry with headers containing `X-Email-User` and `X-Atlassian-Token`
3. Store both values for use in subsequent phases

**If `mcp.json` is not found or missing credentials:**
- Ask the user: "I need your Atlassian email and API token. You can generate a token at https://id.atlassian.com/manage-profile/security/api-tokens"
- Store the values they provide

**Validation:**
- Test JIRA access by running:

```bash
curl -s -w "%{http_code}" -o /dev/null \
  "https://drivenets.atlassian.net/rest/api/2/myself" \
  -H "Authorization: Basic $(echo -n '<email>:<token>' | base64)"
```

- If HTTP 200 → credentials are valid
- If HTTP 401/403 → tell user credentials are invalid, ask them to re-check

## Step 2: MCP Server Detection

Check which MCP servers are available.

### 2a — JIRA MCP (`dn-mcp-server`)

1. Try listing MCP tool descriptors at the project's MCP folder
2. Look for a tool named `atlassian_jira_get_issue` or similar

**If available:**
- Use MCP for JIRA read operations (faster, structured output)
- Use `curl` for JIRA write operations (create, update, link, comment)

**If not available:**
- Use `curl` for all JIRA operations (both reads and writes)

### 2b — SSH MCP (`dn-connect-mcp`)

Check for `ssh_connect`, `ssh_exec`, `ssh_shell_start`, `ssh_shell_send`, `ssh_shell_read`, `ssh_profile_create`, `ssh_profile_list` tools.

**If available:**
- Use `dn-connect-mcp` for all SSH/router operations in Phase 3 (preferred method)
- **MUST use profiles** — `ssh_exec` does NOT work without a profile with `type: "dnos"`. Inline credentials will silently fail.
- `ssh_profile_create` with `type: "dnos"` and `jumpHostProfile` chaining for multi-hop
- `ssh_exec` returns structured output with auto-injected CLI reference guides
- `ssh_shell_start/send/read` for interactive investigation
- Uses `waitForPrompt` for deterministic prompt detection — no sleep/poll hacks
- No tmux or expect required

**Check for existing profiles:**

```
CallMcpTool: user-dn-connect-mcp / ssh_profile_list
{ "type": "dnos" }
```

If profiles exist, list them for the user: "Found saved router profiles: <list>. You can reuse these in Phase 3."

**If not available:**
- Fall back to tmux + expect for Phase 3 (legacy method)
- Requires tmux installed (auto-install via `brew install tmux`)

## Step 3: Repository Detection

Detect local repositories for DNOS, DNOR, and DAP. Only the repo relevant to the ticket's product matters — but detect all available ones upfront.

**Detection method for each product:**

| Product | Identifier | Search for |
|---------|-----------|------------|
| DNOS | `VERSION` file containing `DNOS` | Scan home directory for repos with this file |
| DNOR | `VERSION` file containing `DNOR` | Scan home directory for repos with this file |
| DAP | `VERSION` file containing `DAP` or repo named `dap` | Scan home directory |

**Search strategy:**

```bash
# Check common locations first
for dir in ~/cheetah-1 ~/dnos ~/DNOS ~/cheetah ~/dnor ~/DNOR ~/dap ~/DAP; do
  if [ -d "$dir" ]; then
    echo "$dir: $(head -1 $dir/VERSION 2>/dev/null || echo 'no VERSION file')"
  fi
done
```

Also check the current Cursor workspace — it's likely one of the repos.

**If a repo is not found:**
- Ask: "Where is your <product> repository? (full path, or 'skip' if you don't have it)"
- If they skip, that product's codebase investigation will be unavailable — note this but don't block

**For each found repo, record:**
- Path
- Product (DNOS/DNOR/DAP)

Branch checkout and version matching happens in Phase 4, based on the customer's version from the ticket.

## Step 4: Tool Detection

Check for required and optional tools.

**Required:**
- `curl` — always available on macOS
- `python3` — always available on macOS
- `ssh` — always available on macOS
- `expect` — usually available on macOS (`/usr/bin/expect`)

**Conditional (only if SSH MCP is not available):**
- `tmux` — needed for Phase 3 if `dn-connect-mcp` is not available. Check with `which tmux`
  - If missing and SSH MCP is also missing: install it automatically via `brew install tmux`
  - If SSH MCP is available: tmux is not needed

**Optional:**
- `python-pptx` — needed for parsing PPTX attachments. Check with `python3 -c "import pptx" 2>/dev/null`
  - If missing: "python-pptx is not installed. PPTX attachments won't be parsed. Install with `pip3 install python-pptx`"
- `sshpass` — needed for FTP upload if SSH MCP is not available. Check with `which sshpass`
  - If missing: FTP upload step will use expect-based scp instead

## Step 5: Present Environment Summary

Present the results clearly:

```
Environment Check:
  JIRA Access:    ✓ authenticated as <email>
  JIRA MCP:       ✓ available (or: ✗ using curl fallback)
  SSH MCP:        ✓ dn-connect-mcp available (or: ✗ using tmux fallback)
  SSH Profiles:   ✓ <count> DNOS profiles saved (or: ✗ none — will create in Phase 3)
  DNOS Repo:      ✓ <path> (or: ✗ not found — codebase investigation unavailable for DNOS)
  DNOR Repo:      ✓ <path> (or: ✗ not found — codebase investigation unavailable for DNOR)
  DAP Repo:       ✓ <path> (or: ✗ not found — codebase investigation unavailable for DAP)
  python-pptx:    ✓ installed (or: ✗ not installed — PPTX parsing unavailable)

Ready to investigate. Provide a JIRA ticket key to begin.
```

If JIRA credentials failed, stop here — nothing else works without JIRA access.

If everything else passed, proceed to Phase 1 when the user provides a ticket key.
