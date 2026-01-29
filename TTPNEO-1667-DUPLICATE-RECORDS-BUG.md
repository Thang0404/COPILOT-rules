# TTPNEO-1667 - DUPLICATE BATCH ACTION DEVICE RECORDS BUG

## Bug Description

**Symptom**: Cùng 1 device (IMEI: 350031473170554) được ghi 2 records trong `t_batch_action_device`:
- 1 record: is_success = TRUE
- 1 record: is_success = FALSE với error WRONG_SERVICE_TYPE

**Error Message**:
```json
{
  "message": "Please make sure this imei [350031473170554] is with the right Service",
  "error": {
    "code": "WRONG_SERVICE_TYPE",
    "errorMessage": "Please make sure this imei [350031473170554] is with the right Service",
    "properties": {
      "device_type": "imei",
      "device_uid": "350031473170554",
      "action_name": "Register"
    }
  }
}
```

---

## Root Cause Analysis

### File: [batch_action_dispatcher/src/main.py](file:///home/thang/Documents/rsu/alps-ttp3-backend/modules/batch_action_dispatcher/src/main.py#L119)

### Bug Location: Lines 119-188

**Problem**: Device được process 2 lần:
1. **First Pass (Line 119-127)**: Query devices và validate WRONG_SERVICE_TYPE
2. **Second Pass (Line 241-335)**: WORKFLOW_ACTION path xử lý lại cùng devices

---

## Code Analysis

### Phase 1: Initial Device Validation (Line 119-188)

```python
def batch_action_upload(**kawrgs):
    workflow_id = kawrgs.get("workflow_id")
    
    # Line 119: Query devices
    if device_type == DeviceType.SMARTPHONE:
        if workflow_id:
            devices = db.get_devices_for_workflow_bulk_action(
                schema_name, device_uids['query'], service_type_id
            )
            wrong_service_imeis = db.get_wrong_service_type_devices_for_workflow(
                schema_name, device_uids['query'], service_type_id
            )
        else:
            devices = db.get_devices_by_imeis(
                schema_name, device_uids['query'], True, service_type_id
            )
            wrong_service_imeis = db.get_devices_by_imeis(
                schema_name, device_uids['query'], False, service_type_id
            )
        
        # Convert to set
        wrong_service_imeis = set(map(lambda device: device.imei, wrong_service_imeis))
        
        # ... other validations (archive, not found, etc.)
        
        # ❌ Line 180-188: Insert WRONG_SERVICE_TYPE error
        for imei in wrong_service_imeis:
            counter_id = df.loc[imei]['counter_id']
            ext_fields, result = data_failure_device(
                imei, Error.WRONG_SERVICE_TYPE, device_type, counter_id
            )
            db.insert_batch_action_device(
                schema_name, batch_action_id=batch_action_id,
                tenant_id=tenant_id, created_by=user_id,
                is_success=False,  # ❌ FIRST RECORD: FAIL
                ext_fields=ext_fields, 
                result=result
            )
```

**Key Points**:
- Line 122: `get_wrong_service_type_devices_for_workflow()` returns devices with mismatched service_type_id
- Line 180: Loop through `wrong_service_imeis` and insert FAIL records
- **CRITICAL**: Devices in `wrong_service_imeis` are NOT removed from `devices` list!

---

### Phase 2: WORKFLOW_ACTION Processing (Line 241-335)

```python
# Line 241: Check if this is workflow action
if workflow_id and batch_upload_type in [BatchUploadType.WORKFLOW_ACTION, BatchUploadType.EXIT_WORKFLOW]:
    is_exit_workflow = workflow_id == "exit"
    action_id = "exit_solo_workflow" if is_exit_workflow else "assign_solo_workflow"
    
    allowed_states = [StateId.READY_FOR_USE, StateId.ACTIVE, StateId.LOCKED]
    
    # ❌ Line 250: Process ALL devices (includes wrong_service_imeis!)
    for device in devices:
        device_uid = get_device_uid(device, device_type)
        counter_id = df.loc[device_uid]['counter_id']
        
        # Validate tenant
        if device.tenant_id != tenant_id:
            # ... insert error
            continue
        
        # Validate state
        if device.state_id not in allowed_states:
            # ... insert error
            continue
        
        # Check active workflow
        device_workflow_details = get_billing_cycle_device_detail(device_uid, tenant_id)
        has_active_workflow = device_workflow_details and device_workflow_details.get('isActivatingWorkflow') is True
        
        if not is_exit_workflow and has_active_workflow:
            # ... insert error
            continue
        
        if is_exit_workflow and not has_active_workflow:
            # ... insert error
            continue
        
        # ✅ Line 311: Device passed all validations - Insert SUCCESS record
        batch_action_device = db.insert_batch_action_device(
            schema_name, batch_action_id=batch_action_id,
            tenant_id=tenant_id, created_by=user_id, 
            device_id=device.device_id
            # is_success defaults to TRUE  # ❌ SECOND RECORD: SUCCESS
        )
        
        # Build and send payload
        payload = {
            "device_uid": device_uid,
            "action_id": action_id,
            "solo_workflow": {"workflow_id": workflow_id},
            # ... other fields
        }
        
        send_message(payload, {}, BATCH_ACTION_PROCESSOR_SQS)
    
    # Line 336: Return early
    return
```

