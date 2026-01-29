# TTPNEO-1667 - Three Workflow Assignment Flows Analysis

## Overview

There are **3 DIFFERENT workflows** for assigning workflows to devices:

1. **Flow 1**: Batch Upload + Assign Solo Workflow (via Upload page)
2. **Flow 2**: Bulk Action - Assign Workflow to Device (dropdown action)
3. **Flow 3**: New Bulk Action + Register Service + Workflow (YOUR CASE!)

---

## Flow 1: Batch Upload + Assign Solo Workflow

### Entry Point
**Page**: `/device/upload` (Upload Devices page)

### User Journey
```
1. User uploads CSV with device IMEIs
2. User selects "Assign Solo Workflow" option
3. User picks a workflow from dropdown
4. System uploads devices AND assigns workflow
```

### Frontend Code

**File**: `/alps-ttp3-frontend/src/pages/Device/UploadDevice/index.tsx`

```typescript
// During upload process
await finalizeAction({
  uploadId: res.prepareUpload.uploadId,
  batchUploadType: BatchUploadTypes.UPLOAD, // Regular upload
  workflowId: selectedWorkflow.id,  // Include workflow
  serviceTypeId: selectedWorkflow.service_type_id
});
```

### Backend Flow

#### Resolver
**File**: `action_resolver/src/main.py`

```python
def finalize_batch_action(**kwargs, user_pool):
    batch_upload_type = kwargs.get("batch_upload_type")  # "UPLOAD"
    workflow_id = kwargs.get("workflow_id")
    service_type_id = kwargs.get("service_type_id")
    
    # Store workflow info in ext_fields
    ext_fields["pending_workflow_id"] = workflow_id
    ext_fields["pending_workflow_service_type_id"] = service_type_id
    
    # Send to dispatcher
    send_message(batch_action_dispatcher_payload, BATCH_ACTION_DISPATCHER_SQS)
```

#### Dispatcher
**File**: `batch_action_dispatcher/src/main.py`

```python
def batch_upload(**kwargs):
    batch_upload_type = "UPLOAD"
    workflow_id = kwargs.get("workflow_id")  # Present
    
    # Normal upload processing (NOT workflow specific)
    batch_handler.upload_records(...)
```

**Key**: This flow uses normal UPLOAD processing, workflow is assigned DURING normal device registration

---

## Flow 2: Bulk Action - Assign Workflow to Device

### Entry Point
**Component**: BulkAction dropdown → "Assign Workflow"

**Type**: `BulkActionType.WORKFLOW_ACTION`

### User Journey
```
1. User selects "Assign Workflow" from bulk action dropdown
2. User picks service (Prepaid/Postpaid/Sim Control)
3. User uploads CSV with devices ALREADY in valid states
4. System assigns workflow to existing devices
```

### Frontend Code

**File**: `/alps-ttp3-frontend/src/components/BulkAction/index.tsx`

```typescript
// Line 767-780
await finalizeAction({
  batchActionId: res.prepareBatchAction.batchActionId,
  batchUploadType: BatchUploadTypes.WORKFLOW_ACTION,  // ✅ WORKFLOW_ACTION
  workflowId: selectedWorkflow.id,
  serviceTypeId: selectedWorkflow.service_type_id,
  exitWorkflowReason: workflowAction === WORKFLOW_ACTION.EXIT_WORKFLOW 
    ? values?.exitWorkflowReason 
    : undefined,
});
```

### Backend Flow

#### Resolver
**File**: `action_resolver/src/main.py` (Line 805-810)

```python
def finalize_batch_action(**kwargs):
    workflow_id = kwargs.get("workflow_id")  # workflow UUID or "exit"
    
    if workflow_id == "exit":
        batch_upload_type = BatchUploadType.EXIT_WORKFLOW
    # else: batch_upload_type = BatchUploadType.WORKFLOW_ACTION (from FE)
```

#### Dispatcher
**File**: `batch_action_dispatcher/src/main.py` (Lines 242-336)

