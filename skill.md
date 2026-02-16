---
name: tfvc-devops-skill
description: >-
  Use for TFVC repository operations — connect, browse, download, sync, and check in code
  to Azure DevOps TFVC repos. Invoke when user needs to sync, get, download, or check in to TFVC.
---

# TFVC Connection Skill

Use this skill for **TFVC (Team Foundation Version Control)** repository operations — connect, browse, download, sync, and check in code to Azure DevOps TFVC repositories.

## Trigger Conditions

Invoke this skill when the user needs to:
- Connect to a TFVC repository
- Browse TFVC server paths
- Download/sync files from TFVC
- Check in / commit files to TFVC
- Push local changes to the TFVC server
- Troubleshoot TFVC connectivity issues
- Set up a new TFVC workspace mapping

---

## STEP 0: Session Setup (MANDATORY)

**STOP. Before ANY TFVC operation, discover all required variables. Do NOT skip.**

### Required Variables

| Variable | Description | Example |
|:---------|:------------|:--------|
| `{TfExePath}` | Full path to `tf.exe` | `C:\Program Files\Microsoft Visual Studio\2022\Enterprise\Common7\IDE\CommonExtensions\Microsoft\TeamFoundation\Team Explorer\TF.exe` |
| `{OrgDiscovery}` | Azure DevOps organization URL (modern format) | `https://dev.azure.com/{OrgName}` |
| `{OrgTfvc}` | Azure DevOps TFVC URL (classic format) | `https://{orgname}.visualstudio.com` |
| `{ProjectName}` | TFVC project within the org (user-confirmed) | `WMS D365 Implementation` |
| `{Email}` | Azure DevOps user email | `user@domain.com` |
| `{PAT}` | Personal Access Token (sensitive — never store in skill or CLAUDE.md) | _(runtime only)_ |
| `{ServerPath}` | TFVC server path to work with | `$/{ProjectName}/Trunk` |
| `{LocalPath}` | Local directory to map/download to | `C:\Temp\MyProject\Trunk` |
| `{MachineName}` | Current computer hostname (auto-detected) | `CPC-Spark-F1IOA` |
| `{CheckinMethod}` | How this machine checks in to TFVC | `rest-api`, `workspace`, or `unknown` |
| `{WorkspaceName}` | TFVC workspace name (if checkin method is workspace) | `CPC-Spark-F1IOA_ARM` |

### Discovery Workflow

