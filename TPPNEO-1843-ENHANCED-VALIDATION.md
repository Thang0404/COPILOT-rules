# TPPNEO-1843: Enhanced Validation - Contract Mode + Past Date Check

## Updated Logic

### Validation Flow

```
Request → update_device_expiration()
    ↓
1. Feature Flag Check ✓
    ↓
2. Device Validation ✓
    ↓
3. Service Type Check (PREPAID only) ✓
    ↓
4. **NEW: Contract Validation** ✓
    ├─→ get_contract_info(tenant_id, imei)
    │    ├─→ invoke_workflow_service()
    │    │    └─→ Lambda: workflow-service
    │    │         └─→ GET /api/v1/device/{device_uid}/billing-cycle
    │    │              └─→ Returns: {contract_mode, is_activating_contract}
    │    └─→ Return: {contract_mode, is_activating} if is_activating=true, else None
    │
    ├─→ If contract_mode === 'fixed':
    │    └─→ Raise UPDATE_EXPIRATION_NOT_ALLOWED_FOR_FIXED_CONTRACT ❌
    │
    ├─→ If contract_mode === 'variable':
    │    └─→ Check expiration timestamp
    │         ├─→ If expiration < current_time:
    │         │    └─→ Raise EXPIRATION_CANNOT_BE_IN_PAST ❌
    │         └─→ Else: Continue ✓
    │
    └─→ If no contract: Continue ✓
```

---

## Files Modified

### 1. workflow_helper.py - NEW Function ✓

**Added:** `get_contract_info(tenant_id, device_uid)` at line ~76

**Returns:**
```python
{
    'contract_mode': 'fixed' | 'variable',
    'is_activating': True
}
# or None if no active contract
```

**Logic:**
- Calls `get_device_billing_cycle()`
- Returns contract info ONLY if `is_activating_contract = True`
- Returns `None` if no contract or contract not activating

---

### 2. error.py - NEW Error Code ✓

**Added:** Line ~218
```python
EXPIRATION_CANNOT_BE_IN_PAST = (
    "EXPIRATION_CANNOT_BE_IN_PAST", 
    "Expiration date cannot be in the past for device with variable contract"
)
```

---

### 3. update_device_expiration.py - Enhanced Validation ✓

**Changes:**

**Line 7:** Import updated
```python
from utils.workflow_helper import get_contract_info
```

**Line 25:** Added error code
```python
EXPIRATION_CANNOT_BE_IN_PAST
```

**Lines 201-239:** NEW validation logic
```python
contract_info = get_contract_info(tenant_id, imei)
if contract_info:
    contract_mode = contract_info.get('contract_mode')
    
    # Fixed contract → BLOCK
    if contract_mode == 'fixed':
        raise SystemException(
            message=UPDATE_EXPIRATION_NOT_ALLOWED_FOR_FIXED_CONTRACT,
            properties={"device_uid": imei, "contract_mode": "fixed"}
        )
    
    # Variable contract → CHECK PAST DATE
    elif contract_mode == 'variable':
        import time
        current_timestamp = int(time.time())
        expiration_timestamp = int(expiration)
        
        if expiration_timestamp < current_timestamp:
            raise SystemException(
                message=EXPIRATION_CANNOT_BE_IN_PAST,
                properties={
                    "device_uid": imei,
                    "contract_mode": "variable",
                    "expiration": expiration
                }
            )
```

---

## Test Scenarios

### Debug File: 4 Test Cases

| Test Case | Device UID | Contract Mode | Expiration | Expected Result |
|-----------|-----------|---------------|------------|-----------------|
| 1 | 359445338655677 | **fixed** | Future | **BLOCKED** ❌ |
| 2 | R8YW20K0YRT | **variable** | Future | **ALLOWED** ✓ |
| 3 | 350419600000770 | **variable** | **Past** | **BLOCKED** ❌ |
| 4 | 355220090419339 | None | Future | **ALLOWED** ✓ |

---

## API Response Examples

### Test Case 1: Fixed Contract
**Request:**
```json
{
  "updateExpirationList": [{
    "deviceUid": "359445338655677",
    "expiration": 1769504123
  }]
}
```

**Response:**
```json
{
  "updateExpirationResponseList": [{
    "deviceUid": "359445338655677",
    "resultCode": "UPDATE_EXPIRATION_NOT_ALLOWED_FOR_FIXED_CONTRACT",
    "resultMessage": "Cannot update expiration for device with fixed contract assigned"
  }]
}
```

---

### Test Case 2: Variable Contract + Future Date
**Request:**
```json
{
  "updateExpirationList": [{
    "deviceUid": "R8YW20K0YRT",
    "expiration": 1769504123
  }]
}
```

**Response:**
```json
{
  "updateExpirationResponseList": [{
    "deviceUid": "R8YW20K0YRT",
    "resultCode": "REQUEST_SUCCESS",
    "resultMessage": "Request is successful"
  }]
}
```

---

### Test Case 3: Variable Contract + Past Date
**Request:**
```json
{
  "updateExpirationList": [{
    "deviceUid": "350419600000770",
    "expiration": 1737291600
  }]
}
```

**Response:**
```json
{
  "updateExpirationResponseList": [{
    "deviceUid": "350419600000770",
    "resultCode": "EXPIRATION_CANNOT_BE_IN_PAST",
    "resultMessage": "Expiration date cannot be in the past for device with variable contract"
  }]
}
```

---

### Test Case 4: No Contract
**Request:**
```json
{
  "updateExpirationList": [{
    "deviceUid": "355220090419339",
    "expiration": 1769504123
  }]
}
```

**Response:**
```json
{
  "updateExpirationResponseList": [{
    "deviceUid": "355220090419339",
    "resultCode": "REQUEST_SUCCESS",
    "resultMessage": "Request is successful"
  }]
}
```

---

## Business Rules Summary

| Scenario | Contract Mode | Expiration | Result |
|----------|--------------|------------|--------|
| Has Contract | **FIXED** | Any | ❌ BLOCKED |
| Has Contract | **VARIABLE** | Future | ✅ ALLOWED |
| Has Contract | **VARIABLE** | Past | ❌ BLOCKED |
| No Contract | N/A | Any | ✅ ALLOWED |

---

## Debug Command

```bash
cd /home/thang/Documents/rsu/alps-ttp3-backend/modules/north_bound_api/north_bound_proxy_v2/src
python debug_nb.py 2>&1 | tail -200
```

---

## Key Changes from Previous Version

**Before:**
- Only checked if `has_fixed_contract()` = True → BLOCK
- No validation for variable contracts

**After:**
- Get full contract info: `{contract_mode, is_activating}`
- Fixed contract → BLOCK (same as before)
- **NEW:** Variable contract → Check past date → BLOCK if past
- No contract → ALLOW (same as before)

---

## Summary

✅ Fixed contract → Always blocked
✅ Variable contract → Allowed only for future dates
✅ No contract → Always allowed
✅ Past date validation only applies to variable contracts