**Key Points**:
- Line 250: Loops through ALL `devices`
- **CRITICAL BUG**: `devices` list includes devices that were already marked as WRONG_SERVICE_TYPE
- Line 311: Inserts SUCCESS record for the same device
- **Result**: 2 records in database for same device (1 FAIL + 1 SUCCESS)

---

## Why This Happens

### Query Logic Problem

**Line 122**: Query for workflow action:
```python
if workflow_id:
    devices = db.get_devices_for_workflow_bulk_action(
        schema_name, device_uids['query'], service_type_id
    )
    wrong_service_imeis = db.get_wrong_service_type_devices_for_workflow(
        schema_name, device_uids['query'], service_type_id
    )
```

**Problem**:
- `get_devices_for_workflow_bulk_action()` returns devices cho workflow action
- `get_wrong_service_type_devices_for_workflow()` returns devices với wrong service_type
- **BUT**: Có overlap giữa 2 queries!

**Example Scenario**:
- Device IMEI: 350031473170554
- Current service_type_id: 6 (INVENTORY)
- Workflow service_type_id: 2 (PREPAID)
- Device state: READY_FOR_USE

**Query Results**:
```python
# devices list includes this device (state is valid for workflow)
devices = [Device(imei='350031473170554', state_id=2, service_type_id=6)]

# wrong_service_imeis also includes this device (service_type mismatch)
wrong_service_imeis = {'350031473170554'}
```

**Flow**:
1. Line 180: Insert FAIL record (WRONG_SERVICE_TYPE) ✅
2. Line 250: Loop through devices (includes this device) ❌
3. Line 311: Insert SUCCESS record ❌
4. **Result**: 2 records!

---

## Database Queries Analysis

### File: [batch_action_dispatcher/src/db.py](file:///home/thang/Documents/rsu/alps-ttp3-backend/modules/batch_action_dispatcher/src/db.py)

Need to check these 2 queries:

**Query 1**: `get_devices_for_workflow_bulk_action()`
```python
def get_devices_for_workflow_bulk_action(self, schema_name, device_uids, service_type_id):
    # Likely filters by:
    # - imei IN (device_uids)
    # - state_id IN allowed_states (maybe not)
    # - service_type_id = ? (maybe not)
    pass
```

**Query 2**: `get_wrong_service_type_devices_for_workflow()`
```python
def get_wrong_service_type_devices_for_workflow(self, schema_name, device_uids, service_type_id):
    # Filters by:
    # - imei IN (device_uids)
    # - service_type_id != service_type_id
    pass
```

**Problem**: Query 1 không filter by service_type_id correctly, causing overlap!

---

## Solution Approaches

### Solution 1: Remove wrong_service devices from main devices list ✅ RECOMMENDED

**File**: batch_action_dispatcher/src/main.py
**Location**: After Line 188 (after processing wrong_service_imeis)

```python
# Line 180-188: Process wrong_service_imeis
for imei in wrong_service_imeis:
    counter_id = df.loc[imei]['counter_id']
    ext_fields, result = data_failure_device(
        imei, Error.WRONG_SERVICE_TYPE, device_type, counter_id
    )
    db.insert_batch_action_device(
        schema_name, batch_action_id=batch_action_id,
        tenant_id=tenant_id, created_by=user_id,
        is_success=False, ext_fields=ext_fields, result=result
    )

# ✅ NEW: Remove wrong_service devices from main devices list
devices = [device for device in devices if get_device_uid(device, device_type) not in wrong_service_imeis]
logger.info(f"Removed {len(wrong_service_imeis)} devices with wrong service type from processing")
```

