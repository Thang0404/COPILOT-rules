# TTPNEO-1667 - CRITICAL: Workflow Assignment State Logic Analysis

## State Validation Mismatch

### Dispatcher Requirement vs Reality

#### File: [batch_action_dispatcher/src/main.py Line 248](file:///home/thang/Documents/rsu/alps-ttp3-backend/modules/batch_action_dispatcher/src/main.py#L248)

```python
# Dispatcher checks for these states:
allowed_states = [StateId.READY_FOR_USE, StateId.ACTIVE, StateId.LOCKED]
# StateId.READY_FOR_USE = 2
# StateId.ACTIVE = 4  
# StateId.LOCKED = 5

for device in devices:
    if device.state_id not in allowed_states:
        # Insert FAIL record with WRONG_STATE_DEVICE error
        ext_fields, result = data_failure_device(
            device_uid, Error.WRONG_STATE_DEVICE, device_type, counter_id
        )
        db.insert_batch_action_device(..., is_success=False, ...)
        continue
```

**Requirement**: Devices MUST be in state `[READY_FOR_USE, ACTIVE, LOCKED]` to assign workflow

---

### User's Test Case

**Scenario**: INVENTORY → Register Prepaid with Workflow

**Device State**: ALL devices are in **IDLE** state (StateId = 1)

**What Happens**:
```
1. Query Phase:
   - devices = get_devices_for_workflow_bulk_action(...)
   - Returns: All devices (regardless of state)
   
2. State Validation (Line 268):
   - device.state_id = 1 (IDLE)
   - allowed_states = [2, 4, 5] (READY_FOR_USE, ACTIVE, LOCKED)
   - 1 NOT IN [2, 4, 5]
   - Result: Insert FAIL record (WRONG_STATE_DEVICE)
```

**This is CORRECT behavior!** Devices in IDLE state CANNOT be assigned workflows!

---

## But Wait... What About "Register Prepaid" Action?

### The REAL Question

User wants to:
1. **Apply for service**: INVENTORY (current service)
2. **Register Prepaid**: Change service from INVENTORY → PREPAID
3. **Assign Workflow**: Attach workflow to device

**Key Issue**: Should workflow be assigned BEFORE or AFTER service transition?

---

## Service Transition Flow Analysis

### Normal "Register Prepaid" Action (without workflow)

#### File: [action_trigger/src/handlers/message_handler.py](file:///home/thang/Documents/rsu/alps-ttp3-backend/modules/action_trigger/src/handlers/message_handler.py)

**Flow**:
```
1. Device in IDLE state (StateId = 1)
2. Action: REGISTER (ActionType = 1)
3. Service: PREPAID (ServiceType = 2)
4. State transition: IDLE → READY_FOR_USE
5. Service change: INVENTORY → PREPAID
6. Result: Device now in READY_FOR_USE with PREPAID service
```

**After transition**: Device is now eligible for workflow assignment!

---

### "Register Prepaid" + Workflow (current implementation)

#### File: [batch_action_dispatcher/src/main.py Lines 242-336](file:///home/thang/Documents/rsu/alps-ttp3-backend/modules/batch_action_dispatcher/src/main.py#L242)

**Flow**:
```
1. Device in IDLE state (StateId = 1)
2. Dispatcher checks: workflow_id exists
3. Validation: device.state_id NOT IN [READY_FOR_USE, ACTIVE, LOCKED]
4. Result: FAIL (WRONG_STATE_DEVICE)
5. Workflow assignment: NEVER HAPPENS
```

**Problem**: Workflow validation happens BEFORE service transition!

---

## Root Cause: Timing Issue

### Expected Behavior

**User Expectation**:
```
Step 1: Change service (INVENTORY → PREPAID)
Step 2: Transition state (IDLE → READY_FOR_USE) 
Step 3: Assign workflow to device
```

### Actual Behavior

**Current Implementation**:
```
Step 1: Validate state for workflow (FAIL because IDLE)
Step 2: STOP - never reach service transition
```

