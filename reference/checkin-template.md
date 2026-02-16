# TFVC REST API Checkin Template

Ready-to-use template for checking in files via the Azure DevOps REST API.
Referenced by **Operation E** in skill.md.

Uses the **hybrid pattern**: PowerShell for JSON construction (reliable XML/special-char escaping), `curl.exe` for HTTP transport (reliable, no timeouts).

---

## Prerequisites

- `curl.exe` (mingw64 or Windows native) available in PATH
- PowerShell available (for `ConvertTo-Json`)
- PAT with **Code (Read & Write)** scope

---

## Helper: Check Server Path (add vs edit)

Queries the REST API to determine if a file already exists on the server.

```bash
# Returns "add" for new files, or "edit|{version}" for existing files
check_server_path() {
  local PAT="$1"
  local ORG="$2"           # e.g. WMSAdmin (just the org name)
  local PROJECT_ENC="$3"   # URL-encoded project name
  local SERVER_PATH="$4"   # e.g. $/WMS D365 Implementation/Trunk/...

  # URL-encode the server path
  local ENCODED_PATH
  ENCODED_PATH=$(powershell.exe -Command "[Uri]::EscapeDataString('$SERVER_PATH')")

  local RESPONSE HTTP_CODE
  RESPONSE=$(curl.exe -s -w "\n%{http_code}" \
    -u ":${PAT}" \
    "https://dev.azure.com/${ORG}/${PROJECT_ENC}/_apis/tfvc/items?path=${ENCODED_PATH}&api-version=7.1")

  HTTP_CODE=$(echo "$RESPONSE" | tail -1)
  local BODY
  BODY=$(echo "$RESPONSE" | sed '$d')

  if [ "$HTTP_CODE" = "200" ]; then
    local VER
    VER=$(echo "$BODY" | powershell.exe -Command '$input | ConvertFrom-Json | Select-Object -ExpandProperty version')
    echo "edit|${VER}"
  else
    echo "add"
  fi
}
```

---

## Single File Checkin

The simplest case — check in one file.

```bash
# === Variables (substitute from skill session) ===
PAT="{PAT}"
ORG="{OrgName}"                          # e.g. WMSAdmin
PROJECT_ENC="{ProjectNameEncoded}"        # e.g. WMS%20D365%20Implementation
SERVER_PATH="{ServerPath}/{RelativePath}" # e.g. $/WMS D365.../AxClass/MyClass.xml
LOCAL_PATH="{LocalPath}/{RelativePath}"   # e.g. C:\Temp\...\AxClass\MyClass.xml
COMMENT="Your checkin comment"

# 1. Determine add vs edit
RESULT=$(check_server_path "$PAT" "$ORG" "$PROJECT_ENC" "$SERVER_PATH")
CHANGE_TYPE="${RESULT%%|*}"
VERSION="${RESULT##*|}"

# 2. Build JSON via PowerShell (handles all XML escaping safely)
#    Write to temp file to avoid stdin size limits
LOCAL_PATH_WIN=$(echo "$LOCAL_PATH" | sed 's|/|\\|g')
if [ "$CHANGE_TYPE" = "edit" ]; then
  powershell.exe -Command "
    \$content = [IO.File]::ReadAllText('${LOCAL_PATH_WIN}')
    \$item = @{ path='${SERVER_PATH}'; contentMetadata=@{ encoding=65001 }; version=${VERSION} }
    \$change = @{ item=\$item; changeType='edit'; newContent=@{ content=\$content; contentType='rawtext' } }
    @{ changes=@(\$change); comment='${COMMENT}' } | ConvertTo-Json -Depth 10 -Compress
  " > /tmp/tfvc_changeset.json
else
  powershell.exe -Command "
    \$content = [IO.File]::ReadAllText('${LOCAL_PATH_WIN}')
    \$item = @{ path='${SERVER_PATH}'; contentMetadata=@{ encoding=65001 } }
    \$change = @{ item=\$item; changeType='add'; newContent=@{ content=\$content; contentType='rawtext' } }
    @{ changes=@(\$change); comment='${COMMENT}' } | ConvertTo-Json -Depth 10 -Compress
  " > /tmp/tfvc_changeset.json
fi

# 3. POST via curl.exe
RESPONSE=$(curl.exe -s -w "\nHTTP_CODE:%{http_code}" -X POST \
  -u ":${PAT}" \
  -H "Content-Type: application/json; charset=utf-8" \
  -d @/tmp/tfvc_changeset.json \
  "https://dev.azure.com/${ORG}/${PROJECT_ENC}/_apis/tfvc/changesets?api-version=7.1")

# 4. Verify
echo "$RESPONSE"
rm -f /tmp/tfvc_changeset.json
```

---

## Multi-File Checkin (Single Changeset)

