# GitHub Development & Deployment Architecture Guide

A comprehensive guide for managing applications using GitHub version control with VPS deployment.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Components](#components)
3. [Initial Setup](#initial-setup)
4. [Development Workflow](#development-workflow)
5. [Version Management](#version-management)
6. [Deployment Process](#deployment-process)
7. [Rollback Procedures](#rollback-procedures)
8. [Monitoring & Tracking](#monitoring--tracking)
9. [Security Best Practices](#security-best-practices)
10. [Command Reference](#command-reference)

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           DEVELOPMENT FLOW                                   │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────┐         ┌─────────────────┐         ┌─────────────────┐
│   LOCAL PC      │         │     GITHUB      │         │      VPS        │
│   (Development) │         │  (Central Repo) │         │  (Production)   │
├─────────────────┤         ├─────────────────┤         ├─────────────────┤
│                 │         │                 │         │                 │
│  1. Write Code  │         │  Stores:        │         │  Runs:          │
│  2. Test        │  push   │  - All code     │  pull   │  - Application  │
│  3. Commit      │ ──────► │  - History      │ ──────► │  - Cron jobs    │
│  4. Push        │         │  - Versions     │         │  - Services     │
│                 │         │  - Tags         │         │                 │
│  Tools:         │         │                 │         │  Tools:         │
│  - Git          │         │  Tracks:        │         │  - Git          │
│  - Python       │         │  - Commits      │         │  - Python/venv  │
│  - VS Code      │         │  - Contributors │         │  - Cron         │
│  - Claude Code  │         │  - Changes      │         │  - Docker (opt) │
│                 │         │                 │         │                 │
└─────────────────┘         └─────────────────┘         └─────────────────┘
        │                                                       │
        │                      SSH (No Password)                │
        └───────────────────────────────────────────────────────┘
```

---

## Components

### 1. Local Development Environment

| Component | Purpose |
|-----------|---------|
| Git | Version control |
| Python | Application runtime |
| VS Code / Editor | Code editing |
| Claude Code | AI-assisted development |
| SSH Keys | Passwordless VPS access |

**Location**: `D:\31-12-2025\GitHub System\`

### 2. GitHub (Central Repository)

| Feature | Purpose |
|---------|---------|
| Repository | Stores all code |
| Commits | Tracks every change |
| Tags | Marks versions (v1, v1.1, v1.2) |
| Branches | Parallel development (optional) |
| Security Log | Tracks access and PAT usage |

**Access**: Via Personal Access Token (PAT)

### 3. VPS (Production Server)

| Component | Purpose |
|-----------|---------|
| Git | Pull updates from GitHub |
| Python + venv | Isolated runtime environment |
| Cron | Scheduled task execution |
| Logs | Application monitoring |

**Server**: Hostinger VPS (`195.35.21.58`)

---

## Initial Setup

### Step 1: Create GitHub Repository

```bash
# Via GitHub API (using PAT)
curl -X POST https://api.github.com/user/repos \
  -H "Authorization: token YOUR_PAT" \
  -H "Accept: application/vnd.github.v3+json" \
  -d '{"name":"repo-name","private":true,"auto_init":true}'
```

Or manually at: https://github.com/new

### Step 2: Clone Locally

```bash
git clone https://USERNAME:PAT@github.com/USERNAME/REPO_NAME.git
cd REPO_NAME
git config user.name "your-username"
git config user.email "your-email@example.com"
```

### Step 3: Setup VPS

```bash
# SSH into VPS
ssh root@YOUR_VPS_IP

# Create project directory
mkdir -p /home/PROJECT_FOLDER/app-name
cd /home/PROJECT_FOLDER/app-name

# Clone repository
git clone https://USERNAME:PAT@github.com/USERNAME/REPO_NAME.git .

# Create virtual environment
python3 -m venv venv
./venv/bin/pip install -r requirements.txt

# Create .env file with secrets
nano .env

# Test application
./venv/bin/python main.py
```

### Step 4: Setup SSH Key Authentication (Passwordless)

```bash
# On local machine - generate key (if not exists)
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -N ""

# Copy public key to VPS
ssh-copy-id root@YOUR_VPS_IP

# Or manually add to VPS
cat ~/.ssh/id_rsa.pub | ssh root@YOUR_VPS_IP "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"

# Test passwordless login
ssh root@YOUR_VPS_IP "echo 'Connected successfully'"
```

---

## Development Workflow

### Daily Development Cycle

```
┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│   1. PULL          2. CODE          3. TEST         4. PUSH     │
│   ─────────────────────────────────────────────────────────────► │
│                                                                  │
│   git pull         Edit files       python app.py   git add .   │
│                                     pytest          git commit   │
│                                                     git push     │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Commands

```bash
# 1. Start of day - get latest code
git pull

# 2. Make changes to your files

# 3. Test locally
python your_app.py
python -m pytest tests/

# 4. Check what changed
git status
git diff

# 5. Stage and commit
git add .
git commit -m "Description of changes"

# 6. Push to GitHub
git push
```

---

## Version Management

### Semantic Versioning

```
v1.0.0
│ │ │
│ │ └── PATCH: Bug fixes (backwards compatible)
│ └──── MINOR: New features (backwards compatible)
└────── MAJOR: Breaking changes
```

### Creating Versions

```bash
# Simple tag
git tag v1.0

# Annotated tag (recommended)
git tag -a v1.0 -m "Initial release - description here"

# Push tags to GitHub
git push origin v1.0

# Push all tags
git push --tags
```

### Viewing Versions

```bash
# List all tags
git tag

# Show tag details
git show v1.0

# See commit for a tag
git log v1.0 -1
```

### CHANGELOG.md Structure

```markdown
# Changelog

## [v1.2] - YYYY-MM-DD
- Change description
- Another change
- Reason for changes (if reverting)

## [v1.1] - YYYY-MM-DD
- Feature added
- Bug fixed

## [v1.0] - YYYY-MM-DD
- Initial release
```

---

## Deployment Process

### Standard Deployment

```bash
# 1. Push code from local
git add .
git commit -m "v1.x: Description"
git push
git tag -a v1.x -m "Version description"
git push origin v1.x

# 2. Deploy to VPS
ssh root@YOUR_VPS_IP "cd /path/to/app && git pull"

# 3. Test on VPS
ssh root@YOUR_VPS_IP "cd /path/to/app && ./venv/bin/python main.py"
```

### One-Liner Deployment

```bash
# Push and deploy in one command
git push && ssh root@YOUR_VPS_IP "cd /path/to/app && git pull && ./venv/bin/python main.py"
```

### Check VPS is Up-to-Date

```bash
ssh root@YOUR_VPS_IP "cd /path/to/app && git fetch && git status"
```

**Expected output**: `Your branch is up to date with 'origin/main'`

---

## Rollback Procedures

### Option 1: Quick VPS Rollback (Temporary)

```bash
# Switch to specific version on VPS
ssh root@YOUR_VPS_IP "cd /path/to/app && git checkout v1.0"

# Return to latest
ssh root@YOUR_VPS_IP "cd /path/to/app && git checkout main"
```

### Option 2: Revert Commit (Recommended - Permanent)

```bash
# Find commit to revert
git log --oneline

# Revert specific commit
git revert COMMIT_HASH --no-edit

# Push the revert
git push

# Tag as new version
git tag -a v1.x -m "Reverts to vX.X - reason"
git push origin v1.x

# Update VPS
ssh root@YOUR_VPS_IP "cd /path/to/app && git pull"
```

### Option 3: Hard Reset (Destructive - Use Carefully)

```bash
# Reset to specific version
git reset --hard v1.0
git push --force

# WARNING: This permanently removes commits after v1.0
```

### Rollback Decision Guide

| Situation | Use |
|-----------|-----|
| Quick test of old version | Option 1 (checkout) |
| Bug in production, need rollback | Option 2 (revert) |
| Accidentally committed secrets | Option 3 (hard reset) |

---

## Monitoring & Tracking

### Git History

```bash
# View commit history
git log --oneline

# Detailed history with dates
git log --pretty=format:"%h %ad %s" --date=short

# History for specific file
git log --oneline -- filename.py

# Who changed each line
git blame filename.py
```

### GitHub Tracking

| What | Where |
|------|-------|
| Commit history | Repo → Commits |
| Contributors | Repo → Insights → Contributors |
| Traffic | Repo → Insights → Traffic |
| Security events | Settings → Security log |

### VPS Logs

```bash
# Application logs (if configured)
tail -f /var/log/app-name.log

# Cron job logs
tail -f /var/log/cron.log

# View cron jobs
crontab -l
```

---

## Security Best Practices

### PAT Management

| Practice | Why |
|----------|-----|
| Use separate PATs per machine | Track which machine accessed |
| Use Fine-Grained PATs | Limit permissions |
| Set expiration dates | Auto-revoke old tokens |
| Delete unused PATs | Reduce attack surface |

**Create Fine-Grained PAT**: https://github.com/settings/personal-access-tokens/new

### .gitignore (Never Commit Secrets)

```gitignore
# Environment files
.env
*.env

# Credentials
credentials.json
*.pem
*.key

# Python
__pycache__/
venv/
*.pyc
```

### .env File (Store Secrets Locally)

```bash
# .env.example (commit this - template)
API_KEY=your-api-key-here
DATABASE_URL=your-db-url-here

# .env (never commit - actual secrets)
API_KEY=actual-secret-key
DATABASE_URL=actual-db-connection
```

---

## Command Reference

### Git Basics

| Action | Command |
|--------|---------|
| Clone repo | `git clone URL` |
| Check status | `git status` |
| View changes | `git diff` |
| Stage all | `git add .` |
| Commit | `git commit -m "message"` |
| Push | `git push` |
| Pull | `git pull` |

### Version Tags

| Action | Command |
|--------|---------|
| Create tag | `git tag v1.0` |
| Annotated tag | `git tag -a v1.0 -m "description"` |
| Push tag | `git push origin v1.0` |
| List tags | `git tag` |
| Show tag | `git show v1.0` |
| Delete tag | `git tag -d v1.0` |

### History & Tracking

| Action | Command |
|--------|---------|
| View history | `git log --oneline` |
| File history | `git log -- filename` |
| Who changed lines | `git blame filename` |
| Compare versions | `git diff v1.0 v1.1` |

### VPS Operations

| Action | Command |
|--------|---------|
| SSH connect | `ssh root@IP` |
| Check sync status | `git fetch && git status` |
| Pull updates | `git pull` |
| Run app | `./venv/bin/python main.py` |
| View cron | `crontab -l` |
| Edit cron | `crontab -e` |

---

## Project Structure Template

```
project-name/
├── .env.example        # Environment template (commit this)
├── .env                # Actual secrets (DO NOT commit)
├── .gitignore          # Files to ignore
├── CHANGELOG.md        # Version history
├── README.md           # Project documentation
├── Dockerfile          # Container config (optional)
├── requirements.txt    # Python dependencies
├── main.py             # Application entry point
├── tests/
│   └── test_app.py     # Unit tests
└── venv/               # Virtual environment (DO NOT commit)
```

---

## Quick Start Checklist (New Project)

- [ ] Create GitHub repository
- [ ] Clone locally
- [ ] Set up git config (name, email)
- [ ] Create `.gitignore`
- [ ] Create `.env.example`
- [ ] Create `requirements.txt`
- [ ] Create `CHANGELOG.md`
- [ ] Write application code
- [ ] Test locally
- [ ] Commit and push
- [ ] Tag as v1
- [ ] Clone on VPS
- [ ] Create venv on VPS
- [ ] Create `.env` on VPS
- [ ] Set up cron job (if needed)
- [ ] Test on VPS

---

## Credentials Reference

### GitHub

| Item | Value |
|------|-------|
| Username | `brian-egmat` |
| PAT Mgmt | https://github.com/settings/tokens |
| Security Log | https://github.com/settings/security-log |

### VPS (Hostinger)

| Item | Value |
|------|-------|
| IP | `195.35.21.58` |
| User | `root` |
| Auth | SSH Key (passwordless) |

---

*Document created: 2025-12-31*
*Last updated: 2025-12-31*
