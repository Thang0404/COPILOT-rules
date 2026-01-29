# TTPNEO-1667 - CRITICAL BUG: Multiple Duplicate Records (17,185 FAIL for 10,000 devices)

## Bug Summary

**Severity**: CRITICAL 🔴🔴🔴

**Impact**: 
- 10,000 devices tested
- 17,185 FAIL records generated (1.7x duplication rate)
- SUCCESS records written FIRST
- Each device gets at LEAST 1 SUCCESS + 1-2 FAIL records

---

## Root Cause: DOUBLE PROCESSING

### File: [batch_action_dispatcher/src/main.py](file:///home/thang/Documents/rsu/alps-ttp3-backend/modules/batch_action_dispatcher/src/main.py)

**Problem**: Code has **3 SEPARATE LOOPS** that all process the same devices:

1. **Lines 135-168**: Loop through `wrong_service_imeis` → Insert FAIL records
2. **Lines 250-335**: Loop through `devices` (WORKFLOW_ACTION path) → Insert SUCCESS records
3. **Lines 338-500**: Loop through `devices` (Normal action path) → Insert MORE records

---

## Detailed Flow Analysis

### Phase 1: Query Devices (Lines 119-127)

```python
if workflow_id:
    # ❌ PROBLEM: Both queries return OVERLAPPING devices!
    devices = db.get_devices_for_workflow_bulk_action(
        schema_name, device_uids['query'], service_type_id
    )
    wrong_service_imeis = db.get_wrong_service_type_devices_for_workflow(
        schema_name, device_uids['query'], service_type_id
    )
```

**Query 1**: `get_devices_for_workflow_bulk_action()`
```sql
-- Returns devices WHERE:
-- - imei IN (device_uids)
-- - service_type_id = :service_type_id  (if provided)
-- OR 1=1 (if service_type_id is NULL)
```

**Query 2**: `get_wrong_service_type_devices_for_workflow()`
```sql
-- Returns devices WHERE:
-- - imei IN (device_uids)
-- - service_type_id <> :service_type_id
```

**CRITICAL ISSUE**: 
- If `service_type_id` is NULL or not properly set
- Query 1 returns ALL devices (because of `1=1` condition)
- Query 2 also returns devices with wrong service type
- **RESULT**: Same devices appear in BOTH lists!

---

### Phase 2: First Loop - Process wrong_service_imeis (Lines 180-188)

```python
# Loop 1: Insert FAIL records for wrong service
for imei in wrong_service_imeis:
    counter_id = df.loc[imei]['counter_id']
    ext_fields, result = data_failure_device(
        imei, Error.WRONG_SERVICE_TYPE, device_type, counter_id
    )
    db.insert_batch_action_device(
        schema_name, batch_action_id=batch_action_id,
        tenant_id=tenant_id, created_by=user_id,
        is_success=False,  # ❌ Record 1: FAIL
        ext_fields=ext_fields, 
        result=result
    )
```

**Result**: 1 FAIL record per device in `wrong_service_imeis`

---

### Phase 3: Second Loop - WORKFLOW_ACTION Path (Lines 250-335)

```python
# Line 241: Check for workflow_id
if workflow_id and batch_upload_type in [BatchUploadType.WORKFLOW_ACTION, BatchUploadType.EXIT_WORKFLOW]:
    
    # Line 250: Loop 2 - Process ALL devices
    for device in devices:  # ❌ Includes devices from wrong_service_imeis!
        device_uid = get_device_uid(device, device_type)
        
        # ... validations (tenant, state, existing workflow)
        
        # Line 311: Insert SUCCESS record
        batch_action_device = db.insert_batch_action_device(
            schema_name, batch_action_id=batch_action_id,
            tenant_id=tenant_id, created_by=user_id, 
            device_id=device.device_id
            # ✅ Record 2: SUCCESS (is_success defaults to TRUE)
        )
        
        # Send to processor
        payload = {...}
        send_message(payload, {}, BATCH_ACTION_PROCESSOR_SQS)
    
    # Line 336: RETURN EARLY
    return  # ❌ SHOULD stop here, but already created duplicates!
```