```python
def batch_action_upload(**kwargs):
    workflow_id = kwargs.get("workflow_id")
    batch_upload_type = kwargs.get("batch_upload_type")  # "WORKFLOW_ACTION"
    
    if workflow_id and batch_upload_type in [BatchUploadType.WORKFLOW_ACTION, BatchUploadType.EXIT_WORKFLOW]:
        # ✅ THIS PATH IS TAKEN
        
        allowed_states = [StateId.READY_FOR_USE, StateId.ACTIVE, StateId.LOCKED]
        
        for device in devices:
            # Validate state
            if device.state_id not in allowed_states:
                # Insert FAIL record
                db.insert_batch_action_device(..., is_success=False)
                continue
            
            # Validate tenant
            if device.tenant_id != tenant_id:
                # Insert FAIL record
                db.insert_batch_action_device(..., is_success=False)
                continue
            
            # Validate existing workflow
            device_workflow_details = get_billing_cycle_device_detail(device_uid, tenant_id)
            has_active_workflow = device_workflow_details and device_workflow_details.get('isActivatingWorkflow') is True
            
            if not is_exit_workflow and has_active_workflow:
                # Insert FAIL record (already has workflow)
                db.insert_batch_action_device(..., is_success=False)
                continue
            
            # SUCCESS: Insert record and send to processor
            batch_action_device = db.insert_batch_action_device(..., is_success=True)
            
            payload = {
                "action_id": "assign_solo_workflow",  # ✅ Workflow action
                "solo_workflow": {"workflow_id": workflow_id},
                ...
            }
            send_message(payload, BATCH_ACTION_PROCESSOR_SQS)
        
        return  # ✅ Early return - skip normal action processing
```

#### Processor
**File**: `batch_action_processor/src/main.py` (Line 109)

```python
def batch_action_processor(messages):
    for message in messages:
        if message.get("action_id") == "assign_solo_workflow":
            handle_workflow_assignment(db, message)
            continue
```

**File**: `batch_action_processor/src/actions/others/handle_workflow_assignment.py`

```python
def handle_workflow_assignment(db, message):
    # Send to ACTION_TRIGGER_SQS
    send_message(message, {}, queue_url=ACTION_TRIGGER_SQS)
    
    # Update batch_action_device status
    db.update_batch_action_device(schema_name, batch_action_device_id, is_success=True)
```

#### Trigger
**File**: `action_trigger/src/handlers/message_handler.py` (Line 158)

```python
def handle_message(conn, message):
    if payload_dto.action_id == "assign_solo_workflow":
        if payload_dto.solo_workflow and payload_dto.solo_workflow.workflow_id:
            workflow_handler.assign_solo_workflow_to_device(
                tenant_id=schema.tenant_id,
                device=device,  # Device in READY_FOR_USE/ACTIVE/LOCKED
                action=None,  # ❌ No action processing
                workflow_id=payload_dto.solo_workflow.workflow_id,
                ...
            )
        return
```

**File**: `action_trigger/src/workflow_handler.py` (Line 692)

```python
def assign_solo_workflow_to_device(...):
    url = WorkflowConfig.URL_TOKEN + f"/device/{device_uid}/assign_workflow"
    
    body = {
        "serviceType": service_type,
        "provisionType": device.provision_type,
        "stateId": str(device.state_id),  # Uses CURRENT state
        "workflowId": workflow_id,
        ...
    }
    
    response = requests.put(url=url, headers=headers, data=json.dumps(body))
```

**Key**: This flow ONLY assigns workflow, NO service/state transition

---

## Flow 3: New Bulk Action + Register Service + Workflow ⭐ YOUR CASE

### Entry Point
**Component**: BulkAction dropdown → "New bulk action"

**Type**: `BulkActionType.DEFAULT`

### User Journey
```
1. User selects "New bulk action" from dropdown
2. User selects "Apply for service": INVENTORY (current)
3. User selects "Apply for": Register Prepaid (target)
4. User OPTIONALLY selects a solo workflow
5. User uploads CSV with devices in IDLE state
6. System should:
   a. Register devices (IDLE → READY_FOR_USE)
   b. Change service (INVENTORY → PREPAID)
   c. Assign workflow (if selected)
```

### Frontend Code

**File**: `/alps-ttp3-frontend/src/components/BulkAction/index.tsx`

