# TPPNEO-1843: Implementation Summary - Fixed Contract Validation

## Implementation Completed ✓

All changes have been implemented to block update expiration for devices with fixed contracts.

---

## Files Modified

### 1. Error Code Definition ✓
**File:** `validator/error.py`
- **Line:** ~216 (after INVALID_EXPIRATION_FORMAT)
- **Change:** Added `UPDATE_EXPIRATION_NOT_ALLOWED_FOR_FIXED_CONTRACT` error code

```python
UPDATE_EXPIRATION_NOT_ALLOWED_FOR_FIXED_CONTRACT = (
    "UPDATE_EXPIRATION_NOT_ALLOWED_FOR_FIXED_CONTRACT", 
    "Cannot update expiration for device with fixed contract assigned"
)
```

---

### 2. Workflow Helper (NEW) ✓
**File:** `utils/workflow_helper.py` (NEW FILE)
- **Lines:** 1-108
- **Functions:**
  - `invoke_workflow_service()` - Invoke workflow Lambda
  - `get_device_billing_cycle()` - Query billing cycle data
  - `has_fixed_contract()` - Check if device has fixed contract

**Key Logic:**
```python
def has_fixed_contract(tenant_id: str, device_uid: str) -> bool:
    billing_cycle = get_device_billing_cycle(tenant_id, device_uid)
    if not billing_cycle:
        return False
    
    contract_mode = billing_cycle.get('contract_mode', '').lower()
    is_activating = billing_cycle.get('is_activating_contract', False)
    
    return contract_mode == 'fixed' and is_activating
```

---

### 3. Update Expiration Handler ✓
**File:** `handlers/update_device_expiration.py`

**Changes:**
1. **Line 7** - Import helper:
```python
from utils.workflow_helper import has_fixed_contract
```

2. **Line 22** - Import error code:
```python
UPDATE_EXPIRATION_NOT_ALLOWED_FOR_FIXED_CONTRACT
```

3. **Lines 203-211** - Add validation (after service type check):
```python
# Check if device has fixed contract assigned
if has_fixed_contract(tenant_id, imei):
    raise SystemException(
        message=UPDATE_EXPIRATION_NOT_ALLOWED_FOR_FIXED_CONTRACT,
        properties={
            "device_uid": imei,
            "contract_mode": "fixed"
        }
    )
```

---

### 4. Test Cases ✓
**File:** `tests/test_update_device_expiration.py`
- **Lines:** 315-405 (before `if __name__ == "__main__"`)

**Added 3 Test Cases:**

1. **test_fixed_contract_blocks_update_expiration**
   - Mocks `has_fixed_contract` to return `True`
   - Expects: `UPDATE_EXPIRATION_NOT_ALLOWED_FOR_FIXED_CONTRACT`

2. **test_variable_contract_allows_update_expiration**
   - Mocks `has_fixed_contract` to return `False`
   - Expects: `REQUEST_SUCCESS`

3. **test_no_contract_allows_update_expiration**
   - Mocks `has_fixed_contract` to return `False`
   - Expects: `REQUEST_SUCCESS`

---

## Validation Flow

```
Request → update_device_expiration()
    ↓
1. Feature Flag Check (EnablePrepaidEnhancement) ✓
    ↓
2. Device Validation (exists, not archived, etc.) ✓
    ↓
3. Service Type Check (PREPAID only) ✓
    ↓
4. **NEW: Fixed Contract Check** ✓
    ├─→ has_fixed_contract(tenant_id, imei)
    │    ├─→ invoke_workflow_service()
    │    │    └─→ Lambda: workflow-service
    │    │         └─→ GET /api/v1/device/{device_uid}/billing-cycle
    │    │              └─→ Returns: {contract_mode, is_activating_contract}
    │    └─→ Check: contract_mode === 'fixed' AND is_activating === true
    │
    ├─→ If TRUE: Raise UPDATE_EXPIRATION_NOT_ALLOWED_FOR_FIXED_CONTRACT ❌
    └─→ If FALSE: Continue → Update expiration ✓
```