```
┌─────────────────────────────────────────────────────────────────┐
│  1. CHECK PROJECT CONFIG FOR SAVED VALUES                        │
│                                                                  │
│     Search CLAUDE.md (or project config files) for:             │
│     "## TFVC Configuration" section                              │
│                                                                  │
│     Extract any previously saved values:                         │
│     - tf.exe path                                                │
│     - Organization URLs                                          │
│     - Email                                                      │
│     - PAT File path (reference only — read PAT from that file)  │
│     - Server Path                                                │
│     - Local Path                                                 │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  2. AUTO-DISCOVER tf.exe PATH                                    │
│                                                                  │
│     Search known Visual Studio installation locations:           │
│                                                                  │
│     Bash (Git Bash on Windows):                                  │
│     ls "/c/Program Files/Microsoft Visual Studio/"*/            │
│        */Common7/IDE/CommonExtensions/Microsoft/                │
│        TeamFoundation/Team\ Explorer/TF.exe 2>/dev/null         │
│                                                                  │
│     PowerShell (fallback):                                       │
│     Get-ChildItem "C:\Program Files\Microsoft Visual Studio"    │
│       -Recurse -Filter "TF.exe" -ErrorAction SilentlyContinue  │
│       | Select -First 3 -ExpandProperty FullName                │
│                                                                  │
│     Common locations:                                            │
│     • VS 2022 Enterprise: ...\2022\Enterprise\Common7\IDE\...   │
│     • VS 2022 Professional: ...\2022\Professional\Common7\...   │
│     • VS 2019: ...\2019\{Edition}\Common7\IDE\...               │
│                                                                  │
│     If multiple found → ask user which to use                    │
│     If none found → ask user to provide path                     │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  3. RESOLVE PAT                                                  │
│                                                                  │
│     a. Check if CLAUDE.md has a "PAT File:" path reference      │
│        → If YES: Read PAT value from that file                  │
│        → If file missing/empty: warn user, ask for new PAT      │
│                                                                  │
│     b. If no PAT file path in config:                            │
│        → Ask user to provide PAT                                 │
│        → Default save location: Docs/PAT.txt (relative to       │
│          project root). Save automatically unless user declines  │
│          or specifies a different path.                           │
│        → Present to user:                                        │
│          "PAT will be saved to Docs/PAT.txt for future sessions. │
│           You can specify a different path or decline."           │
│          Options:                                                 │
│            1. Save to Docs/PAT.txt (default — recommended)       │
│            2. Save to a different path (user provides path)       │
│            3. Don't save (session only)                           │
│                                                                  │
│        If saving (option 1 or 2):                                │
│        → Create parent directory if needed (mkdir -p)            │
│        → Write PAT value to the chosen file path                 │
│        → Add PAT file path reference to CLAUDE.md               │
│        → EXCLUDE PAT file from source control:                   │
│          • Check .gitignore → add PAT filename pattern           │
│          • Check .tfignore → add PAT filename pattern            │
│          • If neither exists → ask user if want to create one    │
│          • Warn: "PAT files should never be checked in"          │
│                                                                  │
│        If user declines saving (option 3):                       │
│        → Use PAT for this session only                           │
│        → Next session will ask again                             │
│                                                                  │
│     CRITICAL: NEVER store PAT value in CLAUDE.md or skill files │
│     Only the file path reference is stored in CLAUDE.md          │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  4. COLLECT ORG URL, EMAIL                                       │
│                                                                  │
│     For {OrgDiscovery} and {Email}:                              │
│     → If present in CLAUDE.md → use them (trusted)               │
│     → If missing → ASK THE USER, do not guess                    │
│     → Auto-save to CLAUDE.md under "## TFVC Configuration"      │
│                                                                  │
│     Derive {OrgTfvc} from {OrgDiscovery}:                        │
│     → https://dev.azure.com/{OrgName} → https://{orgname}.      │
│       visualstudio.com  (lowercase the org name)                 │
│                                                                  │
│     CRITICAL — NEVER auto-discover org URLs or server paths:     │
│     • Do NOT infer {OrgDiscovery}, {OrgTfvc}, or {ServerPath}   │
│       from local TFVC workspace state (tf workspaces, etc.)     │
│     • The local machine may have workspaces for DIFFERENT        │
│       organizations/projects — using them silently connects to   │
│       the WRONG repo without the user knowing                    │
│     • ONLY trust values from CLAUDE.md "## TFVC Configuration"  │
│       — these were explicitly confirmed by the user              │
│     • If not in CLAUDE.md → ASK the user, do not guess           │
│     • Once the user provides values and they are saved to        │
│       CLAUDE.md, they are trusted for all future sessions        │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  5. TEST CONNECTIVITY                                            │
│                                                                  │
│     Run from OUTSIDE any existing TFVC workspace directory:      │
│     cd /c/Temp  (or any safe path outside workspaces)            │
│                                                                  │
│     & "{TfExePath}" workspaces                                   │
│       /collection:"{OrgDiscovery}"                               │
│       /loginType:OAuth                                           │
│       /login:"{Email}","{PAT}"                                   │
│                                                                  │
│     ✓ Success → proceed to step 5b                               │
│     ✗ TF30063 → PAT invalid/expired, ask user to regenerate     │
│     ✗ 404 → wrong org URL format                                 │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  5b. CHECK MACHINE STATE (for checkin capability)                │
│                                                                  │
│     Get machine hostname:                                        │
│       MACHINE=$(hostname)                                        │
│     Set {MachineName} = $MACHINE                                 │
│                                                                  │
│     Search CLAUDE.md for "## TFVC Machine State" section        │
│     → Find sub-heading "### {MachineName}"                       │
│                                                                  │
│     If found:                                                    │
│       → Parse "Checkin Method:" → set {CheckinMethod}            │
│       → Parse "Workspace:" → set {WorkspaceName} (if any)        │
│       → DONE — no further checkin discovery needed               │
│                                                                  │
│     If NOT found:                                                │
│       → Set {CheckinMethod} = "unknown"                          │
│       → Will be discovered on first checkin (Operation E)        │
│       → Print: "Checkin method not yet determined for            │
│         {MachineName}. Will discover on first checkin."          │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  6. RESOLVE PROJECT & SERVER PATH                                │
│                                                                  │
│     An org can contain MULTIPLE TFVC projects. Do NOT assume     │
│     which project to use — always confirm with the user.         │
│                                                                  │
│     a. If CLAUDE.md has "Server Path:" → use it (trusted)        │
│        → Skip to step 7                                          │
│                                                                  │
│     b. If CLAUDE.md has NO "Server Path:" →                      │
│        → Browse the TFVC root to list available projects:        │
│                                                                  │
│          cd /c/Temp                                               │
│          "{TfExePath}" dir '$/' \                                 │
│            /collection:"{OrgDiscovery}" \                        │
│            /loginType:OAuth \                                    │
│            /login:"{Email}","{PAT}"                              │
│                                                                  │
│        → Present the project list to the user                    │
│        → Ask user to CONFIRM which project to use                │
│        → Then ask for the sub-path within that project           │
│          (e.g., {ProjectName}/Trunk/DEV/Metadata/ARM)            │
│        → Combine into full {ServerPath}                          │
│          e.g., $/{ProjectName}/Trunk/DEV/Metadata/ARM            │
│        → Save confirmed {ServerPath} to CLAUDE.md                │
│                                                                  │
│     IMPORTANT: Even if only ONE project is listed, still         │
│     confirm with the user — do not auto-select.                  │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  7. COLLECT LOCAL PATH                                           │
│                                                                  │
│     For {LocalPath}:                                             │
│     → If present in CLAUDE.md → use it (trusted)                 │
│     → If missing → ask user, suggest a sensible default          │
│       based on current working directory                         │
│     → Auto-save to CLAUDE.md under "## TFVC Configuration"      │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  ✓ SESSION READY — proceed to requested operation                │
└─────────────────────────────────────────────────────────────────┘
```

