# GitHub System Documentation Index

Quick reference to all documentation files. **Start here!**

---

## Documentation Map

```
GitHub System/
|
+-- DOCUMENTATION INDEX (You are here)
|   |
|   +-- [1] DEVELOPMENT_RULES.md        <- Start here for rules
|   +-- [2] GITHUB_ARCHITECTURE_GUIDE.md <- Architecture overview
|   +-- [3] COLLABORATOR_GUIDE_SANDEEP.md <- For team members
|   +-- [4] _templates/BEST_PRACTICES.md <- Coding standards
|
+-- VPS Settings (on server)
    +-- /home/.settings_/claude.md      <- VPS-specific standards
```

---

## Quick Links

| Document | Purpose | When to Use |
|----------|---------|-------------|
| **DEVELOPMENT_RULES.md** | Rules for versioning, deployment, notifications | Before ANY deployment |
| **GITHUB_ARCHITECTURE_GUIDE.md** | How the system works | Understanding the setup |
| **COLLABORATOR_GUIDE_SANDEEP.md** | Step-by-step for team members | New team member onboarding |
| **_templates/BEST_PRACTICES.md** | Coding standards, patterns | Creating new projects |

---

## 1. DEVELOPMENT_RULES.md

**Location:** `GitHub System/DEVELOPMENT_RULES.md`

**Key Topics:**
- Version numbering (v1, v1.1, v2, etc.)
- Git workflow
- Teams notifications
- VPS connection
- Deployment process
- Rollback procedures
- Security rules

**The Golden Rules:**
1. Always read /home/.settings_/claude.md first
2. Always tag versions
3. Always use deploy.sh (never raw git pull)
4. Always revert, never reset
5. Always document in CHANGELOG.md
6. Never commit secrets

---

## 2. GITHUB_ARCHITECTURE_GUIDE.md

**Location:** `GitHub System/GITHUB_ARCHITECTURE_GUIDE.md`

**Key Topics:**
- Architecture diagram (Local -> GitHub -> VPS)
- Component descriptions
- Initial setup steps
- Development workflow
- Version management
- Rollback options

---

## 3. COLLABORATOR_GUIDE_SANDEEP.md

**Location:** `GitHub System/COLLABORATOR_GUIDE_SANDEEP.md`

**Key Topics:**
- One-time setup (Git, credentials)
- Clone a project
- Make changes
- Push to GitHub
- Deploy to VPS
- Revert to previous version
- Quick reference commands
- Troubleshooting

**For:** New team members learning the workflow

---

## 4. BEST_PRACTICES.md

**Location:** `GitHub System/_templates/BEST_PRACTICES.md`

**Key Topics:**
- Database migrations
- Git commit format
- Project structure template
- Configuration management
- Error handling & logging
- Deployment checklist
- Documentation standards

**For:** Creating new projects with proper standards

---

## VPS Settings Reference

**Location:** `/home/.settings_/` on VPS (195.35.21.58)

**How to access:**
```bash
ssh root@195.35.21.58 "cat /home/.settings_/claude.md"
```

**Files:**
| File | Purpose |
|------|---------|
| claude.md | Project structure, coding standards |
| api_keys.json | API keys reference |
| llm-providers-reference.json | LLM configurations |
| WEBHOOK_SETUP_GUIDE.md | Webhook setup |

---

## Quick Commands

### Before Starting Work
```bash
# Read the rules
cat "D:/31-12-2025/GitHub System/DEVELOPMENT_RULES.md"

# Check VPS standards
ssh root@195.35.21.58 "cat /home/.settings_/claude.md"
```

### Creating a New Project
```bash
# Check best practices template
cat "D:/31-12-2025/GitHub System/_templates/BEST_PRACTICES.md"
```

### Onboarding New Team Member
```bash
# Share the collaborator guide
cat "D:/31-12-2025/GitHub System/COLLABORATOR_GUIDE_SANDEEP.md"
```

---

## Credentials Quick Reference

| Service | Details |
|---------|---------|
| **GitHub** | Username: brian-egmat |
| **VPS** | IP: 195.35.21.58, User: root, Auth: SSH Key |
| **Teams** | Notifications via Power Automate |

---

## Project Repositories

| Project | Purpose | Location on VPS |
|---------|---------|-----------------|
| Marketing_GC_Reviews_Monthly | GMAT reviews scraper | /home/Regular-Tasks/Marketing_GC_Reviews_Monthly |
| My-first-app-gh0 | Test project | /home/Testing-SK/my-first-gh-project |
| github-deploy-notifier | Deploy notifications | /home/github-deploy-notifier |

---

*Last Updated: 2026-01-01*