#### Line 798-817: Prepare workflow data
```typescript
const handleSubmit = async () => {
  const values = form.getFieldsValue();
  
  // Prepare batch action config
  const res = await prepareAction({
    rowCount: rowCountValid,
    config: data,  // Contains action states
    deviceType: ...,
    brandId: ...,
    provisionType: ...,
  });
  
  // Upload CSV
  await uploadFileWithPresignS3(res.prepareBatchAction.fileUpload, currentFile);
  
  // Finalize
  await finalizeAction({
    batchActionId: res.prepareBatchAction.batchActionId,
    batchUploadType: BatchUploadTypes.ASSIGN_ACTION,  // ✅ ASSIGN_ACTION (not WORKFLOW_ACTION!)
    billingDetail: {...},
    simControlId: ...,
    simControlName: ...,
    duration: ...,
    workflowId: values?.assignWorkflowId ? selectedWorkflow.id : undefined,  // ✅ Optional workflow
    serviceTypeId: values?.assignWorkflowId ? selectedWorkflow.service_type_id : undefined,
    exitWorkflowReason: ...,
  });
}
```

**Key Points**:
- `batchUploadType`: `ASSIGN_ACTION` (NOT `WORKFLOW_ACTION`)
- `workflowId`: Optional, passed if user selects workflow
- Config contains action states (IDLE device → REGISTER action)

### Backend Flow

#### Resolver
**File**: `action_resolver/src/main.py`

```python
def finalize_batch_action(**kwargs):
    batch_upload_type = "ASSIGN_ACTION"  # From frontend
    workflow_id = kwargs.get("workflow_id")  # Optional
    service_type_id = kwargs.get("service_type_id")  # Optional
    
    # Store workflow in ext_fields if provided
    if workflow_id and workflow_id != "exit":
        ext_fields["pending_workflow_id"] = workflow_id
        if service_type_id:
            ext_fields["pending_workflow_service_type_id"] = service_type_id
    
    # Send to dispatcher
    payload = {
        "batch_upload_type": "ASSIGN_ACTION",
        "workflow_id": workflow_id,  # Can be None
        "service_type_id": service_type_id,  # Can be None
        ...
    }
    send_message(payload, BATCH_ACTION_DISPATCHER_SQS)
```

#### Dispatcher - CURRENT IMPLEMENTATION (BUGGY)

**File**: `batch_action_dispatcher/src/main.py`

```python
def batch_action_upload(**kwargs):
    workflow_id = kwargs.get("workflow_id")  # Present!
    batch_upload_type = "ASSIGN_ACTION"
    service_type_id = kwargs.get("service_type_id")
    
    # Line 121: Query devices
    if workflow_id:
        # ❌ BUG: This branch is taken because workflow_id exists!
        devices = db.get_devices_for_workflow_bulk_action(
            schema_name, device_uids['query'], service_type_id
        )
        wrong_service_imeis = db.get_wrong_service_type_devices_for_workflow(
            schema_name, device_uids['query'], service_type_id
        )
    else:
        devices = db.get_devices_by_imeis(schema_name, device_uids['query'], True, service_type_id)
        wrong_service_imeis = db.get_devices_by_imeis(schema_name, device_uids['query'], False, service_type_id)
    
    # Line 180: Insert FAIL for wrong service
    for imei in wrong_service_imeis:
        db.insert_batch_action_device(..., is_success=False)  # WRONG_SERVICE_TYPE
    
    # Line 242: Check workflow path
    if workflow_id and batch_upload_type in [BatchUploadType.WORKFLOW_ACTION, BatchUploadType.EXIT_WORKFLOW]:
        # ❌ This condition is FALSE because batch_upload_type is "ASSIGN_ACTION"
        # So this block is SKIPPED
        pass
    
    # Line 338: Normal action processing
    for device in devices:
        # ✅ This SHOULD run for ASSIGN_ACTION
        # Process REGISTER action
        # Transition: IDLE → READY_FOR_USE
        # Change service: INVENTORY → PREPAID
        # ...
        
        # But workflow_id is present, so what happens to workflow assignment?
        # Answer: NOTHING! Workflow is ignored!
```

**CRITICAL BUG**: 
- Line 121: Takes `workflow_id` branch for queries
- Line 242: Skips workflow processing (wrong batch_upload_type check)
- Line 338: Runs normal action BUT ignores workflow

**Result**: 
- Devices get FAIL records (wrong service)
- Workflow NEVER assigned
- User expects: Service transition + workflow assignment
- Reality: Neither happens correctly

