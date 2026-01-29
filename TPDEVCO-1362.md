# TPDEVCO-1362: Bulk Workflow Assignment

## Quick Overview
Bulk workflow assignment uses constant `action_id = "assign_solo_workflow"` (not from database). Validates device state [2,4,5] and service_type [1,2,3] in dispatcher, then sends to trigger via processor.

---

## Flow: Single vs Bulk

| Component | Single Flow | Bulk Flow |
|-----------|-------------|-----------|
| **Frontend** | workflowId | workflowId + CSV |
| **action_resolver** | `action_id = "assign_solo_workflow"` → TRIGGER_SQS | `batch_### Testing Scenarios

#### Valid Cases (Should Success)
- ✅ CSV chứa devices state READY_FOR_USE + service_type SIM_CONTROL
- ✅ CSV chứa devices state ACTIVE + service_type PREPAID
- ✅ CSV chứa devices state LOCKED + service_type POSTPAID
- ✅ CSV mix READY_FOR_USE + ACTIVE + LOCKED (all valid service types)

#### Invalid Cases (Should Fail with Errors)
- ❌ Device state IDLE → Error: WRONG_STATE_DEVICE
- ❌ Device state ENROLLED → Error: WRONG_STATE_DEVICE
- ❌ Device state RELEASED → Error: WRONG_STATE_DEVICE
- ❌ Device service_type khác 1,2,3 → Error: INVALID_SERVICE_TYPE
- ❌ Device không tồn tại → Error: DEVICE_NOT_FOUND

---

## Latest Fix: service_type_id Flow from FE to Database

### Problem
User discovered that `ext_fields` in `t_batch_action` had `service_type_id: null` when preparing bulk workflow action. This caused dispatcher to not filter devices correctly.

### Root Cause Analysis
The `service_type_id` wasn't flowing through the complete pipeline:

1. **Frontend Issue**: `prepareBatchAction` mutation wasn't sending `serviceTypeId` for WORKFLOW_ACTION
2. **Resolver Issue**: `prepare_batch_action` function only added `service_type_id` to `ext_fields` when `sim_control_action` existed

### Implementation (3 files)

#### 1. Frontend - BulkAction/index.tsx (2 changes)

**Change 1: Remove service_type_id from config object (Line 551-567)**

**Before**:
```typescript
const formatPrepareAction = () => {
  const newState = [] as BatchActionStates[];
  if (
    bulkActionType === BulkActionType.WORKFLOW_ACTION || 
    // ... other types
  ) {
    return {
      service_type_id: null,  // ❌ PROBLEM: Overrides serviceTypeId parameter
      states: newState,
    };
  }
  // ...
}
```

**After**:
```typescript
const formatPrepareAction = () => {
  const newState = [] as BatchActionStates[];
  if (
    bulkActionType === BulkActionType.WORKFLOW_ACTION || 
    // ... other types
  ) {
    // ✅ FIX: Don't include service_type_id in config
    return {
      states: newState,  // Only states, no service_type_id
    };
  }
  // ...
}
```

**Purpose**: Config object should NOT contain `service_type_id` because it's sent as a separate mutation parameter

**Change 2: Add serviceTypeId parameter (Line 730-732)**

**Change 2: Add serviceTypeId parameter (Line 730-732)**

**Before**:
```typescript
const prepareResult = await prepareAction({
  variables: {
    // ... other fields ...
    // ❌ No serviceTypeId for WORKFLOW_ACTION
  },
});
```

**After** (Line 730-732):
```typescript
const prepareResult = await prepareAction({
  variables: {
    // ... other fields ...
    ...(bulkActionType === BulkActionType.SIM_CONTROL_ACTION && {
      serviceTypeId: parseInt(values.service),
    }),
    // ✅ NEW: Add serviceTypeId for WORKFLOW_ACTION
    ...(bulkActionType === BulkActionType.WORKFLOW_ACTION && {
      serviceTypeId: selectedWorkflow.service_type_id,
    }),
  },
});
```

**Purpose**: Send workflow's `service_type_id` as a **separate mutation parameter**, not inside config object

#### 2. Backend - action_resolver/src/main.py (Line 711-717)

**Before**:
```python
ext_fields = {**config, "total_devices": row_count, "device_type": device_type, "brand_id": brand_id, "provision_type": provision_type}

if assign_action_id:
    ext_fields["assign_action_id"] = assign_action_id

if sim_control_action:
    # ❌ service_type_id only added for sim_control_action
    sim_control_ext_fields = {
        "sim_control_action": sim_control_action,
        "sim_control_rule_id": sim_control_rule_id,
        "service_type_id": service_type_id,
    }
    ext_fields = {**ext_fields, **sim_control_ext_fields}
```

**After** (Line 714-717):
```python
ext_fields = {**config, "total_devices": row_count, "device_type": device_type, "brand_id": brand_id, "provision_type": provision_type}

if assign_action_id:
    ext_fields["assign_action_id"] = assign_action_id

# ✅ NEW: Add service_type_id to ext_fields if provided (for workflow or sim control actions)
if service_type_id is not None:
    ext_fields["service_type_id"] = service_type_id

if sim_control_action:
    sim_control_ext_fields = {
        "sim_control_action": sim_control_action,
        "sim_control_rule_id": sim_control_rule_id,
        "service_type_id": service_type_id,  # Redundant but kept for backwards compatibility
    }
    ext_fields = {**ext_fields, **sim_control_ext_fields}
```

**Purpose**: Save `service_type_id` to `ext_fields` for all action types, not just sim_control

#### 3. Backend - batch_action_dispatcher/src/main.py (Already Fixed Earlier)

**Priority Logic** (Line 97-102):
```python
# Priority: kwargs > ext_fields
service_type_id = kwargs.get('service_type_id') or batch_action.ext_fields.get('service_type_id')

# Use service_type_id for query filtering
query_service_type_id = service_type_id  # Always filter by service_type
```

**Purpose**: 
- Read `service_type_id` from `ext_fields` (saved by resolver)
- Fallback to `kwargs` if provided by `finalizeBatchAction`
- Use for filtering devices by service type

### Complete Flow

**Step 1: Frontend sends prepareBatchAction**
```json
{
  "workflowId": "68a5ae7a4cb9e87b88b7d344",
  "serviceTypeId": 1  // ✅ NEW: Workflow's service_type_id
}
```

**Step 2: Resolver saves to ext_fields**
```python
# prepare_batch_action receives serviceTypeId=1 from FE
service_type_id = kwargs.get("service_type_id")  # 1

ext_fields = {
    "service_type_id": 1,  # ✅ NEW: Saved to ext_fields
    "total_devices": 2,
    "device_type": 1,
    # ... other fields
}

# Insert to t_batch_action
conn.insert_batch_action(
    schema_name,
    tenant_id=tenant_id,
    created_by=user.id,
    upload_channel="PORTAL",
    ext_fields=json.dumps(ext_fields),  # ✅ service_type_id saved
    file_url=file_upload['fields']['key']
)
```

**Step 3: Dispatcher reads from ext_fields**
```python
def execute_batch_action(**kwargs):
    batch_action = db.get_batch_action_by_id(schema_name, batch_action_id)
    
    # Priority: kwargs > ext_fields
    service_type_id = kwargs.get('service_type_id') or batch_action.ext_fields.get('service_type_id')
    # service_type_id = 1 (from ext_fields)
    
    # Filter devices by service_type_id
    query_service_type_id = service_type_id
    
    if device_type == DeviceType.SMARTPHONE:
        devices = db.get_devices_by_imeis(
            schema_name, 
            device_uids['query'], 
            True, 
            service_type_id=query_service_type_id  # ✅ Filter by service_type_id=1
        )
```

**Step 4: Validation catches mismatches**
```python
# wrong_service query in DB only returns devices where:
# device.service_type_id != query_service_type_id

# If workflow has service_type_id=1 (SIM_CONTROL)
# Only devices with service_type_id=1 will be in devices['query']
# Devices with service_type_id=2,3,etc will be in devices['wrong_service_imeis']

for device in devices:
    # Only devices matching service_type_id=1 reach here
    # Assign workflow to matching devices
```

### Database Schema

**t_batch_action.ext_fields** (Before fix):
```json
{
  "service_type_id": null,  // ❌ Missing
  "states": [],
  "total_devices": 2,
  "device_type": 1
}
```

**t_batch_action.ext_fields** (After fix):
```json
{
  "service_type_id": 1,  // ✅ Saved from prepareBatchAction
  "states": [],
  "total_devices": 2,
  "device_type": 1
}
```

### Why This Fix is Critical

**Without service_type_id in ext_fields**:
- Dispatcher reads `service_type_id = None`
- Sets `query_service_type_id = None` 
- Device query **doesn't filter by service_type**
- Returns ALL devices regardless of service type
- wrong_service validation catches mismatches
- But devices already in wrong_service_imeis list get error `WRONG_SERVICE_TYPE`

**With service_type_id in ext_fields**:
- Dispatcher reads `service_type_id = 1` (from ext_fields)
- Sets `query_service_type_id = 1`
- Device query **filters by service_type_id=1**
- Only returns devices matching workflow's service type
- wrong_service_imeis is empty (no mismatches)
- All devices proceed to workflow assignment

### Testing Verification

**Before Fix**:
```bash
# Debug dispatcher
python debug_dispatcher.py

# Output: ext_fields has service_type_id: null
# Devices show WRONG_SERVICE_TYPE error
```

**After Fix**:
```bash
# 1. Frontend sends serviceTypeId in prepareBatchAction ✅
# 2. Check t_batch_action.ext_fields in DB
SELECT ext_fields FROM t_batch_action WHERE id = 1974;
# Expected: {"service_type_id": 1, ...}

# 3. Debug dispatcher
python debug_dispatcher.py
# Expected: query_service_type_id = 1
# Expected: devices only contain service_type_id=1 devices
# Expected: wrong_service_imeis is empty

