# Pending Queue Mechanism - How It Works

## Overview

Pending Queue (table `t_queue_action`) là cơ chế để store các actions khi device OFFLINE, và execute chúng khi device check-in.

## Core Flow

### 1. Action Assignment khi Device Offline

```
User assigns action → Device offline → Action vào pending queue
                                    ↓
                            Store in t_queue_action table
```

**Code**: [batch_action_processor/actions/action.py:478-499](file:///home/thang/Documents/rsu/alps-ttp3-backend/modules/batch_action_processor/src/actions/action.py#L478)

```python
is_queue_action = getattr(device, "is_queue_action", False)
if is_queue_action:
    queue_actions = db.get_pending_actions(schema_name, device_id)
    assigned_action_history_id = insert_milestone(...)
    
    payload["assigned_action_history_id"] = assigned_action_history_id
    payload["trigger_type"] = TriggerType.QUEUE
    
    db.insert_queue_action(schema_name, device, tenant_id, 
                          action_id, user_id, json.dumps(payload))
```

### 2. Device Check-in → Execute Pending Actions

```
Device check-in → action_pending_handler triggered
                       ↓
              Get FIRST pending action ONLY (LIMIT 1)
                       ↓
              Send to action-trigger
                       ↓
              Pop action from queue (DELETE)
                       ↓
              DONE - Wait for next check-in
```

**IMPORTANT**: Chỉ execute **1 ACTION** per check-in, KHÔNG PHẢI tất cả pending actions!

**Code**: [checkin/handlers.py:1593-1613](file:///home/thang/Documents/rsu/alps-ttp3-backend/modules/checkin/src/handlers.py#L1593)

```python
def action_pending_handler(conn, schema_name, device, sender_type):
    # Get ONLY ONE action (LIMIT 1)
    first_pending_action = db.get_pending_action(conn, schema_name, device.id)
    if not first_pending_action:
        return

    action_trigger_payload = first_pending_action.action_trigger_payload
    
    # Remove from queue BEFORE sending
    db.pop_pending_action(conn, schema_name, first_pending_action.id, device.id)
    
    # Send to action-trigger
    sender = get_sender(sender_type)
    response = sender.send(action_trigger_payload)
    
    # DONE - Function returns, other actions stay in queue
```

### 3. Database Operations

**Get Pending Action** - [checkin/db.py:1113-1125](file:///home/thang/Documents/rsu/alps-ttp3-backend/modules/checkin/src/db.py#L1113)
```sql
SELECT * FROM {schema_name}.t_queue_action
WHERE device_id = :device_id
ORDER BY assigned_at ASC  -- FIFO order
LIMIT 1                   -- ONLY ONE ACTION!
```

**Pop Pending Action** - [checkin/db.py:1135-1145](file:///home/thang/Documents/rsu/alps-ttp3-backend/modules/checkin/src/db.py#L1135)
```sql
DELETE FROM {schema_name}.t_queue_action
WHERE id = :queue_action_id AND device_id = :device_id
```

**Insert Queue Action** - [batch_action_processor/db.py:600-625](file:///home/thang/Documents/rsu/alps-ttp3-backend/modules/batch_action_processor/src/db.py#L600)
```sql
INSERT INTO {schema_name}.t_queue_action 
(tenant_id, action_id, device_id, assigned_by, action_trigger_payload)
VALUES (...)
```

---

## Your Question: What Happens with Multiple Actions?

### Scenario: Device has Assign Action + Workflow Action

```
Timeline:
T1: User assigns action (e.g., LOCK)     → Device offline → Goes to pending queue
T2: Workflow scheduled (e.g., Lock Msg)  → Device offline → ???
T3: Device check-in                      → Execute pending actions
```

### Answer: WORKFLOW ACTION ĐƯỢC THÊM VÀO QUEUE SAU ASSIGN ACTION

**WHY?** Because:

1. **Workflow action khi triggered mà device offline → ALSO goes to pending queue**
   - Workflow Processor pushes to SQS
   - Action Trigger consumes from SQS
   - Device offline → Action fails
   - **SHOULD** add to pending queue (this is the bug we found)

2. **Pending queue is FIFO (First In First Out)**
   - `ORDER BY assigned_at ASC LIMIT 1`
   - First action assigned = First action executed

3. **One action at a time**
   - `action_pending_handler` gets FIRST action only
   - Sends to action-trigger
   - Pops from queue
   - Next check-in → Gets next action

### Example Flow - Multiple Pending Actions

**Scenario**: Device has LOCK → LOCK_MSG in queue

```
Device ABC:

t_queue_action table BEFORE any check-in:
┌────┬────────────┬────────────────────┬─────────────┐
│ id │ device_id  │ action_type        │ assigned_at │
├────┼────────────┼────────────────────┼─────────────┤
│ 1  │ ABC        │ LOCK (assign)      │ 10:00:00    │  ← Oldest (execute first)
│ 2  │ ABC        │ LOCK_MSG (workflow)│ 10:05:00    │  ← Stays in queue
└────┴────────────┴────────────────────┴─────────────┘

1st Check-in at 10:15:00:
  → Get action id=1 (LIMIT 1)
  → Execute LOCK action
  → Pop id=1 from queue
  → DONE (id=2 STILL IN QUEUE)

t_queue_action table AFTER 1st check-in:
┌────┬────────────┬────────────────────┬─────────────┐
│ id │ device_id  │ action_type        │ assigned_at │
├────┼────────────┼────────────────────┼─────────────┤
│ 2  │ ABC        │ LOCK_MSG (workflow)│ 10:05:00    │  ← Now first in queue
└────┴────────────┴────────────────────┴─────────────┘

2nd Check-in at 10:20:00:
  → Get action id=2 (LIMIT 1)
  → Execute LOCK_MSG action
  → Pop id=2 from queue
  → DONE (queue now empty)

t_queue_action table AFTER 2nd check-in:
┌────┬────────────┬────────────────────┬─────────────┐
│ id │ device_id  │ action_type        │ assigned_at │
├────┼────────────┼────────────────────┼─────────────┤
│    │            │     (empty)        │             │
└────┴────────────┴────────────────────┴─────────────┘
```

**KEY POINT**: Device cần check-in **2 LẦN** để execute cả 2 actions!

---

## Key Points

### ONE Action Per Check-in

- **Only 1 action executed per check-in** (`LIMIT 1` in SQL)
- Device needs **MULTIPLE check-ins** to process all pending actions
- Actions execute in FIFO order (oldest first)

### Actions Are NOT Rejected

- **Workflow actions are NOT rejected when device has pending actions**
- **They are QUEUED** (or should be, after fix)
- **Executed ONE BY ONE** on subsequent check-ins

### Current Bug

**Problem**: Workflow actions khi fail (device offline > 75 min) → Go to DLQ → LOST

**Should Be**: Workflow actions khi fail → Add to pending queue → Execute on next check-in

### Why This Design? (One Action Per Check-in)

1. **Safer execution**: 
   - One action at a time → easier to track success/failure
   - If action fails, doesn't affect other pending actions

2. **Clear failure tracking**: 
   - Each action has its own execution result
   - Failed action doesn't block other actions

3. **Prevents action loss**: 
   - All actions eventually execute when device online
   - Actions stay in queue until successfully executed

4. **Preserves action order**: 
   - FIFO ensures actions execute in assignment order
   - Important for state-dependent actions (e.g., LOCK before LOCK_MSG)

5. **Simple recovery**: 
   - Just check-in multiple times
   - System automatically processes queue

---

## Related Code References

### Checkin Flow
- [checkin/handlers.py](file:///home/thang/Documents/rsu/alps-ttp3-backend/modules/checkin/src/handlers.py#L1593) - action_pending_handler
- [checkin/db.py](file:///home/thang/Documents/rsu/alps-ttp3-backend/modules/checkin/src/db.py#L1113) - get_pending_action
- [checkin/db.py](file:///home/thang/Documents/rsu/alps-ttp3-backend/modules/checkin/src/db.py#L1135) - pop_pending_action

### Action Assignment
- [batch_action_processor/actions/action.py](file:///home/thang/Documents/rsu/alps-ttp3-backend/modules/batch_action_processor/src/actions/action.py#L478) - Queue action logic
- [batch_action_processor/db.py](file:///home/thang/Documents/rsu/alps-ttp3-backend/modules/batch_action_processor/src/db.py#L600) - insert_queue_action
- [batch_action_processor/devices/device.py](file:///home/thang/Documents/rsu/alps-ttp3-backend/modules/batch_action_processor/src/devices/device.py#L236) - is_queue_action flag

### Milestone Tracking
- [batch_action_processor/milestone_handler.py](file:///home/thang/Documents/rsu/alps-ttp3-backend/modules/batch_action_processor/src/milestone_handler.py#L56) - insert_milestone
- Handles `more_days` calculation for queued actions
- Updates expiration time based on ALL queued actions

---

## Summary

**Question 1**: Khi device đang có assign action (trong pending queue) và workflow action được trigger thì điều gì xảy ra?

> **Answer**: Workflow action **ĐƯỢC THÊM VÀO QUEUE SAU** assign action, **KHÔNG BỊ REJECT**.

**Question 2**: Khi trong queue có LOCK → LOCK_MSG thì khi device check-in cả 2 actions đều được trigger chứ?

> **Answer**: **KHÔNG!** Chỉ **1 action duy nhất** được execute per check-in (`LIMIT 1`). Device cần check-in **2 LẦN** để execute cả 2 actions.
>
> - 1st check-in → Execute LOCK → Pop khỏi queue
> - 2nd check-in → Execute LOCK_MSG → Pop khỏi queue

**Execution Order**: FIFO (First In First Out) based on `assigned_at` timestamp.

**Current Bug**: Workflow actions không được add vào queue khi fail → Need to fix this.
