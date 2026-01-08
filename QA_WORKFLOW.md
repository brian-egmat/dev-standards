# QA Workflow & Testing Best Practices

Standard process for testing changes before production deployment.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Branch Strategy](#2-branch-strategy)
3. [Port Reservation](#3-port-reservation)
4. [VPS QA Environment Setup](#4-vps-qa-environment-setup)
5. [Development Workflow](#5-development-workflow)
6. [Testing Checklist](#6-testing-checklist)
7. [Merging to Production](#7-merging-to-production)
8. [Quick Reference Commands](#8-quick-reference-commands)

---

## 1. Overview

**Principle:** All changes MUST be tested in a QA environment before production deployment.

### Why QA First?

| Risk | Solution |
|------|----------|
| Breaking production | Test in isolated QA environment |
| Untested edge cases | Verify with real data on QA |
| Configuration errors | Catch before users are affected |
| Regression bugs | Compare QA behavior vs production |

### Environment Summary

| Environment | Branch | Port | URL Pattern | Purpose |
|-------------|--------|------|-------------|---------|
| **Production** | `main` | 8003 | `/app-name/` | Live users |
| **QA** | `qa` | 8004 | `/app-name-qa/` | Testing |

---

## 2. Branch Strategy

### Two-Branch Model

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   qa branch                    main branch                      │
│   ──────────                   ───────────                      │
│   (Development)                (Production)                     │
│                                                                 │
│   1. Make changes              3. Merge qa → main               │
│   2. Test on QA URL            4. Deploy to production          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Branch Rules

| Branch | Purpose | Protection |
|--------|---------|------------|
| `main` | Production code | Only merge from `qa` after testing |
| `qa` | Active development | Free to push changes |

### Creating the QA Branch (First Time)

```bash
# Create qa branch from main
git checkout main
git checkout -b qa
git push -u origin qa
```

---

## 3. Port Reservation

### Reserved Port: 8004

**Port 8004 is RESERVED for QA testing across ALL applications.**

| Port | Environment | Notes |
|------|-------------|-------|
| 8003 | Production | Default app port |
| **8004** | **QA** | **Reserved for testing ANY app** |

### Usage

- When setting up QA for any application, use port 8004
- Only ONE app uses QA port at a time (shutdown previous QA when switching)
- Production apps continue running on their assigned ports

### Nginx Configuration Pattern

```nginx
# Production (port 8003)
location /app-name/ {
    proxy_pass http://127.0.0.1:8003/;
    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_connect_timeout 60s;
    proxy_send_timeout 60s;
    proxy_read_timeout 60s;
    proxy_buffering off;
}

# QA (port 8004)
location /app-name-qa/ {
    proxy_pass http://127.0.0.1:8004/;
    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_connect_timeout 60s;
    proxy_send_timeout 60s;
    proxy_read_timeout 60s;
    proxy_buffering off;
}
```

---

## 4. VPS QA Environment Setup

### Directory Structure

```
/home/Project-Folder/
├── AppName_0/              # Production (main branch)
│   ├── venv/
│   ├── secrets/
│   │   └── api_keys.json
│   └── ...
│
└── AppName_0_QA/           # QA (qa branch)
    ├── venv/
    ├── secrets/
    │   └── api_keys.json   # Copy from production
    └── ...
```

### Setting Up QA Directory

```bash
# 1. Clone qa branch to separate directory
cd /home/Project-Folder/
git clone -b qa https://github.com/brian-egmat/repo-name.git AppName_0_QA

# 2. Create virtual environment
cd AppName_0_QA
python3 -m venv venv

# 3. Install dependencies
./venv/bin/pip install -r requirements.txt

# 4. Copy secrets from production
cp ../AppName_0/secrets/api_keys.json ./secrets/

# 5. Start QA server on port 8004
./venv/bin/gunicorn -w 2 -b 127.0.0.1:8004 "v1.src.main:app" --daemon
```

### Nginx Setup

Add QA location block to nginx config:

```bash
# Edit nginx config
nano /etc/nginx/sites-enabled/your-site

# Test config
nginx -t

# Reload nginx
systemctl reload nginx
```

---

## 5. Development Workflow

### Step-by-Step Process

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   LOCAL                    GITHUB                    VPS        │
│   ─────                    ──────                    ───        │
│                                                                 │
│   1. checkout qa    ──►    2. push to qa    ──►    3. pull qa   │
│   4. make changes                                  5. restart   │
│   6. test on QA URL                                             │
│   7. repeat until ready                                         │
│                                                                 │
│   8. merge qa → main ──►   9. push main     ──►   10. deploy    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Detailed Steps

#### Phase 1: Development on QA

```bash
# LOCAL - Switch to qa branch
git checkout qa
git pull origin qa

# Make your changes...

# Commit and push to qa
git add .
git commit -m "WIP: description of change"
git push origin qa
```

#### Phase 2: Deploy to QA Server

```bash
# VPS - Update QA directory
cd /home/Project-Folder/AppName_0_QA
git pull origin qa

# Restart QA server
pkill -f "gunicorn.*8004"
./venv/bin/gunicorn -w 2 -b 127.0.0.1:8004 "v1.src.main:app" --daemon
```

#### Phase 3: Test on QA URL

```
QA URL: https://your-server.com/app-name-qa/
```

Test thoroughly:
- All features work as expected
- No errors in logs
- Edge cases handled
- UI displays correctly

#### Phase 4: Merge to Production

```bash
# LOCAL - Merge qa to main
git checkout main
git pull origin main
git merge qa
git push origin main

# Tag the release
git tag -a v2.X -m "Description of changes"
git push origin v2.X
```

#### Phase 5: Deploy to Production

```bash
# VPS - Deploy using deploy script
/home/github-deploy-notifier/deploy.sh /home/Project-Folder/AppName_0 "Hostinger VPS"
```

---

## 6. Testing Checklist

### Before Merging to Main

```
[ ] All features tested on QA URL
[ ] No console errors
[ ] No server errors in logs
[ ] Edge cases verified
[ ] UI/UX acceptable
[ ] Performance acceptable
[ ] API responses correct
[ ] Data saved correctly
[ ] Existing functionality not broken
```

### Log Verification

```bash
# Check QA server logs
tail -f /home/Project-Folder/AppName_0_QA/logs/*.log

# Check for errors
grep -i error /home/Project-Folder/AppName_0_QA/logs/*.log
```

---

## 7. Merging to Production

### Pre-Merge Checklist

```
[ ] Testing complete on QA
[ ] No pending issues
[ ] CHANGELOG.md updated
[ ] Version number decided
```

### Merge Commands

```bash
# Ensure main is up to date
git checkout main
git pull origin main

# Merge qa into main
git merge qa

# Resolve conflicts if any, then:
git push origin main

# Tag the release
git tag -a v2.X -m "$(cat <<'EOF'
Version 2.X: Brief description

- Change 1
- Change 2
- Change 3
EOF
)"
git push origin v2.X
```

### Post-Merge: Production Deployment

```bash
# SSH to VPS
ssh root@195.35.21.58

# Deploy with notification
/home/github-deploy-notifier/deploy.sh /home/Project-Folder/AppName_0 "Hostinger VPS"

# OR manual deployment
cd /home/Project-Folder/AppName_0
git pull origin main
pkill -f "gunicorn.*8003"
./venv/bin/gunicorn -w 2 -b 127.0.0.1:8003 "v1.src.main:app" --daemon
```

---

## 8. Quick Reference Commands

### Branch Management

| Action | Command |
|--------|---------|
| Switch to qa | `git checkout qa` |
| Switch to main | `git checkout main` |
| Update qa | `git pull origin qa` |
| Update main | `git pull origin main` |
| Merge qa to main | `git checkout main && git merge qa` |

### VPS QA Management

| Action | Command |
|--------|---------|
| Update QA code | `cd /path/to/app_QA && git pull origin qa` |
| Start QA server | `./venv/bin/gunicorn -w 2 -b 127.0.0.1:8004 "v1.src.main:app" --daemon` |
| Stop QA server | `pkill -f "gunicorn.*8004"` |
| Check QA running | `ps aux \| grep 8004` |
| View QA logs | `tail -f /path/to/app_QA/logs/*.log` |

### Testing URLs

| Environment | URL Pattern |
|-------------|-------------|
| Production | `https://your-server.com/app-name/` |
| QA | `https://your-server.com/app-name-qa/` |

---

## Example: Sales Agent Form

### Current Setup

| Environment | Port | Directory | URL |
|-------------|------|-----------|-----|
| Production | 8003 | `/home/Sales-Enhancements/E4_Update_Prospect_Records_0` | `/sales-agent-form/` |
| QA | 8004 | `/home/Sales-Enhancements/E4_Update_Prospect_Records_0_QA` | `/sales-agent-form-qa/` |

### Workflow Example

```bash
# 1. LOCAL: Make changes on qa branch
git checkout qa
# ... edit files ...
git add .
git commit -m "WIP: Add IST timestamp to extraction"
git push origin qa

# 2. VPS: Update and restart QA
ssh root@195.35.21.58
cd /home/Sales-Enhancements/E4_Update_Prospect_Records_0_QA
git pull origin qa
pkill -f "gunicorn.*8004"
./venv/bin/gunicorn -w 2 -b 127.0.0.1:8004 "v1.src.main:app" --daemon

# 3. TEST: Verify on QA URL
# https://srv1140745.hstgr.cloud/sales-agent-form-qa/

# 4. LOCAL: Merge to production
git checkout main
git merge qa
git push origin main
git tag -a v2.6 -m "v2.6: IST timestamp and notes guidance"
git push origin v2.6

# 5. VPS: Deploy to production
/home/github-deploy-notifier/deploy.sh /home/Sales-Enhancements/E4_Update_Prospect_Records_0 "Hostinger VPS"
```

---

## Golden Rules for QA

1. **NEVER push directly to main** - Always develop on `qa` branch
2. **ALWAYS test on QA URL first** - No exceptions
3. **Port 8004 is reserved for QA** - Don't use for production
4. **Merge only after testing** - QA must pass before production
5. **Document in CHANGELOG** - Record what changed in each version
6. **Tag every release** - Version numbers for tracking
7. **Keep QA secrets synced** - Copy api_keys.json from production

---

*Document created: 2026-01-08*
*Last updated: 2026-01-08*
