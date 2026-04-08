# Phase 3 — Live Box Data Collection

Collect data from a live DNOS router via SSH. Always offer this to the user — live data from the box strengthens every investigation.

**DNOS only.** DNOR and DAP do not have CLI access via SSH.

## Prerequisites

- `dn-connect-mcp` available (preferred) **OR** `tmux` + `expect` installed (legacy fallback)
- User provides connection details

## Step 0: Propose Live Investigation

Always ask the user — don't silently skip this phase. Present what you have and what live data would add:

"Based on Phases 1–2, here's what I have so far:
  - <brief summary of data collected>
  - <any gaps or open questions>

I'd like to check the router directly to <specific reasons — e.g., verify current state, check system-events.log, confirm configuration, look at live counters>. This would strengthen the analysis.

Do you want me to connect to a router? I'll only run `show` commands — no config changes. (Yes / No)"

If user declines, skip this phase and proceed to Phase 4 with available data.

## Step 1: Identify the Router

**Step 1a — Look up the router in the topology file:**

Read [topology.md](topology.md) and search for the router hostname mentioned in the ticket. If found, you already have the management IP — present it to the user for confirmation:

"The ticket mentions router <hostname>. From the lab topology, the management IP is <mgmt_ip>. Is this the router you want me to connect to?"

If the hostname is a production/customer name (e.g., "L-Agg-303") that doesn't match any lab device, tell the user:

"The router name <hostname> doesn't match any device in the lab topology. Can you provide the management IP for this router?"

**Step 1b — Ask for the hop path:**

There is no direct SSH access to DNOS routers. Access always goes through one or more intermediate servers (jump hosts).

"To reach the router, I need the servers you hop through to get there. For example:
  'I go through 32.40.217.6 and then 135.16.39.205 to reach the router'

How many hops do you go through, and what are the IPs?"

**Step 1b — Walk through each hop:**

Once the user describes the path, confirm each hop and ask for credentials one at a time:

"Got it. Let me confirm the path:
  Hop 1: <ip> — what's the username and password?
  Hop 2: <ip> — what's the username and password?
  Final router: <ip> — what's the username and password?"

If the user provides all at once, that's fine too — adapt to whatever format they give.

**Step 1c — Confirm before connecting:**

"Here's the plan:
  1. SSH to <hop1> as <user>
  2. SSH to <hop2> as <user>
  3. SSH to <router_ip> as <user>
  4. Run `show` commands only — no config changes

Does this look right?"

Wait for confirmation before connecting.

Store credentials — do not log passwords in output shown to the user.

## Step 2: Look Up Commands

Before connecting to the router — and before running ANY command on it — look up the correct syntax.

**Step 2-PRE — HARD GATE: Read the MCP CLI resource BEFORE running ANY command**

**THIS IS NOT OPTIONAL. DO NOT SKIP THIS STEP. DO NOT GUESS COMMANDS.**

DNOS is NOT Cisco/Juniper. The `dn-connect-mcp` server instructions explicitly state:
> "Anti-Pattern: Any command you consider as Cisco/Juniper is wrong. ALWAYS read the guide before running a command."

**Before you type ANY show command on the router, you MUST first call `FetchMcpResource` for the relevant guide.** No exceptions. If you haven't read the resource for a command category, you are NOT allowed to run commands in that category.

**Required resources — read the ones you need BEFORE connecting:**

| Resource URI | MUST read before running |
|---|---|
| `resource://dn-connect/guide/dnos/core` | `show system`, `show config system`, AAA, login, certificates |
| `resource://dn-connect/guide/dnos/routing-bgp` | `show bgp`, any BGP command |
| `resource://dn-connect/guide/dnos/routing-igp-mpls` | `show isis`, `show ospf`, MPLS, SR, BFD, VRRP |
| `resource://dn-connect/guide/dnos/interfaces` | `show interface`, bundles, L2 |
| `resource://dn-connect/guide/dnos/policy-security` | `show config routing-policy`, ACLs, QoS |
| `resource://dn-connect/guide/dnos/services` | VRF, network services |
| `resource://dn-connect/guide/dnos/operations` | `show file`, tech-support, system ops |
| `resource://dn-connect/guide/dnos/show-reference` | General show command index |
| `resource://dn-connect/guide/dnos/action-reference` | `request` and `clear` commands |
| `resource://dn-connect/guide/dnos/multicast` | PIM, IGMP, MSDP |