# 4. Check CloudWatch logs
# action_resolver: "prepare_batch_action with kwargs: {..., 'serviceTypeId': 1}"
# dispatcher: "service_type_id from ext_fields: 1"
# trigger: "Assign solo workflow to device"
```

### Summary

**Problem**: 
1. `formatPrepareAction()` was returning `service_type_id: null` inside config object
2. This config was sent as the `$config` parameter in GraphQL mutation
3. Backend resolver was reading from **config object** (`kwargs.get('config')`) instead of **serviceTypeId parameter** (`kwargs.get('serviceTypeId')`)
4. Result: `ext_fields` always had `service_type_id: null` even though FE sent `serviceTypeId: 1` as parameter

**Root Cause**: 
- Config object (`$config: BatchAction`) and serviceTypeId parameter (`$serviceTypeId: Int`) are **different parameters** in GraphQL mutation
- Resolver was spreading config: `ext_fields = {**config, ...}` which included `service_type_id: null`
- This **overrode** the separate `service_type_id` that should come from `kwargs.get('serviceTypeId')`

**Solution**: Two-part fix
1. ✅ **Frontend**: Remove `service_type_id` from config object in `formatPrepareAction()`
2. ✅ **Frontend**: Keep `serviceTypeId` as separate parameter (already added)
3. ✅ **Backend**: Read from `kwargs.get('serviceTypeId')` and save to ext_fields (already added)

**Result**: Clean separation between config and serviceTypeId parameter

**Files Modified**: 2 files (3 changes total)
1. `alps-ttp3-frontend/src/components/BulkAction/index.tsx` 
   - Line 551-567: Removed `service_type_id: null` from config
   - Line 730-732: Added `serviceTypeId` parameter for WORKFLOW_ACTION
2. `alps-ttp3-backend/modules/action_resolver/src/main.py` 
   - Line 714-717: Read `serviceTypeId` parameter and save to ext_fields

**Status**: ✅ Complete - Ready for testing




````"WORKFLOW_ACTION"` → DISPATCHER_SQS |
| **dispatcher** | — | Validate + `action_id = "assign_solo_workflow"` → PROCESSOR_SQS |
| **processor** | — | Forward → TRIGGER_SQS |
| **trigger** | `if action_id == "assign_solo_workflow"` → Call workflow API | Same |

**Key**: Both converge at trigger with same `action_id`!

---## Key Database Tables

### t_action
- Stores REGISTER actions for each service_type_id
- Columns: `id`, `name`, `actiontype_id`, `service_type_id`, `from_state_id`
- Example:
  ```
  id: "360b7474-49ed-4757-8b8c-4370e64c6596"
  name: "Register"
  actiontype_id: "REGISTER"
  service_type_id: 1  (SIM_CONTROL)
  from_state_id: 1
  ```

### t_batch_action
- Stores batch action metadata
- Columns: `id`, `tenant_id`, `ext_fields`, `dispatched`
- ext_fields format:
  ```json
  {
    "states": [
      {
        "state_id": 1,
        "action_id": "360b7474-49ed-4757-8b8c-4370e64c6596",
        "options": {}
      }
    ],
    "service_type_id": 1,
    "total_devices": 2,
    "action_type": "Workflow Assignment"
  }
  ```

### t_batch_action_device
- Stores per-device processing status
- Columns: `id`, `batch_action_id`, `device_id`, `is_success`, `result`
- Updated by processor after each device processed

### t_device
- Device information
- Columns: `id`, `uid`, `state_id`, `service_type_id`, `tenant_id`
- Updated by action_trigger after workflow assigned

---

## Key Design Decisions

### 1. Two-tier Action Resolution
- **User-specified**: Use `assignActionId` if provided
- **Auto-find**: Query REGISTER action by `service_type_id` if not provided
- Reason: Flexibility for both manual and automatic assignment

### 2. Don't Filter Devices by Service Type for Workflow
- When `workflow_id` present: `query_service_type_id = None`
- Reason: Workflows can be assigned across different service types
- `service_type_id` only used to find correct REGISTER action

### 3. Derive service_type_ids from Action (not payload)
- `apply_service_type_ids = None` for workflow assignment
- Processor derives from `action.service_type_id`
- Reason: Different from normal bulk actions which send explicit service types

### 4. Workflow Payload Format
- Must send `"workflow": {}` (empty dict), NOT `null`
- Reason: dto_handler in action_trigger expects dict for `values.get('workflow', {}).get('next_due')`
- If `null`, crashes with `'NoneType' object has no attribute 'get'`

---

## Critical Bug Fixes

### Bug 1: Frontend Constant Wrong
- **Problem**: `WORKFLOW_ACTION = 'ASSIGN_ACTION'`
- **Fix**: Changed to `WORKFLOW_ACTION = 'WORKFLOW_ACTION'`
- **File**: `src/constants/device.ts`

### Bug 2: Constant Not Defined in Dispatcher
- **Problem**: `BatchUploadType.WORKFLOW_ACTION` not defined
- **Fix**: Added to `batch_action_dispatcher/src/const.py`

### Bug 3: Hardcoded batch_upload_type
- **Problem**: Always sent "ASSIGN_ACTION" instead of "WORKFLOW_ACTION"
- **Fix**: Extract from kwargs and pass through
- **File**: `batch_action_dispatcher/src/main.py`

### Bug 4: Service Type Filtering
- **Problem**: Devices not found because filtered by wrong service_type_id
- **Fix**: Set `query_service_type_id = None` when workflow present
- **File**: `batch_action_dispatcher/src/main.py`

### Bug 5: NoneType Iteration Error
- **Problem**: `service_type_ids = kwargs['apply_service_type_ids']` got None
- **Fix**: Check if None, derive from action or device
- **File**: `batch_action_processor/src/actions/type/register.py`

### Bug 6: Workflow Null Payload
- **Problem**: Sent `"workflow": null`, crashed dto_handler
- **Fix**: `workflow = kwargs.get("workflow") or {}` → always send dict
- **File**: `batch_action_processor/src/actions/type/register.py`

### Bug 7: convert_to_desired_services NoneType
- **Problem**: Tried to iterate over None
- **Fix**: Added null check at start of function
- **File**: `batch_action_processor/src/actions/action.py`

---

## Files Modified (10 files)

### Frontend (1 file)
1. `src/constants/device.ts` - Fixed WORKFLOW_ACTION constant

### Backend GraphQL (1 file)
2. `appsync/main.graphql` - Added serviceTypeId, assignActionId parameters

### action_resolver (3 files)
3. `modules/action_resolver/src/const.py` - Added WORKFLOW_ACTION constant
4. `modules/action_resolver/src/db.py` - Added get_register_action_by_service_type_id()
5. `modules/action_resolver/src/main.py` - Implemented auto-find REGISTER action logic

### batch_action_dispatcher (2 files)
6. `modules/batch_action_dispatcher/src/const.py` - Added WORKFLOW_ACTION constant
7. `modules/batch_action_dispatcher/src/main.py` - Added handler, fixed service type filtering

### batch_action_processor (2 files)
8. `modules/batch_action_processor/src/actions/type/register.py` - Fixed null handling for workflow and service_type_ids
9. `modules/batch_action_processor/src/actions/action.py` - Fixed convert_to_desired_services() null check

### action_trigger (1 file)
10. `modules/action_trigger/src/handlers/message_handler.py` - Fixed workflow.message_template_id access (optional)

---

## Testing Flow

### Local Testing
1. **Dispatcher**: Use `debug_dispatcher.py` with `event_dispatcher.json`
2. **Processor**: Use `debug_processor.py` with `event_processor.json`
3. **Trigger**: Use `debug_trigger.py` with `event.json`

### AWS Testing
1. Upload CSV with device UIDs
2. Select service type (SIM_CONTROL/PREPAID/POSTPAID)
3. Select workflow
4. Monitor CloudWatch logs:
   - action_resolver: "Auto-found REGISTER action"
   - dispatcher: "process_device" for each device
   - processor: Payload with solo_workflow
   - trigger: "Assign solo workflow to device"

