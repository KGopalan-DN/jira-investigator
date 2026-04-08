# Phase 2 — Techsupport Analysis

Download and analyze techsupport bundles referenced in the source ticket.

## Input

Techsupport references collected in Phase 1:
- FTP paths (`dn@ftp.drivenets.com:/path/...`)
- Minio URLs (`http://minio.../...`)
- JIRA attachment URLs (`.tar.gz`, `.tgz` files)
- Paths mentioned in comments

## Step 0: No Techsupport Available

If Phase 1 found no techsupport references:
- Report: "No techsupport found on this ticket. Proceeding to Phase 3 / Phase 4 with ticket data only."
- Skip this phase entirely

## Step 1: Create Working Directory

```bash
mkdir -p ~/AI/tickets/<TICKET_KEY>/techsupport
```

## Step 2: Download Techsupport

Try each source type. Prefer local download; SSH/SCP as fallback for FTP.

### From JIRA Attachments

```bash
curl -s -L -o ~/AI/tickets/<TICKET_KEY>/techsupport/<filename> \
  -H "Authorization: Basic $(echo -n '<email>:<token>' | base64)" \
  "<attachment_content_url>"
```

### From Minio

```bash
wget -q -O ~/AI/tickets/<TICKET_KEY>/techsupport/<filename> \
  "<minio_url>"
```

If `wget` fails (access restricted), note it and try curl:

```bash
curl -s -L -o ~/AI/tickets/<TICKET_KEY>/techsupport/<filename> \
  "<minio_url>"
```

### From FTP Server

**Option A — SCP (preferred, requires sshpass):**

```bash
sshpass -p '1!-HmR_Dg9Te!!4' scp -o StrictHostKeyChecking=no \
  dn@ftp.drivenets.com:<remote_path> \
  ~/AI/tickets/<TICKET_KEY>/techsupport/
```

**Option B — Expect-based SCP (if sshpass unavailable):**

```bash
expect -c '
spawn scp -o StrictHostKeyChecking=no dn@ftp.drivenets.com:<remote_path> ~/AI/tickets/<TICKET_KEY>/techsupport/
expect "*assword:*"
send "1!-HmR_Dg9Te!!4\r"
expect eof
'
```

**If download fails entirely:**
- Report: "Could not download techsupport from <source>. Error: <error>. You can manually place the file in `~/AI/tickets/<TICKET_KEY>/techsupport/` and re-run this phase."

## Step 3: Extract Techsupport

```bash
cd ~/AI/tickets/<TICKET_KEY>/techsupport
tar -xf <filename>
```

If tar fails (corrupt or wrong format), try:

```bash
gunzip <filename>  # might be plain gzip
```

If extraction fails, note it and try to work with the raw file.

After extraction, list the contents:

```bash
find ~/AI/tickets/<TICKET_KEY>/techsupport -type f | head -100
```

Common techsupport structure:
- `var/log/` — system logs
- `var/log/syslog*` — main syslog
- `var/log/dnos/` — DNOS-specific logs
- `etc/` — configuration files
- `proc/` — system info snapshots
- `show_commands/` — captured CLI output
- `core/` — core dumps (if any)

## Step 4: Targeted Log Analysis

Based on the issue described in the ticket, search for relevant evidence in the logs.

### General Approach

1. **Identify keywords** from the ticket summary and description (alarm names, feature names, error messages, component names)
2. **Search syslog** for those keywords around the reported time:

```bash
zgrep -i "<keyword>" ~/AI/tickets/<TICKET_KEY>/techsupport/var/log/syslog* | head -50
```

3. **Search DNOS logs** for specific component errors:

```bash
zgrep -i "<keyword>" ~/AI/tickets/<TICKET_KEY>/techsupport/var/log/dnos/*.log* | head -50
```

4. **Check captured show commands** if present:

```bash
ls ~/AI/tickets/<TICKET_KEY>/techsupport/show_commands/ 2>/dev/null
cat ~/AI/tickets/<TICKET_KEY>/techsupport/show_commands/<relevant_file>
```

### Issue-Type-Specific Searches

**Alarm-related issues:**
- `zgrep -i "alarm\|ALARM" <syslog>` — alarm raise/clear events
- `zgrep -i "<alarm_name>" <syslog>` — specific alarm
- Look for alarm oscillation patterns (rapid raise/clear cycles)

**Crash/restart issues:**
- `zgrep -i "segfault\|core dump\|panic\|abort\|killed" <syslog>`
- `zgrep -i "restart\|respawn\|exited" <syslog>`
- Check for core dumps in `core/` directory

**Configuration issues:**
- Check `etc/` for running configuration
- `zgrep -i "commit\|config\|error" <syslog>`

**Connectivity/protocol issues:**
- `zgrep -i "bgp\|ospf\|isis\|interface\|link" <syslog>` (adjust per ticket)
- `zgrep -i "down\|flap\|timeout" <syslog>`

**Hardware issues:**
- `zgrep -i "temperature\|sensor\|fan\|power\|hardware" <syslog>`
- Check `proc/` for hardware state snapshots

## Step 5: Timeline Reconstruction

From the log evidence found:

1. **Build a timeline** of relevant events:
   - When did the issue first appear?
   - Was there a trigger event (config change, upgrade, hardware event)?
   - Is the issue intermittent or persistent?
   - When was the last known good state?

2. **Correlate with ticket timeline:**
   - When was the ticket created?
   - Do the log timestamps match the reported issue timeframe?
   - Any discrepancy between reported time and log evidence?

## Step 6: Present Findings

Present the techsupport analysis:

```
Techsupport Analysis:
  Source: <download source>
  Extracted: <number of files>

Log Evidence:
  1. <timestamp> — <event description>
  2. <timestamp> — <event description>
  ...

Timeline:
  - First occurrence: <time>
  - Trigger: <if identified>
  - Pattern: <intermittent/persistent/escalating>

Relevant Configurations:
  - <config snippet if relevant>

Observations:
  - <observation 1>
  - <observation 2>

Data Gaps:
  - <anything missing that would help: e.g., "No core dump present", "Logs only cover last 24h">

Initial Theory:
  <brief hypothesis based on logs — to be validated in Phase 4>
```

**Wait for user confirmation before proceeding to Phase 3 or Phase 4.**
