# TPPNEO-1843: North Bound API Verification - Update Expiration

## API Endpoint Analysis

### Endpoint Details
- **Path:** `/api/v2/device/updateExpiration`
- **Method:** POST
- **Handler:** [update_device_expiration.py](file:///home/thang/Documents/rsu/alps-ttp3-backend/modules/north_bound_api/north_bound_proxy_v2/src/handlers/update_device_expiration.py)

---

## Current Implementation - Validation Logic

### Entry Point (Lines 51-67)
```python
@authorized
@permission_required
@request_validation(schema=UPDATE_EXPIRATION_SCHEMA, is_restrict_device_type=False)
@event_logger
def update_device_expiration(
    tenant_id, tenant_name, tenant_ids, requests, device_type, **kwargs
):
```

### Existing Validations (Lines 100-200)

#### 1. Feature Flag Check (Lines 100-108)
```python
enabled_prepaid_enhancement = settings.settings.get("Settings", {}).get(
    "EnablePrepaidEnhancement", False
)
if enabled_prepaid_enhancement is False:
    raise HTTPException(
        status_code=HTTPStatus.FORBIDDEN.value,
        message=FEATURE_SYSTEM_UNSUPPORTED,
    )
```
**Status:** ✓ Present

#### 2. Service Type Check (Lines 186-194)
```python
service_type_id = service_type_row[0][0] if service_type_row else None
if service_type_id and service_type_id != ServiceType.PREPAID:
    service_name = ServiceType.get_service_by_id(service_type_id)
    raise ActionException(
        DeviceValidationError.ERR_SERVICE_NOT_SUPPORT,
        DeviceValidationError.error_message(
            DeviceValidationError.ERR_SERVICE_NOT_SUPPORT, service_name
        ),
        {"service_name": service_name}
    )
```
**Status:** ✓ Present - Only PREPAID allowed

#### 3. Other Validations
- Device not found ✓
- Device archived ✓
- Service not enabled ✓
- Tenant ownership ✓
- Expiration format ✓

---

## MISSING VALIDATION: Fixed Contract Check

### Current Gap

**NO validation** to check if device has **fixed contract assigned**

### Search Results
```bash
grep -r "contract|billing|cycle|fixed|variable|is_fixed" update_device_expiration.py
# NO MATCHES ❌
```

**Validation NOT found:**
- No check for `contract_mode`
- No check for `is_fixed` flag
- No query to billing_cycle table
- No block for fixed contracts

---

## Contract Mode Definition

### Backend Constant
File: [workflow constants.py line 215](file:///home/thang/Documents/rsu/alps-ttp3-workflow/src/apiservice/models/constants.py#L215)

```python
class ContractMode:
    VARIABLE = "variable"
    FIX = "fixed"
```

### Usage in Workflow Service
File: [assign_billing_cycle.py line 388](file:///home/thang/Documents/rsu/alps-ttp3-workflow/src/apiservice/controller/assign_billing_cycle.py#L388)

```python
if service_type == ServiceType.PREPAID and billing_cycle_versioning.contract_mode == ContractMode.VARIABLE:
    # Variable contract logic
```

---

## Test Coverage Analysis

### Test File
[test_update_device_expiration.py](file:///home/thang/Documents/rsu/alps-ttp3-backend/modules/north_bound_api/north_bound_proxy_v2/src/tests/test_update_device_expiration.py)

### Existing Tests
1. `test_device_archived_returns_archived_error` ✓
2. `test_device_not_found` ✓
3. `test_expiration_disabled` ✓
4. `test_expiration_iso_datetime_accepted` ✓
5. `test_failed_not_prepaid` ✓

### Missing Test
- **NO TEST** for fixed contract validation ❌

---

## Root Cause

**API allows update expiration for fixed contracts because:**
1. No database query to `device_bill_cycles` table
2. No check for `contract_mode` field
3. No validation against `ContractMode.FIX`

---

## Required Fix - Add Fixed Contract Validation

### Step 1: Query Device Billing Cycle

Need to check if device has active billing cycle with fixed mode:

```python
# After line 186 (service type check)
# Query device billing cycle from workflow MongoDB
device_billing_cycle = db_workflow.get_device_billing_cycle_by_device_uid(
    tenant_id=tenant_id,
    device_uid=imei
)

if device_billing_cycle and device_billing_cycle.contract_mode == ContractMode.FIX:
    raise SystemException(
        message=UPDATE_EXPIRATION_NOT_ALLOWED_FOR_FIXED_CONTRACT,
        properties={
            "device_uid": imei,
            "contract_mode": "fixed"
        }
    )
```

### Step 2: Add Error Code

File: `validator/error.py`

```python
UPDATE_EXPIRATION_NOT_ALLOWED_FOR_FIXED_CONTRACT = (
    "UPDATE_EXPIRATION_NOT_ALLOWED_FOR_FIXED_CONTRACT",
    "Cannot update expiration for device with fixed contract assigned"
)
```

### Step 3: Add Database Helper

Need MongoDB connection to workflow service database to query `device_bill_cycles` collection:

```python
def get_device_billing_cycle_by_device_uid(tenant_id, device_uid):
    """Query device billing cycle from workflow MongoDB"""
    # Implementation needed
    pass
```

---

## Expected Behavior After Fix

### Test Scenario 1: Fixed Contract Assigned

**Request:**
```json
{
  "updateExpirationList": [{
    "deviceUid": "868424060000992",
    "expiration": "2026-12-31T23:59:59Z"
  }]
}
```

**Current Response:**
```json
{
  "updateExpirationResponseList": [{
    "device_uid": "868424060000992",
    "result_code": "REQUEST_SUCCESS",
    "result_message": "Request is successful"
  }]
}
```
**Status:** ✗ WRONG - Should reject

**Expected Response:**
```json
{
  "updateExpirationResponseList": [{
    "device_uid": "868424060000992",
    "result_code": "UPDATE_EXPIRATION_NOT_ALLOWED_FOR_FIXED_CONTRACT",
    "result_message": "Cannot update expiration for device with fixed contract assigned"
  }]
}
```

### Test Scenario 2: Variable Contract Assigned

**Request:**
```json
{
  "updateExpirationList": [{
    "deviceUid": "868424060000993",
    "expiration": "2026-12-31T23:59:59Z"
  }]
}
```

**Expected Response:**
```json
{
  "updateExpirationResponseList": [{
    "device_uid": "868424060000993",
    "result_code": "REQUEST_SUCCESS",
    "result_message": "Request is successful"
  }]
}
```
**Status:** ✓ Should allow

---

## Verification Checklist

| Item | Status | Details |
|------|--------|---------|
| Feature flag check | ✓ Present | EnablePrepaidEnhancement |
| Service type check | ✓ Present | Only PREPAID allowed |
| Contract mode check | ✗ MISSING | No validation for fixed contract |
| Error code defined | ✗ MISSING | Need new error code |
| Test coverage | ✗ MISSING | No test for fixed contract |
| MongoDB connection | ✗ MISSING | Need workflow DB access |

---

## Conclusion

**North Bound API Validation: INCORRECT ❌**

**Missing:**
1. Query to check device billing cycle
2. Validation for `contract_mode === 'fixed'`
3. Error code for fixed contract rejection
4. Test coverage for fixed contract scenario
5. MongoDB workflow database connection

**Impact:** API allows update expiration for fixed contracts, causing inconsistency with UI

**Next Step:** Implement validation to block fixed contracts (requires MongoDB connection to workflow service)
