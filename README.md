# Web Portal Retraining Trigger Integration Guide

## 📋 Overview

This guide explains how to integrate automatic retraining triggers into the web portal. When doctors provide feedback that overrides AI predictions, the system can automatically trigger model retraining if the override count exceeds a threshold.

---

## 🎯 Two Approaches

### Approach 1: Simple API Endpoint (Recommended for Demo)

**How it works:**
- Web portal checks override count in database
- If threshold exceeded → calls Vision Inference API endpoint
- Endpoint triggers GitHub Actions workflow for retraining

**Pros:**
- ✅ Simple to implement
- ✅ Immediate triggering
- ✅ Easy to test
- ✅ Good for demo

---

## 🔌 API Endpoint Details

### Endpoint: `POST /trigger-retraining`

**Base URL:** Your Vision Inference API URL (e.g., `https://vision-inference-api-xxx.run.app`)

**Request:**
```json
{
  "reason": "Override threshold exceeded",
  "override_count": 15,
  "threshold": 5
}
```

**Response (Success):**
```json
{
  "success": true,
  "message": "Retraining workflow triggered successfully",
  "reason": "Override threshold exceeded: 15 overrides (threshold: 5)",
  "timestamp": "2024-12-11T12:00:00Z"
}
```

**Response (Error):**
```json
{
  "success": false,
  "error": "GITHUB_TOKEN not set. Cannot trigger retraining workflow."
}
```

---

## 💻 Implementation for Web Portal

### Step 1: Add Function to Check Override Count

Add this function to your web portal backend (e.g., `MedScanPortal/backend/app/api/radiologist.py` or a new monitoring service):

```python
from sqlalchemy import text
from sqlalchemy.orm import Session
from datetime import datetime, timedelta

def check_override_threshold(
    db: Session,
    threshold: int = 5,
    lookback_hours: int = 24
) -> dict:
    """
    Check if override count exceeds threshold.
    
    Args:
        db: Database session
        threshold: Maximum number of overrides allowed
        lookback_hours: Hours to look back for overrides
    
    Returns:
        Dictionary with override count and whether threshold is exceeded
    """
    end_time = datetime.utcnow()
    start_time = end_time - timedelta(hours=lookback_hours)
    
    # Count overrides (full_override or partial_override)
    result = db.execute(text("""
        SELECT COUNT(*) as override_count
        FROM radiologist_feedback
        WHERE feedback_timestamp >= :start_time
          AND feedback_timestamp <= :end_time
          AND feedback_type IN ('full_override', 'partial_override')
    """), {
        'start_time': start_time,
        'end_time': end_time
    })
    
    row = result.fetchone()
    override_count = row.override_count if row else 0
    
    threshold_exceeded = override_count >= threshold
    
    return {
        'override_count': override_count,
        'threshold': threshold,
        'threshold_exceeded': threshold_exceeded,
        'lookback_hours': lookback_hours
    }
```

### Step 2: Add Function to Trigger Retraining

Add this function to call the Vision Inference API:

```python
import requests
import os

def trigger_vision_retraining(
    reason: str,
    override_count: int,
    threshold: int,
    vision_api_url: str = None
) -> dict:
    """
    Call Vision Inference API to trigger retraining.
    
    Args:
        reason: Reason for retraining
        override_count: Number of overrides detected
        threshold: Threshold that was exceeded
        vision_api_url: Vision Inference API URL (or set VISION_API_URL env var)
    
    Returns:
        Response from API
    """
    vision_api_url = vision_api_url or os.getenv(
        'VISION_API_URL',
        'https://vision-inference-api-xxx.run.app'  # Replace with your actual URL
    )
    
    url = f"{vision_api_url}/trigger-retraining"
    
    payload = {
        'reason': reason,
        'override_count': override_count,
        'threshold': threshold
    }
    
    try:
        response = requests.post(url, json=payload, timeout=30)
        response.raise_for_status()
        return response.json()
    except Exception as e:
        logger.error(f"Failed to trigger retraining: {e}")
        return {
            'success': False,
            'error': str(e)
        }
```

### Step 3: Add Background Task or Scheduled Check

**Option A: Check After Each Feedback Submission**

Add to your feedback submission endpoint (`/scans/{scan_id}/feedback`):