**Pros**:
- Simple fix, minimal code changes
- Maintains existing logic flow
- Prevents duplicate records

**Cons**:
- Need to import `get_device_uid` function if not already available

---

### Solution 2: Skip wrong_service devices in workflow loop

**File**: batch_action_dispatcher/src/main.py
**Location**: Line 250 (inside workflow processing loop)

```python
# Line 241: Workflow action processing
if workflow_id and batch_upload_type in [BatchUploadType.WORKFLOW_ACTION, BatchUploadType.EXIT_WORKFLOW]:
    # ... setup code
    
    for device in devices:
        device_uid = get_device_uid(device, device_type)
        
        # ✅ NEW: Skip if already marked as wrong service
        if device_uid in wrong_service_imeis:
            logger.debug(f"Skipping device {device_uid} - already marked as WRONG_SERVICE_TYPE")
            continue
        
        counter_id = df.loc[device_uid]['counter_id']
        
        # ... rest of validation logic
```

**Pros**:
- Explicit check, easy to understand
- Logs which devices are skipped

**Cons**:
- Requires passing `wrong_service_imeis` into scope
- More checks in loop

---

### Solution 3: Fix database queries to be mutually exclusive

**File**: batch_action_dispatcher/src/db.py

**Fix Query 1**: `get_devices_for_workflow_bulk_action()`
```python
def get_devices_for_workflow_bulk_action(self, schema_name, device_uids, service_type_id):
    # Add filter to EXCLUDE wrong service types
    query = f"""
        SELECT * FROM {schema_name}.t_device
        WHERE imei IN ({device_uids})
        AND service_type_id = {service_type_id}  -- ✅ Add this filter
        AND state_id IN (2, 4, 5)  -- Optional: pre-filter by allowed states
    """
    return self.execute(query)
```

**Pros**:
- Fixes root cause at query level
- No overlap between queries
- Better performance (fewer devices returned)

**Cons**:
- Need to verify query logic
- May affect other code paths using same query

---

## Recommended Fix

**Use Solution 1 + Solution 3 together**:

### Step 1: Fix at dispatcher level (immediate fix)

