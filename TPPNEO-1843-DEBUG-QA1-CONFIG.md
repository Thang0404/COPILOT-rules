# Debug Configuration Update - QA1 Tenant

## Changes Made ✅

Updated debug file to match your QA environment configuration.

---

## Updated Configuration

### Tenant ID
**Before:** `deveco`  
**After:** `/qa1`

### Authorization Token
**Before:** `Bearer test_token_12345`  
**After:** `Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ0ZW5hbnRfaWQiOiIvcWExIiwidXNlcl9pZCI6MjUzLCJleHAiOjE3Njk0MjE1MDh9.c-wAFubzmzy7FSYHMzytRZ0F2xTac--WGj6icKIBdb4`

### User Context
```python
user_obj = SimpleNamespace(
    id=253,
    user_id=253,
    tenant_id='/qa1',
    tenant_name='qa1'
)
```

---

## Test Devices in QA1

### Test Case 1: Fixed Contract (BLOCKED)
- **Device:** `359445338655677`
- **Contract:** Fixed
- **Expiration:** `1769504123` (future)
- **Expected:** ❌ BLOCKED - `UPDATE_EXPIRATION_NOT_ALLOWED_FOR_FIXED_CONTRACT`

### Test Case 2: Variable Contract + Future Date (ALLOWED)
- **Device:** `R8YW20K0YRT`
- **Contract:** Variable
- **Expiration:** `1769504123` (future)
- **Expected:** ✅ ALLOWED - `REQUEST_SUCCESS`

### Test Case 3: Variable Contract + Past Date (BLOCKED)
- **Device:** `350419600000770`
- **Contract:** Variable
- **Expiration:** `current_time - 86400` (yesterday)
- **Expected:** ❌ BLOCKED - `EXPIRATION_CANNOT_BE_IN_PAST`

### Test Case 4: No Contract (ALLOWED)
- **Device:** `355220090419339`
- **Contract:** None
- **Expiration:** `1769504123` (future)
- **Expected:** ✅ ALLOWED - `REQUEST_SUCCESS`

---

## Database Connection

Debug will now connect to:
```python
schema_name = db.get_schema(tenant_id='/qa1', ...)
```

This means it will query devices from the **QA1 schema** in PostgreSQL.

---

## Mock Configuration

Contract info mock remains the same:
```python
CONTRACT_DEVICES = {
    "359445338655677": {"contract_mode": "fixed", "is_activating": True},
    "868424060000992": {"contract_mode": "fixed", "is_activating": True},
    "R8YW20K0YRT": {"contract_mode": "variable", "is_activating": True},
    "350419600000770": {"contract_mode": "variable", "is_activating": True},
}
```

---

## Important Prerequisites

For debug to work, these devices **MUST exist** in QA1 database with:
- ✅ `deleted_at IS NULL` (not archived)
- ✅ `service_enabled = true`
- ✅ `tenant_id = '/qa1'`
- ✅ `service_type = PREPAID`

If any device doesn't meet these conditions, you'll get error **before** contract validation:
- `DEVICE_NOT_FOUND`
- `DEVICE_IS_ARCHIVED`
- `SERVICE_NOT_FOUND`
- `DEVICE_UID_OWNED_BY_OTHER`

---

## Run Debug

```bash
cd /home/thang/Documents/rsu/alps-ttp3-backend/modules/north_bound_api/north_bound_proxy_v2/src
python debug_nb.py
```

---

## Expected Output Format

```
================================================================================

TEST CASE 1: Device WITH Fixed Contract - Should be BLOCKED
================================================================================
[MOCK] get_contract_info called with tenant_id=/qa1, device_uid=359445338655677
[MOCK] Contract info for 359445338655677: {'contract_mode': 'fixed', 'is_activating': True}

[RESULT]
{
  "statusCode": 200,
  "body": {
    "updateExpirationResponseList": [
      {
        "deviceUid": "359445338655677",
        "resultCode": "UPDATE_EXPIRATION_NOT_ALLOWED_FOR_FIXED_CONTRACT",
        "resultMessage": "Cannot update expiration for device with fixed contract assigned"
      }
    ]
  }
}

================================================================================
```

---

## Summary

✅ Updated tenant to `/qa1`  
✅ Updated auth token to match your QA environment  
✅ Updated user context (user_id=253)  
✅ Database queries will use QA1 schema  
✅ All 4 test cases configured with correct devices

Ready to test!
