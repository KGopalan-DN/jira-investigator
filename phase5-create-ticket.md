# Phase 5 — Create R&D Ticket

Create the follow-up ticket in the user's chosen JIRA project, link it to the source ticket, and optionally upload techsupport.

## Input

- Full analysis from Phases 1–4
- Classification: Bug or NFR
- User-confirmed analysis and draft

## Step 0: Check for Existing R&D Tickets

Before creating, search for duplicates in the target project:

```
atlassian_jira_search: summary ~ "<key_terms>" AND project = <TARGET_PROJECT>
```

Also check if the source ticket already has linked SW/WS/AR tickets.

**If a potential duplicate is found:**
- Present it: "Found existing ticket <KEY>: <summary>. Status: <status>."
- Ask: "Do you want to: (A) Skip creation and link to this existing ticket, (B) Create a new ticket anyway, (C) Let me check this ticket first?"
- Wait for user decision.

## Step 0b: Confirm Target Project

Ask the user:
"Which JIRA project should I create the follow-up ticket in?"
- SW (DNOS R&D)
- WS (DNOR R&D)
- AR (DAP R&D)
- Other: specify project key

Also confirm:
- Issue type: Bug or New Feature (based on classification, but let user override)
- Component (e.g., "DNOS alarms", "DNOS CLI", etc.)

## Step 0c: Present Full Draft for Approval

**Before any API call**, present the complete ticket draft to the user:

```
--- DRAFT ---
Project: <TARGET_PROJECT>
Type: <Bug / New Feature>
Summary: <SOURCE_KEY>: <summary>
Priority: <priority>
Component: <component>

Description:
<full description in readable format — NOT wiki markup, show it human-readable>

Support Tab (Bug only):
  Customer Issue: Yes
  Customer Name: <name>
  Customer Severity: <sev>
--- END DRAFT ---

Does this look good? (Yes / Edit / Cancel)
```

- If "Yes" → proceed to Step 1
- If "Edit" → ask what to change, update, show again
- If "Cancel" → stop, do not create

**NEVER create a ticket without showing the draft first.**

## Step 1: Map Priority

Map customer severity to R&D priority:

| Customer Severity | R&D Priority |
|------------------|-------------|
| 1 | Highest |
| 2 | High |
| 3 | Medium |
| Not specified | Medium (default) |

## Step 2: Build Description

Format the ticket description in JIRA Wiki Markup.

**Formatting rules (reference: SW-253878):**
- `*Bold text*` for section headings AND sub-labels before code blocks — do NOT use `h1.`, `h2.`, `h3.` headings
- `{code}` blocks for ALL CLI/router output and tabular data
- `{{inline code}}` for file paths, function names, commands, variable names
- Plain text for explanations between code blocks
- Brief observation line after each code block summarizing what it shows
- NO numbered steps ("Step 1", "Step 2") — present evidence naturally
- NO `{noformat}` blocks — always use `{code}`
- NO procedural walkthroughs — use bold context label → code block → observation
- Blank line between sections

**Example pattern from SW-253878:**
```
*Section Heading*

*Device/context label* — platform, version info

{code}
CLI output here
{code}

Brief plain text observation about the output above.
```

### Bug Template

```
*Customer Ticket*
<SOURCE_KEY>: <source_summary>
Customer: <customer_name>
Severity: <customer_severity>
Version: <customer_version>

*Problem Description*
<Clear explanation of the issue in 2-3 sentences. What is happening, when, and the impact.>

*Environment*
Product: <product>
Version: <version>
Platform: <platform/hardware if known>

*Log Evidence*
{code}
<relevant log snippets — timestamps, error messages, alarm events>
{code}

*Root Cause Analysis*
<Explanation of the code path and what is going wrong. Reference specific files and functions using {{file:function}} format.>

<If regression:>
Regression introduced in commit {{<commit_hash>}} (<commit message summary>).
Affects branches: <list of affected version branches>.

*Suggested Fix*
<Brief description of what needs to change.>
<Reference specific files/functions.>

*Steps to Reproduce*
<If known from the investigation — conditions that trigger the issue.>

*Workaround*
<If identified during investigation, describe it. Otherwise: "No known workaround.">
```

