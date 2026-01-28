# Temporary Lock Action Assignment Flow

## Overview

Quy trình assign Temporary Lock action từ Frontend thông qua AppSync, TTP Platform, TTP Workflow, tới DocumentDB và các background processors.

## System Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                         PHASE 1: List Actions                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Frontend                                                        │
│    ↓ (GraphQL Query)                                             │
│  AppSync (listWorkflowActions)                                   │
│    ↓                                                             │
│  TTP Platform - list_workflow_actions()                         │
│    ↓                                                             │
│  DocumentDB - Check workflow activation                         │
│    ↓ (YES/NO)                                                    │
│  Add/Remove Temp Lock from action list                          │
│    ↓                                                             │
│  Return to Frontend                                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    PHASE 2: Assign Action                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Frontend (User selects Temp Lock)                               │
│    ↓ (GraphQL Mutation)                                          │
│  AppSync → assign_temp_lock endpoint                             │
│    ↓                                                             │
│  TTP Workflow API PUT /device/{device_uid}/action/assign/temp-lock
│    ↓                                                             │
│  START TRANSACTION                                               │
│    ├─ Find Device by device_uid                                 │
│    ├─ Create Device if not exists                               │
│    ├─ Get current action assigned                               │
│    ├─ Update old action status → CANCELLED                      │
│    ├─ Create DeviceActionDB (status=WAITING)                    │
│    ├─ Calculate next_process_time = now + (duration * 60)       │
│    ├─ Insert DeviceActionHistory                                │
│    └─ Invoke platform_data_sync Lambda                          │
│  END TRANSACTION                                                 │
│    ↓                                                             │
│  DocumentDB Save & Commit                                        │
│    ↓                                                             │
│  Response to Frontend (Success)                                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│               PHASE 3: Background Processing                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  SQS (device-temporary-lock-entry-sqs)                           │
│    ↓                                                             │
│  device-temporary-lock-entry Lambda                             │
│    ↓                                                             │
│  Step Function (device-temporary-lock-timing-processor-sfn)     │
│    ├─ Check: next_process_time <= now() ?                       │
│    ├─ Update status WAITING → ACTIVATED (on device check-in)    │
│    └─ Update status ACTIVATED → DEACTIVATED (when duration end) │
│    ↓                                                             │
│  Scheduler (15min interval)                                      │
│  device-temporary-lock-schedule-processor-sfn                   │
│    ├─ Query all devices with lock_duration pending              │
│    └─ Update status DEACTIVATED if time expired                 │
│    ↓                                                             │
│  SNS: device-monitoring-status-changed                          │
│    ↓                                                             │
│  Device Monitoring Event Processor                               │
│    └─ Trigger check-in action on RP platform                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Implementation Details

### Phase 1: List Available Actions