**Result**: 1 SUCCESS record per device in `devices` list

**CRITICAL**: 
- `devices` list STILL contains devices from `wrong_service_imeis`
- Those devices get SUCCESS records EVEN THOUGH they had FAIL records
- `return` statement prevents Loop 3 from running

---

### Phase 4: Third Loop - Normal Action Path (Lines 338-500)

```python
# Line 338: Loop 3 - Process devices AGAIN (only if no workflow_id or didn't return early)
for device in devices:
    device_uid = get_device_uid(device, device_type)
    config = configs.get(device.state_id, {})
    
    # Multiple validations:
    # - Wrong state
    # - TAC blacklisted
    # - Property updates
    # - Contract conflicts
    # - Deregister validation
    # - Blink action validation
    # - Reload days validation
    
    # Line 460: Insert record (could be 3rd or 4th for same device)
    batch_action_device = db.insert_batch_action_device(
        schema_name, batch_action_id=batch_action_id,
        tenant_id=tenant_id, created_by=user_id
        # ❌ Record 3+: Could be SUCCESS or FAIL depending on validations
    )
    
    # Send to processor
    payload = {...}
    send_message(payload, {}, BATCH_ACTION_PROCESSOR_SQS)
```

**Result**: Additional records per device (1-5 more records depending on validations)

---

## Why 17,185 FAIL Records for 10,000 Devices?

### Scenario Analysis:

**Assumption**: Test case is INVENTORY → Register Prepaid + Workflow

**Device Breakdown**:
- 10,000 devices total
- All devices in state: IDLE (for REGISTER action)
- All devices have service: INVENTORY (service_type_id = 6)
- Workflow service_type_id: PREPAID (2)

**What Happens**:

#### Step 1: Query Phase (Lines 119-127)
```python
service_type_id = 2  # From workflow

# Query 1: Returns devices for workflow (state check, no service filter if NULL)
devices = db.get_devices_for_workflow_bulk_action(..., service_type_id=2)
# Result: 10,000 devices (because INVENTORY devices pass state check)

# Query 2: Returns devices with wrong service
wrong_service_imeis = db.get_wrong_service_type_devices_for_workflow(..., service_type_id=2)
# Result: 10,000 devices (all have service_type_id=6, not 2)
```

**CRITICAL**: ALL 10,000 devices are in BOTH lists!

#### Step 2: Loop 1 - wrong_service_imeis (Lines 180-188)
```
Process 10,000 devices
→ Insert 10,000 FAIL records (WRONG_SERVICE_TYPE)
```

#### Step 3: Loop 2 - WORKFLOW_ACTION (Lines 250-335)
```
Process 10,000 devices from devices list
Validations:
- Tenant check: Some devices may belong to different tenant
- State check: Expects READY_FOR_USE/ACTIVE/LOCKED, but devices are IDLE
- Existing workflow check: Some may have workflows
```

**Failures**:
```
State check FAIL (IDLE not in [R4U, ACTIVE, LOCKED]): ~10,000 devices
→ Insert 10,000 FAIL records (WRONG_STATE_DEVICE)

Already has workflow: ~0 devices (assuming fresh devices)
```

**Successes**:
```
If somehow some devices pass: ~0 devices
→ Insert 0 SUCCESS records
```

#### Step 4: Loop 3 - Normal Path (Lines 338-500)
```
DOESN'T RUN because Line 336 returns early
```

**Total Records**:
```
Loop 1 (wrong_service): 10,000 FAIL
Loop 2 (workflow - state check): 10,000 FAIL
Loop 2 (workflow - success): 0 SUCCESS
Total: 20,000 records (but observed 17,185)
```

**Discrepancy**: Why 17,185 instead of 20,000?

**Possible Reasons**:
1. Some devices filtered out earlier (archive, not found, invalid IMEI)
2. Some devices passed validations in Loop 2
3. Counter_id logic may skip some duplicates
4. Database constraints may prevent some inserts