### Verification
- Check `t_batch_action.ext_fields` has states array
- Check `t_batch_action_device` all devices processed
- Check workflow service: Device has assigned workflow
- Check device state remains IDLE (REGISTER doesn't change state)

---

## Payload Examples

### Frontend → action_resolver
```json
{
  "serviceTypeId": 1,
  "workflowId": "68a5ae7a4cb9e87b88b7d344"
}
```

### action_resolver → dispatcher
```json
{
  "batch_upload_type": "WORKFLOW_ACTION",
  "batch_action_id": 1974,
  "workflow_id": "68a5ae7a4cb9e87b88b7d344"
}
```

### dispatcher → processor (per device)
```json
{
  "device_uid": "f4057468931f154fb47a9acf...",
  "action_id": "360b7474-49ed-4757-8b8c-4370e64c6596",
  "solo_workflow": {"workflow_id": "68a5ae7a4cb9e87b88b7d344"},
  "apply_service_type_ids": null,
  "batch_action_device_id": 10787
}
```

### processor → trigger
```json
{
  "device_uid": "f4057468931f154fb47a9acf...",
  "action_id": "360b7474-49ed-4757-8b8c-4370e64c6596",
  "workflow": {},
  "solo_workflow": {"workflow_id": "68a5ae7a4cb9e87b88b7d344"},
  "trigger_type": "API",
  "channel": "PORTAL"
}
```

---

## Notes

- Bulk flow has extra step through dispatcher/processor vs single flow going direct to trigger
- Both flows end up at action_trigger with same payload format
- Key difference: Bulk needs to process multiple devices, single is one-shot
- States config in ext_fields allows different actions per device state
- Service type filtering disabled for workflow to support cross-service assignment

---

---

## REDESIGN: Use Special Action ID "assign_solo_workflow"

### Overview
Instead of using REGISTER action from t_action table, workflow assignment now uses a **special constant action_id** that doesn't exist in database. This simplifies the flow and makes it consistent with single device workflow assignment.

### Key Changes from Original Design

**BEFORE (Old Design)**:
- action_resolver auto-finds REGISTER action by service_type_id
- Creates states config with REGISTER action_id
- Dispatcher reads states config from ext_fields
- Processor uses Register.handle() logic

**AFTER (New Design)**:
- action_resolver DOES NOT create states config
- Only forwards `batch_upload_type: "WORKFLOW_ACTION"` to dispatcher
- Dispatcher creates constant `action_id = "assign_solo_workflow"`
- Processor forwards payload to trigger
- Trigger handles special action_id directly

### Flow Comparison: Single vs Bulk

| Step | Single Flow | Bulk Flow |
|------|-------------|-----------|
| **Frontend** | Send workflowId | Send workflowId + CSV |
| **action_resolver** | Create payload with `action_id = "assign_solo_workflow"` | Forward `batch_upload_type = "WORKFLOW_ACTION"` |
| **Dispatcher** | ❌ No dispatcher | Validate state/service_type, create `action_id = "assign_solo_workflow"` |
| **Processor** | ❌ No processor | Forward payload to trigger |
| **Trigger** | Check `action_id == "assign_solo_workflow"` → Call workflow API | Same as single |

**Both flows converge at trigger with identical action_id!**

---

## Business Rules & Validations

## Business Rules

### Allowed States & Service Types

**States** (3 allowed):
- ✅ `2` READY_FOR_USE - Registered, not checked in
- ✅ `4` ACTIVE - Checked in, stable
- ✅ `5` LOCKED - Active but locked

**Service Types** (3 allowed):
- ✅ `1` SIM_CONTROL
- ✅ `2` PREPAID  
- ✅ `3` POSTPAID

**Rejected States**:
- ❌ `1` IDLE - Not registered
- ❌ `3` ENROLLED - Syncing MongoDB (unstable)
- ❌ `6` RELEASED

### Error Codes
- `WRONG_STATE_DEVICE` - State not in [2,4,5]
- `WRONG_SERVICE_TYPE` - Service type not in [1,2,3]
- `DEVICE_NOT_FOUND` - Device doesn't exist

---

## Implementation Guide

### Step 1: action_resolver (Remove old logic)

**File**: `modules/action_resolver/src/main.py`

**Change**:
```python
# OLD: Create states config with REGISTER action
# NEW: Just forward batch_upload_type

if batch_upload_type == BatchUploadType.WORKFLOW_ACTION:
    workflow_id = kwargs.get('workflow_id')
    
    # Send to dispatcher - NO states config needed
    message = {
        "batch_upload_type": "WORKFLOW_ACTION",
        "batch_action_id": batch_action_id,
        "workflow_id": workflow_id
    }
    send_to_dispatcher(message)
```

---

### Step 2: batch_action_dispatcher (Main logic)

**File**: `modules/batch_action_dispatcher/src/main.py`

**Add handler in `batch_action_upload()`**:

```python
def batch_action_upload(**kwargs):
    # ... existing code ...
    
    workflow_id = kwargs.get("workflow_id")
    
    # NEW: Check if this is workflow assignment
    if workflow_id:
        action_id = "assign_solo_workflow"  # Constant
        allowed_states = [State.READY_FOR_USE, State.ACTIVE, State.LOCKED]  # [2, 4, 5]
        allowed_service_types = [ServiceType.SIM_CONTROL, ServiceType.PREPAID, ServiceType.POSTPAID]  # [1, 2, 3]
        
        # Extract all fields from resolver payload
        user_id = kwargs.get("user_id")
        tenant_id = kwargs.get("tenant_id")
        batch_action_id = kwargs.get("batch_action_id")
        batch_upload_type = kwargs.get("batch_upload_type")
        billing_detail = kwargs.get("billing_detail")
        sim_control = kwargs.get("sim_control")
        duration = kwargs.get("duration")
        blink_frequency = kwargs.get("blink_frequency")
        blink_time_unit = kwargs.get("blink_time_unit")
        
        # Query devices (don't filter by service_type)
        devices = db.get_devices_by_imeis(schema_name, device_uids['query'], True, service_type_id=None)
        
        for device in devices:
            device_uid = device.imei
            counter_id = df.loc[device_uid]['counter_id']
            
            # Validate state
            if device.state_id not in allowed_states:
                ext_fields, result = data_failure_device(
                    device_uid, Error.WRONG_STATE_DEVICE, device_type, counter_id
                )
                db.insert_batch_action_device(
                    schema_name, batch_action_id, tenant_id, user_id,
                    is_success=False, ext_fields=ext_fields, result=result, device_id=device.device_id
                )
                continue
            
            # Validate service type
            if device.service_type_id not in allowed_service_types:
                ext_fields, result = data_failure_device(
                    device_uid, Error.WRONG_SERVICE_TYPE, device_type, counter_id
                )
                db.insert_batch_action_device(
                    schema_name, batch_action_id, tenant_id, user_id,
                    is_success=False, ext_fields=ext_fields, result=result, device_id=device.device_id
                )
                continue
            
            # Valid device - send to processor
            batch_action_device = db.insert_batch_action_device(
                schema_name, batch_action_id, tenant_id, user_id
            )
            
            # Keep ALL fields from resolver + add new fields
            payload = {
                "device_uid": device_uid,
                "user_id": user_id,
                "tenant_id": tenant_id,
                "batch_action_id": batch_action_id,
                "batch_upload_type": batch_upload_type,
                "billing_detail": billing_detail,
                "sim_control": sim_control,
                "duration": duration,
                "workflow_id": workflow_id,
                "blink_frequency": blink_frequency,
                "blink_time_unit": blink_time_unit,
                "action_id": action_id,  # NEW: Add constant
                "solo_workflow": {"workflow_id": workflow_id},  # NEW: Add solo_workflow
                "batch_action_device_id": batch_action_device.id  # NEW: Add batch_action_device_id
            }
            
            send_message(payload, {}, BATCH_ACTION_PROCESSOR_SQS)
        
        return  # Done for workflow
    
    # ... existing code for other actions ...
```

**Helper functions available**:
- ✅ `data_failure_device()` - Creates error ext_fields/result
- ✅ `db.insert_batch_action_device()` - Inserts to DB
- ✅ `Error.WRONG_STATE_DEVICE`, `Error.WRONG_SERVICE_TYPE` - Constants

**Key Points**:
- ✅ Keep ALL fields from resolver payload
- ✅ Only validate state_id [2,4,5] and service_type_id [1,2,3]
- ✅ Add 3 new fields: action_id, solo_workflow, batch_action_device_id
- ✅ Forward complete payload to processor

---

### Step 3: batch_action_processor (No changes needed)

Existing processor should forward payload correctly to trigger.

**Processor receives from dispatcher**:
```json
{
  "device_uid": "f4057468931f154fb47a9acf...",
  "user_id": 123,
  "tenant_id": "tenant-uuid",
  "batch_action_id": 456,
  "batch_upload_type": "WORKFLOW_ACTION",
  "billing_detail": null,
  "sim_control": null,
  "duration": null,
  "workflow_id": "68a5ae7a4cb9e87b88b7d344",
  "blink_frequency": null,
  "blink_time_unit": null,
  "action_id": "assign_solo_workflow",
  "solo_workflow": {"workflow_id": "68a5ae7a4cb9e87b88b7d344"},
  "batch_action_device_id": 10787
}
```

**Processor forwards to trigger**: Same payload (unchanged)

---

### Step 4: action_trigger (Handle special action_id)

**File**: `modules/action_trigger/src/handlers/message_handler.py`

**Add handler after line 58**:

```python
def handle_message(conn, message, **kwargs):
    payload_dto = PayloadDTO(**message)
    
    # ... existing validation (schema, device, etc.) ...
    
    # NEW: Handle assign_solo_workflow
    if payload_dto.action_id == "assign_solo_workflow":
        logger.info("Assign solo workflow to device (bulk)")
        
        if payload_dto.solo_workflow and payload_dto.solo_workflow.workflow_id:
            workflow_handler.assign_solo_workflow_to_device(
                tenant_id=schema.tenant_id,
                username=payload_dto.username,
                device=device,
                action=None,  # No database action
                device_uid=device.uid,
                workflow_id=payload_dto.solo_workflow.workflow_id,
                schema_name=schema.name,
                device_handler=device_handler
            )
        return  # Done
    
    # ... existing code for other actions ...
```

**Payload received from processor**:
```json
{
  "device_uid": "f4057468931f154fb47a9acf...",
  "user_id": 123,
  "tenant_id": "tenant-uuid",
  "batch_action_id": 456,
  "batch_upload_type": "WORKFLOW_ACTION",
  "billing_detail": null,
  "sim_control": null,
  "duration": null,
  "workflow_id": "68a5ae7a4cb9e87b88b7d344",
  "blink_frequency": null,
  "blink_time_unit": null,
  "action_id": "assign_solo_workflow",
  "solo_workflow": {"workflow_id": "68a5ae7a4cb9e87b88b7d344"},
  "batch_action_device_id": 10787
}
```

**Trigger uses**:
- ✅ `action_id` - To route to correct handler
- ✅ `solo_workflow.workflow_id` - To assign workflow
- ✅ `tenant_id`, `device_uid` - For workflow API call
- ❌ Other fields ignored (not needed for workflow assignment)

---

## Testing Scenarios

### Valid Cases (Should Success)
- ✅ Device state=2, service_type=1
- ✅ Device state=4, service_type=2
- ✅ Device state=5, service_type=3
- ✅ Mix states [2,4,5], all valid service types

### Invalid Cases (Should Fail)
- ❌ State=1 → WRONG_STATE_DEVICE
- ❌ State=3 → WRONG_STATE_DEVICE  
- ❌ State=6 → WRONG_STATE_DEVICE
- ❌ service_type=4 → WRONG_SERVICE_TYPE
- ❌ Device not exist → DEVICE_NOT_FOUND

---

## Summary

**Total files modified**: 3
1. ✅ **action_resolver/src/main.py** - Removed states config creation
2. ✅ **batch_action_dispatcher/src/main.py** - Added workflow validation logic with early return
3. ✅ **action_trigger/src/handlers/message_handler.py** - Handle `assign_solo_workflow`

**Frontend changes**: 1 file
4. ✅ **BulkAction/index.tsx** - Removed serviceTypeId from finalizeBatchAction

**Key implementation points**:
- ✅ Use constant `action_id = "assign_solo_workflow"`
- ✅ Validate in dispatcher: states [2,4,5] using `StateId` constants
- ✅ Validate in dispatcher: service_types [1,2,3] using `ServiceType` constants
- ✅ Keep ALL fields from resolver payload in dispatcher
- ✅ Add 3 new fields: action_id, solo_workflow, batch_action_device_id
- ✅ Early return in dispatcher after processing workflow to skip normal flow
- ✅ Both single and bulk flows use same action_id at trigger
- ✅ No database query for action needed
- ✅ Processor forwards payload unchanged (no modifications needed)

### Allowed Service Types

Chỉ cho phép workflow assignment cho **3 service types**:

1. **SIM_CONTROL (service_type_id = 1)**
2. **PREPAID (service_type_id = 2)**
3. **POSTPAID (service_type_id = 3)**

Devices thuộc service types khác sẽ bị **REJECTED** với error code `INVALID_SERVICE_TYPE`.

### Validation Points

#### 1. Backend - batch_action_dispatcher

**Purpose**: Validate device eligibility and create constant action_id

**Logic**:
```python
if batch_upload_type == BatchUploadType.WORKFLOW_ACTION:
    workflow_id = message.get('workflow_id')
    
    # Define constant action_id (NOT from database)
    action_id = "assign_solo_workflow"
    
    # Hardcode allowed states
    allowed_states = [State.READY_FOR_USE, State.ACTIVE, State.LOCKED]  # [2, 4, 5]
    
    # Hardcode allowed service types
    allowed_service_types = [ServiceType.SIM_CONTROL, ServiceType.PREPAID, ServiceType.POSTPAID]  # [1, 2, 3]
    
    # Query devices (DON'T filter by service_type)
    devices = db.get_devices_by_imeis(schema_name, device_uids, service_type_id=None)
    
    for device in devices:
        # Validate state
        if device.state_id not in allowed_states:
            ext_fields, result = data_failure_device(device.uid, Error.WRONG_STATE_DEVICE, device_type)
            db.insert_batch_action_device(
                schema_name, batch_action_id, tenant_id, user_id,
                is_success=False, ext_fields=ext_fields, result=result, device_id=device.id
            )
            continue
        
        # Validate service type
        if device.service_type_id not in allowed_service_types:
            ext_fields, result = data_failure_device(device.uid, Error.WRONG_SERVICE_TYPE, device_type)
            db.insert_batch_action_device(
                schema_name, batch_action_id, tenant_id, user_id,
                is_success=False, ext_fields=ext_fields, result=result, device_id=device.id
            )
            continue
        
        # Device valid - Send to processor
        batch_action_device = db.insert_batch_action_device(
            schema_name, batch_action_id, tenant_id, user_id
        )
        
        payload = {
            "device_uid": device.uid,
            "tenant_id": tenant_id,
            "user_id": user_id,
            "action_id": action_id,  # "assign_solo_workflow"
            "solo_workflow": {"workflow_id": workflow_id},
            "batch_action_device_id": batch_action_device.id
            # NOTE: batch_upload_type NOT needed in processor/trigger
        }
        
        send_message(payload, {}, BATCH_ACTION_PROCESSOR_SQS)
```

**Available Helper Functions**:
- ✅ `data_failure_device(device_uid, error_code, device_type, counter_id)` - Creates error ext_fields & result
- ✅ `db.insert_batch_action_device(...)` - Inserts to t_batch_action_device
- ✅ Error constants: `Error.WRONG_STATE_DEVICE`, `Error.WRONG_SERVICE_TYPE`

#### 2. Backend - batch_action_processor

**Purpose**: Forward payload to trigger (minimal processing)

**Logic**:
```python
# Processor just forwards to trigger
# No special handling needed for "assign_solo_workflow"
# Existing processor logic should work
```

#### 3. Backend - action_trigger

**Purpose**: Handle special action_id and call workflow service

**Logic**:
```python
def handle_message(conn, message, **kwargs):
    payload_dto = PayloadDTO(**message)
    
    # ... existing validation ...
    
    # NEW: Handle assign_solo_workflow action
    if payload_dto.action_id == "assign_solo_workflow":
        logger.info("Handle bulk workflow assignment (assign_solo_workflow)")
        
        if payload_dto.solo_workflow and payload_dto.solo_workflow.workflow_id:
            workflow_handler.assign_solo_workflow_to_device(
                tenant_id=schema.tenant_id,
                username=payload_dto.username,
                device=device,
                action=None,  # No action from database
                device_uid=device.uid,
                workflow_id=payload_dto.solo_workflow.workflow_id,
                schema_name=schema.name,
                device_handler=device_handler
            )
        return  # Done
    
    # ... existing flow for other actions ...
```

**Note**: `workflow_handler.assign_solo_workflow_to_device()` does NOT use `action` parameter, so passing `None` is safe.

### Implementation Summary

**Files Modified** (4 total):

1. **Frontend - BulkAction/index.tsx**:
   - ❌ Removed `serviceTypeId` from finalizeBatchAction for WORKFLOW_ACTION
   - ✅ Keep `workflowId` unchanged
   - ✅ Added comment explaining dispatcher handles validation

2. **action_resolver/src/main.py**:
   - ❌ Removed entire states config creation block (lines 825-877)
   - ✅ Added comment: "For WORKFLOW_ACTION, no states config creation needed"
   - ✅ Simplified to just forward `batch_upload_type` and `workflow_id` to dispatcher

3. **batch_action_dispatcher/src/main.py**:
   - ✅ Added special workflow handler **BEFORE** normal device loop (after line 216)
   - ✅ Validates state [2, 4, 5] using `StateId.READY_FOR_USE, StateId.ACTIVE, StateId.LOCKED`
   - ✅ Validates service_type [1, 2, 3] using `ServiceType.SIM_CONTROL, ServiceType.PREPAID, ServiceType.POSTPAID`
   - ✅ Creates `action_id = "assign_solo_workflow"` constant
   - ✅ Keeps ALL fields from resolver: user_id, tenant_id, batch_action_id, batch_upload_type, billing_detail, sim_control, duration, workflow_id, blink_frequency, blink_time_unit
   - ✅ Adds 3 new fields: action_id, solo_workflow, batch_action_device_id
   - ✅ Sends complete payload to BATCH_ACTION_PROCESSOR_SQS
   - ✅ **Returns early** to skip normal flow (critical!)

4. **action_trigger/src/handlers/message_handler.py**:
   - ✅ Added handler for `action_id == "assign_solo_workflow"` after line 145
   - ✅ Calls `workflow_handler.assign_solo_workflow_to_device()` with `action=None`
   - ✅ Returns after handling (no further processing)

**Processor** (NO CHANGES):
   - ✅ Existing logic forwards payload unchanged to trigger
   - ✅ No modifications needed

### Why This Design is Better

1. **Simpler**: No database query for REGISTER action
2. **Consistent**: Both single and bulk use same `action_id = "assign_solo_workflow"`
3. **Independent**: Workflow assignment doesn't depend on service_type_id
4. **Maintainable**: Logic centralized in dispatcher and trigger

---

### Error Handling (Unchanged)

User sẽ thấy kết quả chi tiết trong **Action History** sau khi bulk action hoàn thành:

**Success devices**: Devices đã assign workflow thành công
- State: READY_FOR_USE, ACTIVE, hoặc LOCKED
- Service type: SIM_CONTROL, PREPAID, hoặc POSTPAID

**Failed devices** với error codes:
- `WRONG_STATE_DEVICE`: Device ở state IDLE, ENROLLED, hoặc RELEASED
- `INVALID_SERVICE_TYPE`: Device không thuộc 3 service types cho phép
- `DEVICE_NOT_FOUND`: Device không tồn tại trong hệ thống

**Frontend notification**: Popup thông báo "Actions will be now applied to all eligible devices..."
- Không block UI
- User check chi tiết trong Action History (Reference No: 1986)

### Testing Scenarios

#### Valid Cases (Should Success)
- ✅ CSV chứa devices state READY_FOR_USE + service_type SIM_CONTROL
- ✅ CSV chứa devices state ACTIVE + service_type PREPAID
- ✅ CSV chứa devices state LOCKED + service_type POSTPAID
- ✅ CSV mix READY_FOR_USE + ACTIVE + LOCKED (all valid service types)

#### Invalid Cases (Should Fail with Errors)
- ❌ Device state IDLE → Error: WRONG_STATE_DEVICE
- ❌ Device state ENROLLED → Error: WRONG_STATE_DEVICE
- ❌ Device state RELEASED → Error: WRONG_STATE_DEVICE
- ❌ Device service_type khác 1,2,3 → Error: INVALID_SERVICE_TYPE
- ❌ Device không tồn tại → Error: DEVICE_NOT_FOUND

```````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````
```````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````
```````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````
```````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````
```````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````
```````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````
```````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````
```````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````
```````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````
```````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````
```````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````
```````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````
```````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````
```````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````
```````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````
```````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````
```````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````
```````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````
```````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````
```````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````
```````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````
```````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````

### flow chung, bám sát vào, cấm mày sửa khúc này
Frontend → finalizeBatchAction(exitWorkflowReason)
  ↓
  resolver
  ↓
Dispatcher → exit_solo_workflow
  ↓
Processor → handle_workflow_exit
  ↓
Trigger → remove_workflow_and_contract_from_device(reason)
  ↓
Workflow Service → DELETE /device/{id}/remove_workflow_and_contract_from_device?reason=xxx

---

## FIX: Bulk REGISTER with Workflow Assignment for SIM_CONTROL

### Problem Statement
When user performs Bulk Register for SIM_CONTROL service and selects a workflow to assign:
- Device successfully registers (state: IDLE → READY_FOR_USE) ✅
- But workflow is NOT assigned to device ❌

### Root Cause
**File**: `batch_action_dispatcher/src/main.py` (line 439-457)

Normal REGISTER flow doesn't include `solo_workflow` field in payload sent to processor, even when `workflow_id` exists in kwargs from resolver.

**Before fix**:
```python
payload = {
    "device_uid": device_uid,
    "action_id": action_id,
    "billing_detail": billing_detail,
    # ... other fields ...
    # ❌ MISSING: solo_workflow field
}
```

### Solution
Add `solo_workflow` field to payload when `workflow_id` is present in kwargs (similar to WORKFLOW_ACTION flow).

**After fix** (line 458-462):
```python
# Add workflow assignment if workflow_id provided (for REGISTER with workflow)
workflow_id = kawrgs.get("workflow_id")
if workflow_id and workflow_id != "exit":
    payload["solo_workflow"] = {"workflow_id": workflow_id}
```

### Complete Flow: REGISTER + Workflow Assignment

#### Phase 1: Frontend → action_resolver
- User selects **Register Device** bulk action
- Selects service type (SIM_CONTROL/PREPAID/POSTPAID)
- Selects workflow from dropdown
- Uploads CSV with device UIDs in IDLE state

#### Phase 2: action_resolver → Dispatcher
- `prepareBatchAction` saves workflow_id to ext_fields
- `finalizeBatchAction` sends to dispatcher with workflow_id in payload

#### Phase 3: Dispatcher → Processor (per device)
- Validates device state = IDLE (required for REGISTER)
- Builds payload with action_id from states config
- **NEW**: Adds `solo_workflow: {workflow_id}` to payload
- Sends to PROCESSOR_SQS

#### Phase 4: Processor → Trigger
- Processor reads `solo_workflow` from payload (line 94)
- Adds to trigger payload if exists (line 211-212)
- Forwards to ACTION_TRIGGER_SQS

#### Phase 5: Trigger → Workflow Service
- Trigger processes REGISTER action
- If `solo_workflow` exists, calls workflow API to assign workflow
- Device state: IDLE → READY_FOR_USE
- Workflow assigned to device

### Files Modified
1. **batch_action_dispatcher/src/main.py** (line 458-462)
   - Added logic to include `solo_workflow` in payload when `workflow_id` present

### Supported Service Types
This fix works for all 3 service types that support workflow assignment:
- ✅ SIM_CONTROL (service_type_id = 1)
- ✅ PREPAID (service_type_id = 2)
- ✅ POSTPAID (service_type_id = 3)

### Testing Scenarios

#### Valid Cases (Should Success)
- ✅ Bulk Register IDLE devices to SIM_CONTROL + assign workflow
- ✅ Bulk Register IDLE devices to PREPAID + assign workflow
- ✅ Bulk Register IDLE devices to POSTPAID + assign workflow
- ✅ Workflow should appear in device details after registration
- ✅ Device state should change: IDLE → READY_FOR_USE

#### Invalid Cases (Should Fail)
- ❌ Device not in IDLE state → Error: WRONG_STATE_DEVICE
- ❌ Workflow belongs to different service type → Workflow not assigned
- ❌ Device already has workflow → Follow existing workflow logic

### Comparison: WORKFLOW_ACTION vs REGISTER with Workflow

| Aspect | WORKFLOW_ACTION | REGISTER with Workflow |
|--------|----------------|------------------------|
| **Device State** | Must be [2,4,5] (READY_FOR_USE, ACTIVE, LOCKED) | Must be [1] (IDLE) |
| **Action ID** | Constant: "assign_solo_workflow" | From database: t_action (REGISTER) |
| **Workflow Field** | `solo_workflow: {workflow_id}` | `solo_workflow: {workflow_id}` (same) |
| **State Transition** | No state change | IDLE → READY_FOR_USE |
| **Primary Purpose** | Assign workflow only | Register device + assign workflow |
| **Validation** | Dispatcher validates state + service_type | Normal REGISTER validation |

### Key Implementation Notes
- ✅ Processor already supports `solo_workflow` field (no changes needed)
- ✅ Trigger already handles workflow assignment via `solo_workflow` (no changes needed)
- ✅ Only dispatcher needed modification to pass field through
- ✅ Condition `workflow_id != "exit"` prevents conflict with exit workflow flow
- ✅ Uses same workflow assignment logic as WORKFLOW_ACTION

---

## BULK ASSIGN SOLO WORKFLOW - Detailed 6-Phase Flow

This section tracks the happy path where a CSV-driven bulk job assigns a solo workflow to every eligible device. It mirrors the exit flow that calls `workflow_handler.remove_workflow_and_contract_from_device()` (documented below) but funnels into `workflow_handler.assign_solo_workflow_to_device()` instead. Every hop is triggered by a GraphQL mutation on the portal and then propagated through SQS queues.

### Phase 1: Frontend → action_resolver (prepare + finalize)

**Files**: `alps-ttp3-frontend/src/components/BulkAction/index.tsx`, `alps-ttp3-frontend/src/graphql/mutations.ts`

1. User picks **Workflow Assignment** in the Bulk Action modal, chooses a workflow (which exposes `workflowId`, `service_type_id`, `workflow_name`, etc.), and uploads a CSV of device UIDs.
2. `prepareBatchAction` GraphQL mutation sends:
   ```graphql
   mutation prepareBatchAction(
     $tenantId: String,
     $actionType: String,
     $workflowId: String,
     $serviceTypeId: Int
   ) {
     prepareBatchAction(
       tenantId: "deveco"
       actionType: "WORKFLOW_ACTION"
       workflowId: "68a5ae7a4cb9e87b88b7d344"
       serviceTypeId: 1
     ) {
       batchActionId
       fileUpload { url, fields }
     }
   }
   ```
   The backend replies with an S3 pre-signed form `fileUpload`, and the UI uploads the CSV directly to S3 using those fields.
3. When the user clicks **Submit**, `finalizeBatchAction` fires with `batchActionId`, `workflowId`, and `batchUploadType = "WORKFLOW_ACTION"`. This GraphQL boundary is the only direct FE→BE hop; everything else rides through queues.

### Phase 2: action_resolver → DISPATCHER_SQS

**File**: `alps-ttp3-backend/modules/action_resolver/src/main.py`

- `prepare_batch_action` stores metadata in `t_batch_action.ext_fields`: total devices, workflow id, service type, device type, etc.
- `finalize_batch_action` pulls that row back, injects user context (`user_id`, `tenant_id`), and publishes a JSON message to **`BATCH_ACTION_DISPATCHER_SQS`**. Critical fields:
  ```json
  {
    "tenant_id": "deveco",
    "user_id": 123,
    "batch_action_id": 1974,
    "batch_upload_type": "WORKFLOW_ACTION",
    "workflow_id": "68a5ae7a4cb9e87b88b7d344",
    "service_type_id": 1,
    "assign_action_id": "assign_solo_workflow"
  }
  ```
- No devices are touched yet; the dispatcher owns CSV parsing and validation.

### Phase 3: Dispatcher → PROCESSOR_SQS (per device)

**File**: `alps-ttp3-backend/modules/batch_action_dispatcher/src/main.py`

1. Downloads the CSV using the `file_url` from `t_batch_action` and hydrates a DataFrame with IMEIs/Uids.
2. Validates each device through the DB layer:
   - `state_id` must be in `[2 (READY_FOR_USE), 4 (ACTIVE), 5 (LOCKED)]`.
   - `service_type_id` must match the workflow’s service type `[1,2,3]`.
3. For every valid device, inserts a `t_batch_action_device` row and emits a message to **`BATCH_ACTION_PROCESSOR_SQS`**:
   ```json
   {
     "device_uid": "f4057468931f154fb47a9acf5d8e3b21a",
     "tenant_id": "deveco",
     "user_id": 123,
     "batch_action_id": 1974,
     "batch_action_device_id": 10787,
     "action_id": "assign_solo_workflow",
     "solo_workflow": { "workflow_id": "68a5ae7a4cb9e87b88b7d344" },
     "service_type_id": 1,
     "device_type": 1,
     "batch_upload_type": "WORKFLOW_ACTION"
   }
   ```
4. Failed validations short-circuit with `Error.WRONG_STATE_DEVICE`, `Error.WRONG_SERVICE_TYPE`, or `Error.DEVICE_NOT_FOUND`, keeping the action history accurate.

### Phase 4: Processor → ACTION_TRIGGER_SQS

**File**: `alps-ttp3-backend/modules/batch_action_processor/src/actions/others/handle_workflow_assignment.py`

- `handle_workflow_assignment` receives each device payload, enriches it with tenant schema info, and updates `t_batch_action_device` status once the downstream publish succeeds.
- The message pushed to **`ACTION_TRIGGER_SQS`** keeps the same contract fields (`solo_workflow`, `action_id`, `device_uid`, etc.) so the trigger can be stateless.
- Any exception while sending is recorded back into `t_batch_action_device.result` so the portal shows a per-device failure reason.

### Phase 5: Trigger → Workflow Service

**Files**: `alps-ttp3-backend/modules/action_trigger/src/handlers/message_handler.py`, `alps-ttp3-backend/modules/action_trigger/src/workflow_handler.py`

1. `message_handler.handle_message` switches on `payload_dto.action_id` and, when it equals `assign_solo_workflow`, delegates to `workflow_handler.assign_solo_workflow_to_device`.
2. The handler crafts an HTTP **POST** to the workflow service:
   ```http
   POST /api/v1/device/{device_uid}/assign_workflow
   Headers:
     tenant-id: deveco
     user-id: sinh.vo
     user-group-roles: ["Admin"]
   Body:
     {
       "workflowId": "68a5ae7a4cb9e87b88b7d344",
       "reason": "Bulk workflow assignment",
       "source": "PORTAL",
       "channel": "BULK_ACTION"
     }
   ```
3. Successful responses mark `t_batch_action_device.is_success = true`; failures (4xx/5xx) bubble up so operators can re-run affected devices.

### Phase 6: Workflow Service → MongoDB

**Files**: `alps-ttp3-workflow/src/apiservice/api/api_v1/endpoints/device.py`, `alps-ttp3-workflow/src/apiservice/controller/device.py`

- The workflow microservice validates the tenant headers, loads the Device document from MongoDB, and calls `WorkflowConfig.assign_workflow_to_device`.
- MongoDB fields updated:
  ```json
  {
    "is_activating_workflow": true,
    "workflow.workflowId": "68a5ae7a4cb9e87b88b7d344",
    "workflow.workflowName": "Solo Setup",
    "workflow.assignedAt": "2024-03-10T08:15:00Z",
    "workflow.assignedBy": "sinh.vo"
  }
  ```
- The service emits audit logs and returns `200 OK` to the trigger, closing the loop.

---

## BULK EXIT WORKFLOW - Detailed 6-Phase Flow

### Phase 1: Frontend → action_resolver

**File**: `alps-ttp3-frontend/src/components/BulkAction/index.tsx`

**What to do**:
- User selects EXIT_WORKFLOW radio button in bulk action modal
- User enters exit reason text in TextArea field (required, max 255 characters)
- TextArea validation checks: not empty, length <= 255
- User clicks Submit button
- Frontend sends GraphQL mutation `finalizeBatchAction` to action_resolver
- Include all required fields: tenantId, batchActionId, batchUploadType, exitWorkflowReason
- batchActionId comes from previous prepareBatchAction response
- tenantId comes from user context
- batchUploadType is hardcoded constant "WORKFLOW_ACTION"
- exitWorkflowReason is user input from TextArea

**Payload sent** (GraphQL mutation):
```graphql
mutation finalizeBatchAction(
  $tenantId: String
  $batchActionId: String
  $batchUploadType: String
  $exitWorkflowReason: String
) {
  finalizeBatchAction(
    tenantId: "deveco"
    batchActionId: "1974"
    batchUploadType: "WORKFLOW_ACTION"
    exitWorkflowReason: "User requested to exit workflow for maintenance"
  )
}
```

**Complete payload fields**:
- `tenantId`: "deveco" - Current user's tenant
- `batchActionId`: "1974" - ID from prepareBatchAction
- `batchUploadType`: "WORKFLOW_ACTION" - Constant for workflow actions
- `exitWorkflowReason`: "User requested to exit workflow for maintenance" - User input text

---

### Phase 2: action_resolver → Dispatcher

**File**: `alps-ttp3-backend/modules/action_resolver/src/main.py`

**What to do**:
- Receive GraphQL mutation parameters from frontend (tenantId, batchActionId, batchUploadType, exitWorkflowReason)
- Extract user_id from context (authenticated user)
- Query t_batch_action table using batch_action_id to get batch_action record
- Read ext_fields JSON from batch_action record
- Extract workflow_id from ext_fields (value will be "exit" for exit workflow)
- Detect this is exit workflow action when workflow_id == "exit"
- Extract exit_workflow_reason from kwargs (GraphQL mutation parameter)
- Extract batch_upload_type from kwargs (value: "WORKFLOW_ACTION")
- Create message dictionary with all necessary fields
- Add batch_action_id to message
- Add tenant_id to message
- Add user_id to message (from context, not from kwargs)
- Add workflow_id to message (value: "exit")
- Add exit_workflow_reason to message (pass through unchanged)
- Add batch_upload_type to message (value: "WORKFLOW_ACTION")
- Send complete message to DISPATCHER_SQS queue
- No database updates needed in this phase
- Return success response to frontend

**Payload to Dispatcher** (SQS message):
```json
{
  "user_id": 123,
  "tenant_id": "deveco",
  "batch_action_id": 1974,
  "batch_upload_type": "WORKFLOW_ACTION",
  "billing_detail": null,
  "sim_control": null,
  "duration": null,
  "workflow_id": "exit",
  "exit_workflow_reason": "User requested to exit workflow for maintenance",
  "blink_frequency": null,
  "blink_time_unit": null,
  "service_type_id": 1
}
```

**Complete payload fields**:
- `user_id`: 123 - User ID from authenticated context
- `tenant_id`: "deveco" - Tenant ID from kwargs
- `batch_action_id`: 1974 - ID of batch action from t_batch_action
- `batch_upload_type`: "WORKFLOW_ACTION" - Routing identifier for dispatcher
- `billing_detail`: null - Billing information (not used for workflow exit)
- `sim_control`: null - SIM control config (not used for workflow exit)
- `duration`: null - Duration config (not used for workflow exit)
- `workflow_id`: "exit" - Special value indicating exit workflow (from ext_fields)
- `exit_workflow_reason`: "User requested to exit workflow for maintenance" - User input passed through
- `blink_frequency`: null - Blink frequency config (not used for workflow exit)
- `blink_time_unit`: null - Blink time unit config (not used for workflow exit)
- `service_type_id`: 1 - Service type ID (optional, from kwargs or ext_fields)

---

### Phase 3: Dispatcher → Processor (per device)

**File**: `alps-ttp3-backend/modules/batch_action_dispatcher/src/main.py`

**What to do**:
- Receive message from DISPATCHER_SQS with all fields from Phase 2
- Extract workflow_id from message kwargs
- Detect this is exit workflow when workflow_id == "exit"
- Set action_id constant to "exit_solo_workflow" (hardcoded, not from database)
- Extract exit_workflow_reason from kwargs (pass through unchanged)
- Extract batch_upload_type from kwargs (value: "WORKFLOW_ACTION")
- Extract batch_action_id, tenant_id, user_id from kwargs
- Query t_batch_action to get CSV file URL and device list
- Parse CSV file to get all device UIDs
- Query t_device table to get device records for all UIDs (filter by service_type_id)
- Loop through each device record
- **Validate device state**: device.state_id must be in [2, 4, 5] (READY_FOR_USE, ACTIVE, LOCKED)
  - If invalid state, create error record with `Error.WRONG_STATE_DEVICE`
- **Validate device has workflow**: device.is_activating_workflow must be true
  - If false (no workflow), create error record with `Error.DEVICE_HAS_NO_WORKFLOW`
- For valid devices (correct state + has workflow), insert new record in t_batch_action_device table
- Get generated batch_action_device_id from insert
- Create per-device message payload
- Add device_uid to payload (from device record)
- Keep tenant_id, user_id unchanged from kwargs
- Add action_id = "exit_solo_workflow" to payload
- Add exit_workflow_reason to payload (unchanged from kwargs)
- Add batch_action_device_id to payload (from database insert)
- Add batch_action_id to payload (unchanged from kwargs)
- Add batch_upload_type to payload (value: "WORKFLOW_ACTION")
- Send one message per device to PROCESSOR_SQS
- After processing all devices, update t_batch_action.dispatched = true

**Payload to Processor** (one message per device):
```json
{
  "device_uid": "f4057468931f154fb47a9acf5d8e3b21a",
  "user_id": 123,
  "tenant_id": "deveco",
  "batch_action_id": 1974,
  "batch_upload_type": "WORKFLOW_ACTION",
  "billing_detail": null,
  "sim_control": null,
  "duration": null,
  "lock_time_unit": null,
  "blink_frequency": null,
  "blink_time_unit": null,
  "action_id": "exit_solo_workflow",
  "batch_action_device_id": 10787,
  "device_type": 1,
  "exit_workflow_reason": "User requested to exit workflow for maintenance"
}
```

**Complete payload fields**:
- `device_uid`: "f4057468931f154fb47a9acf5d8e3b21a" - Individual device UID from CSV
- `user_id`: 123 - Unchanged from dispatcher input
- `tenant_id`: "deveco" - Unchanged from dispatcher input
- `batch_action_id`: 1974 - Unchanged from dispatcher input
- `batch_upload_type`: "WORKFLOW_ACTION" - Unchanged from dispatcher input
- `billing_detail`: null - Billing information (not used, forwarded from resolver)
- `sim_control`: null - SIM control config (not used, forwarded from resolver)
- `duration`: null - Duration config (not used, forwarded from resolver)
- `lock_time_unit`: null - Lock time unit config (not used, forwarded from resolver)
- `blink_frequency`: null - Blink frequency config (not used, forwarded from resolver)
- `blink_time_unit`: null - Blink time unit config (not used, forwarded from resolver)
- `action_id`: "exit_solo_workflow" - Hardcoded constant for exit workflow
- `batch_action_device_id`: 10787 - Generated ID from t_batch_action_device insert
- `device_type`: 1 - Device type (1=SMARTPHONE, 2=FEATURE_PHONE, etc.)
- `exit_workflow_reason`: "User requested to exit workflow for maintenance" - Exit reason from user input

---

### Phase 4: Processor → Trigger

**File**: `alps-ttp3-backend/modules/batch_action_processor/src/actions/others/handle_workflow_exit.py`

**What to do**:
- Receive message from PROCESSOR_SQS with all fields from Phase 3
- Extract batch_action_device_id from message
- Extract tenant_id from message to get schema_name
- Extract device_uid from message
- Extract user_id from message
- Extract exit_workflow_reason from message (default to "Bulk exit workflow" if missing)
- Extract action_id from message (should be "exit_solo_workflow")
- Query database to get schema_name from tenant_id
- Forward complete message to ACTION_TRIGGER_SQS (unchanged, all fields passed through)
- After sending to trigger, update t_batch_action_device record
- Set is_success = True (optimistic update, trigger will handle errors)
- Update ext_fields JSON with device_uid, exit_reason, assign_action_id
- Update result JSON with message "Workflow exit sent to trigger"
- Update created_by field with user_id
- If error occurs during send, catch exception
- Update t_batch_action_device with is_success = False
- Update result JSON with error message
- Do NOT retry or resend message

**Payload to Trigger** (SQS message, unchanged from Phase 3):
```json
{
  "device_uid": "f4057468931f154fb47a9acf5d8e3b21a",
  "user_id": 123,
  "tenant_id": "deveco",
  "batch_action_id": 1974,
  "batch_upload_type": "WORKFLOW_ACTION",
  "billing_detail": null,
  "sim_control": null,
  "duration": null,
  "lock_time_unit": null,
  "blink_frequency": null,
  "blink_time_unit": null,
  "action_id": "exit_solo_workflow",
  "batch_action_device_id": 10787,
  "device_type": 1,
  "exit_workflow_reason": "User requested to exit workflow for maintenance"
}
```

**Complete payload fields** (all fields forwarded unchanged):
- `device_uid`: "f4057468931f154fb47a9acf5d8e3b21a" - Same from Phase 3
- `user_id`: 123 - Same from Phase 3
- `tenant_id`: "deveco" - Same from Phase 3
- `batch_action_id`: 1974 - Same from Phase 3
- `batch_upload_type`: "WORKFLOW_ACTION" - Same from Phase 3
- `billing_detail`: null - Same from Phase 3 (not used by trigger)
- `sim_control`: null - Same from Phase 3 (not used by trigger)
- `duration`: null - Same from Phase 3 (not used by trigger)
- `lock_time_unit`: null - Same from Phase 3 (not used by trigger)
- `blink_frequency`: null - Same from Phase 3 (not used by trigger)
- `blink_time_unit`: null - Same from Phase 3 (not used by trigger)
- `action_id`: "exit_solo_workflow" - Same from Phase 3 (routing field)
- `batch_action_device_id`: 10787 - Same from Phase 3 (tracking field)
- `device_type`: 1 - Same from Phase 3 (device type identifier)
- `exit_workflow_reason`: "User requested to exit workflow for maintenance" - Same from Phase 3 (critical field for MongoDB)

**Database update** (t_batch_action_device):
- Update WHERE id = batch_action_device_id
- SET is_success = True
- SET ext_fields = JSON with {device_uid, exit_reason, assign_action_id}
- SET result = JSON with {message: "Workflow exit sent to trigger"}
- SET created_by = user_id

---

### Phase 5: Trigger → Workflow Service

**File**: `alps-ttp3-backend/modules/action_trigger/src/handlers/message_handler.py`

**What to do**:
- Receive message from ACTION_TRIGGER_SQS with all fields from Phase 4
- Parse message into PayloadDTO Pydantic model for validation
- Extract action_id from PayloadDTO
- Detect this is exit workflow when action_id == "exit_solo_workflow"
- Extract device_uid from PayloadDTO
- Extract tenant_id from PayloadDTO
- Query database to get schema record from tenant_id
- Extract user_id from PayloadDTO (convert to username if needed)
- Extract exit_workflow_reason from PayloadDTO
- Query t_device table to get device record by device_uid
- Validate device exists in database
- Call workflow_handler.remove_workflow_and_contract_from_device() function
- Pass device_uid as parameter
- Pass tenant_id (from schema.tenant_id) as parameter
- Pass user_id as parameter
- Pass exit_workflow_reason as reason parameter (critical field)
- Workflow handler builds HTTP DELETE request URL
- URL format: {WORKFLOW_SERVICE_URL}/device/{device_uid}/remove_workflow_and_contract_from_device
- Add reason as query parameter in URL
- URL encode reason parameter properly
- Add HTTP headers: tenant-id, user-id, user-group-roles
- Set user-group-roles to ["Admin"] for permission
- Send HTTP DELETE request to workflow service
- Set timeout to 5 seconds
- Disable SSL verification (verify=False)
- If response status != 200, log warning with response text
- If HTTP request throws exception, log error
- Return from handler (do not process further)

**HTTP Request to Workflow Service**:
```http
DELETE https://workflow-service.example.com/api/v1/device/f4057468931f154fb47a9acf5d8e3b21a/remove_workflow_and_contract_from_device?reason=User%20requested%20to%20exit%20workflow%20for%20maintenance
Host: workflow-service.example.com
tenant-id: deveco
user-id: sinh.vo
user-group-roles: ["Admin"]
Content-Length: 0
```

**Complete request details**:
- **Method**: DELETE
- **URL Path**: /api/v1/device/{device_uid}/remove_workflow_and_contract_from_device
- **device_uid**: f4057468931f154fb47a9acf5d8e3b21a - From PayloadDTO
- **Query Parameters**:
  - `reason`: "User requested to exit workflow for maintenance" - URL encoded exit_workflow_reason
- **Headers**:
  - `tenant-id`: "deveco" - From schema.tenant_id  
  - `user-id`: "sinh.vo" - From PayloadDTO.username (string, not ID)
  - `user-group-roles`: ["Admin"] - Hardcoded for permission
- **Body**: Empty (DELETE request, reason in query parameter)
- **Timeout**: 5 seconds
- **SSL Verify**: False (disabled for internal services)

**Important Notes**:
- ⚠️ `reason` is passed as **query parameter**, not in body
- ⚠️ `user-id` header is **username** (e.g. "sinh.vo"), not numeric ID
- ⚠️ Same behavior as single device exit workflow

---

### Phase 6: Workflow Service Processing

**File**: `alps-ttp3-workflow/src/apiservice/api/api_v1/endpoints/device.py`

**What to do**:
- Receive HTTP DELETE request from action_trigger
- Extract device_id from URL path parameter
- Extract reason from query parameter (optional, may be None)
- Extract tenant_id from request header "tenant-id"
- Extract user_id from request header "user-id"
- Validate identity headers (tenant_id and user_id are required)
- Call controller function transactional_exit_workflow_and_contract_from_device()
- Pass device_id as parameter
- Pass tenant_id as parameter
- Pass user_id as parameter
- Pass reason as parameter (may be None)
- Controller queries MongoDB DeviceDB collection by device_uid and tenant_id
- Get device document from MongoDB
- Check device.is_activating_contract field to determine device type
- Format exit_reason string based on device type:
  - If is_activating_contract == True (contract device): exit_reason = "{reason}" (raw reason)
  - If is_activating_contract == False (solo workflow): exit_reason = "User {user_id} have requested to exit workflow manually with reason: {reason}"
- Update device document in MongoDB
- Set device.exit_reason = formatted string (only if reason is not None)
- Set device.is_activating_workflow = False
- Set device.is_activating_contract = False
- Save device document to MongoDB (atomic update within transaction)
- Transaction ensures atomicity of workflow exit and contract removal
- Return HTTP 200 response to trigger
- Response body: {"message": "Successfully removed workflow"}
- If device not found, return HTTP 404
- If error occurs, transaction rolls back, return HTTP 500

**MongoDB Update** (DeviceDB collection):
```json
{
  "_id": "ObjectId(...)",
  "deviceUid": "f4057468931f154fb47a9acf5d8e3b21a",
  "tenantId": "deveco",
  "exit_reason": "User 123 have requested to exit workflow manually with reason: User requested to exit workflow for maintenance",
  "is_activating_workflow": false,
  "is_activating_contract": false,
  "updatedAt": "2024-01-15T10:30:00Z"
}
```

**Complete MongoDB update fields**:
- `exit_reason`: "User 123 have requested to exit workflow manually with reason: User requested to exit workflow for maintenance" - Formatted string with user_id and reason
- `is_activating_workflow`: false - Device no longer has active workflow
- `is_activating_contract`: false - Device no longer has active contract
- `updatedAt`: Timestamp of update

**HTTP Response to Trigger**:
```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "message": "Successfully removed workflow"
}
```

**Exit Reason Format Logic**:
- **Contract device** (is_activating_contract = true):
  - Format: `"{reason}"`
  - Example: `"User requested to exit workflow for maintenance"`
- **Solo workflow device** (is_activating_contract = false):
  - Format: `"User {user_id} have requested to exit workflow manually with reason: {reason}"`
  - Example: `"User 123 have requested to exit workflow manually with reason: User requested to exit workflow for maintenance"`

---

## Summary - Files Modified for Bulk Exit Workflow

### Frontend (1 file)
1. **BulkAction/index.tsx**
   - Added EXIT_WORKFLOW radio option
   - Added exit reason TextArea (maxLength=255, required)
   - Added logic to send `workflowId: 'exit'` when EXIT_WORKFLOW selected
   - Removed workflow start date for exit flow
   - Service selector kept (required to filter devices by service_type_id)

### Backend (8 files)

2. **action_resolver/src/main.py**
   - Forward `exit_workflow_reason` from kwargs to dispatcher SQS

3. **batch_action_dispatcher/src/const.py** (NEW ERROR CONSTANT)
   - Added `DEVICE_HAS_NO_WORKFLOW = "DEVICE_HAS_NO_WORKFLOW"` error constant
   - Added error message: "This {device_type} [{device_uid}] does not have an active workflow to exit"

4. **batch_action_dispatcher/src/main.py**
   - Detect `workflow_id == "exit"` from kwargs
   - Set `action_id = "exit_solo_workflow"` (constant)
   - Extract `exit_workflow_reason` from kwargs
   - Validate `device.state_id` in [2, 4, 5] (READY_FOR_USE, ACTIVE, LOCKED)
   - Validate `device.is_activating_workflow == true`
   - Create error records for invalid states or devices without workflow
   - Conditionally add `exit_workflow_reason` to payload (only if present)
   - Pass `exit_workflow_reason` to processor per device

5. **batch_action_processor/src/main.py**
   - Import `handle_workflow_exit` function
   - Route `action_id == "exit_solo_workflow"` to `handle_workflow_exit()`

6. **batch_action_processor/src/actions/others/handle_workflow_exit.py** (NEW FILE)
   - Extract `exit_workflow_reason` from message
   - Update `t_batch_action_device` with success status
   - Save `exit_reason` in `ext_fields` JSON
   - Forward complete message to ACTION_TRIGGER_SQS
   - Handle exceptions and update failure status

7. **action_trigger/src/dto_handler.py**
   - Added `exit_workflow_reason: Optional[str] = None` field to `PayloadDTO`

8. **action_trigger/src/handlers/message_handler.py**
   - Handle `action_id == "exit_solo_workflow"` case
   - Call `workflow_handler.remove_workflow_and_contract_from_device()`
   - Pass `device_uid`, `tenant_id`, `user_id`, `reason` parameters

9. **action_trigger/src/workflow_handler.py** (NEW FUNCTION)
   - Added `remove_workflow_and_contract_from_device()` function
   - Build DELETE request URL with device_uid
   - Pass `reason` as **query parameter** (URL encoded)
   - Set headers: `tenant-id`, `user-id` (username), `user-group-roles`
   - Send HTTP DELETE to workflow service
   - Handle response and log results

### Workflow Service (2 files)

10. **alps-ttp3-workflow/src/apiservice/api/api_v1/endpoints/device.py**
    - Endpoint already exists: `DELETE /{device_id}/remove_workflow_and_contract_from_device`
    - Receives `reason` as **query parameter** (Optional[str])
    - Uses existing `IdentityHeaders` dependency for tenant_id/user_id
    - Calls `transactional_exit_workflow_and_contract_from_device()`

11. **alps-ttp3-workflow/src/apiservice/controller/device.py**
    - Function already exists: `remove_workflow_and_contract_from_device()`
    - Queries MongoDB DeviceDB by device_uid and tenant_id
    - **If device has contract** (`is_activating_contract == true`):
      - Sets `exit_reason = f"{reason}"` (reason only)
    - **If device has workflow only** (`is_activating_contract == false`):
      - Sets `exit_reason = f"User {user_id} have requested to exit workflow manually with reason: {reason}"`
    - Sets `is_activating_workflow = False`
    - Sets `is_activating_contract = False`
    - Saves to MongoDB with transaction

---

## Key Payload Fields Flow

| Phase | Key Field | Value | Purpose |
|-------|-----------|-------|---------|
| Frontend → Resolver | `exitWorkflowReason` | User input text | Exit reason from user |
| Resolver → Dispatcher | `exit_workflow_reason` | Same | Pass to dispatcher |
| Dispatcher → Processor | `exit_workflow_reason` | Same | Per-device message |
| Processor → Trigger | `exit_workflow_reason` | Same | Forward unchanged |
| Trigger → Workflow API | `reason` (query param) | Same | Save to MongoDB |
| MongoDB | `exit_reason` | Formatted string | History display |

---

## COMPREHENSIVE TEST SCENARIOS

### Test Suite 1: Bulk REGISTER with Workflow Assignment

#### Scenario 1.1: SIM_CONTROL Register + Workflow (Happy Path)
**Setup**:
- Service: SIM_CONTROL (service_type_id = 1)
- Workflow: "Device Setup" (service_type_id = 1)
- CSV: 5 devices in IDLE state

**Expected**:
- All 5 devices: state IDLE → READY_FOR_USE
- All 5 devices: workflow "Device Setup" assigned
- Action History: 5 success, 0 failed

**Verification**:
```sql
SELECT state_id, is_activating_workflow FROM t_device WHERE uid IN (...);
-- state_id = 2 (READY_FOR_USE), is_activating_workflow = true
```

#### Scenario 1.2: PREPAID Register + Workflow (Happy Path)
**Setup**:
- Service: PREPAID (service_type_id = 2)
- Workflow: "Prepaid Onboarding" (service_type_id = 2)
- CSV: 3 devices in IDLE state

**Expected**:
- All 3 devices: state IDLE → READY_FOR_USE
- All 3 devices: workflow assigned
- Action History: 3 success, 0 failed

#### Scenario 1.3: POSTPAID Register + Workflow (Happy Path)
**Setup**:
- Service: POSTPAID (service_type_id = 3)
- Workflow: "Contract Setup" (service_type_id = 3)
- CSV: 10 devices in IDLE state

**Expected**:
- All 10 devices: state IDLE → READY_FOR_USE
- All 10 devices: workflow assigned
- Action History: 10 success, 0 failed

#### Scenario 1.4: Mixed Device States (Partial Failure)
**Setup**:
- Service: SIM_CONTROL
- Workflow: "Device Setup"
- CSV: 10 devices
  - 5 in IDLE state (valid)
  - 3 in READY_FOR_USE state (invalid for REGISTER)
  - 2 in ACTIVE state (invalid for REGISTER)

**Expected**:
- 5 devices IDLE → READY_FOR_USE + workflow assigned ✅
- 3 devices READY_FOR_USE → Error: WRONG_STATE_DEVICE ❌
- 2 devices ACTIVE → Error: WRONG_STATE_DEVICE ❌
- Action History: 5 success, 5 failed

#### Scenario 1.5: Wrong Service Type Workflow
**Setup**:
- Service: SIM_CONTROL (service_type_id = 1)
- Workflow: "Prepaid Onboarding" (service_type_id = 2) ← Mismatch!
- CSV: 5 devices in IDLE state

**Expected**:
- Devices register successfully (IDLE → READY_FOR_USE)
- Workflow assignment may fail or skip (workflow service validation)
- Check logs for workflow API errors

---

### Test Suite 2: Bulk WORKFLOW_ACTION (Assign to Registered Devices)

#### Scenario 2.1: SIM_CONTROL Workflow Assignment (Happy Path)
**Setup**:
- Workflow: "Device Setup" (service_type_id = 1)
- CSV: 10 devices
  - 3 in READY_FOR_USE state
  - 4 in ACTIVE state
  - 3 in LOCKED state
  - All have service_type_id = 1

**Expected**:
- All 10 devices: workflow assigned ✅
- No state change (remain in current state)
- Action History: 10 success, 0 failed

#### Scenario 2.2: Invalid States for Workflow Assignment
**Setup**:
- Workflow: "Device Setup" (service_type_id = 1)
- CSV: 8 devices
  - 2 in IDLE state (invalid)
  - 3 in ENROLLED state (invalid)
  - 3 in READY_FOR_USE state (valid)

**Expected**:
- 2 IDLE devices → Error: WRONG_STATE_DEVICE ❌
- 3 ENROLLED devices → Error: WRONG_STATE_DEVICE ❌
- 3 READY_FOR_USE devices → Workflow assigned ✅
- Action History: 3 success, 5 failed

#### Scenario 2.3: Wrong Service Type Devices
**Setup**:
- Workflow: "Device Setup" (service_type_id = 1, SIM_CONTROL)
- CSV: 6 devices in ACTIVE state
  - 3 devices with service_type_id = 1 (valid)
  - 2 devices with service_type_id = 2 (PREPAID - invalid)
  - 1 device with service_type_id = 4 (SUPPLY_CHAIN - invalid)

**Expected**:
- 3 SIM_CONTROL devices → Workflow assigned ✅
- 2 PREPAID devices → Error: WRONG_SERVICE_TYPE ❌
- 1 SUPPLY_CHAIN device → Error: WRONG_SERVICE_TYPE ❌
- Action History: 3 success, 3 failed

#### Scenario 2.4: Large Batch Performance Test
**Setup**:
- Workflow: "Device Setup"
- CSV: 1000 devices in ACTIVE state (all service_type_id = 1)

**Expected**:
- All 1000 devices: workflow assigned
- Check CloudWatch logs for processing time
- Verify no timeouts or SQS queue issues
- Action History: 1000 success, 0 failed

---

### Test Suite 3: Bulk EXIT_WORKFLOW

#### Scenario 3.1: Exit Workflow with Valid Reason (Happy Path)
**Setup**:
- Exit Reason: "Maintenance required for firmware update"
- CSV: 5 devices in ACTIVE state
  - All have is_activating_workflow = true

**Expected**:
- All 5 devices: is_activating_workflow = false ✅
- MongoDB: exit_reason field populated
- Action History: 5 success, 0 failed

**Verification**:
```javascript
db.devices.find({uid: {$in: [...]}}).forEach(d => print(d.exit_reason))
// "User 123 have requested to exit workflow manually with reason: Maintenance required..."
```

#### Scenario 3.2: Exit Workflow - Devices Without Workflow
**Setup**:
- Exit Reason: "User requested exit"
- CSV: 8 devices
  - 5 devices with is_activating_workflow = true (valid)
  - 3 devices with is_activating_workflow = false (invalid)

**Expected**:
- 5 devices with workflow → Workflow exited ✅
- 3 devices without workflow → Error: DEVICE_HAS_NO_WORKFLOW ❌
- Action History: 5 success, 3 failed

#### Scenario 3.3: Exit Workflow - Invalid States
**Setup**:
- Exit Reason: "Testing exit flow"
- CSV: 6 devices (all have workflow)
  - 2 in IDLE state (invalid for exit)
  - 4 in ACTIVE state (valid)

**Expected**:
- 2 IDLE devices → Error: WRONG_STATE_DEVICE ❌
- 4 ACTIVE devices → Workflow exited ✅
- Action History: 4 success, 2 failed

#### Scenario 3.4: Exit Reason Edge Cases
**Test Cases**:

a) **Empty reason**:
- Exit Reason: "" (empty string)
- Expected: Frontend validation should block submission

