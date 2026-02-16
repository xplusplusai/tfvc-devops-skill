# TFVC AI Agent Skill

Enterprise-grade AI automation for Azure DevOps TFVC repositories.
Designed for enterprise environments where TFVC remains a critical part of development infrastructure.

Built and maintained by **XPLUSPLUS.AI**
AI tooling for D365 F&O and enterprise development environments
https://xplusplus.ai

---

## Why This Skill Exists

Modern AI coding assistants are built primarily for Git-based workflows.
However, many enterprise environments — especially Microsoft Dynamics 365 F&O teams — still operate on Azure DevOps TFVC.

This skill bridges that gap by enabling AI agents to:

- Authenticate securely with Azure DevOps
- Perform TFVC operations programmatically
- Automate check-ins, workspaces, and branch interactions
- Integrate TFVC into AI-driven workflows

It is designed specifically for enterprise DevOps scenarios where Git migration is not an option.

## Who This Is For

- Enterprise developers working with Azure DevOps TFVC
- D365 F&O development teams
- Organizations with legacy TFVC repositories
- AI automation workflows interacting with Microsoft DevOps environments

## What This Skill Does

- Connects to Azure DevOps TFVC repositories
- Browses server paths and lists contents
- Downloads files (with or without workspace permissions)
- **Checks in files** to TFVC (via workspace or REST API — auto-detected per machine)
- Remembers checkin method per machine — skips rediscovery in future sessions
- Automatically saves connection config for future sessions
- Handles PAT authentication securely (never stores tokens in source control)
- Includes troubleshooting for common TFVC errors (TF30063, TF14044, etc.)

## High-Level Architecture

```
AI Agent
→ TFVC Skill
→ Azure DevOps REST APIs
→ TFVC Repository
```

The skill handles authentication, workspace logic, and command orchestration, allowing AI agents to operate safely within enterprise DevOps environments.

## Prerequisites

- A development environment supporting agent skills
- Visual Studio 2022 (or 2019) with Team Explorer — provides `tf.exe`
- An Azure DevOps PAT with **Code (Read & Write)** scope (Read-only is sufficient for browse/download, but checkin requires Write)

## Installation

### Option A: Copy & Paste (Simple)

1. Download or copy the files from this repository
2. In your project, create the skill directory:
   ```
   .claude/skills/tfvc-devops-skill/
   ```
3. Copy these files into it:
   ```
   .claude/skills/tfvc-devops-skill/
   ├── skill.md
   └── reference/
       └── troubleshooting.md
   ```
4. Done. Claude Code will detect the skill on next session.

### Option B: Clone + Symlink (Recommended — auto-updates)

This approach clones the repo once, then symlinks it into any project workspace. Future updates are a `git pull` away.

1. Clone this repository to a shared location:
   ```bash
   git clone https://github.com/xplusplusai/tfvc-devops-skill C:\repos\tfvc-devops-skill
   ```

2. In your project workspace, create the skills directory and symlink:

   **Windows (elevated Command Prompt):**
   ```cmd
   mkdir .claude\skills
   mklink /D .claude\skills\tfvc-devops-skill C:\repos\tfvc-devops-skill
   ```

   **Windows (Git Bash — creates junction, functionally equivalent):**
   ```bash
   mkdir -p .claude/skills
   cmd //c "mklink /D .claude\skills\tfvc-devops-skill C:\repos\tfvc-devops-skill"
   ```

3. To update the skill later:
   ```bash
   cd C:\repos\tfvc-devops-skill
   git pull
   ```
   All symlinked projects get the update automatically.

### AI Auto-Install Prompt

Copy and paste the following prompt into Claude Code to automatically install this skill:

```
Clone the TFVC DevOps skill from https://github.com/xplusplusai/tfvc-devops-skill
to C:\repos\tfvc-devops-skill (if not already cloned), then create a symlink from
.claude/skills/tfvc-devops-skill in the current workspace pointing to
C:\repos\tfvc-devops-skill. Create the .claude/skills directory if it doesn't exist.
After symlinking, verify the skill.md file is accessible through the symlink.
```

## Usage

Once installed, Claude Code will use this skill when you ask it to:

- "Connect to TFVC and download the code"
- "Sync files from the TFVC repository"
- "Browse the TFVC server to see what projects are available"
- "Check in this file to TFVC"
- "Commit my changes to TFVC"
- "Set up TFVC connection for this project"

On first use, the skill will:
1. Auto-discover `tf.exe` on your machine
2. Ask for your Azure DevOps organization URL, email, and PAT
3. Save non-sensitive config to your project's CLAUDE.md for future sessions
4. Offer to save your PAT to a local file (excluded from source control)
5. Test connectivity before proceeding

On subsequent sessions, saved config is loaded automatically — you only need to provide PAT if it wasn't saved to a file.

## Skill Files

| File | Purpose |
|:-----|:--------|
| `skill.md` | Main skill prompt — connection workflows, variable discovery, download and checkin operations |
| `reference/troubleshooting.md` | Common TFVC errors and solutions |
| `reference/checkin-template.md` | Reusable checkin script template with multi-file support |

## Security Considerations

- Uses Azure DevOps Personal Access Tokens (PAT)
- No credentials are hardcoded
- PAT values are **never** stored in CLAUDE.md or skill files
- Only the file path reference to a PAT file is stored in CLAUDE.md (safe to commit)
- When saving PAT to a file, the skill automatically adds the filename to `.gitignore` and `.tfignore`
- Tokens are stored locally and never committed
- Designed for secure enterprise usage

Users are responsible for managing and rotating PAT credentials according to their organization's policies.

## Roadmap

- Enhanced workspace automation
- Improved branch detection
- Better error handling and retry logic
- Integration with autonomous DevOps workflows

## License

MIT

---

## Related Tools

If you are working with D365 F&O and AI-assisted development, you may also be interested in:

**XPLUSPLUS.AI Development Toolkit**
AI development guardrails for X++ and D365 F&O
https://xplusplus.ai