### NFR Template

```
*Customer Ticket*
<SOURCE_KEY>: <source_summary>
Customer: <customer_name>
Severity: <customer_severity>
Version: <customer_version>

*Feature Request*
<Clear explanation of what the customer is requesting. 2-3 sentences.>

*Customer Requirements*
<Bullet list of specific requirements extracted from the ticket, comments, and internal discussions.>
- <requirement 1>
- <requirement 2>
- <requirement 3>

*Current Behavior*
<What the system does today. Reference YANG model if relevant: "No YANG model support exists for <feature>.">

*Expected Behavior*
<What the customer expects to happen.>

*Use Case*
<Why the customer needs this — business context from the ticket.>

*Design Considerations*
<Any technical notes from the codebase investigation — where this would need to be implemented, what components are affected.>
```

## Step 3: Create Ticket

Write the JSON payload to a temporary file to avoid shell escaping issues:

```bash
cat > ~/AI/tickets/<SOURCE_KEY>/ticket_payload.json << 'PAYLOAD'
{
  "fields": {
    "project": { "key": "<TARGET_PROJECT>" },
    "summary": "<SOURCE_KEY>: <summary>",
    "issuetype": { "name": "<Bug or New Feature>" },
    "priority": { "name": "<mapped_priority>" },
    "description": "<description_placeholder>",
    "components": [{ "name": "<component>" }],
    "versions": [{ "name": "<affects_version>" }]
  }
}
PAYLOAD
```

Note: Use a short placeholder for description in the initial creation. JIRA's default template may overwrite the description on creation.

```bash
curl -s -X POST \
  "https://drivenets.atlassian.net/rest/api/2/issue" \
  -H "Authorization: Basic $(echo -n '<email>:<token>' | base64)" \
  -H "Content-Type: application/json" \
  -d @~/AI/tickets/<SOURCE_KEY>/ticket_payload.json
```

Extract the created ticket key from the response (`key` field).