---

## Code Evidence

### Dispatcher Flow (batch_action_dispatcher)

#### Phase 1: Validation (Lines 242-336)
```python
if workflow_id and batch_upload_type in [BatchUploadType.WORKFLOW_ACTION, BatchUploadType.EXIT_WORKFLOW]:
    # ❌ This runs FIRST, BEFORE service transition
    
    allowed_states = [StateId.READY_FOR_USE, StateId.ACTIVE, StateId.LOCKED]
    
    for device in devices:
        if device.state_id not in allowed_states:
            # FAIL here - device is IDLE
            db.insert_batch_action_device(..., is_success=False)
            continue
        
        # This SUCCESS path never reached for IDLE devices
        batch_action_device = db.insert_batch_action_device(..., is_success=True)
        send_message(payload, {}, BATCH_ACTION_PROCESSOR_SQS)
    
    return  # ✅ Early return - normal action processing skipped
```

#### Phase 2: Normal Action (Lines 338-500)
```python
# This code NEVER runs when workflow_id is present
# Because Line 336 returns early

for device in devices:
    # Service transition logic
    # State transition logic
    # Action processing
```

**CRITICAL**: When `workflow_id` is present, the normal action processing (including service transition) is SKIPPED!

---

### Processor Flow (batch_action_processor)

#### File: [batch_action_processor/src/main.py Line 109](file:///home/thang/Documents/rsu/alps-ttp3-backend/modules/batch_action_processor/src/main.py#L109)

```python
if message.get("action_id") == "assign_solo_workflow":
    handle_workflow_assignment(db, message)
    continue
```

**What it does**: Simply forwards to ACTION_TRIGGER_SQS, doesn't handle service transition!

---

### Trigger Flow (action_trigger)

#### File: [action_trigger/src/workflow_handler.py Line 692](file:///home/thang/Documents/rsu/alps-ttp3-backend/modules/action_trigger/src/workflow_handler.py#L692)

```python
def assign_solo_workflow_to_device(...):
    url = WorkflowConfig.URL_TOKEN + f"/device/{device_uid}/assign_workflow"
    
    body = {
        "serviceType": service_type,  # Uses CURRENT service_type from device
        "provisionType": device.provision_type,
        "stateId": str(device.state_id),  # Uses CURRENT state_id
        "workflowId": workflow_id,
        # ...
    }
    
    response = requests.put(url=url, headers=headers, data=json.dumps(body))
```

**What it does**: 
- Calls workflow service API
- Uses CURRENT device state and service
- Does NOT transition service or state

---

## The Bug: Wrong Workflow Type

### Current Implementation: WORKFLOW_ACTION