b) **Very long reason**:
- Exit Reason: 300 characters long
- Expected: Frontend validation should trim to 255 chars

c) **Special characters**:
- Exit Reason: "Exit due to: $pecial <chars> & symbols!"
- Expected: Properly URL encoded in HTTP request

d) **Unicode characters**:
- Exit Reason: "Lý do: Bảo trì hệ thống 维护"
- Expected: UTF-8 encoded correctly in MongoDB

---

### Test Suite 4: Error Handling & Edge Cases

#### Scenario 4.1: Device Not Found
**Setup**:
- Action: WORKFLOW_ACTION
- CSV: Contains non-existent device UIDs

**Expected**:
- Error: DEVICE_NOT_FOUND
- No t_batch_action_device record created

#### Scenario 4.2: Concurrent Workflow Assignment
**Setup**:
- Start Bulk WORKFLOW_ACTION (batch 1)
- Before batch 1 completes, start another WORKFLOW_ACTION (batch 2)
- Both batches target same devices

**Expected**:
- First batch processes normally
- Second batch may conflict (check workflow service behavior)
- Verify no data corruption in MongoDB

#### Scenario 4.3: Network Timeout to Workflow Service
**Setup**:
- Mock workflow service to delay 10 seconds
- Trigger timeout in action_trigger