**Workflow for every command:**
1. Identify the category (BGP? ISIS? system? file?)
2. `FetchMcpResource` for that category's guide — find the exact command in the "Show Commands" table
3. Use the exact syntax from the guide, appending `| no-more`
4. Only if the guide doesn't cover your specific need: `cmd help <command>` on the router
5. Only for discovery of unknown commands: `cmd search <keyword>` on the router

**If MCP is unavailable**, check RST files at `prod/dnos_monolith/dnos_cli/`. Then `cmd help`/`cmd search`.

**Violations to catch yourself on:**
- `show isis adjacency` → WRONG (Cisco). Correct: `show isis neighbors`
- `show ip bgp` → WRONG (Cisco). Correct: `show bgp summary`
- `show running-config` → WRONG (Cisco). Correct: `show config`
- `show version` → WRONG (Cisco). Correct: `show system version`
- `show ip route` → WRONG (Cisco). Correct: `show route`
- `show file list` → WRONG. Correct: `show file <category> list`
- `terminal length 0` → DOES NOT EXIST. Use `| no-more` per command.

If you catch yourself about to type a command without having read the resource first — STOP. Read the resource. Then type the command.

**Step 2-CONFIG — Quick reference: common `show config` paths**

When checking configuration, you MUST use the correct hierarchy path. Here are the most common ones:

| What | `show config` path |
|---|---|
| Route policy | `routing-policy policy <name>` |
| Prefix list | `routing-policy prefix-list ipv4 <name>` |
| Community list | `routing-policy community-list <name>` |
| AS-path ACL | `routing-policy as-path-acl-list <name>` |
| BGP | `protocols bgp <asn>` |
| BGP neighbor | `protocols bgp <asn> neighbor <ip>` |
| ISIS | `protocols isis instance <name>` |
| OSPF | `protocols ospf instance <name>` |
| Static routes | `protocols static` |
| BFD | `protocols bfd` |
| VRRP | `protocols vrrp` |
| LLDP | `protocols lldp` |
| Interfaces | `interfaces <if-name>` |
| ACLs (IPv4) | `access-lists ipv4 <name>` |
| ACLs (IPv6) | `access-lists ipv6 <name>` |
| AAA | `system aaa` |
| TACACS | `system tacacs-server` |
| RADIUS | `system radius-server` |
| SSH server | `system ssh server` |
| gRPC | `system grpc` |
| NETCONF | `system netconf` |
| Telemetry | `system telemetry` |
| NTP | `system ntp` |
| Logging | `system logging` |
| SNMP | `system snmp` |
| CPRL | `system cprl` |
| Config groups | `group <name>` |
| VRF | `network-services vrf instance <name>` |
| QoS | `qos` |
| Global ACL | `forwarding-options global-acl` |
| Keychains | `system keychain <name>` |
| NACM | `nacm` |

**Useful `show config` modifiers:**
- `show config | flatten` — one-line-per-leaf (full path, great for grep)
- `show config display-inherited` — shows group-inherited config
- `show config defaults` — shows default values in `[brackets]`
- `show config compare` — diff candidate vs running (config mode only)

**Common mistake:** `show config policy ...` is WRONG. The path is `routing-policy`, not `policy`.

**Step 2a — Standard baseline commands (ALWAYS run, every ticket):**

These commands apply to every investigation regardless of issue type. Run them first, every time:

| Command | Why |
|---------|-----|
| `show system version \| no-more` | Confirm software version, uptime, hardware model |
| `show file log routing_engine/system-events.log \| no-more` | **The universal event history** — captures all system events (AAA, BFD, BGP, alarms, hardware, NETCONF, protocol state changes, rate limits, etc.). Always check this first. |

To find specific events within `system-events.log`, use filters:

```
show file log routing_engine/system-events.log | include <keyword> | no-more
```

Common filter keywords: `AAA`, `TACACS`, `BFD`, `BGP`, `OSPF`, `ALARM`, `NETCONF`, `ERROR`, `WARNING`, `CRITICAL`, `interface`, `link`, `reboot`, `restart`.

**Step 2b — Issue-specific commands:**

Based on the reported issue, add commands from the relevant categories:

| Issue Type | Commands to Look Up |
|-----------|-------------------|
| AAA/Auth | `show system aaa-servers tacacs`, `show config system aaa-server`, `show file log routing_engine/login`, `show file log routing_engine/sshd_cli_inband` |
| Alarms | `show system alarms severity critical`, `show system alarms severity major` |
| Hardware | `show system hardware temperature`, `show system hardware` |
| Configuration | `show config <path>` |
| Protocol | `show route`, `show bgp`, `show isis`, `show ospf` |
| Interface | `show interface`, `show system interface` |
| gRPC | `show system grpc`, `show system security certificate` |
| System | `show system`, `show system processes` |

**Step 2c — Discover additional logs if needed:**

To find available log files for deeper investigation:

```
show file log list | no-more
show file log list | include <keyword> | no-more
```

Log files are stored under container directories (e.g., `routing_engine/`, `management-engine/`, `node-manager/`). Always use the full path when reading them.

**Step 2d — Verify syntax before every new command category:**

Before running commands in a new category (e.g., switching from system commands to BGP commands), re-check the MCP CLI resource for that category. This is Step 2-PRE applied throughout the investigation, not just at the start.

If MCP is not available, check local RST files:

```bash
cat ~/cheetah-1/prod/dnos_monolith/dnos_cli/Show\ Commands/<command>.rst
```

**If command syntax is unknown after checking resources:** use `cmd search <keyword>` and `cmd help <command>` on the router.

## Step 3: Establish Connection

### Method A — dn-connect-mcp (preferred)

Use `dn-connect-mcp` MCP tools. This handles multi-hop SSH natively, detects DNOS prompts automatically, and provides CLI reference guides with every response.

**CRITICAL: Always use profiles, not inline credentials.**

`ssh_exec` DOES NOT WORK with inline credentials on DNOS devices. Without a profile with `type: "dnos"`, the MCP treats the device as Linux and `ssh_exec` fails. Profiles also enable:
- Auto-injected CLI reference guides with every `ssh_exec` response
- DNOS capsule resource on connect (safety rules, command discovery help)
- Reusable connections — no credentials needed after first setup
- Clean jump host chaining via `jumpHostProfile`

**Step 3a — Create profiles (first time only):**

Create a profile chain: jump hosts first, then the DNOS device referencing them.

```
CallMcpTool: user-dn-connect-mcp / ssh_profile_create
{
  "name": "<hop1-name>",
  "type": "jump-host",
  "host": "<hop1_ip>",
  "username": "<hop1_user>",
  "password": "<hop1_password>",
  "displayName": "Jump Host 1"
}
```

```
CallMcpTool: user-dn-connect-mcp / ssh_profile_create
{
  "name": "<hop2-name>",
  "type": "jump-host",
  "host": "<hop2_ip>",
  "username": "<hop2_user>",
  "password": "<hop2_password>",
  "jumpHostProfile": "<hop1-name>",
  "displayName": "Jump Host 2"
}
```

```
CallMcpTool: user-dn-connect-mcp / ssh_profile_create
{
  "name": "<router-hostname-lowercase>",
  "type": "dnos",
  "host": "<router_ip>",
  "username": "<router_user>",
  "password": "<router_password>",
  "jumpHostProfile": "<hop2-name>",
  "displayName": "<Router Hostname>",
  "tags": ["<lab-name>", "<role>"]
}
```

Profile names must be kebab-case (e.g., `e2e-mse-02`). Profiles persist across sessions — create once, reuse forever.

Check existing profiles first: `ssh_profile_list` (optionally filter by `type: "dnos"`).

**Step 3b — Connect using profile:**

```
CallMcpTool: user-dn-connect-mcp / ssh_connect
{
  "profileName": "<router-profile-name>"
}
```

This returns a `sessionId` and the DNOS capsule resource. Store the sessionId for all subsequent commands.

**Step 3c — Start interactive shell (if using ssh_shell mode):**

```
CallMcpTool: user-dn-connect-mcp / ssh_shell_start
{
  "sessionId": "<sessionId>"
}
```

This returns a `shellId`. Store it for send/read operations.

**Step 3d — Verify connection:**

```
CallMcpTool: user-dn-connect-mcp / ssh_shell_send
{
  "sessionId": "<sessionId>",
  "shellId": "<shellId>",
  "input": "show system version | no-more\n"
}
```