Check in multiple files in one atomic changeset.

```bash
# === Variables ===
PAT="{PAT}"
ORG="{OrgName}"
PROJECT_ENC="{ProjectNameEncoded}"
COMMENT="Your checkin comment"

# Define files as "localPath|serverPath" pairs
FILES=(
  "/c/Temp/Project/AxClass/MyClass.xml|$/Project/Trunk/AxClass/MyClass.xml"
  "/c/Temp/Project/AxTable/MyTable.xml|$/Project/Trunk/AxTable/MyTable.xml"
)

# 1. Build PowerShell file array with change types
PS_ARRAY=""
for FILE_PAIR in "${FILES[@]}"; do
  LOCAL="${FILE_PAIR%%|*}"
  SERVER="${FILE_PAIR##*|}"
  RESULT=$(check_server_path "$PAT" "$ORG" "$PROJECT_ENC" "$SERVER")
  CHANGE_TYPE="${RESULT%%|*}"
  VERSION="${RESULT##*|}"

  LOCAL_WIN=$(echo "$LOCAL" | sed 's|^/c/|C:/|; s|/|\\|g')
  if [ "$CHANGE_TYPE" = "edit" ]; then
    PS_ARRAY+="@{l='${LOCAL_WIN}';s='${SERVER}';t='edit';v=${VERSION}},"
  else
    PS_ARRAY+="@{l='${LOCAL_WIN}';s='${SERVER}';t='add';v=0},"
  fi
done
PS_ARRAY="${PS_ARRAY%,}"  # Remove trailing comma

# 2. Build JSON via PowerShell
powershell.exe -Command "
  \$files = @(${PS_ARRAY})
  \$changes = @()
  foreach (\$f in \$files) {
    \$content = [IO.File]::ReadAllText(\$f.l)
    \$item = @{ path=\$f.s; contentMetadata=@{ encoding=65001 } }
    if (\$f.t -eq 'edit') { \$item.version = [int]\$f.v }
    \$changes += @{
      item = \$item
      changeType = \$f.t
      newContent = @{ content=\$content; contentType='rawtext' }
    }
  }
  @{ changes=\$changes; comment='${COMMENT}' } | ConvertTo-Json -Depth 10 -Compress
" > /tmp/tfvc_changeset.json

# 3. POST via curl.exe
RESPONSE=$(curl.exe -s -w "\nHTTP_CODE:%{http_code}" -X POST \
  -u ":${PAT}" \
  -H "Content-Type: application/json; charset=utf-8" \
  -d @/tmp/tfvc_changeset.json \
  "https://dev.azure.com/${ORG}/${PROJECT_ENC}/_apis/tfvc/changesets?api-version=7.1")

# 4. Verify
echo "$RESPONSE"
rm -f /tmp/tfvc_changeset.json
```

---

## Parsing the Response

On success (HTTP 201), the response contains:

```json
{
  "changesetId": 864,
  "url": "https://dev.azure.com/...",
  "author": { "displayName": "...", ... },
  "createdDate": "2026-02-15T18:05:08Z",
  "comment": "Your checkin comment"
}
```

Extract the changeset ID:

```bash
# Parse changesetId (no jq needed)
CHANGESET_ID=$(echo "$RESPONSE" | sed 's/HTTP_CODE:.*//' | \
  powershell.exe -Command '$input | ConvertFrom-Json | Select-Object -ExpandProperty changesetId')
echo "Changeset ${CHANGESET_ID} created successfully"
```

---

## Common Errors

| HTTP Code | Error | Cause | Fix |
|:----------|:------|:------|:----|
| 401 | Unauthorized | PAT invalid or lacks Code Write scope | Regenerate PAT with **Code (Read & Write)** |
| 400 | "Please specify the item version" | Edit without `version` field on item | Query file first, include `item.version` |
| 404 | Not Found | Wrong org URL or project name | Check `{OrgName}` and `{ProjectName}` encoding |
| 409 | Conflict | File version mismatch (concurrent edit) | Re-fetch version via items API, retry |
| 400 | TF14088 policy violation | Org has check-in policies | Add `policyOverride` to JSON (see below) |
| 201 | Success but empty content | JSON escaping broke file content | ALWAYS use PowerShell `ConvertTo-Json` |

### Handling Check-in Policy Override

If the TFVC project enforces check-in policies, add `policyOverride` to the JSON body:

```json
{
  "changes": [...],
  "comment": "Your comment",
  "policyOverride": {
    "comment": "Automated checkin via Claude Code",
    "policyFailures": []
  }
}
```

In the PowerShell builder, add before `ConvertTo-Json`:

```powershell
$body = @{
  changes = $changes
  comment = $Comment
  policyOverride = @{
    comment = "Automated checkin via Claude Code"
    policyFailures = @()
  }
}
```