**Expected**:
- HTTP timeout exception caught
- t_batch_action_device.is_success = false
- Error logged in CloudWatch
- Device state unchanged

#### Scenario 4.4: CSV File Size Limits
**Test Cases**:

a) **Small CSV** (10 devices):
- Expected: Single chunk, processed immediately

b) **Medium CSV** (500 devices):
- Expected: Multiple chunks, processed with delays

c) **Large CSV** (5000 devices):
- Expected: Many chunks, check CHUNK_SIZE handling

---

### Test Suite 5: Integration Tests (End-to-End)

#### Scenario 5.1: Full Lifecycle - Register → Assign → Exit
**Steps**:
1. Bulk REGISTER 10 IDLE devices to SIM_CONTROL + assign workflow A
2. Verify: devices in READY_FOR_USE state + workflow A assigned
3. Bulk WORKFLOW_ACTION: assign workflow B to same 10 devices
4. Verify: workflow changed from A to B
5. Bulk EXIT_WORKFLOW with reason "Test complete"
6. Verify: is_activating_workflow = false, exit_reason populated

**Expected**:
- All steps succeed
- Device history shows complete lifecycle
- MongoDB audit trail complete

#### Scenario 5.2: Mixed Operations in Parallel
**Setup**:
- Start Bulk REGISTER (batch 1) for 100 devices
- Start Bulk WORKFLOW_ACTION (batch 2) for different 100 devices
- Start Bulk EXIT_WORKFLOW (batch 3) for another 100 devices

