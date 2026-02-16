# TFVC Troubleshooting Guide

Common errors encountered when working with TFVC via `tf.exe` and their solutions.

---

## TF30063: You are not authorized to access the server

**Full message:** `TF30063: You are not authorized to access https://dev.azure.com/{org}.`

**Causes and fixes:**

| Cause | Fix |
|:------|:----|
| PAT expired | Regenerate PAT in Azure DevOps → User Settings → Personal Access Tokens |
| PAT lacks permissions | PAT needs **Code (Read)** scope at minimum; **Code (Read & Write)** for checkins |
| Missing `/loginType:OAuth` | Add `/loginType:OAuth` before `/login:{Email},{PAT}` |
| Using wrong login format | Use `/login:{Email},{PAT}` (email, not username) |
| Wrong org URL | Verify the organization URL — try both `https://dev.azure.com/{Org}` and `https://{org}.visualstudio.com` |

**Diagnostic command:**
```bash
# Test PAT via REST API (does not require tf.exe)
curl -s -o /dev/null -w "%{http_code}" \
  -u ":{PAT}" \
  "https://dev.azure.com/{OrgName}/_apis/projects?api-version=7.0"
# 200 = PAT works; 401 = PAT invalid/expired
```

---

## TF14044: You need the AdminWorkspaces permission

**Full message:** `TF14044: You need the AdminWorkspaces global permission to use the /location option with the workspace command.`

**Cause:** The Azure DevOps organization restricts workspace creation. This is common in managed/enterprise environments.

**Fix:** Use `tf view` bulk download instead of workspace-based `tf get`.

```bash
# Instead of workspace creation:
#   tf workspace /new ...      ← BLOCKED
#   tf workfold /map ...
#   tf get /recursive

# Use tf view for each file:
"{TfExePath}" view '{ServerPath}/{FileName}' \
  /collection:"{OrgDiscovery}" \
  /loginType:OAuth \
  /login:"{Email}","{PAT}" \
  > "{LocalPath}/{FileName}"
```

See **Operation C** in skill.md for the complete bulk download pattern.

---

## TF10121: Path not found or not recognized

**Full message:** `TF10121: The path '...' does not exist in the repository.`

**Causes and fixes:**

| Cause | Fix |
|:------|:----|
| Bash expanded `$` in path | Wrap TFVC paths in **single quotes**: `'$/Project/Trunk'` |
| Wrong project/path name | Use `tf dir '$/` to list all projects at root |
| Typo in path | TFVC paths are case-insensitive but verify spelling |

**Example of `$` expansion problem:**
```bash
# WRONG — bash expands $/WMS as variable (empty)
tf dir $/WMS D365 Implementation/Trunk ...

# CORRECT — single quotes prevent expansion
tf dir '$/WMS D365 Implementation/Trunk' ...
```

---

## /collection flag silently ignored

**Symptom:** `tf` command connects to wrong server or returns unexpected results despite providing `/collection` flag.

**Cause:** Running `tf.exe` from **inside** an existing TFVC workspace directory. When inside a workspace, `tf.exe` ignores the `/collection` flag and uses the workspace's collection.

**Fix:** Always `cd` to a directory **outside** any TFVC workspace before running commands:

```bash
cd /c/Temp  # or any path not under a TFVC workspace mapping
"{TfExePath}" dir '$/...' /collection:"{OrgDiscovery}" ...
```

---

## 404 Not Found on collection URL

**Symptom:** HTTP 404 when connecting to the Azure DevOps organization.

**Causes and fixes:**

| Cause | Fix |
|:------|:----|
| Using project URL instead of org URL | Use `https://dev.azure.com/{Org}` (not `https://dev.azure.com/{Org}/{Project}`) |
| Using `_versionControl` URL | Use the org root URL only |
| Org name case mismatch | Try exact casing from Azure DevOps portal |

---

## tf get: Unrecognized option '/collection'

**Symptom:** `tf get` fails with argument/option error when `/collection` is specified.

**Cause:** `tf get` does **not** support the `/collection` flag. It uses the workspace context to determine the collection.

**Fix:** Remove `/collection` from `tf get` commands:

```bash
# WRONG
"{TfExePath}" get "{LocalPath}" /recursive /collection:"{OrgTfvc}" ...

# CORRECT
"{TfExePath}" get "{LocalPath}" /recursive /loginType:OAuth /login:"{Email}","{PAT}"
```

---

## workfold/get fails with modern URL but works with classic URL

**Symptom:** `tf workfold` or `tf get` fails with `https://dev.azure.com/{Org}` but works with `https://{org}.visualstudio.com`.

**Cause:** Some `tf.exe` versions handle the two URL formats differently for certain operations.

**Fix:** Use the dual-URL pattern:

| Operation | URL Format |
|:----------|:-----------|
| `workspaces`, `dir`, `workspace /new` | `https://dev.azure.com/{Org}` (discovery URL) |
| `workfold`, `get` | `https://{org}.visualstudio.com` (classic TFVC URL) |

---

## tf view outputs binary garbage

**Symptom:** `tf view` output contains binary characters instead of readable text.

**Cause:** The file is a binary file (DLL, image, etc.), or encoding mismatch.

**Fix:**
- For text files: `tf view` should work correctly. Check if the file is actually text.
- For binary files: `tf view` with output redirection (`> file`) preserves binary content.
- If encoding is wrong: Try `/console` flag or redirect to file and check encoding.

---

## Workspace already exists

**Symptom:** `TF14061: The workspace {name} already exists on computer {machine}.`

**Fix options:**
1. Use a different workspace name
2. Delete the existing workspace first:
   ```bash
   "{TfExePath}" workspace /delete {WorkspaceName} \
     /collection:"{OrgDiscovery}" \
     /loginType:OAuth \
     /login:"{Email}","{PAT}"
   ```
3. List existing workspaces to see what's mapped:
   ```bash
   "{TfExePath}" workspaces \
     /collection:"{OrgDiscovery}" \
     /loginType:OAuth \
     /login:"{Email}","{PAT}"
   ```

---

## General Debugging Tips

1. **Always test connectivity first** before attempting complex operations:
   ```bash
   "{TfExePath}" workspaces /collection:"{OrgDiscovery}" /loginType:OAuth /login:"{Email}","{PAT}"
   ```

2. **Test PAT independently** via REST API:
   ```bash
   curl -s -u ":{PAT}" "https://dev.azure.com/{Org}/_apis/projects?api-version=7.0"
   ```

3. **Check tf.exe version:**
   ```bash
   "{TfExePath}" vc help
   ```

4. **Run from a clean directory** (outside any workspace) to avoid implicit context issues.

5. **Use single quotes** for all TFVC server paths in bash to prevent `$` expansion.

---

## REST API checkin: 409 Conflict (version mismatch)

**Symptom:** POST to `_apis/tfvc/changesets` returns HTTP 409.

**Cause:** The `version` in the `item` object does not match the current server version. Someone else checked in a change between when you fetched the version and when you submitted.

**Fix:** Re-fetch the item version and retry:

```bash
curl.exe -s -u ":{PAT}" \
  "https://dev.azure.com/{Org}/{ProjectEncoded}/_apis/tfvc/items?path={encodedPath}&api-version=7.1"
```

Use the `version` from the response in your next changeset POST.

---

## REST API checkin: empty content in changeset

**Symptom:** Changeset created successfully (HTTP 201) but file content is empty or corrupted on server.

**Cause:** JSON content escaping failed. This happens when building JSON manually with bash string manipulation (`echo`, `sed`, `awk`). XML files contain `<`, `>`, `"`, `\`, newlines, and CDATA sections — all of which break manual JSON construction.

**Fix:** ALWAYS use PowerShell `ConvertTo-Json` to build the JSON body. It handles all escaping automatically:

```powershell
$content = [IO.File]::ReadAllText("C:\path\to\file.xml")
@{ content = $content; contentType = "rawtext" } | ConvertTo-Json -Depth 5
```

NEVER attempt to embed file content into JSON via bash string interpolation or heredocs.

---

## REST API checkin: TF14088 policy violation

**Symptom:** POST to `_apis/tfvc/changesets` returns error mentioning check-in policy failure (TF14088 or similar).

**Cause:** The TFVC project has check-in policies (e.g., require work item, require comment, require build).

**Fix:** Add `policyOverride` to the changeset JSON body:

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

---

## PowerShell Invoke-RestMethod timeout

**Symptom:** `Invoke-RestMethod` hangs indefinitely or times out when POSTing to Azure DevOps REST API.

**Cause:** Known issue on some Windows environments — proxy settings, TLS negotiation, or large payload sizes can cause PowerShell's HTTP stack to hang.

**Fix:** Use the **hybrid pattern** — `curl.exe` for HTTP calls, PowerShell only for JSON construction:

```bash
# PowerShell builds JSON (reliable escaping)
powershell.exe -Command "... | ConvertTo-Json -Compress" > /tmp/payload.json

# curl.exe sends HTTP (reliable transport)
curl.exe -s -X POST -u ":{PAT}" \
  -H "Content-Type: application/json" \
  -d @/tmp/payload.json \
  "https://dev.azure.com/{Org}/{Project}/_apis/tfvc/changesets?api-version=7.1"
```

This uses each tool for what it does best. See [checkin-template.md](./checkin-template.md) for the full pattern.