**File**: [batch_action_dispatcher/src/main.py](file:///home/thang/Documents/rsu/alps-ttp3-backend/modules/batch_action_dispatcher/src/main.py#L188)

**After Line 188**, add filter:

```python
for imei in wrong_service_imeis:
    counter_id = df.loc[imei]['counter_id']
    ext_fields, result = data_failure_device(
        imei, Error.WRONG_SERVICE_TYPE, device_type, counter_id
    )
    db.insert_batch_action_device(
        schema_name, batch_action_id=batch_action_id,
        tenant_id=tenant_id, created_by=user_id,
        is_success=False, ext_fields=ext_fields, result=result
    )

# ✅ NEW: Filter out wrong_service devices from main devices list
if device_type == DeviceType.SMARTPHONE:
    devices = [d for d in devices if d.imei not in wrong_service_imeis]
elif device_type == DeviceType.TABLET:
    devices = [d for d in devices if d.serial_number not in wrong_service_devices]

logger.info(f"Filtered devices: {len(devices)} valid, {len(wrong_service_imeis)} wrong service type")
```

### Step 2: Fix database query (proper fix)

**File**: batch_action_dispatcher/src/db.py

Review and fix `get_devices_for_workflow_bulk_action()` query to ensure it doesn't return wrong service type devices.

---

## Testing Plan

### Test Case 1: INVENTORY → Register Prepaid (Device has INVENTORY service)

**Setup**:
- Device IMEI: 350031473170554
- Current state: READY_FOR_USE (valid for workflow)
- Current service_type_id: 6 (INVENTORY)
- Action: Register Prepaid
- Workflow service_type_id: 2 (PREPAID)

**Expected Behavior**:
- Device should be processed by REGISTER action (change service INVENTORY → PREPAID)
- Then assign workflow
- **NO duplicate records**

**Current Bug**:
- Device marked as WRONG_SERVICE_TYPE (service_type = 6, expected = 2)
- Device also processed by workflow action (SUCCESS)
- **Result**: 2 records (1 FAIL + 1 SUCCESS)

**After Fix**:
- Device filtered out from workflow processing
- Only 1 record: FAIL with WRONG_SERVICE_TYPE (correct behavior for this scenario)

---

### Test Case 2: Direct Workflow Assignment (Device already has correct service)

**Setup**:
- Device IMEI: 123456789012345
- Current state: READY_FOR_USE
- Current service_type_id: 2 (PREPAID) 
- Action: Assign Workflow (WORKFLOW_ACTION)
- Workflow service_type_id: 2 (PREPAID)

**Expected Behavior**:
- Device processed successfully
- 1 record: SUCCESS
- Workflow assigned

**Current Behavior**: Works correctly (no duplicate)

**After Fix**: Still works correctly

---

### Test Case 3: Mixed Devices (Some valid, some wrong service)

**Setup**:
- Device 1: service_type = 2 (PREPAID) ✅ Valid
- Device 2: service_type = 6 (INVENTORY) ❌ Wrong service
- Device 3: service_type = 2 (PREPAID) ✅ Valid
- Workflow service_type_id: 2 (PREPAID)

**Expected Behavior**:
- Device 1: SUCCESS (1 record)
- Device 2: FAIL - WRONG_SERVICE_TYPE (1 record)
- Device 3: SUCCESS (1 record)
- **Total: 3 records, no duplicates**

**Current Bug**:
- Device 1: SUCCESS (1 record) ✅
- Device 2: FAIL + SUCCESS (2 records) ❌
- Device 3: SUCCESS (1 record) ✅
- **Total: 4 records, 1 duplicate**

**After Fix**:
- Device 1: SUCCESS (1 record) ✅
- Device 2: FAIL only (1 record) ✅
- Device 3: SUCCESS (1 record) ✅
- **Total: 3 records, no duplicates**

---

## Impact Analysis

### Current Impact

**Database**:
- Duplicate records in `t_batch_action_device`
- Incorrect success/failure counts
- Confusing for users (same device shows both success and fail)

**User Experience**:
- Action History shows mixed results
- Unclear which status is correct
- May trigger unnecessary retries

**Data Integrity**:
- Batch action statistics incorrect
- Reporting/analytics affected

---

### After Fix

**Database**:
- Clean records, no duplicates
- Correct success/failure counts

**User Experience**:
- Clear status for each device
- Accurate action history

**Data Integrity**:
- Correct batch action statistics
- Reliable reporting

---

## Related Issues

### Issue 1: INVENTORY → Register with Workflow Flow

**Reference**: [TTPNEO-1667-INVENTORY-REGISTER-WORKFLOW.md](file:///home/thang/Documents/rsu/copilot-rules/TTPNEO-1667-INVENTORY-REGISTER-WORKFLOW.md)

**Related Bug**: Dispatcher takes wrong path for INVENTORY → Register + Workflow
- Should use REGISTER path (normal action flow)
- Currently takes WORKFLOW_ACTION path (direct assignment)
- Causes IDLE devices to be rejected with WRONG_STATE_DEVICE

**Connection**: Both bugs occur in same code section (Line 241-335)

---

### Issue 2: Service Type Validation Logic

**Problem**: Overlapping queries return same devices
- `get_devices_for_workflow_bulk_action()` vs `get_wrong_service_type_devices_for_workflow()`
- Need clear separation between valid and invalid devices

---

## Files to Modify

### Primary Fix:
1. [batch_action_dispatcher/src/main.py](file:///home/thang/Documents/rsu/alps-ttp3-backend/modules/batch_action_dispatcher/src/main.py#L188)
   - Add filter after Line 188
   - Remove wrong_service devices from main devices list

### Secondary Fix (Recommended):
2. [batch_action_dispatcher/src/db.py](file:///home/thang/Documents/rsu/alps-ttp3-backend/modules/batch_action_dispatcher/src/db.py)
   - Review `get_devices_for_workflow_bulk_action()` query
   - Ensure mutually exclusive with `get_wrong_service_type_devices_for_workflow()`

### Testing:
3. [batch_action_dispatcher/src/tests/](file:///home/thang/Documents/rsu/alps-ttp3-backend/modules/batch_action_dispatcher/src/tests/)
   - Add test for duplicate prevention
   - Test mixed valid/invalid devices
   - Test INVENTORY → Register scenarios

---

## Next Steps

1. ✅ **Immediate Fix**: Add filter at Line 188 in main.py
2. **Review Queries**: Check db.py query logic
3. **Test Thoroughly**: All 3 test cases above
4. **Verify Database**: Check existing duplicate records
5. **Document**: Update flow diagrams with fix
6. **Deploy**: Roll out fix to all environments
