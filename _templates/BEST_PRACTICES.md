# Software Development Best Practices

Reference guide for all future projects. Apply these standards when creating new repositories.

---

## 1. Database Management

### Schema Versioning with Migrations

**Problem:** Database schema changes between code versions can break applications.

**Solution:** Use migration files to track and apply schema changes.

### Migration File Template

```python
# migrations/001_initial_schema.py
def upgrade(cursor):
    cursor.execute("CREATE TABLE IF NOT EXISTS ...")

def downgrade(cursor):
    cursor.execute("DROP TABLE IF EXISTS ...")

VERSION = 1
DESCRIPTION = "Initial schema"
```

### Database Best Practices Checklist

- [ ] Database files in `.gitignore`
- [ ] Use `CREATE TABLE IF NOT EXISTS`
- [ ] Use migrations for schema changes
- [ ] Always backup before schema changes
- [ ] Keep changes backward-compatible when possible
- [ ] Document breaking changes in CHANGELOG

---

## 2. Git and Version Control

### Commit Message Format

```
<type>: <short description>

<detailed description if needed>

Generated with [Claude Code](https://claude.com/claude-code)
Co-Authored-By: Claude Opus 4.5 <noreply@anthropic.com>
```

**Types:** feat, fix, docs, refactor, test, chore

### Version Numbering (Semantic Versioning)

- MAJOR: Breaking changes
- MINOR: New features (backward-compatible)
- PATCH: Bug fixes (backward-compatible)

### Standard .gitignore

```
__pycache__/
venv/
.env
*.db
logs/
output/
```

---

## 3. Project Structure

```
project-name/
+-- src/
|   +-- main.py            # Entry point
|   +-- config.py          # Configuration
|   +-- utils.py
+-- migrations/             # Database migrations
+-- data/                   # Database (gitignored)
+-- logs/                   # Logs (gitignored)
+-- deploy/                 # Deployment configs
+-- .env.example
+-- .gitignore
+-- requirements.txt
+-- README.md
+-- CHANGELOG.md
```

---

## 4. Configuration Management

**Never commit secrets!** Use `.env` files:

```bash
# .env.example (commit this)
API_KEY=your_api_key_here

# .env (gitignored)
API_KEY=sk-actual-secret-key
```

---

## 5. Error Handling and Logging

```python
import logging
from datetime import datetime

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s - %(levelname)s - %(message)s",
    handlers=[
        logging.FileHandler(f"logs/app.log"),
        logging.StreamHandler()
    ]
)
```

---

## 6. Deployment Checklist

### Pre-Deployment
- [ ] All tests passing
- [ ] README updated
- [ ] Version bumped

### Post-Deployment
- [ ] Verify application starts
- [ ] Run migrations
- [ ] Send notification

### Systemd Service Template

```ini
[Unit]
Description=My Application
After=network.target

[Service]
Type=simple
WorkingDirectory=/path/to/app
ExecStart=/path/to/venv/bin/python src/main.py
Restart=always

[Install]
WantedBy=multi-user.target
```

---

## 7. Notifications

After git push and deployment:

```bash
python notify.py push /path/to/repo
python notify.py deploy /path/to/repo "Server Name"
```

---

## 8. Documentation Standards

### README.md Must Include
1. Project Description
2. Features
3. Installation
4. Configuration
5. Usage
6. Troubleshooting
7. Version History

---

## Quick Reference

```bash
# New project
git init && python -m venv venv && source venv/bin/activate

# Release
git commit -m "feat: description"
git tag -a v1.0.0 -m "Version 1.0.0"
git push origin main --tags

# Backup DB
cp data/app.db data/app.db.backup
```

---

*Last Updated: 2026-01-01*