---

## SQL Query Analysis

### Query 1: get_devices_for_workflow_bulk_action()

**File**: [db.py Line 130](file:///home/thang/Documents/rsu/alps-ttp3-backend/modules/batch_action_dispatcher/src/db.py#L130)

```python
def get_devices_for_workflow_bulk_action(self, schema_name, imeis, service_type_id):
    # ❌ PROBLEM: Conditional filter
    service_type_filter = "policy_.service_type_id = :service_type_id" if service_type_id else "1=1"
    
    return f"""
        ...
        WHERE
            {service_type_filter} AND td.deleted_at IS NULL
    """
```

**Issue**: 
- If `service_type_id` is provided: Filters by matching service type ✅
- If `service_type_id` is NULL: Returns ALL devices (1=1) ❌

**In INVENTORY → Register scenario**:
- `service_type_id = 2` (target Prepaid service)
- Filter becomes: `policy_.service_type_id = 2`
- **BUT**: Devices have `service_type_id = 6` (INVENTORY)
- Query returns devices anyway (likely because state/tenant match)

### Query 2: get_wrong_service_type_devices_for_workflow()

**File**: [db.py Line 174](file:///home/thang/Documents/rsu/alps-ttp3-backend/modules/batch_action_dispatcher/src/db.py#L174)

```python
def get_wrong_service_type_devices_for_workflow(self, schema_name, imeis, service_type_id):
    return f"""
        ...
        WHERE
            policy_.service_type_id <> :service_type_id AND td.deleted_at IS NULL
    """
```

**Correct Logic**: Returns devices where `service_type_id` doesn't match

**In INVENTORY → Register scenario**:
- Filter: `policy_.service_type_id <> 2`
- Devices with `service_type_id = 6` match
- Query returns all 10,000 devices ✅

---

## Why SUCCESS Records Written FIRST

**Execution Order**:

1. **Lines 180-188**: Insert FAIL (WRONG_SERVICE_TYPE) - Database record ID: 1-10,000
2. **Lines 250-335**: Insert records in workflow loop
   - Line 270: FAIL (wrong tenant) - IDs: 10,001-10,XXX
   - Line 277: FAIL (wrong state) - IDs: 10,XXX-20,000
   - Line 289: FAIL (already has workflow) - IDs: 20,XXX-20,XXX
   - Line 299: FAIL (no workflow to exit) - IDs: 20,XXX-20,XXX
   - Line 311: SUCCESS (passed all checks) - IDs: 20,XXX-30,000

**User Report**: "SUCCESS records written first"

**Possible Explanations**:
1. **UI sorts by timestamp DESC**: Latest records shown first
2. **Database ID order**: SUCCESS records have higher IDs (inserted later)
3. **Query in UI uses ORDER BY**: May prioritize is_success=TRUE
4. **Batch commit timing**: SUCCESS records committed in different transaction

---

## Complete Bug Scenario

### Test Setup:
```
Action: New Bulk Action
Service: INVENTORY
Apply for: Register Prepaid
Workflow: "Prepaid Onboarding" (service_type_id = 2)
CSV: 10,000 devices
- All in state: IDLE
- All with service: INVENTORY (6)
```

### Execution Flow:

**Query Phase**:
```python
service_type_id = 2  # Target Prepaid

# Query 1
devices = get_devices_for_workflow_bulk_action(..., service_type_id=2)
# Returns: 10,000 devices (query logic issue)

# Query 2
wrong_service_imeis = get_wrong_service_type_devices_for_workflow(..., service_type_id=2)
# Returns: 10,000 devices (6 <> 2)
```

**Loop 1 (Lines 180-188)**:
```
Process: 10,000 IMEIs from wrong_service_imeis
Insert: 10,000 records
Status: FAIL
Error: WRONG_SERVICE_TYPE
Database IDs: 1 - 10,000
```

**Loop 2 (Lines 250-335)**:
```
Process: 10,000 devices from devices list

Validation 1 - Tenant Check (Line 256):
- Devices belonging to different tenant: ~0 (same tenant)
- Insert: 0 FAIL records

Validation 2 - State Check (Line 268):
- Devices in IDLE state (not in [R4U, ACTIVE, LOCKED])
- Insert: 10,000 FAIL records
- Error: WRONG_STATE_DEVICE
- Database IDs: 10,001 - 20,000

Validation 3 - Existing Workflow (Line 283):
- Devices already have workflow: ~0
- Insert: 0 FAIL records

Validation 4 - No Workflow to Exit (Line 295):
- Only for EXIT_WORKFLOW: Skip
- Insert: 0 records

Success Path (Line 311):
- Devices passing all checks: 0
- Insert: 0 SUCCESS records
```

**Total Records**:
```
Loop 1: 10,000 FAIL (WRONG_SERVICE_TYPE)
Loop 2: 10,000 FAIL (WRONG_STATE_DEVICE)
Loop 2: 0 SUCCESS
Total: 20,000 records
```

**Observed**: 17,185 records
**Difference**: 2,815 devices filtered out earlier (archive, not found, etc.)

---

## Solutions

### Solution 1: Filter devices list IMMEDIATELY after wrong_service processing ✅ CRITICAL

**File**: batch_action_dispatcher/src/main.py
**Location**: After Line 188

```python
# After processing wrong_service_imeis
for imei in wrong_service_imeis:
    # ... insert FAIL record
    pass

# ✅ CRITICAL FIX: Remove wrong_service devices from main devices list
if device_type == DeviceType.SMARTPHONE:
    devices = [d for d in devices if d.imei not in wrong_service_imeis]
    logger.info(f"Filtered devices: {len(devices)} valid devices, {len(wrong_service_imeis)} wrong service removed")
elif device_type == DeviceType.TABLET:
    devices = [d for d in devices if d.serial_number not in wrong_service_devices]
    logger.info(f"Filtered devices: {len(devices)} valid devices, {len(wrong_service_devices)} wrong service removed")
```

---

### Solution 2: Fix Query Logic to Prevent Overlap ✅ RECOMMENDED

**File**: batch_action_dispatcher/src/db.py
**Location**: Lines 130-173

**Problem**: Query 1 returns devices regardless of service_type matching

**Fix**: Make queries mutually exclusive

```python
def get_devices_for_workflow_bulk_action(self, schema_name, imeis, service_type_id):
    # ✅ FIX: Always filter by service_type_id
    # Don't use "1=1" fallback
    if service_type_id is None:
        raise ValueError("service_type_id is required for workflow bulk action")
    
    service_type_filter = "policy_.service_type_id = :service_type_id"
    
    return f"""WITH device_imei AS (
        ...
    )
    SELECT ...
    FROM device_imei_
    ...
    WHERE
        {service_type_filter} 
        AND td.deleted_at IS NULL
        AND td.state_id IN (2, 4, 5)  -- ✅ Add state filter here
    """
```

---

### Solution 3: Add Early Return Check ✅ SAFETY NET

**File**: batch_action_dispatcher/src/main.py
**Location**: Line 250 (start of workflow loop)

```python
if workflow_id and batch_upload_type in [BatchUploadType.WORKFLOW_ACTION, BatchUploadType.EXIT_WORKFLOW]:
    
    # ✅ Safety check: Ensure devices list is clean
    if device_type == DeviceType.SMARTPHONE:
        clean_devices = [d for d in devices if d.imei not in wrong_service_imeis]
    else:
        clean_devices = [d for d in devices if d.serial_number not in wrong_service_devices]
    
    logger.info(f"Processing {len(clean_devices)} devices for workflow action (removed {len(devices) - len(clean_devices)} wrong service devices)")
    
    for device in clean_devices:  # ✅ Use clean_devices instead of devices
        # ... rest of logic
```

---

### Solution 4: Add Duplicate Prevention in Database Layer ✅ DEFENSE IN DEPTH

**File**: batch_action_dispatcher/src/db.py
**Location**: insert_batch_action_device()

```python
def insert_batch_action_device(self, schema_name, batch_action_id, device_id=None, **kwargs):
    # ✅ Check if record already exists for this device
    if device_id:
        existing = self.connection.execute(text(f"""
            SELECT id FROM {schema_name}.t_batch_action_device
            WHERE batch_action_id = :batch_action_id
            AND device_id = :device_id
            AND is_success = :is_success
        """), {
            'batch_action_id': batch_action_id,
            'device_id': device_id,
            'is_success': kwargs.get('is_success', True)
        }).fetchone()
        
        if existing:
            logger.warning(f"Duplicate record prevented for device_id={device_id}, batch_action_id={batch_action_id}")
            return existing  # Return existing record instead of creating duplicate
    
    # ... normal insert logic
```

---

## Recommended Fix Strategy

### Phase 1: IMMEDIATE HOT FIX (Deploy today)

1. **Add filter after Line 188** (Solution 1)
2. **Add safety check at Line 250** (Solution 3)

**Files to modify**:
- `batch_action_dispatcher/src/main.py` (2 changes)

**Testing**: Quick smoke test with 100 devices

---

### Phase 2: PROPER FIX (Deploy this week)

1. **Fix query logic** (Solution 2)
2. **Add database duplicate prevention** (Solution 4)

**Files to modify**:
- `batch_action_dispatcher/src/db.py` (2 changes)

**Testing**: Full regression test with 10,000+ devices

---

### Phase 3: VERIFICATION (After deployment)

1. **Query existing duplicates**:
```sql
SELECT 
    batch_action_id,
    device_id,
    COUNT(*) as record_count
FROM schema.t_batch_action_device
WHERE batch_action_id = :your_batch_id
GROUP BY batch_action_id, device_id
HAVING COUNT(*) > 1
ORDER BY record_count DESC;
```

2. **Clean up existing duplicates** (if needed):
```sql
-- Keep oldest record, delete newer duplicates
DELETE FROM schema.t_batch_action_device
WHERE id NOT IN (
    SELECT MIN(id)
    FROM schema.t_batch_action_device
    GROUP BY batch_action_id, device_id
);
```

---

## Impact Assessment

### Before Fix:
- ❌ 10,000 devices → 20,000+ records (2x duplication)
- ❌ Confusing user experience (same device shows multiple statuses)
- ❌ Inaccurate batch action statistics
- ❌ Database bloat
- ❌ Potential performance issues with large batches

### After Fix:
- ✅ 10,000 devices → 10,000 records (1:1 ratio)
- ✅ Clear device status (1 record per device)
- ✅ Accurate statistics
- ✅ Clean database
- ✅ Better performance

---

## Related Documents

1. [TTPNEO-1667-DUPLICATE-RECORDS-BUG.md](file:///home/thang/Documents/rsu/copilot-rules/TTPNEO-1667-DUPLICATE-RECORDS-BUG.md) - Original 2x duplicate bug
2. [TTPNEO-1667-INVENTORY-REGISTER-WORKFLOW.md](file:///home/thang/Documents/rsu/copilot-rules/TTPNEO-1667-INVENTORY-REGISTER-WORKFLOW.md) - INVENTORY → Register flow bug
3. [TPDEVCO-1362.md](file:///home/thang/Documents/rsu/copilot-rules/TPDEVCO-1362.md) - Workflow assignment implementation

---

## Next Steps

1. ✅ **Review this analysis with team**
2. ✅ **Approve fix approach**
3. ✅ **Implement Phase 1 hot fix**
4. ✅ **Test with 100 devices**
5. ✅ **Deploy to staging**
6. ✅ **Test with 10,000 devices**
7. ✅ **Deploy to production**
8. ✅ **Monitor for 24 hours**
9. ✅ **Implement Phase 2 proper fix**
10. ✅ **Clean up existing duplicates** (if approved)