### Session Summary Output (MANDATORY)

After completing setup, ALWAYS print:

```
┌──────────────────────────────────────────────────────────────┐
│  TFVC Connection Ready                                        │
└──────────────────────────────────────────────────────────────┘

  {TfExePath}    = [value]
  {OrgDiscovery} = [value]
  {OrgTfvc}      = [value]
  {ProjectName}  = [value]
  {Email}        = [value]
  {PAT}          = ****[last4] (from [source: file/user input])
  {ServerPath}      = [value]
  {LocalPath}       = [value]
  {MachineName}     = [hostname]
  {CheckinMethod}   = [rest-api | workspace | unknown]
  {WorkspaceName}   = [name | n/a]

Status: [✓ Connected | ✗ Connection failed — see error above]
```

---

## Core Operations

### Operation A: Test Connectivity

Verify authentication and access to the TFVC server.

```bash
# IMPORTANT: Run from OUTSIDE any TFVC workspace directory
cd /c/Temp

"{TfExePath}" workspaces \
  /collection:"{OrgDiscovery}" \
  /loginType:OAuth \
  /login:"{Email}","{PAT}"
```

**Success indicators:** No error, lists workspaces (or "No workspace" message).
**Failure:** See [troubleshooting.md](./reference/troubleshooting.md).

### Operation B: Browse Server Path