---

## Correct Flow 3 Implementation

### How It SHOULD Work

#### Option A: Use transition_new_service_handler ✅ RECOMMENDED

**File**: `action_trigger/src/transition_new_service_handler.py` (Line 92)

```python
def handle_transition_new_service(...):
    # 1. Process normal REGISTER action first
    # 2. Transition state: IDLE → READY_FOR_USE
    # 3. Change service: INVENTORY → PREPAID
    
    # 4. Then check for workflow
    if solo_workflow and solo_workflow.workflow_id and is_support_workflow_assignment_to_new_service(action.service_type_id):
        # ✅ Assign workflow AFTER service transition
        workflow_handler.assign_solo_workflow_to_device(
            tenant_id=tenant_id,
            device=device,  # Device now in READY_FOR_USE with PREPAID
            action=action,
            workflow_id=solo_workflow.workflow_id,
            ...
        )
```

**This is the CORRECT flow used when**:
- User registers device via UI
- Selects "Assign workflow" checkbox
- Device transitions service/state FIRST
- Workflow assigned AFTER transition

#### Option B: Fix Dispatcher Logic

**File**: `batch_action_dispatcher/src/main.py`

**Change 1**: Fix query logic (Line 121)
```python
# ❌ WRONG: if workflow_id:
# ✅ CORRECT: if workflow_id and batch_upload_type in [WORKFLOW_ACTION, EXIT_WORKFLOW]:

if workflow_id and batch_upload_type in [BatchUploadType.WORKFLOW_ACTION, BatchUploadType.EXIT_WORKFLOW]:
    devices = db.get_devices_for_workflow_bulk_action(...)
    wrong_service_imeis = db.get_wrong_service_type_devices_for_workflow(...)
else:
    devices = db.get_devices_by_imeis(schema_name, device_uids['query'], True, service_type_id)
    wrong_service_imeis = db.get_devices_by_imeis(schema_name, device_uids['query'], False, service_type_id)
```

**Change 2**: Pass workflow to normal action processing
```python
# Line 338: Normal action loop
for device in devices:
    # Process action
    payload = {
        "device_uid": device_uid,
        "action_id": action_id,
        ...
    }
    
    # ✅ ADD: Include workflow if present
    if workflow_id and batch_upload_type == BatchUploadType.ASSIGN_ACTION:
        payload["solo_workflow"] = {"workflow_id": workflow_id}
    
    send_message(payload, BATCH_ACTION_PROCESSOR_SQS)
```

**Change 3**: Processor forwards to trigger with workflow
```python
# batch_action_processor/src/main.py
# Normal action processing includes solo_workflow in payload
# Trigger's transition_new_service_handler picks it up
```

---

## Comparison Table

| Feature | Flow 1: Upload | Flow 2: Workflow Action | Flow 3: New Bulk Action |
|---------|----------------|------------------------|-------------------------|
| **Entry Point** | Upload page | Bulk action dropdown | Bulk action dropdown |
| **Bulk Action Type** | `UPLOAD` | `WORKFLOW_ACTION` | `ASSIGN_ACTION` |
| **Device State** | New (IDLE) | Existing (R4U/ACTIVE/LOCKED) | IDLE → R4U |
| **Service Transition** | Yes (during upload) | No | Yes (REGISTER) |
| **State Transition** | Yes (IDLE → R4U) | No | Yes (IDLE → R4U) |
| **Workflow Assignment** | During registration | Standalone | After transition |
| **Dispatcher Path** | Normal upload | Workflow-specific (Line 242) | Normal action (Line 338) |
| **Validates State** | No | Yes (must be R4U/ACTIVE/LOCKED) | No (will transition) |
| **Action Processed** | Register/Activate | None | Register/Change Service |
| **Current Status** | ✅ Working | ✅ Working | ❌ BROKEN |

---

## The Bug Explained

### What User Expects (Flow 3)
```
1. Upload CSV with 10,000 devices (INVENTORY, IDLE state)
2. Select "Register Prepaid" action
3. Select workflow to assign
4. Submit

Expected Result:
- All devices: IDLE → READY_FOR_USE
- All devices: INVENTORY service → PREPAID service
- All devices: Workflow assigned
- Result: 10,000 SUCCESS records
```