**File:** [main.graphql](file:///home/thang/Documents/rsu/alps-ttp3-backend/appsync/main.graphql#L2140)
- GraphQL Query: `listWorkflowActions(tenantId, serviceType, isSoloWorkflow)`

**File:** [platform_data_sync.py](file:///home/thang/Documents/rsu/alps-ttp3-workflow/src/sam/functions/workflow_resolver/tasks/platform_data_sync.py#L14)
- Function: `list_workflow_actions()` - Fetch actions từ TTP Platform
- Logic: Nếu có workflow_steps → Call filtered endpoint, else → Call all actions endpoint

### Phase 2: Assign Temporary Lock

**File:** [device.py - API Endpoint](file:///home/thang/Documents/rsu/alps-ttp3-workflow/src/apiservice/api/api_v1/endpoints/device.py#L162)
- Endpoint: `PUT /device/{device_uid}/action/assign/temp-lock`
- Payload: `AssignDeviceTempLockActionRequest`

**File:** [temp_lock.py - Controller](file:///home/thang/Documents/rsu/alps-ttp3-workflow/src/apiservice/controller/temp_lock.py#L153)
- Function: `transactional_assign_temp_lock_to_device()` - Wrap transaction
- Function: `assign_temp_lock_to_device()` (Line 50) - Main logic
  - Find or create device
  - Cancel old action if exists
  - Create new DeviceActionDB with status=WAITING
  - Set next_process_time = now + (duration * 60 seconds)

**File:** [temp_lock.py - Status Update](file:///home/thang/Documents/rsu/alps-ttp3-workflow/src/apiservice/controller/temp_lock.py#L24)
- Function: `update_device_action_status()` - Update action status
  - If status=ACTIVATED & action=TEMP_LOCK → Calculate next_process_time

### Phase 3: Update Device Action Status

**File:** [device.py - Update Status Endpoint](file:///home/thang/Documents/rsu/alps-ttp3-workflow/src/apiservice/api/api_v1/endpoints/device.py#L173)
- Endpoint: `PUT /device/{device_uid}/action/status`
- Payload: `UpdateDeviceActionStatusRequest`
- Logic: Validate, find action, update status → Response

**File:** [temp_lock.py - Status Update Logic](file:///home/thang/Documents/rsu/alps-ttp3-workflow/src/apiservice/controller/temp_lock.py#L25)
- Function: `update_device_action_status()` (Line 25)
  - If status=ACTIVATED & action=TEMP_LOCK → Calculate `next_process_time = now + (duration * 60 seconds)`
  - Save to DocumentDB

### Phase 4: Background Processors

**Device Temporary Lock Timing Processor** (Step Function)
- Query working hours configuration
- Calculate next available timing
- If `next_process_time > 15min` → Prepare SQS events for delays
- Update devices `next_process_time`, `status=WAITING`
- Send to `device-temporary-lock-process-sqs`

**Device Temporary Lock Schedule Processor** (15min Scheduler)
- Pull devices: `action=temp_lock, status in [WAITING, ACTIVATED], next_process_time < 15min`
- Prepare SQS events với `delays=timing`
- Send to `device-temporary-lock-process-sqs`

**Device Temporary Lock Requests Executor** (SQS Consumer)
- Map devices action (ACTIVATE or DEACTIVATE)
- Group devices by (tenant, provision_type, service_type)
- Filter empty groups
- For each group: Prepare request payload
- Call `cf-util-api` to send lock/unlock command
- Update device action status to ACTIVATING/DEACTIVATING
- Call `workflow-service-api` to update status

### Phase 5: Cleanup & Unlock

**File:** [postpaid_handler.py](file:///home/thang/Documents/rsu/alps-ttp3-backend/modules/checkin/src/postpaid_handler.py#L30)
- Function: `handle()` - Device check-in handler
- Logic: Check if `is_templock_checkin_not_end()` → Keep lock or unlock

**File:** [utils.py - Duration Check](file:///home/thang/Documents/rsu/alps-ttp3-backend/modules/checkin/src/utils.py#L311)
- Function: `is_templock_checkin_not_end()` (Line 311)
  - Check: `now() - first_temp_lock_checkin_at < lock_duration * 60` ?
  - Return True nếu vẫn trong duration, False nếu hết hạn

**File:** [utils.py - First Time Calculation](file:///home/thang/Documents/rsu/alps-ttp3-backend/modules/checkin/src/utils.py#L484)
- Function: `get_first_time_templock()` - Calculate actual lock start time
  - Handle working hours logic
  - Handle holidays logic

## Data Models

### DeviceActionDB Schema (DocumentDB)
```
{
  tenant_id: String (indexed)
  device_id: String
  device_uid: String (indexed)
  action: Action.TEMP_LOCK
  status: ActionStatus (WAITING|ACTIVATED|DEACTIVATED)
  duration: Int (minutes)
  next_process_time: Int (Unix timestamp in seconds)
  notification: Dict
  is_enabled_working_hour: Boolean
  created_at: DateTime
  updated_at: DateTime
}
```

### Device ext_fields Storage
```
{
  lock_duration: Int (minutes)
  first_temp_lock_checkin_at: Int (Unix timestamp - set on first check-in)
}
```

## Key Points

1. **Transaction:** `transactional_assign_temp_lock()` wraps entire operation trong MongoDB transaction
2. **Duration Unit:** Stored in MINUTES, but `next_process_time` calculated in SECONDS
3. **Status Flow:** WAITING → ACTIVATED (device check-in) → DEACTIVATED (hết hạn hoặc override)
4. **Working Hours:** Logic xử lý trong `get_first_time_templock()` - nếu enable, start time adjust to working hours
5. **Cleanup:** `first_temp_lock_checkin_at` & `lock_duration` removed từ ext_fields khi hết hạn