List contents of a TFVC server path.

```bash
# IMPORTANT: Run from OUTSIDE any TFVC workspace directory
cd /c/Temp

"{TfExePath}" dir '{ServerPath}' \
  /collection:"{OrgDiscovery}" \
  /loginType:OAuth \
  /login:"{Email}","{PAT}"
```

**Note:** Use single quotes around `{ServerPath}` in bash to prevent `$` expansion.

To browse recursively (list files in subdirectories):

```bash
"{TfExePath}" dir '{ServerPath}/{SubFolder}' \
  /collection:"{OrgDiscovery}" \
  /loginType:OAuth \
  /login:"{Email}","{PAT}" \
  /recursive
```

### Operation C: Download Files via `tf view` (PRIMARY method)

This is the **recommended download method** because it works without workspace creation permissions.

#### Step 1: List all files in target path

```bash
cd /c/Temp

"{TfExePath}" dir '{ServerPath}' \
  /collection:"{OrgDiscovery}" \
  /loginType:OAuth \
  /login:"{Email}","{PAT}" \
  /recursive
```

#### Step 2: Parse the directory listing

The `tf dir` output format:
```
$/Project/Trunk/Folder:
$File1.xml
$File2.xml

$/Project/Trunk/Folder/SubFolder:
$File3.xml
```

Parse this to build a list of `{serverFilePath}` → `{localFilePath}` pairs.

#### Step 3: Download each file via `tf view`

```bash
"{TfExePath}" view '{serverFilePath}' \
  /collection:"{OrgDiscovery}" \
  /loginType:OAuth \
  /login:"{Email}","{PAT}" \
  > "{localFilePath}"
```

#### Step 4: Run downloads in parallel batches

- Create local directory structure first (`mkdir -p`)
- Run 5-10 `tf view` commands in parallel for speed
- Verify file count after download matches server count

#### Complete Bulk Download Pattern

```bash
# 1. Create local directory structure
mkdir -p "{LocalPath}"

# 2. List server contents
cd /c/Temp
"{TfExePath}" dir '{ServerPath}' \
  /collection:"{OrgDiscovery}" \
  /loginType:OAuth \
  /login:"{Email}","{PAT}" \
  /recursive > /tmp/tfvc_listing.txt

# 3. Parse listing and download files
# For each directory block in the listing:
#   a. Extract the server directory path
#   b. Map to local directory path
#   c. mkdir -p the local directory
#   d. For each file in that directory:
#      tf view '{serverDir}/{fileName}' ... > '{localDir}/{fileName}'

# 4. Verify
echo "Downloaded $(find "{LocalPath}" -type f | wc -l) files"
```

### Operation D: Workspace-based Get (if permitted)

Attempt workspace creation. If blocked by permissions, fall back to Operation C.

```bash
cd /c/Temp

# Step 1: Create workspace
"{TfExePath}" workspace /new {WorkspaceName} \
  /collection:"{OrgDiscovery}" \
  /location:local \
  /permission:Private \
  /filetime:current \
  /loginType:OAuth \
  /login:"{Email}","{PAT}"

# If TF14044 error → Fall back to Operation C (tf view bulk download)

# Step 2: Map server path to local path
"{TfExePath}" workfold /map '{ServerPath}' "{LocalPath}" \
  /collection:"{OrgTfvc}" \
  /workspace:{WorkspaceName} \
  /loginType:OAuth \
  /login:"{Email}","{PAT}"

# Step 3: Get latest (NO /collection flag — tf get does not support it)
"{TfExePath}" get "{LocalPath}" /recursive \
  /loginType:OAuth \
  /login:"{Email}","{PAT}"
```

**Important notes:**
- `workfold` and `get` may require the classic TFVC URL (`{OrgTfvc}`) instead of the modern URL (`{OrgDiscovery}`)
- `tf get` does **NOT** support the `/collection` flag — omit it
- If `TF14044: You need the AdminWorkspaces permission` → use Operation C instead