**Expected**:
- All 3 batches process independently
- No cross-contamination
- Correct devices affected by each batch

---

### Test Suite 6: Performance & Load Tests

#### Scenario 6.1: Throughput Test
**Setup**:
- Submit 10 bulk actions simultaneously
- Each with 100 devices

**Expected**:
- All batches complete within acceptable time
- No SQS queue backlog
- CloudWatch metrics show healthy Lambda execution

#### Scenario 6.2: Database Connection Pool
**Setup**:
- Large batch (1000 devices)
- Monitor database connections during processing

**Expected**:
- No connection pool exhaustion
- No deadlocks
- Transactions complete successfully

---

### Test Checklist Summary

**Pre-deployment Validation**:
- [ ] All 3 service types work for REGISTER + workflow
- [ ] All 3 service types work for WORKFLOW_ACTION
- [ ] EXIT_WORKFLOW validates device has workflow
- [ ] Error messages are clear and actionable
- [ ] CloudWatch logs show correct flow
- [ ] MongoDB updates are atomic
- [ ] Action History displays correct status

**Post-deployment Monitoring**:
- [ ] Check SQS queue metrics (dead letter queue)
- [ ] Monitor Lambda execution times
- [ ] Track error rates in CloudWatch
- [ ] Verify database query performance
- [ ] Review MongoDB indexes for workflow queries
  



  {
"query": "mutation PrepareBatchAction($tenantId: String, $rowCount: Int, $config: BatchAction, $deviceType: String, $brandId: String, $provisionType: String, $targetWorkspaceId: String, $simControlAction: String, $simControlRuleId: String, $serviceTypeId: Int, $blockFactoryResetAction: String, $blockAEProvisioningAction: String, $blockADBCommandAction: String, $workflowId: String) {\n prepareBatchAction(\n tenantId: $tenantId\n rowCount: $rowCount\n config: $config\n deviceType: $deviceType\n brandId: $brandId\n provisionType: $provisionType\n targetWorkspaceId: $targetWorkspaceId\n simControlAction: $simControlAction\n simControlRuleId: $simControlRuleId\n serviceTypeId: $serviceTypeId\n blockFactoryResetAction: $blockFactoryResetAction\n blockAEProvisioningAction: $blockAEProvisioningAction\n blockADBCommandAction: $blockADBCommandAction\n workflowId: $workflowId\n ) {\n batchActionId\n fileUpload {\n url\n fields\n __typename\n }\n __typename\n }\n}",
"variables": {
"tenantId": "deveco",
"rowCount": 1,
"config": {
"service_type_id": "5",
"states": [
{
"action_id": "693f07c1-a8a5-40d0-8d73-99855385f89f",
"state_id": 1,
"options": {
"activity_days": null,
"template_id": null,
"apply_service_type_ids": [
2
]
}
}
]
},
"deviceType": "Smartphone",
"brandId": null,
"provisionType": null
}
}