**File**: [batch_action_dispatcher/src/main.py Line 90](file:///home/thang/Documents/rsu/alps-ttp3-backend/modules/batch_action_dispatcher/src/main.py#L90)

```python
batch_upload_type = kawrgs.get("batch_upload_type", BatchUploadType.WORKFLOW_ACTION)
```

**BatchUploadType.WORKFLOW_ACTION**: 
- For devices ALREADY in correct service/state
- For standalone workflow assignment
- NO service transition
- NO state transition

---

### Should Use: ASSIGN_ACTION + solo_workflow

**File**: [action_trigger/src/transition_new_service_handler.py Line 92](file:///home/thang/Documents/rsu/alps-ttp3-backend/modules/action_trigger/src/transition_new_service_handler.py#L92)

```python
def handle_transition_new_service(...):
    # ... do service transition first
    
    # Then check for workflow
    if solo_workflow and solo_workflow.workflow_id and is_support_workflow_assignment_to_new_service(action.service_type_id):
        # ✅ Assign workflow AFTER service transition
        workflow_handler.assign_solo_workflow_to_device(
            tenant_id=tenant_id,
            username=username,
            device=device,
            action=action,
            device_uid=device_uid,
            workflow_id=solo_workflow.workflow_id,
            schema_name=schema_name,
            root_tenant_id=root_tenant_id,
            device_handler=device_handler,
            milestone_handler=milestone_handler,
            user_id=user_id,
            schema=schema
        )
```

**Correct Flow**:
1. Device starts in IDLE state
2. REGISTER action processes
3. Service changes: INVENTORY → PREPAID
4. State changes: IDLE → READY_FOR_USE
5. Workflow assigned to device (now eligible)

---

## Why 17,185 FAIL Records Make Sense Now

### Updated Understanding

**10,000 devices in INVENTORY IDLE → Register Prepaid + Workflow**

#### Phase 1: wrong_service_imeis Check
```python
# Query devices with wrong service
wrong_service_imeis = get_wrong_service_type_devices_for_workflow(
    schema_name, device_uids['query'], service_type_id=2  # Prepaid
)
# Returns: All 10,000 devices (currently INVENTORY service_type_id=6)

# Insert FAIL records
for imei in wrong_service_imeis:
    db.insert_batch_action_device(..., is_success=False)  # WRONG_SERVICE_TYPE
```

**Result**: 10,000 FAIL records (WRONG_SERVICE_TYPE)

#### Phase 2: Workflow State Check
```python
# Query devices for workflow
devices = get_devices_for_workflow_bulk_action(
    schema_name, device_uids['query'], service_type_id=2
)
# Returns: Some devices (query logic unclear, but likely some overlap)

# Check state
for device in devices:
    if device.state_id not in [READY_FOR_USE, ACTIVE, LOCKED]:  # [2, 4, 5]
        # Device is IDLE (1)
        db.insert_batch_action_device(..., is_success=False)  # WRONG_STATE_DEVICE
```

**Result**: More FAIL records (WRONG_STATE_DEVICE)

#### Total FAIL Records
```
Hypothesis:
- ~10,000 devices: WRONG_SERVICE_TYPE (all devices)
- ~7,185 devices: WRONG_STATE_DEVICE (overlap from devices list)
- Total: 17,185 FAIL records

Why only 7,185 in second check?
- get_devices_for_workflow_bulk_action() query filters by service_type_id=2
- Only devices that somehow match this filter get checked again
- Query overlap or LEFT JOIN returning partial results
```

---

## Solutions

### Solution 1: Change batch_upload_type to ASSIGN_ACTION ✅ RECOMMENDED

**File**: Frontend - BulkAction component

**Change**:
```javascript
// Current (WRONG):
batch_upload_type: BatchUploadType.WORKFLOW_ACTION

// Correct:
batch_upload_type: BatchUploadType.ASSIGN_ACTION
```

**With**:
```javascript
// Include workflow_id in normal action payload
action_data: {
  action_id: register_action_id,
  solo_workflow: {
    workflow_id: selected_workflow_id
  }
}
```

**Result**: 
- Normal action processing runs (service + state transition)
- Workflow assigned AFTER transition
- Devices eligible for workflow

---

### Solution 2: Fix Dispatcher Logic (More Complex)

**File**: [batch_action_dispatcher/src/main.py](file:///home/thang/Documents/rsu/alps-ttp3-backend/modules/batch_action_dispatcher/src/main.py)

**Option A**: Skip state check for REGISTER actions
```python
if workflow_id and batch_upload_type in [BatchUploadType.WORKFLOW_ACTION, BatchUploadType.EXIT_WORKFLOW]:
    allowed_states = [StateId.READY_FOR_USE, StateId.ACTIVE, StateId.LOCKED]
    
    # ✅ ADD: Check if action will transition to valid state
    action = db.get_action(schema_name, action_id)
    will_transition_to_valid_state = (
        action.actiontype_id == ActionType.REGISTER and 
        device.state_id == StateId.IDLE
    )
    
    for device in devices:
        if device.state_id not in allowed_states and not will_transition_to_valid_state:
            # Only FAIL if device won't be eligible after transition
            db.insert_batch_action_device(..., is_success=False)
            continue
```

**Option B**: Process action FIRST, then assign workflow
```python
# Remove early return at Line 336
# Let normal action processing run
# Then assign workflow at the end

for device in devices:
    # Process normal action (service transition)
    # ... existing logic
    
    # After action completes, assign workflow
    if workflow_id:
        payload = {
            "action_id": "assign_solo_workflow",
            "workflow_id": workflow_id,
            # ...
        }
        send_message(payload, {}, BATCH_ACTION_PROCESSOR_SQS)
```

---

### Solution 3: Fix Query Logic ✅ STILL NEEDED

**File**: [batch_action_dispatcher/src/main.py Line 188](file:///home/thang/Documents/rsu/alps-ttp3-backend/modules/batch_action_dispatcher/src/main.py#L188)

**Add filter** (regardless of solution chosen):
```python
# After processing wrong_service_imeis
for imei in wrong_service_imeis:
    db.insert_batch_action_device(..., is_success=False)

# ✅ CRITICAL: Filter devices list
if device_type == DeviceType.SMARTPHONE:
    devices = [d for d in devices if d.imei not in wrong_service_imeis]
elif device_type == DeviceType.TABLET:
    devices = [d for d in devices if d.serial_number not in wrong_service_devices]
```

**Why**: Prevents duplicate FAIL records even with correct solution

---

## Recommended Fix Strategy

### Phase 1: Frontend Change (EASIEST)

1. **Change BulkAction component**:
   - Use `BatchUploadType.ASSIGN_ACTION` instead of `WORKFLOW_ACTION`
   - Pass `solo_workflow` in action payload
   
2. **Files to modify**:
   - `/alps-ttp3-frontend/src/components/BulkAction/index.tsx`

3. **Testing**:
   - INVENTORY IDLE → Register Prepaid + Workflow
   - Expected: Service transition happens, workflow assigned after
   - Result: 10,000 SUCCESS records

---

### Phase 2: Backend Safety Fix (CRITICAL)

1. **Filter devices list after wrong_service check**:
   - Prevents duplicate FAIL records
   
2. **File to modify**:
   - `/alps-ttp3-backend/modules/batch_action_dispatcher/src/main.py` (Line 188)

3. **Testing**:
   - Same test as Phase 1
   - Verify NO duplicate records

---

### Phase 3: Add Validation (OPTIONAL)

1. **Add warning for invalid workflow assignments**:
   - If user tries WORKFLOW_ACTION on IDLE devices
   - Show warning: "Devices must be in READY_FOR_USE, ACTIVE, or LOCKED state"

2. **File to modify**:
   - Frontend validation logic

---

## Impact Assessment

### Current Behavior (WRONG)
- ❌ INVENTORY IDLE → Register Prepaid + Workflow: 17,185 FAIL records
- ❌ No service transition
- ❌ Workflow never assigned
- ❌ Confusing user experience

### After Fix (CORRECT)
- ✅ INVENTORY IDLE → Register Prepaid + Workflow: 10,000 SUCCESS records
- ✅ Service transition: INVENTORY → PREPAID
- ✅ State transition: IDLE → READY_FOR_USE
- ✅ Workflow assigned after transition
- ✅ Clear user experience

---

## Related Files

1. [TTPNEO-1667-CHUNK-PROCESSING-BUG.md](file:///home/thang/Documents/rsu/copilot-rules/TTPNEO-1667-CHUNK-PROCESSING-BUG.md)
2. [TTPNEO-1667-MASSIVE-DUPLICATE-BUG.md](file:///home/thang/Documents/rsu/copilot-rules/TTPNEO-1667-MASSIVE-DUPLICATE-BUG.md)
3. [TTPNEO-1667-INVENTORY-REGISTER-WORKFLOW.md](file:///home/thang/Documents/rsu/copilot-rules/TTPNEO-1667-INVENTORY-REGISTER-WORKFLOW.md)