### Operation E: Check In Files to TFVC

Check in one or more local files to the TFVC server. Two paths: workspace-based (if available) or REST API (recommended fallback).

**Full template with reusable code blocks:** See [checkin-template.md](./reference/checkin-template.md).

#### Decision Tree

```
┌─────────────────────────────────────────────────────────────────┐
│  1. CHECK SAVED STATE                                            │
│                                                                  │
│     Read {CheckinMethod} from session (populated in Step 5b).   │
│                                                                  │
│     a. "rest-api" → skip to REST API Path (below)               │
│     b. "workspace" → skip to Workspace Path (below)             │
│     c. "unknown" → run discovery (step 2)                       │
└─────────────────────────────────────────────────────────────────┘
                            │ (c)
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  2. DISCOVER CHECKIN METHOD                                      │
│                                                                  │
│     Step 2a: Check for existing workspace on this machine       │
│                                                                  │
│     cd /c/Temp                                                   │
│     "{TfExePath}" workspaces \                                   │
│       /collection:"{OrgDiscovery}" \                             │
│       /computer:{MachineName} \                                  │
│       /loginType:OAuth \                                         │
│       /login:"{Email}","{PAT}"                                   │
│                                                                  │
│     → If workspace listed with {ServerPath} mapping:             │
│       Set {CheckinMethod}="workspace", {WorkspaceName}=name     │
│       Save to CLAUDE.md "## TFVC Machine State"                  │
│       → Use Workspace Path                                       │
│                                                                  │
│     Step 2b: If no workspace, try creating one:                  │
│                                                                  │
│     "{TfExePath}" workspace /new {MachineName}_ARM \             │
│       /collection:"{OrgDiscovery}" \                             │
│       /location:local /permission:Private /noprompt \            │
│       /loginType:OAuth \                                         │
│       /login:"{Email}","{PAT}"                                   │
│                                                                  │
│     → If created: map folder, set {CheckinMethod}="workspace"   │
│     → If TF14044: set {CheckinMethod}="rest-api"                │
│                                                                  │
│     Save result to CLAUDE.md "## TFVC Machine State"             │
│     Save lesson to auto memory if discovery was non-trivial      │
└─────────────────────────────────────────────────────────────────┘
```

#### Workspace Path (if workspace available)

```bash
# For NEW files:
"{TfExePath}" add "{LocalFilePath}" \
  /loginType:OAuth \
  /login:"{Email}","{PAT}"

# For EXISTING files (pend edit):
"{TfExePath}" checkout "{LocalFilePath}" \
  /loginType:OAuth \
  /login:"{Email}","{PAT}"

# Then check in (works for both add and edit):
"{TfExePath}" checkin "{LocalFilePath}" \
  /comment:"{Comment}" \
  /loginType:OAuth \
  /login:"{Email}","{PAT}" \
  /noprompt
```

#### REST API Path (PRIMARY — no workspace needed)

Uses the **hybrid pattern**: PowerShell for JSON construction, `curl.exe` for HTTP.

**CRITICAL**: NEVER build JSON manually in bash when embedding file content. Always use PowerShell `ConvertTo-Json`. See Critical Rule 9.

##### Step 1: Determine change type per file

```bash
# Check if file exists on server (200 = edit, 404 = add)
ENCODED_PATH=$(powershell.exe -Command "[Uri]::EscapeDataString('{ServerFilePath}')")

HTTP_CODE=$(curl.exe -s -o /dev/null -w "%{http_code}" \
  -u ":{PAT}" \
  "https://dev.azure.com/{OrgName}/{ProjectNameEncoded}/_apis/tfvc/items?path=${ENCODED_PATH}&api-version=7.1")

# If 200 → edit (also fetch version number from response)
# If 404 → add
```

##### Step 2: Build JSON via PowerShell, POST via curl.exe

