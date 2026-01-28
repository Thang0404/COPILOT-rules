# TPPNEO-1843: Debug Setup Ready

## Files Created/Modified

### 1. workflow_helper.py ✓
**Location:** `utils/workflow_helper.py`

**Functions:**
- `invoke_workflow_service()` - Lambda invocation helper
- `get_device_billing_cycle()` - Get billing cycle from workflow service
- `has_fixed_contract()` - Check if device has fixed contract (returns True/False)

---

### 2. Debug Setup ✓
**File:** `debug_nb.py`

**Mock Added:**
```python
def mock_has_fixed_contract(tenant_id: str, device_uid: str) -> bool:
    print(f"[MOCK] has_fixed_contract called with tenant_id={tenant_id}, device_uid={device_uid}")
    return True  # Change to False to test variable contract scenario
```

**Test Event:**
```python
event = {
    'resource': '/api/v2/device/updateExpiration',
    'path': '/api/v2/device/updateExpiration',
    'httpMethod': 'POST',
    'headers': {
        'accept': 'application/json',
        'Authorization': 'Bearer test_token_12345',
        'content-type': 'application/json',
        'tenantid': 'deveco',
        'User-Agent': 'Mozilla/5.0'
    },
    'body': '''{
    "updateExpirationList": [
        {
            "deviceUid": "868424060000992",
            "expiration": 1759925447
        }
    ]
}''',
    'isBase64Encoded': False
}
```

---

## Expected Results

### Scenario 1: Fixed Contract (mock returns True)
**Expected Response:**
```json
{
  "updateExpirationResponseList": [{
    "deviceUid": "868424060000992",
    "resultCode": "UPDATE_EXPIRATION_NOT_ALLOWED_FOR_FIXED_CONTRACT",
    "resultMessage": "Cannot update expiration for device with fixed contract assigned"
  }]
}
```

### Scenario 2: Variable Contract (mock returns False)
**Expected Response:**
```json
{
  "updateExpirationResponseList": [{
    "deviceUid": "868424060000992",
    "resultCode": "REQUEST_SUCCESS",
    "resultMessage": "Request is successful"
  }]
}
```

---

## Debug Steps

1. **Run debug:**
```bash
cd /home/thang/Documents/rsu/alps-ttp3-backend/modules/north_bound_api/north_bound_proxy_v2/src
python debug_nb.py
```

2. **Test Fixed Contract:**
   - Mock returns `True`
   - Should see error: `UPDATE_EXPIRATION_NOT_ALLOWED_FOR_FIXED_CONTRACT`

3. **Test Variable Contract:**
   - Change mock to return `False`
   - Should succeed with `REQUEST_SUCCESS`

---

## Validation Flow in Code

```
handler(event) 
  ↓
update_device_expiration()
  ↓
Line 186-200: Check service type (PREPAID)
  ↓
Line 203-211: NEW VALIDATION
  ├─→ has_fixed_contract(tenant_id, imei)
  │    └─→ MOCKED: Returns True/False
  │
  ├─→ If True: 
  │    └─→ Raise SystemException(UPDATE_EXPIRATION_NOT_ALLOWED_FOR_FIXED_CONTRACT)
  │
  └─→ If False: 
       └─→ Continue to update expiration
```

---

## Files Modified Summary

| File | Status | Changes |
|------|--------|---------|
| `validator/error.py` | ✓ | Added error code line ~217 |
| `utils/workflow_helper.py` | ✓ | NEW FILE with 3 functions |
| `handlers/update_device_expiration.py` | ✓ | Import + validation at lines 203-211 |
| `tests/test_update_device_expiration.py` | ✓ | 3 new test cases |
| `debug_nb.py` | ✓ | Mock + test event ready |

---

## Mock Behavior

**Current Setup:**
```python
patch('utils.workflow_helper.has_fixed_contract', side_effect=mock_has_fixed_contract)
```

**To Test Different Scenarios:**
1. **Block Update (Fixed):** `return True`
2. **Allow Update (Variable):** `return False`
3. **Allow Update (No Contract):** `return False`

---

## Ready to Debug!

Run command:
```bash
python debug_nb.py
```

Expected output will show:
- Mock function call logs
- API response with either error or success
- JSON formatted result