### What Actually Happens
```
1. Upload CSV with 10,000 devices
2. Dispatcher receives:
   - batch_upload_type = "ASSIGN_ACTION"
   - workflow_id = "abc-123-def"  ← Present!
   - service_type_id = 2  ← Prepaid

3. Line 121: if workflow_id:  ← TRUE, takes this branch
   - Queries devices with workflow-specific queries
   - wrong_service_imeis = all 10,000 (INVENTORY ≠ PREPAID)
   
4. Line 180: Insert 10,000 FAIL (WRONG_SERVICE_TYPE)

5. Line 242: if workflow_id and batch_upload_type in [WORKFLOW_ACTION, EXIT_WORKFLOW]:
   - FALSE (batch_upload_type is ASSIGN_ACTION)
   - Skip workflow processing

6. Line 338: Normal action loop
   - Processes devices (has overlap with wrong_service_imeis)
   - Some devices get SECOND FAIL (state validation)
   - Workflow is ignored (not passed to processor)

Result:
- 17,185 FAIL records (duplicates)
- NO service transition
- NO workflow assignment
- ❌ COMPLETE FAILURE
```

---

## Solution

### Recommended Fix: Update Dispatcher Logic

**File**: `batch_action_dispatcher/src/main.py`

#### Change 1: Fix Line 121 condition
```python
# Before:
if workflow_id:
    devices = db.get_devices_for_workflow_bulk_action(...)

# After:
if workflow_id and batch_upload_type in [BatchUploadType.WORKFLOW_ACTION, BatchUploadType.EXIT_WORKFLOW]:
    devices = db.get_devices_for_workflow_bulk_action(...)
else:
    devices = db.get_devices_by_imeis(...)
```

#### Change 2: Filter devices after wrong_service check (Line 188)
```python
for imei in wrong_service_imeis:
    db.insert_batch_action_device(..., is_success=False)

# ✅ ADD THIS
if device_type == DeviceType.SMARTPHONE:
    devices = [d for d in devices if d.imei not in wrong_service_imeis]
elif device_type == DeviceType.TABLET:
    devices = [d for d in devices if d.serial_number not in wrong_service_devices]
```

#### Change 3: Pass workflow to normal action (Line 338+)
```python
for device in devices:
    payload = {
        "device_uid": device_uid,
        "action_id": action_id,
        ...
    }
    
    # ✅ ADD: Include workflow for ASSIGN_ACTION
    if workflow_id and batch_upload_type == BatchUploadType.ASSIGN_ACTION:
        payload["solo_workflow"] = {"workflow_id": workflow_id}
    
    send_message(payload, BATCH_ACTION_PROCESSOR_SQS)
```

---

## Testing Plan

### Test Case 1: Flow 2 (Workflow Action) - Should still work
```
Input: 1,000 devices in READY_FOR_USE with PREPAID service
Action: Assign Workflow
Expected: 1,000 SUCCESS, all get workflow
```

### Test Case 2: Flow 3 (New Bulk Action) - Should now work
```
Input: 10,000 devices in IDLE with INVENTORY service
Action: Register Prepaid + Assign Workflow
Expected: 
- 10,000 SUCCESS
- All transition to READY_FOR_USE
- All change to PREPAID service
- All get workflow assigned
```

### Test Case 3: Flow 3 without workflow - Should work
```
Input: 10,000 devices in IDLE with INVENTORY service
Action: Register Prepaid (NO workflow selected)
Expected:
- 10,000 SUCCESS
- All transition to READY_FOR_USE
- All change to PREPAID service
- No workflow assigned
```

---

## Related Files

1. [TTPNEO-1667-STATE-LOGIC-BUG.md](file:///home/thang/Documents/rsu/copilot-rules/TTPNEO-1667-STATE-LOGIC-BUG.md)
2. [TTPNEO-1667-CHUNK-PROCESSING-BUG.md](file:///home/thang/Documents/rsu/copilot-rules/TTPNEO-1667-CHUNK-PROCESSING-BUG.md)
3. [TTPNEO-1667-INVENTORY-REGISTER-WORKFLOW.md](file:///home/thang/Documents/rsu/copilot-rules/TTPNEO-1667-INVENTORY-REGISTER-WORKFLOW.md)