**Error handling:**
- If HTTP 400: check the response body — usually means a required field is missing or a field value is invalid (e.g., component doesn't exist in that project). Show the error to the user and ask how to fix.
- If HTTP 401/403: credentials are invalid or user lacks permission in the target project. Re-check credentials.
- If HTTP 404: project key is wrong. Ask user to confirm the project.
- If creation succeeds but key is empty/null: something went wrong — show the full response to the user.

## Step 4: Update Description

After creation, update the description with the full content to avoid JIRA template overwrite:

```bash
cat > ~/AI/tickets/<SOURCE_KEY>/ticket_update.json << 'PAYLOAD'
{
  "fields": {
    "description": "<full_wiki_markup_description>"
  }
}
PAYLOAD

curl -s -X PUT \
  "https://drivenets.atlassian.net/rest/api/2/issue/<CREATED_KEY>" \
  -H "Authorization: Basic $(echo -n '<email>:<token>' | base64)" \
  -H "Content-Type: application/json" \
  -d @~/AI/tickets/<SOURCE_KEY>/ticket_update.json
```

## Step 4b: Copy Labels from Source Ticket

Copy all labels from the source ticket to the newly created ticket. Labels carry context (product area, customer tags, feature flags) that R&D uses for filtering and triage.

```bash
# Labels were already extracted in Phase 1. Apply them:
cat > ~/AI/tickets/<SOURCE_KEY>/labels_update.json << 'PAYLOAD'
{
  "update": {
    "labels": [
      {"add": "<label1>"},
      {"add": "<label2>"}
    ]
  }
}
PAYLOAD

curl -s -X PUT \
  "https://drivenets.atlassian.net/rest/api/2/issue/<CREATED_KEY>" \
  -H "Authorization: Basic $(echo -n '<email>:<token>' | base64)" \
  -H "Content-Type: application/json" \
  -d @~/AI/tickets/<SOURCE_KEY>/labels_update.json
```

If the source ticket has no labels, skip this step.

## Step 5: Set Support Tab Fields (Bug only)

These custom fields are only available on the Bug issue screen, not New Feature.

**The Customer Name and Customer Severity MUST match the source ticket exactly.** Do not re-interpret or change them. Copy the values as-is from what Phase 1 extracted.

**IMPORTANT: The SW project has TWO "Customer Issue" fields.** Both must be set to "Yes" for the Support Tab to display correctly:
- `customfield_10501` — Customer Issue (legacy/shared field)
- `customfield_11575` — Customer Issue (SW project-specific field)

If you skip `customfield_11575`, the Support Tab will show "Customer Issue: No" even though `customfield_10501` is "Yes".

**Discovery tip:** If unsure which custom fields a project uses, query the editmeta endpoint first:
```bash
curl -s "https://drivenets.atlassian.net/rest/api/2/issue/<CREATED_KEY>/editmeta" \
  -H "Authorization: Basic $(echo -n '<email>:<token>' | base64)" \
  -H "Content-Type: application/json" | python3 -c "
import sys, json
fields = json.load(sys.stdin)['fields']
for k, v in sorted(fields.items()):
    name = v.get('name','').lower()
    if any(kw in name for kw in ['customer','support','severity']):
        print(f'{k}: {v[\"name\"]} (required={v.get(\"required\",False)})')
"
```

```bash
cat > ~/AI/tickets/<SOURCE_KEY>/support_fields.json << 'PAYLOAD'
{
  "fields": {
    "customfield_10501": { "value": "Yes" },
    "customfield_11575": { "value": "Yes" },
    "customfield_11630": [{ "value": "<customer_name_from_source>" }],
    "customfield_11533": { "value": "<customer_severity_from_source>" }
  }
}
PAYLOAD

curl -s -X PUT \
  "https://drivenets.atlassian.net/rest/api/2/issue/<CREATED_KEY>" \
  -H "Authorization: Basic $(echo -n '<email>:<token>' | base64)" \
  -H "Content-Type: application/json" \
  -d @~/AI/tickets/<SOURCE_KEY>/support_fields.json
```

- `customfield_10501` — Customer Issue (legacy): always "Yes"
- `customfield_11575` — Customer Issue (SW-specific): always "Yes"
- `customfield_11630` — Customer Name: **exact value from source ticket** (e.g., "AT&T - Artemis"). Format: `[{"value": "<name>"}]`
- `customfield_11533` — Customer Severity: **exact value from source ticket** (e.g., "1", "2", or "3")

**If the source ticket does not have Customer Name or Customer Severity set:**
- Ask the user: "The source ticket doesn't have Customer Name / Severity set. What should I use?"
- Do NOT guess or leave blank — these fields are important for R&D triage.

**If issue type is New Feature, skip this step** — these fields are not available on the NFR screen.

## Step 6: Link to Source Ticket

```bash
curl -s -X POST \
  "https://drivenets.atlassian.net/rest/api/2/issueLink" \
  -H "Authorization: Basic $(echo -n '<email>:<token>' | base64)" \
  -H "Content-Type: application/json" \
  -d '{
    "type": { "name": "Relates" },
    "inwardIssue": { "key": "<SOURCE_KEY>" },
    "outwardIssue": { "key": "<CREATED_KEY>" }
  }'
```

## Step 7: Add Comment on Source Ticket

Post a **generic** comment on the source ticket. Source tickets (ART, etc.) are customer-facing — never expose internal R&D ticket keys, URLs, or details in the comment.

```bash
curl -s -X POST \
  "https://drivenets.atlassian.net/rest/api/2/issue/<SOURCE_KEY>/comment" \
  -H "Authorization: Basic $(echo -n '<email>:<token>' | base64)" \
  -H "Content-Type: application/json" \
  -d '{
    "body": "Opened an internal ticket for investigation. Will revert once we have more feedback."
  }'
```

**Rules:**
- Do NOT include the internal ticket key (e.g., SW-XXXXX) in the comment
- Do NOT include URLs to internal tickets
- Do NOT describe the internal investigation details
- Keep it brief and customer-appropriate
- The JIRA link between the tickets (Step 6) already provides internal traceability — no need to duplicate it in comments

## Step 8: Upload Techsupport to FTP (Optional)

If techsupport was downloaded in Phase 2, offer to upload it for R&D:

"Do you want me to upload the techsupport to the FTP server for R&D access? It will be placed at `/ftpdisk/dn/att-uploads/<CREATED_KEY>/`"

If user confirms:

```bash
# Create directory on FTP
sshpass -p '1!-HmR_Dg9Te!!4' ssh -o StrictHostKeyChecking=no dn@ftp.drivenets.com \
  "mkdir -p /ftpdisk/dn/att-uploads/<CREATED_KEY>"

# Upload techsupport tarball
sshpass -p '1!-HmR_Dg9Te!!4' scp -o StrictHostKeyChecking=no \
  ~/AI/tickets/<SOURCE_KEY>/techsupport/<tarball_filename> \
  dn@ftp.drivenets.com:/ftpdisk/dn/att-uploads/<CREATED_KEY>/
```

If `sshpass` is unavailable, use expect:

```bash
expect -c '
spawn ssh -o StrictHostKeyChecking=no dn@ftp.drivenets.com "mkdir -p /ftpdisk/dn/att-uploads/<CREATED_KEY>"
expect "*assword:*"
send "1!-HmR_Dg9Te!!4\r"
expect eof
'

expect -c '
spawn scp -o StrictHostKeyChecking=no ~/AI/tickets/<SOURCE_KEY>/techsupport/<tarball_filename> dn@ftp.drivenets.com:/ftpdisk/dn/att-uploads/<CREATED_KEY>/
expect "*assword:*"
send "1!-HmR_Dg9Te!!4\r"
expect eof
'
```

After upload, add a comment on the R&D ticket with the FTP path:

```bash
curl -s -X POST \
  "https://drivenets.atlassian.net/rest/api/2/issue/<CREATED_KEY>/comment" \
  -H "Authorization: Basic $(echo -n '<email>:<token>' | base64)" \
  -H "Content-Type: application/json" \
  -d '{
    "body": "Techsupport uploaded to FTP:\ndn@ftp.drivenets.com:/ftpdisk/dn/att-uploads/<CREATED_KEY>/<tarball_filename>"
  }'
```

## Step 8b: Verify Created Ticket

After all updates, read back the created ticket to verify everything was applied:

```
atlassian_jira_get_issue: <CREATED_KEY>
```

Check:
- Description is populated (not empty or placeholder)
- Priority matches
- Component matches
- Support tab fields are set (Bug only) — verify BOTH `customfield_10501` AND `customfield_11575` are "Yes"
- Labels copied from source ticket
- Link to source ticket exists

If anything is missing, attempt to fix it. If it can't be fixed automatically, tell the user what needs manual attention.

## Step 9: Present Summary

```
Ticket Created:
  Key: <CREATED_KEY>
  URL: https://drivenets.atlassian.net/browse/<CREATED_KEY>
  Project: <TARGET_PROJECT>
  Type: <Bug / New Feature>
  Priority: <priority>
  Component: <component>

Actions Completed:
  ✓ Ticket created with full description
  ✓ Support tab populated (Bug only)
  ✓ Linked to <SOURCE_KEY>
  ✓ Comment added on <SOURCE_KEY>
  ✓ Techsupport uploaded to FTP (if applicable)

Investigation complete for <SOURCE_KEY>.
```