```python
@router.post("/scans/{scan_id}/feedback", response_model=FeedbackResponse)
async def submit_feedback(
    scan_id: UUID,
    feedback: FeedbackCreate,
    background_tasks: BackgroundTasks,
    current_user: User = Depends(require_role(["radiologist"])),
    db: Session = Depends(get_db)
):
    """Submit diagnosis feedback."""
    # ... existing feedback submission code ...
    
    # After saving feedback, check override threshold
    override_check = check_override_threshold(db, threshold=10, lookback_hours=24)
    
    if override_check['threshold_exceeded']:
        logger.warning(
            f"Override threshold exceeded: {override_check['override_count']} >= {override_check['threshold']}"
        )
        
        # Trigger retraining in background
        background_tasks.add_task(
            trigger_vision_retraining,
            reason="Override threshold exceeded",
            override_count=override_check['override_count'],
            threshold=override_check['threshold']
        )
    
    return FeedbackResponse(...)
```

**Option B: Scheduled Check (Every Hour)**

Create a scheduled task (using Celery, APScheduler, or similar):

```python
from apscheduler.schedulers.background import BackgroundScheduler

def check_and_trigger_retraining():
    """Scheduled task to check override threshold."""
    from app.core.database import SessionLocal
    
    db = SessionLocal()
    try:
        override_check = check_override_threshold(db, threshold=5, lookback_hours=24)
        
        if override_check['threshold_exceeded']:
            result = trigger_vision_retraining(
                reason="Override threshold exceeded",
                override_count=override_check['override_count'],
                threshold=override_check['threshold']
            )
            
            if result.get('success'):
                logger.info(f"✅ Retraining triggered: {result.get('reason')}")
            else:
                logger.error(f"❌ Failed to trigger retraining: {result.get('error')}")
    finally:
        db.close()

# Schedule to run every hour
scheduler = BackgroundScheduler()
scheduler.add_job(
    check_and_trigger_retraining,
    'interval',
    hours=1
)
scheduler.start()
```

---

## 🔧 Configuration

### Environment Variables

Add to your web portal `.env`:

```bash
# Vision Inference API URL
VISION_API_URL=https://vision-inference-api-xxx.run.app

# Override threshold (optional, defaults to 5)
OVERRIDE_THRESHOLD=5

# Lookback hours (optional, defaults to 24)
OVERRIDE_LOOKBACK_HOURS=24
```

### Threshold Configuration

You can make the threshold configurable:

```python
OVERRIDE_THRESHOLD = int(os.getenv('OVERRIDE_THRESHOLD', '5'))
OVERRIDE_LOOKBACK_HOURS = int(os.getenv('OVERRIDE_LOOKBACK_HOURS', '24'))
```

---

## 📝 Example Integration

### Complete Example: Check After Feedback

```python
@router.post("/scans/{scan_id}/feedback", response_model=FeedbackResponse)
async def submit_feedback(
    scan_id: UUID,
    feedback: FeedbackCreate,
    background_tasks: BackgroundTasks,
    current_user: User = Depends(require_role(["radiologist"])),
    db: Session = Depends(get_db)
):
    """Submit diagnosis feedback."""
    try:
        # ... existing code to save feedback ...
        
        # Check override threshold
        threshold = int(os.getenv('OVERRIDE_THRESHOLD', '5'))
        lookback_hours = int(os.getenv('OVERRIDE_LOOKBACK_HOURS', '24'))
        
        override_check = check_override_threshold(db, threshold, lookback_hours)
        
        if override_check['threshold_exceeded']:
            logger.warning(
                f"⚠️  Override threshold exceeded: "
                f"{override_check['override_count']} overrides in last {lookback_hours}h "
                f"(threshold: {threshold})"
            )
            
            # Trigger retraining in background
            background_tasks.add_task(
                trigger_vision_retraining,
                reason="Override threshold exceeded",
                override_count=override_check['override_count'],
                threshold=override_check['threshold']
            )
        
        return FeedbackResponse(...)
        
    except Exception as e:
        logger.error(f"Error in feedback submission: {e}")
        raise
```

---

## 🧪 Testing

### Test 1: Manual API Call

```bash
curl -X POST https://vision-inference-api-xxx.run.app/trigger-retraining \
  -H "Content-Type: application/json" \
  -d '{
    "reason": "Test retraining trigger",
    "override_count": 15,
    "threshold": 5
  }'
```

### Test 2: Check Override Count

```python
from app.core.database import SessionLocal
from app.api.radiologist import check_override_threshold

db = SessionLocal()
result = check_override_threshold(db, threshold=5, lookback_hours=24)
print(f"Overrides: {result['override_count']}, Threshold: {result['threshold']}")
print(f"Exceeded: {result['threshold_exceeded']}")
```

