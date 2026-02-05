# GitHub System Documentation Index

Quick reference to all documentation files. **Start here!**

---

## Documentation Map

```
GitHub System/
|
+-- DOCUMENTATION INDEX (You are here)
|   |
|   +-- [1] DEVELOPMENT_RULES.md         <- Start here for rules
|   +-- [2] GITHUB_ARCHITECTURE_GUIDE.md <- Architecture overview
|   +-- [3] _templates/
|   |       +-- BEST_PRACTICES.md        <- Coding standards
|   |       +-- AI_USAGE_MONITORING.md   <- AI/Claude API usage tracking (REQUIRED)
|   |       +-- DATABASE_SCHEMA_GUIDE.md <- Database & backward compatibility
|   |       +-- notify-on-push.yml       <- GitHub Actions workflow template
|   |       +-- api_keys_reference.json  <- API keys for all services
|   +-- [4] QA_WORKFLOW.md               <- QA testing before production
|   +-- [5] DEPLOYMENT_TESTING_CHECKLIST.md <- Pre/post deploy verification
|
+-- VPS Settings (on server)
    +-- /home/.settings_/claude.md       <- VPS-specific standards
```

---

## Quick Links

| Document | Purpose | When to Use |
|----------|---------|-------------|
| **DEVELOPMENT_RULES.md** | Rules for versioning, deployment, notifications | Before ANY deployment |
| **GITHUB_ARCHITECTURE_GUIDE.md** | How the system works | Understanding the setup |
| **_templates/BEST_PRACTICES.md** | Coding standards, patterns | Creating new projects |
| **_templates/AI_USAGE_MONITORING.md** | Claude API usage tracking | **ALL apps using Claude API** |
| **_templates/DATABASE_SCHEMA_GUIDE.md** | Schema changes, migrations, rollbacks | Database modifications |
| **_templates/notify-on-push.yml** | Teams notification workflow | Setting up new repos |
| **_templates/api_keys_reference.json** | API keys for Customer.io, Segment, Mixpanel, etc. | Configuring new apps |
| **QA_WORKFLOW.md** | QA testing workflow, port 8004 reserved | Before pushing to production |
| **DEPLOYMENT_TESTING_CHECKLIST.md** | Pre/post deployment verification | After every deployment |

---

## 1. DEVELOPMENT_RULES.md

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

**Key Topics:**
- Architecture diagram (Local -> GitHub -> VPS)
- Component descriptions
- Initial setup steps
- Development workflow
- Version management
- Rollback options

---

## 3. Templates

### BEST_PRACTICES.md

**Key Topics:**
- Database migrations
- Git commit format
- Project structure template
- Configuration management
- Error handling & logging
- Deployment checklist
- Documentation standards
- Remote server file modification patterns

**For:** Creating new projects with proper standards

### AI_USAGE_MONITORING.md (REQUIRED for Claude API apps)

**Key Topics:**
- AI usage logging integration
- Cost tracking and budget alerts
- Shared logger module usage
- Error handling best practices
- Testing and verification steps
- Common pitfalls and solutions
- Model pricing reference

**For:** ALL applications using Anthropic Claude API

**The Core Rule:**
> Every Claude API call MUST be logged to the central monitoring system at `/home/.ai_monitoring/`

**Quick Integration:**
```python
import sys
sys.path.insert(0, '/home/.ai_monitoring/lib')
from ai_usage_logger import AIUsageLogger

logger = AIUsageLogger(app_name="Your_App")
# After API call:
logger.log_from_response(response, purpose="query", latency_ms=latency)
```

### DATABASE_SCHEMA_GUIDE.md

**Key Topics:**
- The Expand-Contract Pattern (industry standard)
- Safe vs unsafe database operations
- Migration strategies for zero downtime
- Rollback scenarios and safety
- Version compatibility matrix
- Migration file templates
- Tools (Flyway, Liquibase, Alembic, etc.)
- Step-by-step examples

**For:** Any database schema changes that need to be backward compatible

**The Core Rule:**
> Never make schema changes that break currently running code.
> Old code and new code must BOTH work with the database at every step.

### notify-on-push.yml

**Purpose:** GitHub Actions workflow that sends Teams notifications on every push to main/master.

**Setup for any repo:**
1. Copy `_templates/notify-on-push.yml` to `.github/workflows/notify-on-push.yml`
2. Add GitHub Secrets (Settings > Secrets > Actions):
   - `POWER_AUTOMATE_URL`
   - `TEAMS_CHAT_ID`
   - `AGENT_EMAIL`
3. Push to main branch - notification will be sent!

**For:** Enabling push notifications on any repository

### api_keys_reference.json

**Purpose:** Centralized API keys reference for all third-party services.

**Services included:**
| Service | Keys Available |
|---------|----------------|
| **Customer.io** | Track API (Site ID, API Key), App API Key |
| **Segment** | Write Key |
| **Mixpanel** | Project ID, Secret, Username |
| **Airtable** | Personal Access Token |
| **Anthropic** | Forum Responses key, Sales Operations key |
| **OpenAI** | API Key, Org ID |

**For:** Configuring new applications with required API credentials

**Usage:**
```bash
# View all available keys
cat _templates/api_keys_reference.json

# Example: Get Customer.io env vars for .env file
# CIO_SITE_ID=f08b2621c1e5dd0345b8
# CIO_API_KEY=2b48fe86d4fe6dfcde4d
# CIO_TRACK_URL=https://track.customer.io/api/v1
# CIO_APP_API_KEY=3acea18efa40180ec76ea74fe366e3ae
```

---

## 4. QA_WORKFLOW.md

**Location:** `GitHub System/QA_WORKFLOW.md`

**Key Topics:**
- Branch strategy (`main` = production, `qa` = testing)
- Port reservation (8004 for QA)
- VPS QA environment setup
- Development workflow (qa → test → merge → production)
- Testing checklist
- Merge and deployment process

**For:** Testing changes before production deployment

**Golden Rules:**
1. NEVER push directly to main - develop on `qa` branch
2. ALWAYS test on QA URL first - no exceptions
3. Port 8004 is reserved for QA testing
4. Merge only after testing passes

---

## 5. DEPLOYMENT_TESTING_CHECKLIST.md

**Location:** `GitHub System/DEPLOYMENT_TESTING_CHECKLIST.md`

**Key Topics:**
- Pre-deployment checklist
- Complete user flow testing
- Post-deployment verification
- API endpoint testing
- Common issues to check
- Regression test script

**For:** Verifying deployments work correctly

**Critical Lesson (2026-01-08):**
> A bug where notes weren't saved was introduced because the complete user flow wasn't tested after an endpoint change. Always verify ALL fields are saved correctly, not just that the API returns success.

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
cat DEVELOPMENT_RULES.md

# Check VPS standards
ssh root@195.35.21.58 "cat /home/.settings_/claude.md"
```

### Creating a New Project
```bash
# Check best practices template
cat _templates/BEST_PRACTICES.md
```

### Making Database Changes
```bash
# Read the database schema guide first!
cat _templates/DATABASE_SCHEMA_GUIDE.md

# Key pattern: Expand -> Migrate -> Contract
# Never drop columns that running code uses
```

### Setting Up Notifications for a Repo
```bash
# Copy workflow to your repo
mkdir -p .github/workflows
cp _templates/notify-on-push.yml .github/workflows/

# Then add secrets in GitHub UI
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

*Last Updated: 2026-02-05*