Then read:

```
CallMcpTool: user-dn-connect-mcp / ssh_shell_read
{
  "sessionId": "<sessionId>",
  "shellId": "<shellId>",
  "waitForPrompt": "\\S+#\\s*$",
  "waitTimeoutMs": 30000
}
```

Verify the output shows the router hostname and version.

**Key MCP rules:**
- Always append `\n` to the `input` in `ssh_shell_send` to execute the command
- Always use `waitForPrompt` with regex `\\S+#\\s*$` in `ssh_shell_read` — do NOT poll with sleep
- For long-running outputs (like `system-events.log`), increase `waitTimeoutMs` to 60000 or higher
- Append `| no-more` to every show command — DNOS CLI does NOT support `terminal length 0`

**Step 3e — Choose ssh_exec vs ssh_shell:**

Two modes for running commands. Use the right one:

**`ssh_exec` (preferred for most commands):**
- Returns clean, structured output (stdout/stderr/exitCode)
- Auto-injects the relevant DNOS CLI reference guide with every response (`_cliReference` field)
- If a command fails, the injected guide helps you find the correct syntax immediately
- Best for: individual commands, healthchecks, quick lookups
- **Requires a profile with `type: "dnos"`** — does NOT work with inline credentials

```
CallMcpTool: user-dn-connect-mcp / ssh_exec
{
  "profileName": "<router-profile-name>",
  "command": "show system version | no-more"
}
```

**`ssh_shell_start/send/read` (for iterative investigation):**
- Interactive shell with persistent state
- You control the flow — run a command, analyze output, decide what to run next
- No auto-injected CLI guides — you must read MCP resources manually (Step 2-PRE)
- Best for: multi-step troubleshooting, following a chain of evidence, config exploration

Use `ssh_exec` by default. Switch to `ssh_shell` only when you need iterative, dependent commands where the next command depends on the previous output.

### Method B — tmux (legacy fallback, only if MCP unavailable)

Only use this if `dn-connect-mcp` is not available.

**Step 1 — Create tmux session:**

```bash
tmux new-session -d -s "investigation-<TICKET_KEY>"
```

**Step 2 — Connect using chained SSH via expect:**

Write an expect script with chained hops (not `ssh -J`) and `interact` at the end. Run it inside the tmux session.

**Step 3 — Send commands via tmux send-keys:**

```bash
tmux send-keys -t "investigation-<TICKET_KEY>" "show system version | no-more" Enter
sleep 5
tmux capture-pane -t "investigation-<TICKET_KEY>" -p -S -100
```

This method is timing-dependent and fragile. Use MCP whenever possible.

## Step 3f: Handle Large/Unexpected Output

**If a command produces unexpectedly massive output (megabytes), the command is probably wrong.** Stop and verify:

1. `show system tech-support` (without a filename) dumps the ENTIRE system state to screen — never run this. Use `show file tech-support list | no-more` to check for existing files, or `request system tech-support <filename>` to generate one.
2. `show route | no-more` on a router with 1.5M routes will produce massive output. Use `show route summary | no-more` instead, or filter with `show route <prefix> | no-more`.
3. `show config | no-more` on a fully configured router can be very large. Use `show config <specific-path> | no-more` to target what you need.

**Rules:**
- If `ssh_shell_read` returns `truncated: true`, the output was cut off — the command produced more than the buffer can hold
- If `ssh_exec` returns `stdoutTruncated: true`, same issue
- Use `| include <keyword>` to filter large outputs before they hit the buffer
- If you get an unexpected flood of output in a shell, wait for the prompt to return — don't send more commands until the current one finishes

**Session recovery — if the connection drops:**

1. Check: `ssh_list_sessions` — look for `connected: true/false`
2. If dropped: call `ssh_connect` with `profileName` again (profiles persist, just reconnect)
3. Start a new shell with `ssh_shell_start`
4. Tell the user: "Session dropped, reconnected. Continuing investigation."

Do NOT panic or restart the investigation — just reconnect and resume from where you left off.

## Step 4: Run Show Commands

**CRITICAL: Only `show` commands and `cmd search`/`cmd help`. NEVER enter config mode. NEVER run commands that modify state.**

