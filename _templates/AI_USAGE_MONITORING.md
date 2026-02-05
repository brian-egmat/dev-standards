# AI Usage Monitoring Integration Guide

**MANDATORY for all applications using Anthropic Claude API.**

This guide explains how to integrate AI usage tracking into your application. All new apps MUST include AI usage logging to ensure cost visibility and budget management.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Architecture](#2-architecture)
3. [Quick Start](#3-quick-start)
4. [Integration Methods](#4-integration-methods)
5. [Code Examples](#5-code-examples)
6. [Error Handling Best Practices](#6-error-handling-best-practices)
7. [Testing & Verification](#7-testing--verification)
8. [Common Pitfalls & Solutions](#8-common-pitfalls--solutions)
9. [Model Pricing Reference](#9-model-pricing-reference)
10. [Checklist](#10-checklist)

---

## 1. Overview

### What This System Does

The AI Usage Monitoring System tracks all Anthropic Claude API calls across VPS applications:

- **Cost Tracking**: Automatic cost calculation per request
- **Token Counting**: Input, output, cache, and thinking tokens
- **Daily Reports**: Automated Teams notifications at 11:30 PM IST
- **Budget Alerts**: Warnings when spending exceeds thresholds
- **Error Monitoring**: Track failed API calls

### Why It's Required

| Reason | Impact |
|--------|--------|
| Cost visibility | Know exactly what each app spends |
| Budget management | Alerts before overspending |
| Debugging | Track errors and latency |
| Optimization | Identify high-cost operations |

### System Location

```
VPS: /home/.ai_monitoring/
├── config/
│   └── settings.json      # App configurations
├── lib/
│   ├── ai_usage_logger.py # Shared logging module
│   └── log_adapters.py    # Log format adapters
├── logs/
│   └── {App_Name}.jsonl   # Per-app log files
├── services/
│   └── daily_report_v2.py # Daily report generator
└── docs/
    └── INTEGRATION_GUIDE.md
```

---

## 2. Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Your Application                              │
│                                                                      │
│   ┌─────────────┐     ┌─────────────┐     ┌─────────────────────┐   │
│   │ User        │ --> │ Claude API  │ --> │ AIUsageLogger       │   │
│   │ Request     │     │ Call        │     │ (logs to JSONL)     │   │
│   └─────────────┘     └─────────────┘     └─────────────────────┘   │
│                                                    │                 │
└────────────────────────────────────────────────────│─────────────────┘
                                                     │
                                                     v
┌─────────────────────────────────────────────────────────────────────┐
│                    Central Monitoring System                         │
│                    /home/.ai_monitoring/                             │
│                                                                      │
│   ┌─────────────┐     ┌─────────────┐     ┌─────────────────────┐   │
│   │ JSONL Logs  │ --> │ Daily       │ --> │ Teams               │   │
│   │ (per app)   │     │ Report v2   │     │ Notification        │   │
│   └─────────────┘     └─────────────┘     └─────────────────────┘   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. Quick Start

### Minimal Integration (5 minutes)

```python
# 1. Add to your imports
import sys
sys.path.insert(0, '/home/.ai_monitoring/lib')
from ai_usage_logger import AIUsageLogger

# 2. Initialize logger (once, at module level)
usage_logger = AIUsageLogger(app_name="Your_App_Name")

# 3. Log after each API call
response = client.messages.create(...)
usage_logger.log_from_response(
    response=response,
    purpose="your_purpose",
    latency_ms=latency
)
```

### After Integration

Add your app to `/home/.ai_monitoring/config/settings.json`:

```json
{
  "apps": {
    "Your_App_Name": {
      "adapter_type": "jsonl",
      "log_file": "/home/.ai_monitoring/logs/Your_App_Name.jsonl",
      "description": "Your App Description"
    }
  }
}
```

---

## 4. Integration Methods

### Method A: Shared Logger (Recommended)

Best for: New apps or apps being updated

**Pros:**
- Automatic cost calculation
- Standardized log format
- Thread-safe writes
- Fallback handling built-in

**Implementation:**

```python
import sys
import time
import anthropic

# Add monitoring library to path
sys.path.insert(0, '/home/.ai_monitoring/lib')
from ai_usage_logger import AIUsageLogger

# Initialize once
client = anthropic.Anthropic()
usage_logger = AIUsageLogger(app_name="My_App")

def call_claude(prompt: str, purpose: str = "api_call") -> str:
    """Call Claude with automatic usage logging."""
    start_time = time.time()

    try:
        response = client.messages.create(
            model="claude-sonnet-4-5-20250929",
            max_tokens=2048,
            messages=[{"role": "user", "content": prompt}]
        )

        latency_ms = int((time.time() - start_time) * 1000)

        # Log successful call
        usage_logger.log_from_response(
            response=response,
            purpose=purpose,
            latency_ms=latency_ms,
            question=prompt[:500]  # Truncate for logging
        )

        return response.content[0].text

    except Exception as e:
        # Log failed call
        usage_logger.log_error(
            error=str(e),
            model="claude-sonnet-4-5-20250929",
            purpose=purpose,
            question=prompt[:500]
        )
        raise
```

### Method B: Custom Adapter (No Code Changes)

Best for: Apps with existing logging that can't be modified

**Available Adapters:**

| Adapter | Use Case | Config Required |
|---------|----------|-----------------|
| `jsonl` | Single JSONL file | `log_file` |
| `daily_rotating` | Daily log files | `log_dir`, `file_pattern` |
| `supabase` | Database logging | `table`, credentials |
| `python_logger` | Python logger output | `log_file` |

**Configuration Example:**

```json
{
  "apps": {
    "Legacy_App": {
      "adapter_type": "daily_rotating",
      "log_dir": "/home/app/logs",
      "file_pattern": "ai_requests_*.jsonl",
      "description": "Legacy app with daily rotating logs"
    }
  }
}
```

---

## 5. Code Examples

### Example 1: Basic Text Query

```python
def process_user_query(question: str) -> str:
    start_time = time.time()

    response = client.messages.create(
        model="claude-sonnet-4-5-20250929",
        max_tokens=2048,
        messages=[{"role": "user", "content": question}]
    )

    usage_logger.log_from_response(
        response=response,
        purpose="user_query",
        latency_ms=int((time.time() - start_time) * 1000),
        question=question
    )

    return response.content[0].text
```

### Example 2: Vision/Image Analysis

```python
def analyze_image(image_data: str, prompt: str) -> str:
    start_time = time.time()

    response = client.messages.create(
        model="claude-sonnet-4-5-20250929",
        max_tokens=2048,
        messages=[{
            "role": "user",
            "content": [
                {"type": "image", "source": {"type": "base64", "media_type": "image/png", "data": image_data}},
                {"type": "text", "text": prompt}
            ]
        }]
    )

    usage_logger.log_from_response(
        response=response,
        purpose="image_analysis",
        latency_ms=int((time.time() - start_time) * 1000),
        question=prompt,
        metadata={"has_image": True}
    )

    return response.content[0].text
```

### Example 3: Extended Thinking (Claude Opus)

```python
def complex_reasoning(question: str) -> str:
    start_time = time.time()

    response = client.messages.create(
        model="claude-opus-4-5-20251101",
        max_tokens=16000,
        thinking={
            "type": "enabled",
            "budget_tokens": 10000
        },
        messages=[{"role": "user", "content": question}]
    )

    # log_from_response automatically handles thinking tokens
    usage_logger.log_from_response(
        response=response,
        purpose="complex_reasoning",
        latency_ms=int((time.time() - start_time) * 1000),
        question=question
    )

    # Extract text from response (skip thinking blocks)
    for block in response.content:
        if hasattr(block, 'text'):
            return block.text
    return ""
```

### Example 4: Manual Logging (When Not Using log_from_response)

```python
def custom_api_call():
    # Your custom API handling
    input_tokens = 1500
    output_tokens = 500

    usage_logger.log_call(
        model="claude-sonnet-4-5-20250929",
        input_tokens=input_tokens,
        output_tokens=output_tokens,
        cache_creation_tokens=0,
        cache_read_tokens=0,
        thinking_tokens=0,
        purpose="custom_call",
        latency_ms=1200,
        success=True,
        question="User's question",
        response_preview="Response preview..."
    )
```

### Example 5: Wrapper Function for Existing Claude Client

If you have an existing `claude_client.py`, add a helper function:

```python
# Add to your existing claude_client.py

import sys
sys.path.insert(0, '/home/.ai_monitoring/lib')

# Try to import logger, but don't fail if unavailable
try:
    from ai_usage_logger import AIUsageLogger
    _usage_logger = AIUsageLogger(app_name="Your_App_Name")
    _logging_enabled = True
except ImportError:
    _logging_enabled = False

def _log_usage(response, purpose: str, latency_ms: int, question: str = None):
    """Log AI usage if monitoring is available."""
    if _logging_enabled:
        try:
            _usage_logger.log_from_response(
                response=response,
                purpose=purpose,
                latency_ms=latency_ms,
                question=question
            )
        except Exception as e:
            # Don't let logging failures break the app
            print(f"[Warning] Failed to log AI usage: {e}")
```

---

## 6. Error Handling Best Practices

### Lesson 1: Never Let Logging Break Your App

```python
# WRONG - Logging failure crashes the app
usage_logger.log_from_response(response, purpose="query")
return response.content[0].text

# CORRECT - Wrap logging in try/except
try:
    usage_logger.log_from_response(response, purpose="query")
except Exception as e:
    print(f"[Warning] Failed to log AI usage: {e}")
return response.content[0].text
```

### Lesson 2: Graceful Degradation When Logger Unavailable

```python
# At module level - don't fail if logger not available
try:
    from ai_usage_logger import AIUsageLogger
    _usage_logger = AIUsageLogger(app_name="My_App")
    _logging_enabled = True
except ImportError:
    _logging_enabled = False
    print("[Warning] AI usage logging not available")

# In your function
def call_api():
    response = client.messages.create(...)

    if _logging_enabled:
        try:
            _usage_logger.log_from_response(response, purpose="api_call")
        except:
            pass  # Silent fail for logging

    return response
```

### Lesson 3: Log Errors Too

```python
try:
    response = client.messages.create(...)
    usage_logger.log_from_response(response, purpose="query")
    return response.content[0].text
except anthropic.APIError as e:
    # Log the failed attempt
    usage_logger.log_error(
        error=str(e),
        model="claude-sonnet-4-5-20250929",
        purpose="query",
        question=prompt
    )
    raise
```

### Lesson 4: Handle Service Restarts

After modifying code, always restart the service:

```bash
# Restart and verify
sudo systemctl restart your-service
sleep 2
sudo systemctl status your-service

# Check logs for errors
journalctl -u your-service -n 50 --no-pager
```

---

## 7. Testing & Verification

### Step 1: Verify Logger Import

```bash
ssh server_0 "python3 -c \"
import sys
sys.path.insert(0, '/home/.ai_monitoring/lib')
from ai_usage_logger import AIUsageLogger
logger = AIUsageLogger(app_name='Test')
print('Logger initialized successfully')
\""
```

### Step 2: Test Log Writing

```bash
# Create test entry
ssh server_0 "python3 -c \"
import sys
sys.path.insert(0, '/home/.ai_monitoring/lib')
from ai_usage_logger import AIUsageLogger
logger = AIUsageLogger(app_name='Test_App')
entry = logger.log_call(
    model='claude-sonnet-4-5-20250929',
    input_tokens=100,
    output_tokens=50,
    purpose='test'
)
print('Logged:', entry)
\""

# Verify log file
ssh server_0 "cat /home/.ai_monitoring/logs/Test_App.jsonl"
```

### Step 3: Preview Daily Report

```bash
ssh server_0 "cd /home/.ai_monitoring && python3 services/daily_report_v2.py --preview"
```

### Step 4: Verify Your App Appears

Check that your app shows in the report output with correct:
- Request count
- Token counts
- Cost calculation

### Step 5: End-to-End Test

1. Trigger an API call in your app
2. Check the log file immediately:
   ```bash
   ssh server_0 "tail -5 /home/.ai_monitoring/logs/Your_App.jsonl"
   ```
3. Verify the entry has correct data

---

## 8. Common Pitfalls & Solutions

### Pitfall 1: Log File Not Created

**Symptom:** No JSONL file appears for your app

**Causes & Solutions:**

| Cause | Solution |
|-------|----------|
| Wrong app_name | Ensure `AIUsageLogger(app_name="X")` matches settings.json |
| Import failed silently | Add print statement to verify import works |
| Permission denied | Check `/home/.ai_monitoring/logs/` is writable |
| Logger initialized but never called | Verify `log_from_response` is actually being called |

### Pitfall 2: Zero Cost in Reports

**Symptom:** App shows in report but cost is $0.00

**Causes & Solutions:**

| Cause | Solution |
|-------|----------|
| Missing token data | Use `log_from_response` which extracts tokens automatically |
| Unknown model | Add model to pricing in `ai_usage_logger.py` |
| Null token values | Ensure tokens are integers, not None |

### Pitfall 3: Daily Rotating Adapter Errors

**Symptom:** `'str' object has no attribute 'get'`

**Cause:** Log format doesn't match expected structure

**Solution:** Check your log format:

```python
# Expected: Single JSON per line
{"timestamp": "...", "model": "...", "tokens": {...}}

# If using multi-line JSON with separator:
# Configure adapter with correct parsing
```

### Pitfall 4: Service Won't Start After Changes

**Symptom:** Service fails immediately after adding logging

**Debug Steps:**

```bash
# 1. Check service status
sudo systemctl status your-service

# 2. Check recent logs
journalctl -u your-service -n 100 --no-pager

# 3. Test Python syntax
python3 -m py_compile /path/to/modified_file.py

# 4. Test import manually
python3 -c "import sys; sys.path.insert(0, '/path/to/app'); from your_module import *"
```

### Pitfall 5: Duplicate Logging

**Symptom:** Same request logged multiple times

**Cause:** Logger initialized multiple times or called in multiple places

**Solution:**
- Initialize logger ONCE at module level
- Call logging in ONE place (preferably right after API response)

### Pitfall 6: Port Conflicts After Restart

**Symptom:** Service fails with "Address already in use"

**Solution:**

```bash
# Find process using the port
sudo lsof -i :YOUR_PORT

# Kill orphaned process
sudo kill PID

# Restart service
sudo systemctl restart your-service
```

---

## 9. Model Pricing Reference

Current pricing per 1 million tokens (as of 2026):

| Model | Input | Output | Cache Write | Cache Read |
|-------|-------|--------|-------------|------------|
| claude-opus-4-5-20251101 | $15.00 | $75.00 | $18.75 | $1.50 |
| claude-sonnet-4-5-20250929 | $3.00 | $15.00 | $3.75 | $0.30 |
| claude-haiku-4-5-20251001 | $0.80 | $4.00 | $1.00 | $0.08 |
| claude-3-opus-20240229 | $15.00 | $75.00 | $18.75 | $1.50 |
| claude-3-haiku-20240307 | $0.25 | $1.25 | $0.30 | $0.03 |

**Note:** Extended thinking tokens are billed at output token rates.

---

## 10. Checklist

### New App Integration Checklist

- [ ] Add `sys.path.insert(0, '/home/.ai_monitoring/lib')` to imports
- [ ] Import `AIUsageLogger` with try/except fallback
- [ ] Initialize logger with correct `app_name`
- [ ] Add `log_from_response()` after ALL API calls
- [ ] Add `log_error()` in exception handlers
- [ ] Wrap logging in try/except to prevent app failures
- [ ] Add app to `/home/.ai_monitoring/config/settings.json`
- [ ] Test with `daily_report_v2.py --preview`
- [ ] Verify logs appear in `/home/.ai_monitoring/logs/`
- [ ] Restart service and verify it starts correctly
- [ ] Test end-to-end with real API call

### Pre-Deployment Verification

- [ ] Logger imports without errors
- [ ] Test API call logs correctly
- [ ] Service restarts cleanly
- [ ] App appears in daily report preview
- [ ] Cost calculation looks correct

### Post-Deployment Monitoring

- [ ] Check daily report includes your app
- [ ] Verify request counts match expectations
- [ ] Monitor for any logging errors in app logs

---

## Quick Reference Commands

```bash
# View app logs
ssh server_0 "tail -20 /home/.ai_monitoring/logs/YOUR_APP.jsonl"

# Preview daily report
ssh server_0 "cd /home/.ai_monitoring && python3 services/daily_report_v2.py --preview"

# Send test report to Teams
ssh server_0 "cd /home/.ai_monitoring && python3 services/daily_report_v2.py"

# Check all monitored apps
ssh server_0 "cat /home/.ai_monitoring/config/settings.json | python3 -c \"import sys,json; print('\\n'.join(json.load(sys.stdin)['apps'].keys()))\""

# Test logger manually
ssh server_0 "python3 -c \"
import sys
sys.path.insert(0, '/home/.ai_monitoring/lib')
from ai_usage_logger import AIUsageLogger
logger = AIUsageLogger(app_name='Test')
print(logger.calculate_cost('claude-sonnet-4-5-20250929', 1000, 500))
\""
```

---

## Support

- **Daily Report Issues**: Check `/home/.ai_monitoring/services/daily_report_v2.py`
- **Logger Issues**: Check `/home/.ai_monitoring/lib/ai_usage_logger.py`
- **Adapter Issues**: Check `/home/.ai_monitoring/lib/log_adapters.py`
- **Configuration**: `/home/.ai_monitoring/config/settings.json`

---

*Last Updated: 2026-02-05*
