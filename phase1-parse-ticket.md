# Phase 1 — Parse Ticket

Read and extract all relevant data from the source JIRA ticket(s).

## Input

One or more JIRA ticket keys provided by the user (e.g., `ART-9684`, `CS-1234 CS-1235 CS-1236`).

### Batch Mode

When the user provides multiple tickets:
1. Fetch all tickets in parallel (concurrent MCP calls or curl requests)
2. Identify relationships — are they related (same root cause, same feature area, cloned from the same parent)?
3. If related: present a combined intake summary grouped by theme, note shared fields (reporter, product, version, labels), highlight what's unique per ticket
4. If unrelated: treat as separate investigations, present individual summaries, ask the user which to investigate first
5. For related tickets, subsequent phases can share context (e.g., one codebase investigation covers all tickets)

### External Evidence

The user may provide additional context beyond what's in JIRA:
- **PDF/PPTX documents**: customer presentations, security reviews, test reports. Read and extract key findings, map them to specific tickets.
- **Email threads**: internal dev discussions, customer communications. Cross-reference with code analysis.
- **CLI output / screenshots**: pasted directly or as images. Parse for error messages, config snippets, version info.
- **Links to SharePoint/Confluence/external docs**: note them, attempt to fetch if accessible, ask user for content if not.

Incorporate all external evidence into the intake summary alongside JIRA ticket data. Flag any contradictions between the ticket description and the external evidence.

## Step 1: Fetch Ticket Data

Using MCP (if available) or curl, fetch the ticket with all fields.

**Via MCP:**

```
CallMcpTool: atlassian_jira_get_issue
  issue_key: <TICKET_KEY>
```

**Via curl (if MCP unavailable):**

```bash
curl -s -L \
  "https://drivenets.atlassian.net/rest/api/2/issue/<TICKET_KEY>?fields=summary,description,status,priority,components,versions,fixVersions,issuelinks,labels,assignee,reporter,comment,attachment,customfield_11533,customfield_11630,customfield_10501,created,updated" \
  -H "Authorization: Basic $(echo -n '<email>:<token>' | base64)"
```

**Extract and record:**

| Field | Source | Notes |
|-------|--------|-------|
| Key | `key` | |
| Summary | `summary` | |
| Description | `description` | Full text — scan for techsupport links |
| Status | `status.name` | |
| Priority | `priority.name` | |
| Components | `components[].name` | Helps identify product area |
| Affects Versions | `versions[].name` | Customer's version — critical for branch matching |
| Fix Versions | `fixVersions[].name` | |
| Labels | `labels[]` | May indicate security, milestone info |
| Assignee | `assignee.displayName` | Check if assigned to someone else |
| Reporter | `reporter.displayName` | Customer contact |
| Customer Severity | `customfield_11533.value` | 1, 2, or 3 |
| Customer Name | `customfield_11630[].value` | |
| Created | `created` | |
| Attachments | `attachment[]` | Filenames, sizes, URLs |
| Links | `issuelinks[]` | Existing related tickets |

## Step 2: Read Comments

Fetch all comments from the ticket. Comments often contain:
- Techsupport links posted by customer or JesterAI bot
- Additional context not in the description
- Internal discussion about the issue
- Requirements additions (especially for NFRs)

Parse each comment — record author, date, and body.

## Step 3: Download and Parse Attachments

For each attachment on the ticket:

**Download:**

```bash
curl -s -L -o "<local_path>/<filename>" \
  -H "Authorization: Basic $(echo -n '<email>:<token>' | base64)" \
  "<attachment_content_url>"
```

Store downloads in `~/AI/tickets/<TICKET_KEY>/attachments/`.

**Parse by type:**

| File Type | How to Parse |
|-----------|-------------|
| `.csv` | `python3` with `csv` module — extract columns, counts, summaries |
| `.pptx` | `python3` with `python-pptx` — extract text and tables from all slides |
| `.txt`, `.log` | Read directly |
| `.png`, `.jpg` | Note presence, show to user if relevant |
| `.tar.gz`, `.tgz` | This is techsupport — hand off to Phase 2 |
| `.pdf` | Read tool supports PDFs — read directly |
| Other | Note the file, don't attempt to parse |

If `python-pptx` is not installed (detected in Phase 0), skip PPTX parsing and note it.

## Step 4: Scan for Techsupport Links

Search description, comments, and attachment filenames for techsupport references.

**Patterns to look for:**
- `dn@ftp.drivenets.com:/path/...` or `dn@ftp:/path/...` — FTP server path
- `http://minio...` or `https://minio...` — Minio object storage URL
- `techsupport` in attachment filenames
- `.tar.gz` or `.tgz` attachments
- Links to other JIRA tickets that may contain techsupport

Record all found techsupport references for Phase 2.

## Step 5: Identify Product and Version

Determine the product (DNOS, DNOR, DAP) and the customer's **exact version** including patch/build number from:

1. **Components field** — `DNOS`, `DNOR`, `DAP` directly
2. **Affects Versions field** — version string often contains product name (e.g., `v25.4.13 (DNOS)`)
3. **Fix Versions field** — if a fix was delivered, this tells you which version branch to check
4. **Description/comments** — may mention product or version explicitly (e.g., "running 25.4.133", "upgraded to v25.4.13")
5. **Labels** — may contain product identifiers
6. **Linked tickets** — linked SW/WS/AR tickets often specify the target version

**CRITICAL: Extract the FULL version string, not just major.minor.**

- `v25.4` is NOT enough — branch `dev_v25_4` and `dev_v25_4_13` can have completely different code
- Look for patterns like `25.4.13`, `25.4.133`, `v25.4.13`, `25_4_13`
- If the ticket mentions a fix version (e.g., "fix delivered in v25.4.13"), record BOTH the customer's running version AND the fix version

**Record:**
- `product`: DNOS / DNOR / DAP
- `customer_version`: exact version customer is running (e.g., `25.4.133`)
- `fix_version`: version where a fix was delivered, if mentioned (e.g., `25.4.13`)
- `target_branch`: will be resolved in Phase 4 from these versions

**If product cannot be determined:**
- Flag it: "Could not determine product from ticket fields. Which product is this for? (DNOS / DNOR / DAP / Other)"

**If version cannot be determined:**
- Flag it: "Customer version not specified. What version is the customer running? (Need full version including patch number, e.g., 25.4.13, not just 25.4)"

Do not proceed to Phase 4 (codebase investigation) without product and version.

## Step 6: Check Assignee

If the ticket is assigned to someone other than the current user:
- Flag: "This ticket is assigned to <name>. Do you want to proceed with investigation anyway?"
- Wait for user confirmation

## Step 7: Duplicate Detection

Check if follow-up tickets already exist for this issue.

**Check 1 — Existing links on the ticket:**

Review `issuelinks` for outward/inward links to other projects (SW, WS, AR, etc.). If found:
- "This ticket is already linked to <KEY>: <summary>. Do you want to: (a) view the existing ticket, (b) create a new one anyway, (c) stop?"

**Check 2 — Search for similar tickets:**

```bash
curl -s -L \
  "https://drivenets.atlassian.net/rest/api/2/search?jql=summary~\"<key_words>\" AND project in (SW,WS,AR) ORDER BY created DESC&maxResults=5" \
  -H "Authorization: Basic $(echo -n '<email>:<token>' | base64)"
```

Extract 2-3 key words from the summary for the search. If potential duplicates are found, present them and ask the user.

## Step 8: Present Intake Summary

Present a structured summary to the user. For batch investigations, present a combined summary.

**Single ticket:**

```
Ticket: <KEY>
Summary: <summary>
Status: <status>
Priority: <priority>
Severity: <customer_severity>
Reporter: <reporter>
Assignee: <assignee>
Product: <product> (or: ⚠ not determined)
Version: <version> (or: ⚠ not specified)
Components: <components>
Customer: <customer_name>
Created: <date>

Attachments: <count> files
  - <filename1> (<size>)
  - <filename2> (<size>)

Techsupport: <found/not found>
  - <link or filename if found>

Linked Tickets:
  - <link_type> <KEY>: <summary>

Flags:
  - <any issues: missing version, assigned to someone else, duplicate found, etc.>
```

**Batch (multiple related tickets):**

```
Investigation: <common theme>
Tickets: <KEY1>, <KEY2>, <KEY3>
Reporter: <shared reporter>
Product: <product>
Version: <version>
Customer: <customer_name>
Shared Labels: <labels common to all>

Per-Ticket Summary:
  <KEY1>: <summary> — <classification hint: bug/NFR/hardening>
  <KEY2>: <summary> — <classification hint>
  <KEY3>: <summary> — <classification hint>

External Evidence:
  - <document/email/CLI output provided by user>

Existing R&D Tickets:
  - <any SW/WS/AR tickets already covering these concerns>

Flags:
  - <issues, gaps, contradictions>
```

**Wait for user confirmation before proceeding to Phase 2.**