**The SSH session stays alive throughout this step.** Run commands, analyze the output, decide if more commands are needed, run those — all within the same session. Do NOT disconnect between commands. Do NOT disconnect while waiting for user input or presenting findings. Only disconnect in Step 7 when the user confirms they have all the data they need.

### With MCP (Method A):

**For baseline/independent commands — use `ssh_exec` (preferred):**

```
ssh_exec: { "profileName": "<router-profile>", "command": "<show_command> | no-more" }
```

Each response auto-injects the relevant CLI reference guide. Check `_cliReference` in the response. If the command fails, the injected guide shows the correct syntax.

**For iterative investigation — use `ssh_shell`:**

```
ssh_shell_send: { "input": "<show_command> | no-more\n" }
ssh_shell_read: { "waitForPrompt": "\\S+#\\s*$" }
```

Analyze the output. Decide if follow-up commands are needed. Run them.

### With tmux (Method B):

```bash
tmux send-keys -t "investigation-<TICKET_KEY>" "<show_command> | no-more" Enter
sleep 5
tmux capture-pane -t "investigation-<TICKET_KEY>" -p -S -100
```

### Iterative Investigation

This is the key advantage of interactive mode. Based on initial output:

1. **Analyze** the output from each command
2. **Decide** if additional commands are needed based on what you see
3. **Look up** new command syntax via MCP CLI resources, RST files, or `cmd search`/`cmd help`
4. **Run** follow-up commands

Example flow:
- `show system alarms` reveals an alarm → check `show system hardware temperature` for specifics
- Temperature looks abnormal → check `show system hardware` for platform details
- Need to verify config → `show config system <path>`

### Using cmd search / cmd help

If you need a command not found in RST files or MCP resources:

```
cmd search <keyword>
```

This returns available commands matching the keyword. Then:

```
cmd help <full_command_path>
```

This returns the syntax and parameters for that command.

## Step 5: Capture and Save Output

Save all collected output locally:

```bash
mkdir -p ~/AI/tickets/<TICKET_KEY>
```

If using MCP, the outputs are already captured in the tool responses. Save them to files for reference:

```bash
# Save each command output to a file for the investigation record
echo "<output>" > ~/AI/tickets/<TICKET_KEY>/output_<command_name>.txt
```

## Step 6: Collect Techsupport from Router (Sub-flow, Optional)

This sub-flow generates a techsupport bundle directly on the router and uploads it to the FTP server. It uses `request` commands (not `show`), so it **requires explicit user authorization** at every step.

**NEVER run this sub-flow without user approval. NEVER use the `inclusive` flag — it is traffic-affecting and requires a system restart.**

### Step 6a: Ask for Authorization

Present this to the user:

"I can collect a techsupport bundle directly from this router. This will:
1. Run `request system tech-support` — generates a tarball in the background (safe, no traffic impact)
2. Poll `show system tech-support status` until generation is complete (may take several minutes)
3. Upload the file from the router to the FTP server using `request file upload`

This uses `request` commands (operator-level), not config changes. No service impact.

Do you want me to proceed? (Yes / No)"

**If user says No → skip to Step 7.**

### Step 6b: Check for Existing Techsupport

Before generating, check if any techsupport bundles already exist on the box:

```
show file tech-support list | no-more
```

This returns a table with Type, Id, File name, Size, and Last modified. Empty table = no files.

Then check if one is currently being generated:

```
show system tech-support status
```

- If "another process is already running" → tell user and wait, or skip
- If a previous file exists → `force` flag will auto-delete it

### Step 6c: Generate Techsupport

Look up exact syntax in RST file or MCP CLI resource first.

Run the command on the router:

```
request system tech-support <TICKET_KEY> force
```

The `force` flag avoids the interactive "delete previous file?" prompt. The router returns control immediately while generating in the background.

**Confirm to user:** "Techsupport generation started. Polling for completion..."

### Step 6d: Poll for Completion

Poll every 30 seconds until generation is done:

```
show system tech-support status
```

Look for:
- "running for X minutes" + progress like "22/27 archives collected" → still running
- "successfully generated" → done
- The output filename (e.g., `ts_<TICKET_KEY>_HH_MM_SS_DD-MM-YYYY.tar`)

Report progress to user: "Techsupport generation in progress: <progress>. Estimated size: <size>."

**Timeout:** If generation exceeds 30 minutes, warn the user and ask if they want to keep waiting or abort.

