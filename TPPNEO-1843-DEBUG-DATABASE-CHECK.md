# Debug Database Check - Confirmation

## Question
Does the debug actually check device status in the database?

---

## Answer: YES ✅

The debug **DOES** query the real PostgreSQL database for device validation.

---

## Evidence

### 1. Database Connection Initialized

**File:** `debug_nb.py` lines 71-73

```python
from db import Dao
db = Dao()
schema_name = db.get_schema(tenant_id='deveco', callback_func=lambda schema: schema.schema_name)
```

This creates a **real database connection** to PostgreSQL.

---

### 2. Handler Uses Real Database Queries

**File:** `update_device_expiration.py` lines 150-183

The handler performs **REAL database queries**:

#### Query 1: Get Device
```python
device = db.get_device_by_imeis(schema_name, imei=str(imei))
if not device:
    raise SystemException(message=DEVICE_NOT_FOUND)
```
**Checks:** Device exists in database

#### Query 2: Check Archived Status
```python
if device.deleted_at is not None:
    raise SystemException(message=DEVICE_IS_ARCHIVED)
```
**Checks:** `deleted_at` column in database

#### Query 3: Check Service Enabled
```python
if not device.service_enabled:
    raise SystemException(message=SERVICE_NOT_FOUND)
```
**Checks:** `service_enabled` column in database

#### Query 4: Check Tenant Ownership
```python
if device.tenant_id not in tenant_ids:
    raise SystemException(message=DEVICE_UID_OWNED_BY_OTHER)

if device.tenant_id != tenant_id or original_tenant_id != tenant_id:
    raise SystemException(message=DEVICE_NOT_ELIGIBLE_FOR_UPDATE_PROPERTY)
```
**Checks:** `tenant_id` column in database

#### Query 5: Get Service Type
```python
service_type_row = db.get_service_type_id_by_imei(schema_name, imei=imei)
service_type_id = service_type_row[0][0] if service_type_row else None
if service_type_id and service_type_id != ServiceType.PREPAID:
    raise ActionException(...)
```
**Checks:** Service type from database (must be PREPAID)

---

### 3. What is MOCKED vs What is REAL

| Component | Status | Purpose |
|-----------|--------|---------|
| **Database (PostgreSQL)** | ✅ REAL | Device validation queries |
| **AWS Lambda Authorization** | 🔶 MOCKED | Skip auth token validation |
| **SQS Queue** | 🔶 MOCKED | Skip actual message sending |
| **Workflow Service Lambda** | 🔶 MOCKED | Return fake contract info |

---

## Validation Flow in Debug

```
Debug Script
    ↓
[REAL] Connect to PostgreSQL Database
    ↓
[REAL] Query: db.get_device_by_imeis()
    ├─→ Check device exists
    ├─→ Check deleted_at IS NULL
    ├─→ Check service_enabled = true
    ├─→ Check tenant_id matches
    └─→ Get device properties
    ↓
[REAL] Query: db.get_service_type_id_by_imei()
    └─→ Check service type = PREPAID
    ↓
[MOCK] get_contract_info() returns fake contract data
    └─→ Returns hardcoded: {contract_mode: 'fixed'/'variable', is_activating: True}
    ↓
[REAL] Validation logic with real device data
    ↓
Result
```

---

## Test Devices Status

To confirm database is used, test with these devices:

### Device 1: 359445338655677
- Must exist in `deveco` schema
- Must NOT be archived (`deleted_at IS NULL`)
- Must have `service_enabled = true`
- Must have `service_type = PREPAID`
- **Mock returns:** Fixed contract

### Device 2: R8YW20K0YRT
- Must exist in `deveco` schema
- Must NOT be archived
- Must have `service_enabled = true`
- Must have `service_type = PREPAID`
- **Mock returns:** Variable contract

### Device 3: 350419600000770
- Must exist in `deveco` schema
- Must NOT be archived
- Must have `service_enabled = true`
- Must have `service_type = PREPAID`
- **Mock returns:** Variable contract

### Device 4: 355220090419339
- Must exist in `deveco` schema
- Must NOT be archived
- Must have `service_enabled = true`
- Must have `service_type = PREPAID`
- **Mock returns:** No contract (None)

---

## What Happens if Device Not in Database?

If any device doesn't exist or fails validation:

```python
device = db.get_device_by_imeis(schema_name, imei=str(imei))
if not device:
    raise SystemException(message=DEVICE_NOT_FOUND)
```

**Result:**
```json
{
  "updateExpirationResponseList": [{
    "deviceUid": "INVALID_DEVICE",
    "resultCode": "DEVICE_NOT_FOUND",
    "resultMessage": "Device not found"
  }]
}
```

---

## Conclusion

✅ **YES**, debug checks real device status in PostgreSQL database

🔶 **Only mocked:** Auth token, SQS, and Workflow Lambda contract info

📝 **All device validations are REAL:**
- Device exists
- Not archived
- Service enabled
- Tenant ownership
- Service type = PREPAID

⚠️ **Important:** Test devices MUST exist in `deveco` schema with valid status, otherwise you'll get `DEVICE_NOT_FOUND` error before reaching contract validation.