{
"query": "mutation FinalizeBatchAction($tenantId: String, $batchActionId: String, $batchUploadType: String, $simControlName: String, $simControlId: String, $billingDetail: BillingCycleAssignment, $duration: Int, $blinkFrequency: Int, $blinkTimeUnit: String, $workflowId: String, $serviceTypeId: Int, $exitWorkflowReason: String) {\n finalizeBatchAction(\n tenantId: $tenantId\n batchActionId: $batchActionId\n batchUploadType: $batchUploadType\n simControlName: $simControlName\n simControlId: $simControlId\n billingDetail: $billingDetail\n duration: $duration\n blinkFrequency: $blinkFrequency\n blinkTimeUnit: $blinkTimeUnit\n workflowId: $workflowId\n serviceTypeId: $serviceTypeId\n exitWorkflowReason: $exitWorkflowReason\n )\n}",
"variables": {
"tenantId": "deveco",
"batchActionId": "2354",
"batchUploadType": "ASSIGN_ACTION",
"billingDetail": {
"billingStartDate": "2026-01-28 17:48:43"
},
"workflowId": "6927f76d2adbe70b9019c310",
"serviceTypeId": 2
}
}

đây là 2 mutation được gọi khi bấm submit cho New bulk action" + Register Prepaid + Assign Workflow , kiểm tra với những giá trị này thì nó chạy như nào , có đúng không