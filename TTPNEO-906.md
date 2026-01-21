Workflow Processor 
  ↓ (trigger action)
Action Trigger 
  ↓ (send to CF-Utils)
CF-Utils process_action 
  ↓ (call OEM API)
OGuard/RGuard Service 
  ↓ (response with result=SUCCESS + error object)
OGuardErrorParserStrategy 
  ↓ (parse error incorrectly)
ErrorHandler 
  ↓ (send to ERROR_HANDLER_SQS)
error_handler lambda 
  ↓ (insert error record)
t_device_errors table ❌







┌─────────────────────────────────────────────────────────────────┐
│ 1. Backend Action Trigger Lambda                                │
│    - Executes action (Lock/Message)                            │
│    - Calls CF-Utils → Samsung API                              │
│    - Samsung returns: {result: SUCCESS, error: DUPLICATE}      │
└──────────────────┬──────────────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. Backend Marks Action Complete in DB                         │
│    - Updates t_assigned_action_history → APPLIED               │
│    - Updates device state                                       │
│    - Updates device.assigned_action_id                         │
│    - Updates device.expiry_time (if applicable)                │
└──────────────────┬──────────────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. Workflow Service - Device Monitoring Webhook                │
│    File: device_monitoring.py::ingest_check_in_device()        │
│    - Backend calls: POST /device-monitoring/check-in           │
│    - Updates DeviceDB with new action_assigned_id              │
│    - Detects change: action_assigned_id CHANGED                │
│    - Publishes SNS: DEVICE_STATUS_CHANGED                      │
└──────────────────┬──────────────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. Lambda: workflow_device_event_processor                     │
│    File: workflow_device_event_processor/app.py                │
│    - Consumes SNS event                                         │
│    - Calls: device_status_change_handler.handle()              │
└──────────────────┬──────────────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────────────┐
│ 5. Handler Detects Multiple Conditions                         │
│    File: device_status_change_handler.py                       │
│                                                                  │
│    ⚠️ BUG HERE - Multiple triggers in ONE handler call:        │
│                                                                  │
│    A. Status Change Check:                                      │
│       if item.new_status != item.old_status:                   │
│          → calculate_workflow_step(STATE_CONDITION)            │
│          → Calls update_device_solo_workflow_step API          │
│          → Generates workflow steps                             │
│          → TRIGGERS ACTION #1 🔴                                │
│                                                                  │
│    B. Expiry Time Change (for Prepaid):                        │
│       if item.expiry_time != item.old_expiry_time:             │
│          → calculate_workflow_step(EXPIRY_TIME_CONDITION)      │
│          → Calls update_device_solo_workflow_step API          │
│          → Generates workflow steps                             │
│          → TRIGGERS ACTION #2 🔴                                │
│                                                                  │
│    C. Action Assigned Check:                                    │
│       if item.action_assigned_id:                              │
│          → calculate_workflow_step(ACTION_CONDITION)           │
│          → Calls update_device_solo_workflow_step API          │
│          → Generates workflow steps                             │
│          → COULD TRIGGER ACTION #3 🔴                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