```bash
# PowerShell builds JSON with proper escaping → writes to temp file
LOCAL_PATH_WIN="C:\\path\\to\\file.xml"
powershell.exe -Command "
  \$content = [IO.File]::ReadAllText('${LOCAL_PATH_WIN}')
  \$item = @{ path='{ServerFilePath}'; contentMetadata=@{ encoding=65001 } }
  # For edits, add: \$item.version = {VERSION}
  \$change = @{ item=\$item; changeType='{add|edit}'; newContent=@{ content=\$content; contentType='rawtext' } }
  @{ changes=@(\$change); comment='{Comment}' } | ConvertTo-Json -Depth 10 -Compress
" > /tmp/tfvc_changeset.json

# curl.exe sends the request (reliable HTTP)
RESPONSE=$(curl.exe -s -w "\nHTTP_CODE:%{http_code}" -X POST \
  -u ":{PAT}" \
  -H "Content-Type: application/json; charset=utf-8" \
  -d @/tmp/tfvc_changeset.json \
  "https://dev.azure.com/{OrgName}/{ProjectNameEncoded}/_apis/tfvc/changesets?api-version=7.1")

rm -f /tmp/tfvc_changeset.json
```

##### Step 3: Verify

```bash
# Success: HTTP 201, response contains changesetId
# Parse: echo "$RESPONSE" | powershell.exe -Command '$input | ConvertFrom-Json | Select -Expand changesetId'
```

**For multi-file checkins**, see the full template in [checkin-template.md](./reference/checkin-template.md).

---

## Critical Rules

1. **Always `cd` to a safe path** OUTSIDE any existing TFVC workspace before running `tf` commands with `/collection`. Running from inside a workspace causes `/collection` to be ignored — the command silently uses the workspace's collection instead.

2. **Always use `/loginType:OAuth`** with `/login:{Email},{PAT}`. Using just `/login:{user},{pat}` without `/loginType:OAuth` may return `TF30063` unauthorized.

3. **`tf get` does NOT support `/collection`**. Remove `/collection` from any `tf get` command.

4. **Two org URL formats may be needed:**
   - Discovery/auth: `https://dev.azure.com/{OrgName}` — used for `workspaces`, `dir`, `workspace /new`
   - TFVC operations: `https://{orgname}.visualstudio.com` — may be needed for `workfold`, `get`
   - Test both if one fails.

5. **Escape `$` in bash.** TFVC paths start with `$` which bash expands. Always wrap TFVC server paths in **single quotes**: `'$/Project/Trunk'`

6. **Never store PAT values** in skill files, CLAUDE.md, or any source-controlled file. Only store the file path reference to where PAT can be read.

7. **Prefer `tf view` bulk download** (Operation C) over workspace-based get (Operation D). Workspace creation is often blocked by organizational permissions.

8. **Never auto-discover org URLs or server paths from local state.** Do NOT use `tf workspaces` (without `/collection`), environment variables, or any local workspace metadata to infer `{OrgDiscovery}`, `{OrgTfvc}`, or `{ServerPath}`. The local machine may have workspaces for completely different organizations. Only trust values explicitly saved in the project's CLAUDE.md `## TFVC Configuration` section. If not present there, **always ask the user**. Once confirmed and saved to CLAUDE.md, values are trusted for future sessions.