### Step 6e: Ask Authorization for Upload

Once generation is complete, ask:

"Techsupport file generated: `<filename>` (<size>).
I need to get this file off the router so I can analyze the logs. The plan:

1. Upload from router to FTP server (`ftp.drivenets.com:/ftpdisk/dn/att-uploads/<TICKET_KEY>/`)
2. Download from FTP to your local machine for log analysis
3. The file stays on FTP for R&D access too

Do you want me to proceed? (Yes / No / Custom destination)"

**Wait for user response.**

### Step 6f: Upload from Router to FTP

**Upload to FTP server:**

```
request file upload tech-support <filename> dn@ftp.drivenets.com://ftpdisk/dn/att-uploads/<TICKET_KEY>/<filename>
```

The router will prompt for the FTP password. Send: `1!-HmR_Dg9Te!!4`

**If user skipped:** Note the file is on the router at `/techsupport/<filename>` for manual retrieval. Skip Steps 6g and 6h.

**Confirm to user:** "Techsupport uploaded to FTP. Downloading to local machine..."

### Step 6g: Download from FTP to Local Machine

Pull the file from FTP to the local working directory:

```bash
mkdir -p ~/AI/tickets/<TICKET_KEY>/techsupport
sshpass -p '1!-HmR_Dg9Te!!4' scp -o StrictHostKeyChecking=no \
  dn@ftp.drivenets.com:/ftpdisk/dn/att-uploads/<TICKET_KEY>/<filename> \
  ~/AI/tickets/<TICKET_KEY>/techsupport/
```

If `sshpass` is unavailable, use expect-based scp.

If download fails, report the error and note the FTP path for manual download.

### Step 6h: Extract and Analyze Locally

Once downloaded, run the same analysis as Phase 2:

```bash
cd ~/AI/tickets/<TICKET_KEY>/techsupport
tar -xf <filename>
```

Then apply the Phase 2 analysis steps:
- Search syslog for keywords related to the issue
- Search DNOS-specific logs
- Check captured show command outputs
- Build a timeline of relevant events

Re-read [phase2-techsupport.md](phase2-techsupport.md) Steps 4–5 and execute them against this newly downloaded techsupport. This gives a fresh, complete log analysis even if Phase 2 had no techsupport to work with earlier.

Present the new findings alongside the live `show` command data from Step 4.

### Step 6i: Record Techsupport Location

Store the techsupport locations for Phase 5:
- FTP path: `dn@ftp.drivenets.com:/ftpdisk/dn/att-uploads/<TICKET_KEY>/<filename>`
- Local path: `~/AI/tickets/<TICKET_KEY>/techsupport/<filename>`
- If custom destination: record the custom path

Both the FTP path (for R&D reference) and the local analysis results will be included in the R&D ticket description in Phase 5.

## Step 7: Disconnect

### With MCP (Method A):

```
CallMcpTool: user-dn-connect-mcp / ssh_shell_close
{ "sessionId": "<sessionId>", "shellId": "<shellId>" }

CallMcpTool: user-dn-connect-mcp / ssh_disconnect
{ "sessionId": "<sessionId>" }
```

### With tmux (Method B):

```bash
tmux send-keys -t "investigation-<TICKET_KEY>" "exit" Enter
sleep 2
tmux send-keys -t "investigation-<TICKET_KEY>" "exit" Enter
sleep 2
tmux kill-session -t "investigation-<TICKET_KEY>"
```

## Step 8: Present Collected Data

```
Live Data Collection:
  Router: <hostname>
  Method: <dn-connect-mcp / tmux>
  Commands Executed: <count>

Show Command Findings:
  1. <command>:
     <key observations from output>

  2. <command>:
     <key observations from output>

  ...

Techsupport Collection:
  Generated: <Yes/No>
  Filename: <filename>
  FTP: <ftp_path or N/A>
  Local: <local_path or N/A>
  Analyzed: <Yes/No>

Techsupport Log Analysis:
  <findings from extracted logs — same format as Phase 2 output>
  <timeline of events>
  <relevant configurations found>
  <data gaps>

Combined Analysis:
  <how show command output + techsupport logs + Phase 2 data all fit together>
  <updated or new theory based on all data>

Next: Proceeding to codebase investigation with this data.
```

**Wait for user confirmation before proceeding to Phase 4.**