---

## Test Scenarios

| Scenario | has_fixed_contract() | Expected Result |
|----------|---------------------|-----------------|
| Fixed contract + Active | `True` | BLOCKED ❌ |
| Variable contract + Active | `False` | ALLOWED ✓ |
| No contract | `False` | ALLOWED ✓ |
| Fixed contract + Inactive | `False` | ALLOWED ✓ |
| Workflow service error | `False` (fail-open) | ALLOWED ✓ |

---

## API Response Examples

### Fixed Contract - BLOCKED
**Request:**
```json
{
  "updateExpirationList": [{
    "deviceUid": "868424060000992",
    "expiration": 1759925447
  }]
}
```

**Response:**
```json
{
  "updateExpirationResponseList": [{
    "deviceUid": "868424060000992",
    "resultCode": "UPDATE_EXPIRATION_NOT_ALLOWED_FOR_FIXED_CONTRACT",
    "resultMessage": "Cannot update expiration for device with fixed contract assigned"
  }]
}
```

### Variable Contract - ALLOWED
**Request:**
```json
{
  "updateExpirationList": [{
    "deviceUid": "868424060000993",
    "expiration": 1759925447
  }]
}
```

**Response:**
```json
{
  "updateExpirationResponseList": [{
    "deviceUid": "868424060000993",
    "resultCode": "REQUEST_SUCCESS",
    "resultMessage": "Request is successful"
  }]
}
```

---

## Environment Configuration Required

### Lambda Environment Variables
```bash
WORKFLOW_SERVICE_ARN=arn:aws:lambda:<region>:<account>:function:workflow-service
```

### IAM Permissions
North Bound API Lambda needs permission:
```json
{
  "Effect": "Allow",
  "Action": "lambda:InvokeFunction",
  "Resource": "arn:aws:lambda:*:*:function:workflow-service"
}
```

---

## Testing Commands

### Run Unit Tests
```bash
cd /home/thang/Documents/rsu/alps-ttp3-backend/modules/north_bound_api/north_bound_proxy_v2

# Run all tests
pytest src/tests/test_update_device_expiration.py -v

# Run specific test
pytest src/tests/test_update_device_expiration.py::TestUpdateDeviceExpirationIntegration::test_fixed_contract_blocks_update_expiration -v
```

---

## Deployment Checklist

- [ ] Code review completed
- [ ] Unit tests passing
- [ ] Environment variables configured
- [ ] IAM permissions granted
- [ ] Workflow service API endpoint verified
- [ ] Integration testing with real devices
- [ ] Rollback plan documented
- [ ] Monitoring/logging setup

---

## Known Limitations

1. **Fail-Open Design:** If workflow service is unavailable, validation is skipped (returns `False`) to maintain availability
2. **Lambda Invocation:** Adds ~100-300ms latency per request
3. **Workflow API Dependency:** Requires workflow service to have GET billing cycle endpoint

---

## Next Steps

1. **Verify Workflow Service API**
   - Confirm endpoint exists: `GET /api/v1/device/{device_uid}/billing-cycle`
   - Verify response format matches expected schema

2. **Integration Testing**
   - Test with real fixed contract device
   - Test with real variable contract device
   - Test workflow service error scenarios

3. **Deployment**
   - Deploy to dev environment first
   - Monitor logs for errors
   - Verify API responses
   - Deploy to production

---

## Rollback Procedure

If issues occur in production:

1. **Quick Fix:** Comment out validation block in `update_device_expiration.py` lines 203-211
2. **Redeploy** without validation
3. **Investigate** workflow service integration
4. **Fix** and redeploy

---

## Summary

✓ **UI:** Already blocks fixed contracts (verified earlier)
✓ **North Bound API:** Now blocks fixed contracts (implemented)
⏳ **Bulk Action:** Next phase (not in this ticket scope)

**Status:** Implementation complete, ready for testing & deployment