### Test 3: Trigger from Web Portal

1. Submit feedback with `feedback_type='full_override'` multiple times
2. Check logs to see if threshold check runs
3. Verify GitHub Actions workflow is triggered

---

## 📊 Monitoring

### Log Messages

The system will log:
- `⚠️  Override threshold exceeded: X overrides in last Yh (threshold: Z)`
- `✅ Retraining triggered: Override threshold exceeded: X overrides (threshold: Z)`
- `❌ Failed to trigger retraining: [error]`

### Check GitHub Actions

After triggering, check:
- `https://github.com/rjaditya-2702/MedScan_ai/actions`
- Look for `vision-training.yaml` workflow run
- Check `triggered_by=web_portal` in workflow inputs

---

## ⚙️ Configuration Options

### Adjustable Thresholds

```python
# In your web portal config
OVERRIDE_THRESHOLD = 5        # Number of overrides to trigger retraining
OVERRIDE_LOOKBACK_HOURS = 24   # Hours to look back for overrides
```

### Different Thresholds for Different Feedback Types

```python
def check_override_threshold_detailed(db: Session):
    """Check thresholds for different override types."""
    result = db.execute(text("""
        SELECT 
            feedback_type,
            COUNT(*) as count
        FROM radiologist_feedback
        WHERE feedback_timestamp >= NOW() - INTERVAL '24 hours'
          AND feedback_type IN ('full_override', 'partial_override')
        GROUP BY feedback_type
    """))
    
    full_overrides = 0
    partial_overrides = 0
    
    for row in result:
        if row.feedback_type == 'full_override':
            full_overrides = row.count
        elif row.feedback_type == 'partial_override':
            partial_overrides = row.count
    
    # Different thresholds
    full_threshold = 3   # Full overrides are more serious
    partial_threshold = 8  # Partial overrides are less serious
    
    if full_overrides >= full_threshold:
        return {
            'trigger': True,
            'reason': f'Full override threshold exceeded: {full_overrides} >= {full_threshold}',
            'count': full_overrides,
            'threshold': full_threshold
        }
    
    if partial_overrides >= partial_threshold:
        return {
            'trigger': True,
            'reason': f'Partial override threshold exceeded: {partial_overrides} >= {partial_threshold}',
            'count': partial_overrides,
            'threshold': partial_threshold
        }
    
    return {'trigger': False}
```

---

## 🔐 Security Considerations

### API Authentication

The Vision Inference API endpoint should be protected:

1. **API Key Authentication:**
   ```python
   from fastapi import Header, HTTPException
   
   API_KEY = os.getenv('RETRAINING_API_KEY')
   
   @app.post("/trigger-retraining")
   async def trigger_retraining(
       reason: str,
       api_key: str = Header(..., alias="X-API-Key")
   ):
       if api_key != API_KEY:
           raise HTTPException(status_code=401, detail="Invalid API key")
       # ... rest of endpoint
   ```

2. **Or use existing authentication** if Vision API has it

### GitHub Token Security

- Store `GITHUB_TOKEN` as environment variable
- Never commit to code
- Use GitHub Secrets in Cloud Run

---

## 📋 Summary

**What to send to your friend:**

1. **This documentation file** (`WEB_PORTAL_RETRAINING_INTEGRATION.md`)
2. **Vision Inference API URL** (where the endpoint is deployed)
3. **GitHub Token** (if they need to test directly)
4. **Recommended threshold values** (e.g., 5 overrides in 24 hours)

**What your friend needs to do:**

1. Add `check_override_threshold()` function to web portal backend
2. Add `trigger_vision_retraining()` function to call Vision API
3. Integrate check into feedback submission or scheduled task
4. Configure environment variables
5. Test the integration

**Files created:**
- ✅ `ModelDevelopment/VisionInference/retraining_trigger.py` - Retraining trigger logic
- ✅ `ModelDevelopment/VisionInference/app.py` - Updated with `/trigger-retraining` endpoint
- ✅ `docs/WEB_PORTAL_RETRAINING_INTEGRATION.md` - This documentation

---

## 🚀 Quick Start for Your Friend

1. **Copy the functions** from this doc into web portal backend
2. **Set environment variable:** `VISION_API_URL=https://your-vision-api-url`
3. **Add threshold check** to feedback submission endpoint
4. **Test** by submitting multiple overrides
5. **Verify** GitHub Actions workflow is triggered

That's it! Much simpler than the full monitoring system. 🎉

