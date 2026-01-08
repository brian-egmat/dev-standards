# Deployment Testing Checklist

Pre-deployment and post-deployment verification to prevent bugs reaching production.

---

## Why This Exists

On 2026-01-08, a critical bug was discovered where `Interaction_Explanation` (notes) were not being saved to Airtable. The bug was introduced in v2.1 when the endpoint was changed from `/api/submit` to `/api/extract-and-save`, but the new endpoint didn't include the notes field.

**Root Causes:**
1. No regression testing after endpoint changes
2. No verification that all fields were being saved
3. Code was deployed without testing the complete user flow

---

## Pre-Deployment Checklist

### Before Pushing to QA Branch

```
[ ] Code changes compile/run without errors
[ ] Changed files reviewed for obvious issues
[ ] No hardcoded test values left in code
[ ] No console.log/print debug statements left
[ ] Config values use environment/secrets (not hardcoded)
```

### Before Merging QA to Main

```
[ ] COMPLETE USER FLOW TESTED (see below)
[ ] All API endpoints return expected responses
[ ] All database/Airtable fields are updated correctly
[ ] Error cases handled gracefully
[ ] No JavaScript console errors
[ ] No server errors in logs
```

---

## Complete User Flow Test

**For Sales Agent Form specifically:**

### 1. Agent Selection
```
[ ] Agent dropdown loads with all agents
[ ] Selecting agent loads their prospects
```

### 2. Prospect Selection
```
[ ] Prospect list displays correctly
[ ] Clicking prospect enables notes textarea
[ ] Search by email/phone works
```

### 3. Notes Validation
```
[ ] Enter valid notes → Validation passes
[ ] Enter invalid notes → Shows missing items
[ ] Suggested additions display correctly
```

### 4. Save to Airtable (CRITICAL)
```
[ ] Click "Save to Airtable" completes successfully
[ ] VERIFY IN AIRTABLE:
    [ ] Interaction_Explanation contains the notes text
    [ ] Reachout_Channel is set correctly
    [ ] Response is set correctly
    [ ] User_Bucket is set correctly
    [ ] Qualification is set correctly
    [ ] Extraction_Timestamp is set
    [ ] Extraction_Confidence is set
    [ ] Call_Status updated (if applicable)
```

### 5. Edge Cases
```
[ ] Test "Can't Happen" scenarios (Unable to Reach, etc.)
[ ] Test "Reschedule" with date/time
[ ] Test existing student case
[ ] Test no response case
```

---

## Post-Deployment Verification

### Immediately After Deploy

```bash
# 1. Verify server is running
ssh root@195.35.21.58 "ps aux | grep gunicorn | grep 8003"

# 2. Check server responds
curl -s https://srv1140745.hstgr.cloud/sales-agent-form/ | head -5

# 3. Check logs for errors
ssh root@195.35.21.58 "tail -20 /home/Sales-Enhancements/E4_Update_Prospect_Records_0/logs/ai_validation.log"
```

### Functional Verification

```
[ ] Open production URL in browser
[ ] Complete ONE full user flow (select agent → select prospect → write notes → validate → save)
[ ] VERIFY the record was updated correctly in Airtable
[ ] Check server logs for any errors
```

---

## API Endpoint Testing

### Test Each Endpoint Manually

```bash
# Test validation endpoint
curl -X POST http://127.0.0.1:8003/api/validate \
  -H 'Content-Type: application/json' \
  -d '{"email": "test@test.com", "notes": "Test notes here"}'

# Test extract-and-save endpoint (CRITICAL)
curl -X POST http://127.0.0.1:8003/api/extract-and-save \
  -H 'Content-Type: application/json' \
  -d '{"record_id": "recXXX", "email": "test@test.com", "notes": "Test notes"}'
```

### Verify Response Contains

```json
{
  "success": true,
  "fields_updated": [
    "Reachout_Channel",
    "Response",
    "User_Bucket",
    "Qualification",
    "Extraction_Timestamp",
    "Extraction_Confidence",
    "Interaction_Explanation"  // <-- THIS MUST BE PRESENT
  ]
}
```

---

## Common Issues to Check

### 1. Fields Not Being Saved
- Check `build_airtable_update_fields()` includes all required fields
- Check `fields_to_update` dict before `update_airtable_record()` call
- Verify field names match Airtable column names exactly

### 2. Server Won't Start
- Check gunicorn command uses correct module path (`v1.src.main:application`)
- Check all imports resolve correctly
- Check secrets file exists and is readable

### 3. 404 Errors
- Verify nginx config points to correct port
- Verify gunicorn is actually running
- Check nginx was reloaded after config changes

### 4. API Returns Errors
- Check server logs for traceback
- Verify API keys are valid
- Check Airtable permissions

---

## Regression Test Script

Save this as `test_deployment.sh` and run after each deploy:

```bash
#!/bin/bash
# Quick deployment verification script

SERVER="http://127.0.0.1:8003"

echo "=== Testing Sales Agent Form Deployment ==="

# Test 1: Server responds
echo -n "1. Server health check... "
if curl -s "$SERVER/" | grep -q "Sales Agent"; then
    echo "PASS"
else
    echo "FAIL"
    exit 1
fi

# Test 2: Validation endpoint
echo -n "2. Validation endpoint... "
RESPONSE=$(curl -s -X POST "$SERVER/api/validate" \
    -H 'Content-Type: application/json' \
    -d '{"email": "test@test.com", "notes": "Reached out via WhatsApp. Prospect confirmed attendance."}')
if echo "$RESPONSE" | grep -q '"valid"'; then
    echo "PASS"
else
    echo "FAIL"
    exit 1
fi

# Test 3: Check fields_updated includes Interaction_Explanation
echo -n "3. Fields include Interaction_Explanation... "
# This test requires a valid record_id, skip in automated tests
echo "SKIP (manual verification required)"

echo ""
echo "=== Basic tests passed ==="
echo "IMPORTANT: Manually verify a complete save operation updates Airtable correctly"
```

---

## Version Deployment Log

Keep a log of each deployment with test results:

```
| Date | Version | Tested By | Test Result | Notes |
|------|---------|-----------|-------------|-------|
| 2026-01-08 | v2.7 | Claude | PASS | Fixed Interaction_Explanation bug |
| 2026-01-08 | v2.6 | - | FAIL | Notes not saved (bug) |
```

---

## Golden Rules

1. **ALWAYS test the complete user flow** - Don't just test the endpoint you changed
2. **ALWAYS verify Airtable updates** - API success doesn't mean data was saved correctly
3. **Test on QA first** - Never push untested code to production
4. **Check the fields_updated response** - Make sure all expected fields are listed
5. **Keep a deployment log** - Track what was deployed and whether tests passed

---

*Document created: 2026-01-08*
*Last updated: 2026-01-08*
