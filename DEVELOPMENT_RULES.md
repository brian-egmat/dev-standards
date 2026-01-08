# Development & Deployment Rules

Standard processes and best practices for all GitHub-managed applications.

---

## Table of Contents

1. [Version Numbering](#1-version-numbering)
2. [Git Workflow](#2-git-workflow)
3. [Teams Notifications](#3-teams-notifications)
4. [VPS Connection](#4-vps-connection)
5. [Deployment Process](#5-deployment-process)
6. [Rollback Procedures](#6-rollback-procedures)
7. [New Project Setup](#7-new-project-setup)
8. [Repository Standards](#8-repository-standards)
9. [Security Rules](#9-security-rules)
10. [VPS Best Practices Reference](#10-vps-best-practices-reference)
11. [QA Workflow & Port Reservation](#11-qa-workflow--port-reservation)
12. [Quick Reference Commands](#12-quick-reference-commands)

---

## IMPORTANT: VPS Settings Reference

**Location:** `/home/.settings_/` on VPS

Before creating ANY new app, Claude MUST read and follow the best practices in:

| File | Purpose |
|------|---------|
| `claude.md` | **Project structure, coding standards, API key management, error handling** |
| `api_keys.json` | API keys reference |
| `llm-providers-reference.json` | LLM provider configurations |
| `WEBHOOK_SETUP_GUIDE.md` | Webhook setup instructions |
| `UPDATE_GUIDE.md` | Update procedures |
| `GIT_GITHUB_GUIDE.md` | Git/GitHub reference |

**Command to read before starting any new project:**
```bash
ssh root@195.35.21.58 "cat /home/.settings_/claude.md"
```

---

## 1. Version Numbering

### Format: `vMAJOR.MINOR`

| Version | When to Use |
|---------|-------------|
| `v1` | Initial production release |
| `v1.1` | Minor changes, enhancements, bug fixes |
| `v1.2` | More minor changes or reverts |
| `v2` | Major changes, breaking changes, new features |

### Rules

1. **Always tag releases** - Every deployment to production must have a version tag
2. **Use annotated tags** - Include description of what changed
3. **Never reuse tags** - Each version number is unique forever
4. **Document in CHANGELOG.md** - Every version must be documented

### Commands

```bash
# Create annotated tag
git tag -a v1.1 -m "Description of changes"

# Push tag to GitHub
git push origin v1.1

# View all tags
git tag

# View tag details
git show v1.1
```

### Example Version History

```
v1    - Initial release
v1.1  - Added feature X
v1.2  - Bug fix for Y
v1.3  - Reverted v1.2 (bug found)
v2    - Major rewrite with new architecture
v2.1  - Minor enhancement
```

---

## 2. Git Workflow

### Standard Development Cycle

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│   1. PULL       2. CODE       3. TEST       4. COMMIT     5. PUSH   │
│   ────────────────────────────────────────────────────────────────► │
│                                                                     │
│   git pull      Edit files    Run tests     git add .     git push  │
│                                             git commit              │
│                                             git tag                 │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Rules

1. **Always pull before starting work** - Ensure you have the latest code
2. **Test before committing** - Never commit broken code
3. **Write meaningful commit messages** - Describe what and why
4. **Tag after significant changes** - Version every release
5. **Push tags separately** - `git push` then `git push origin <tag>`

### Commit Message Format

```
v1.X: Short description of change

- Detail 1
- Detail 2
- Detail 3

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude Opus 4.5 <noreply@anthropic.com>
```

### Branch Strategy (Simple)

| Branch | Purpose |
|--------|---------|
| `main` | Production code, always deployable |
| `feature/*` | New features (optional, for complex changes) |

---

## 3. Teams Notifications

### Two Types of Notifications

| Type | Trigger | Source |
|------|---------|--------|
| **Push** | Code pushed to GitHub | GitHub Actions |
| **Deploy** | Code deployed to VPS | deploy.sh script |

### Push Notifications (Automatic)

Every repository must have:
1. `.github/workflows/notify-on-push.yml` workflow file
2. Three secrets configured in GitHub:
   - `POWER_AUTOMATE_URL`
   - `TEAMS_CHAT_ID`
   - `AGENT_EMAIL`

### Deploy Notifications (Manual)

Always use the deploy script instead of raw `git pull`:

```bash
# CORRECT - sends notification
/home/github-deploy-notifier/deploy.sh /path/to/app "Server Name"

# WRONG - no notification
cd /path/to/app && git pull
```

### Notification Format (HTML Table)

```
┌─────────────────────────────────┐
│ 📤 Code Push (Blue)             │
│ 🚀 VPS Deployment (Green)       │
├─────────────┬───────────────────┤
│ Repo        │ app-name          │
│ Version     │ v1.2              │
│ Author      │ brian-egmat       │
│ Commit      │ abc1234           │
│ Time        │ 2025-12-31 14:00  │
└─────────────┴───────────────────┘
```

---

## 4. VPS Connection

### SSH Key Authentication (Required)

**Never use passwords** - Always use SSH keys for VPS access.

### Connection Details

| Item | Value |
|------|-------|
| Host | `195.35.21.58` |
| User | `root` |
| Auth | SSH Key (passwordless) |

### Connection Command

```bash
ssh root@195.35.21.58
```

### Setup SSH Key (One-time)

```bash
# Generate key (if not exists)
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -N ""

# Copy to VPS
ssh-copy-id root@195.35.21.58
```

### VPS Directory Structure

```
/home/
├── github-deploy-notifier/     # Notification system
├── Testing-SK/                 # Test projects
│   └── my-first-gh-project/
├── Regular-Tasks/              # Production tasks
│   └── Marketing_Daily_Metrics_Review/
└── [other-projects]/
```

---

## 5. Deployment Process

### Standard Deployment Steps

```bash
# 1. LOCAL: Make changes and push
git add .
git commit -m "v1.X: Description"
git push
git tag -a v1.X -m "Description"
git push origin v1.X

# 2. VPS: Deploy with notification
ssh root@195.35.21.58
/home/github-deploy-notifier/deploy.sh /path/to/app "Hostinger VPS"
```

### One-Command Deployment (from local)

```bash
# Push and deploy in one command
git push && git push origin v1.X && ssh root@195.35.21.58 "/home/github-deploy-notifier/deploy.sh /path/to/app 'Hostinger VPS'"
```

### Check VPS is Up-to-Date

```bash
ssh root@195.35.21.58 "cd /path/to/app && git fetch && git status"
```

Expected: `Your branch is up to date with 'origin/main'`

---

## 6. Rollback Procedures

### ALWAYS Use Option 2: Revert Commit

**Never use hard reset in production.**

### Rollback Steps

```bash
# 1. Find the commit to revert
git log --oneline

# 2. Revert the problematic commit
git revert <COMMIT_HASH> --no-edit

# 3. Push the revert
git push

# 4. Tag as new version
git tag -a v1.X -m "Reverts vX.X - reason for rollback"
git push origin v1.X

# 5. Update CHANGELOG.md
# Add entry explaining the revert

# 6. Deploy to VPS
ssh root@195.35.21.58 "/home/github-deploy-notifier/deploy.sh /path/to/app 'Hostinger VPS'"
```

### Rollback Decision Matrix

| Situation | Action |
|-----------|--------|
| Bug in production | Revert commit → new version |
| Need to test old version | `git checkout <tag>` (temporary) |
| Committed secrets | Hard reset (only this case) |

### Example Revert History

```
v1.0  - Initial release
v1.1  - Added feature X (has bug)
v1.2  - Reverts v1.1 (bug found in feature X)  ← This is correct
v1.3  - Fixed feature X properly
```

---

## 7. New Project Setup

### Checklist for Every New Project

```
[ ] 1. Create GitHub repository
[ ] 2. Clone locally
[ ] 3. Set git config (name, email)
[ ] 4. Create .gitignore
[ ] 5. Create .env.example
[ ] 6. Create CHANGELOG.md
[ ] 7. Create README.md
[ ] 8. Write application code
[ ] 9. Test locally
[ ] 10. Commit and push
[ ] 11. Tag as v1
[ ] 12. Add GitHub secrets (for push notifications)
[ ] 13. Add .github/workflows/notify-on-push.yml
[ ] 14. Clone on VPS
[ ] 15. Create venv on VPS
[ ] 16. Create .env on VPS
[ ] 17. Test on VPS
[ ] 18. Set up cron job (if needed)
```

### Required Files

```
project/
├── .github/
│   └── workflows/
│       └── notify-on-push.yml    # Push notifications
├── .gitignore                     # Ignore secrets, venv, etc.
├── .env.example                   # Template for environment vars
├── .env                           # Actual secrets (NOT committed)
├── CHANGELOG.md                   # Version history
├── README.md                      # Documentation
├── requirements.txt               # Python dependencies
└── main.py                        # Application code
```

### GitHub Secrets (Required for each repo)

| Secret | Value |
|--------|-------|
| `POWER_AUTOMATE_URL` | Power Automate HTTP trigger URL |
| `TEAMS_CHAT_ID` | Teams chat ID |
| `AGENT_EMAIL` | Email to tag in notifications |

---

## 8. Repository Standards

### .gitignore (Minimum)

```gitignore
# Environment
.env

# Python
__pycache__/
*.pyc
venv/
.venv/

# Logs
*.log
logs/

# Secrets
secrets/
*.pem
*.key

# IDE
.vscode/
.idea/

# OS
.DS_Store
Thumbs.db
```

### CHANGELOG.md Format

```markdown
# Changelog

## [vX.X] - YYYY-MM-DD
- Change 1
- Change 2
- Reason (if revert)

## [v1] - YYYY-MM-DD
- Initial release
```

### README.md Sections

```markdown
# Project Name

Brief description.

## Setup
How to install and configure.

## Usage
How to run the application.

## Configuration
Environment variables and settings.

## Version
Current version and history.
```

---

## 9. Security Rules

### Never Commit

- `.env` files with real values
- API keys, passwords, tokens
- Private keys (`.pem`, `.key`)
- `secrets/` folder contents

### PAT (Personal Access Token) Rules

| Rule | Why |
|------|-----|
| Use separate PATs per machine | Track which machine accessed |
| Set expiration dates | Limit exposure time |
| Delete unused PATs | Reduce attack surface |
| Use fine-grained PATs | Limit permissions |

### VPS Security

| Rule | Why |
|------|-----|
| Use SSH keys, not passwords | Stronger authentication |
| Keep SSH keys private | Never share `.ssh/id_rsa` |
| Use venv for Python apps | Isolate dependencies |

---

## 10. VPS Best Practices Reference

### Location

```
/home/.settings_/
├── claude.md                    # Main best practices document
├── api_keys.json                # API keys reference
├── llm-providers-reference.json # LLM provider configs
├── WEBHOOK_SETUP_GUIDE.md       # Webhook setup
├── UPDATE_GUIDE.md              # Update procedures
└── GIT_GITHUB_GUIDE.md          # Git/GitHub guide
```

### Key Standards from claude.md

#### Project Structure (REQUIRED)

```
my_project/
├── v1/                    # Version 1 code
│   ├── src/
│   │   ├── app/
│   │   │   ├── prompts/
│   │   │   ├── models.py
│   │   │   ├── utils.py
│   │   │   ├── error_handler.py
│   │   │   └── config.py
│   │   └── main.py
│   ├── tests/
│   ├── requirements.txt
│   └── config.json
├── common/                # Shared utilities
├── secrets/               # API keys (NEVER commit)
│   └── api_keys.json
├── logs/
├── data/
├── CHANGELOG.md
├── README.md
└── .gitignore
```

#### API Key Management (REQUIRED)

```python
# In common/config.py
import json
import os

def load_api_keys():
    secrets_path = os.path.join(os.path.dirname(__file__), "..", "secrets", "api_keys.json")
    with open(secrets_path, "r") as f:
        return json.load(f)

API_KEYS = load_api_keys()

# Usage in code:
from common.config import API_KEYS
openai_key = API_KEYS["openai"]["api_key"]
```

#### Mixpanel API (REQUIRED - Use Shared Client)

```python
import sys
sys.path.insert(0, '/home/.shared_utils')
from mixpanel import MixpanelClient

client = MixpanelClient(
    username="your_username",
    secret="your_secret",
    project_id="your_project_id",
    app_name="YourAppName",
    use_queue=True
)
```

#### Error Handling (REQUIRED)

```python
MAX_RETRIES = 3
RETRY_DELAY = 5

for attempt in range(MAX_RETRIES):
    try:
        response = requests.get(url)
        response.raise_for_status()
        break
    except requests.exceptions.HTTPError as e:
        if response.status_code == 429:
            wait_time = RETRY_DELAY * (2 ** attempt)
            time.sleep(wait_time)
        else:
            raise
```

#### Cron Job Scheduling (Stagger Times)

```bash
# Project A - Slot: minute 0
0 */2 * * * /usr/bin/python3 /home/project_a/main.py

# Project B - Slot: minute 15
15 */2 * * * /usr/bin/python3 /home/project_b/main.py
```

### Before Creating Any New App

1. Read: `ssh root@195.35.21.58 "cat /home/.settings_/claude.md"`
2. Follow the project structure
3. Use secrets/ folder for API keys
4. Use shared Mixpanel client if needed
5. Implement proper error handling
6. Stagger cron jobs to avoid conflicts

---

## 11. QA Workflow & Port Reservation

**IMPORTANT:** All changes MUST be tested in QA before production deployment.

See **QA_WORKFLOW.md** for detailed instructions.

### Branch Strategy

| Branch | Purpose |
|--------|---------|
| `main` | Production code - only merge from `qa` |
| `qa` | Active development and testing |

### Reserved Ports

| Port | Environment | Usage |
|------|-------------|-------|
| 8003 | Production | Default app port |
| **8004** | **QA** | **Reserved for testing ANY app** |

### QA Process Summary

```
1. Develop on qa branch
2. Push to GitHub (qa branch)
3. Deploy to QA server (port 8004)
4. Test on QA URL (/app-name-qa/)
5. If tests pass: merge qa → main
6. Tag release version
7. Deploy to production (port 8003)
```

### QA Golden Rules

1. **NEVER push directly to main** - Always develop on `qa` branch
2. **ALWAYS test on QA URL first** - No exceptions
3. **Port 8004 is reserved for QA** - Don't use for production
4. **Merge only after testing** - QA must pass before production

---

## 12. Quick Reference Commands

### Git Basics

| Action | Command |
|--------|---------|
| Pull latest | `git pull` |
| Check status | `git status` |
| Stage all | `git add .` |
| Commit | `git commit -m "message"` |
| Push | `git push` |
| Create tag | `git tag -a v1.X -m "description"` |
| Push tag | `git push origin v1.X` |

### Version Management

| Action | Command |
|--------|---------|
| List tags | `git tag` |
| Show tag | `git show v1.X` |
| View history | `git log --oneline` |
| Compare versions | `git diff v1.0 v1.1` |

### VPS Commands

| Action | Command |
|--------|---------|
| Connect | `ssh root@195.35.21.58` |
| Deploy app | `/home/github-deploy-notifier/deploy.sh /path/to/app "Server"` |
| Check sync | `cd /path/to/app && git fetch && git status` |
| Manual pull | `cd /path/to/app && git pull` |

### Notifications

| Action | Command |
|--------|---------|
| Deploy notification | `/home/github-deploy-notifier/deploy.sh /path/to/app "Server"` |
| Manual push notify | `/home/github-deploy-notifier/venv/bin/python /home/github-deploy-notifier/notify.py push /path/to/repo` |
| Manual deploy notify | `/home/github-deploy-notifier/venv/bin/python /home/github-deploy-notifier/notify.py deploy /path/to/repo "Server"` |

---

## Credentials Reference

### GitHub

| Item | Value |
|------|-------|
| Username | `brian-egmat` |
| PAT Management | https://github.com/settings/tokens |
| Security Log | https://github.com/settings/security-log |

### VPS (Hostinger)

| Item | Value |
|------|-------|
| IP | `195.35.21.58` |
| User | `root` |
| Auth | SSH Key |

### Teams Notifications

| Item | Value |
|------|-------|
| Power Automate URL | (stored in .env files) |
| Chat ID | `19:bd3e3506b45a4c4abbaa8fa468d6df87@thread.v2` |
| Agent Email | `sujeev@e-gmat.com` |

---

## Summary: The Golden Rules

1. **Always read /home/.settings_/claude.md first** - Follow project structure & coding standards
2. **Always tag versions** - No deployment without a version tag
3. **Always use deploy.sh** - Never raw `git pull` on VPS
4. **Always revert, never reset** - Preserve history
5. **Always document** - CHANGELOG.md for every version
6. **Always notify** - Teams knows about every change
7. **Never commit secrets** - Use secrets/ folder, not .env in code
8. **Always use SSH keys** - No passwords for VPS
9. **Always use shared Mixpanel client** - `/home/.shared_utils/mixpanel/`
10. **Always stagger cron jobs** - Avoid API rate limit conflicts
11. **Always test on QA first** - Never push directly to main
12. **Port 8004 is reserved for QA** - Use for testing any application

---

*Document created: 2025-12-31*
*Last updated: 2026-01-08*