9. **Always use PowerShell for JSON construction when embedding file content.** XML files contain `<`, `>`, `"`, `\`, newlines, and CDATA sections that will break any manual bash string escaping (sed, awk, echo, heredoc). PowerShell `ConvertTo-Json` handles all escaping automatically and correctly.

10. **Use curl.exe for HTTP, PowerShell for JSON (hybrid pattern).** PowerShell `Invoke-RestMethod` has known timeout issues on some Windows environments. The reliable pattern is: PowerShell builds the JSON payload → writes to temp file → `curl.exe` sends the HTTP request. See [checkin-template.md](./reference/checkin-template.md).

11. **For REST API checkin edits, always include the file version.** Query `_apis/tfvc/items?path={path}` first to get the current `version` number. Include it as `item.version` in the changeset JSON. Omitting it for edits causes HTTP 400 ("Please specify the item version").

12. **URL-encode TFVC paths in REST API URLs.** Server paths contain `$` and spaces. In REST API query parameters: `$` → `%24`, space → `%20`. Use PowerShell `[Uri]::EscapeDataString()` or manual encoding.

---

## CLAUDE.md Integration

### Saving Configuration

When saving TFVC config to the project's CLAUDE.md, use this format:

```markdown
## TFVC Configuration
- tf.exe: {TfExePath}
- Organization: {OrgDiscovery}
- Organization (TFVC): {OrgTfvc}
- Project: {ProjectName}
- Email: {Email}
- PAT File: {relative path to PAT file, default: Docs/PAT.txt}
- Server Path: {ServerPath}
- Local Path: {LocalPath}
```

**Note on Project:** The `Project` field records which TFVC project within the org was confirmed by the user. An org can have multiple projects — this prevents assuming the wrong one in future sessions.

### Rules for Saving
- **Non-PAT variables** → auto-save to CLAUDE.md (non-sensitive config)
- **PAT value** → NEVER in CLAUDE.md; only the file path reference (e.g., `PAT File: Docs/PAT.txt`)
- **PAT file** → always exclude from source control (.gitignore / .tfignore)
- File path references in CLAUDE.md are safe to check into source control

### Reading Configuration

On session start:
1. Read CLAUDE.md → look for `## TFVC Configuration` section
2. Parse each `- Key: Value` line
3. For `PAT File:` → read the actual PAT from that file path
4. Populate session variables from parsed values
5. If `Organization:` exists but `Server Path:` or `Project:` is missing →
   browse `$/` to list projects, ask user to confirm (Step 6 of discovery)
6. For any other missing values → follow discovery workflow above

### Saving Machine State

When checkin method is discovered for a machine (see Operation E), save to CLAUDE.md:

```markdown
## TFVC Machine State
<!-- Auto-populated by tfvc-devops-skill skill. Do not edit manually. -->

### {MachineName}
- Checkin Method: {rest-api | workspace}
- Workspace: {WorkspaceName | none (reason)}
- Discovered: {YYYY-MM-DD}
```

**Rules:**
- One sub-section per machine (`###` heading = hostname)
- Multiple machines can be listed (different devs, different VMs)
- Only update the sub-section for the CURRENT machine
- If method changes (e.g., admin grants workspace permissions), update in place

### Saving Lessons Learned (Auto Memory)

When a TFVC operation fails and you discover the root cause and fix, save the lesson to the project's **auto memory** (`MEMORY.md`) under a `## TFVC Skill Lessons` heading.

**What to save to auto memory (project/environment-specific only):**
- Org-specific behaviors (e.g., "WMSAdmin org does not enforce check-in policies — no policyOverride needed")
- Project-specific quirks (e.g., "This project's server path is case-sensitive for AxClass subfolder")
- Debugging insights tied to this specific environment

**What NOT to save to auto memory:**
- Config values (org URL, server path, PAT file) → those go in CLAUDE.md `## TFVC Configuration`
- Machine state (checkin method, workspace) → those go in CLAUDE.md `## TFVC Machine State`
- Generic reusable patterns (hybrid PS+curl, JSON escaping rules, edit versioning) → those belong in skill reference files (`reference/*.md`) and Critical Rules — NOT in auto memory. If a lesson applies to ALL projects using this skill, update the skill files instead.

---

## Error Handling

When a `tf` command fails, consult [reference/troubleshooting.md](./reference/troubleshooting.md) for known errors and solutions before asking the user.

Common quick fixes:
- **TF30063** → PAT expired or invalid. Ask user to regenerate.
- **TF14044** → No workspace permission. Switch to `tf view` bulk download.
- **TF10121** → Path not found. Check for bash `$` expansion (use single quotes).
- **404** → Wrong organization URL format.
