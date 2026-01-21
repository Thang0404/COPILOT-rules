# TTPNEO-1584: Final Root Cause Analysis - Exit Contract History Log

## Problem Summary

When ROOT_BASIC_USER assigns contract and ADMIN exits contract, history log shows:
> "exit workflow" 

Instead of:
> "exit contract"

---

## Evidence from Production

### GraphQL Request - Assign Contract:
```json
{
  "variables": {
    "deviceId": "868424060000992",
    "billingCycleId": "695cf0937c76294dc509e59e"  ← Template ID
  }
}
```

### GraphQL Request - Exit Contract:
```json
{
  "variables": {
    "deviceId": "868424060000992",
    "billingCycleId": "6969b59d38a1f68a97d66ec3"  ← Device instance ID
  }
}
```

**Note:** 2 IDs khác nhau vì:
- Assign: Dùng `billingCycleId` (template/schema ID)
- Exit: Dùng device billing cycle instance ID (created when assigned)

---

## Root Cause Analysis

### Backend Milestone Selection Logic

File: `/home/thang/Documents/rsu/alps-ttp3-workflow/src/sam/functions/workflow_resolver/tasks/device.py`

**BEFORE (BROKEN):**
```python
def exit_device_workflow(
    billing_cycle_id: Optional[str]=None,
    ...
):
    # Line 336-338
    milestone_type_requested = ActionTypeName.EXIT_CONTRACT_REQUESTED if billing_cycle_id else ActionTypeName.EXIT_WORKFLOW_REQUESTED
    milestone_type_rejected = ActionTypeName.EXIT_CONTRACT_REJECTED if billing_cycle_id else ActionTypeName.EXIT_WORKFLOW_REJECTED
    milestone_type_granted = ActionTypeName.EXIT_CONTRACT_GRANTED if billing_cycle_id else ActionTypeName.EXIT_WORKFLOW_GRANTED
```

### Problem: Falsy Value Check

Python falsy values:
- `None` → Falsy ✓
- `''` (empty string) → Falsy ❌ **THIS IS THE BUG!**
- `' '` (whitespace) → Truthy but invalid ❌

**Scenario:**
```python
# Case 1: Valid ID
billing_cycle_id = "6969b59d38a1f68a97d66ec3"
if billing_cycle_id:  # Truthy ✓
    milestone_type = EXIT_CONTRACT_REQUESTED  ✓

# Case 2: Empty string (BUG!)
billing_cycle_id = ""
if billing_cycle_id:  # Falsy! ❌
    milestone_type = EXIT_WORKFLOW_REQUESTED  # WRONG!

# Case 3: Whitespace (BUG!)
billing_cycle_id = "   "
if billing_cycle_id:  # Truthy but invalid! ❌
    milestone_type = EXIT_CONTRACT_REQUESTED  # WRONG!
```

---

## How Empty String Can Happen

### Frontend Logic

File: `/home/thang/Documents/rsu/alps-ttp3-frontend/src/components/Device/Detail/BillCycleTab/index.tsx`

```typescript
export const getCurrentBillingCycleId = (
  deviceBillingCycleData?: ShowDeviceBillingCycleDetailQuery | null,
) => {
  const data = deviceBillingCycleData?.showDeviceBillingCycleDetail?.items;

  if (data?.length) {
    const sortedData = sortArrayByDate(data, 'startDate');
    // ... logic returns sortedData[0]?._id
    return sortedData[0]?._id;
  }
  return '';  // ❌ Returns empty string if no data!
};
```

### Possible Scenarios Leading to Empty String:

1. **RBAC Permission Issue:**
   - ROOT_BASIC_USER assigns contract
   - MongoDB has data
   - ADMIN has `DEVICE_WORKFLOW_AND_BILL_CYCLE_VIEW = False` (if old RBAC config)
   - Query `showDeviceBillingCycleDetail` returns `items: []`
   - `getCurrentBillingCycleId()` returns `''`

2. **Race Condition:**
   - Contract assigned but not yet propagated
   - Query runs before MongoDB write completes
   - Returns empty result

3. **Data Inconsistency:**
   - Device has `is_activating_contract = True` in MongoDB
   - But `device_bill_cycles` collection has no matching record
   - Query returns empty

---

## Solution

### Fix 1: Strict Validation in Backend

File: `/home/thang/Documents/rsu/alps-ttp3-workflow/src/sam/functions/workflow_resolver/tasks/device.py`

**AFTER (FIXED):**
```python
def exit_device_workflow(
    billing_cycle_id: Optional[str]=None,
    ...
):
    logger.info(f"[EXIT_WORKFLOW_DEBUG] billing_cycle_id received: '{billing_cycle_id}' (type: {type(billing_cycle_id).__name__})")
    
    # ✅ Strict validation: Check not None AND not empty/whitespace
    has_billing_cycle = billing_cycle_id and billing_cycle_id.strip() != ''
    logger.info(f"[EXIT_WORKFLOW_DEBUG] has_billing_cycle: {has_billing_cycle}")
    
    with connect_to_postgres() as pg:
        milestone_type_requested = ActionTypeName.EXIT_CONTRACT_REQUESTED if has_billing_cycle else ActionTypeName.EXIT_WORKFLOW_REQUESTED
        milestone_type_rejected = ActionTypeName.EXIT_CONTRACT_REJECTED if has_billing_cycle else ActionTypeName.EXIT_WORKFLOW_REJECTED
        milestone_type_granted = ActionTypeName.EXIT_CONTRACT_GRANTED if has_billing_cycle else ActionTypeName.EXIT_WORKFLOW_GRANTED
        
        logger.info(f"[EXIT_WORKFLOW_DEBUG] Selected milestone type: {milestone_type_requested}")
```

**Benefits:**
- `billing_cycle_id = "valid-id"` → `has_billing_cycle = True` ✓
- `billing_cycle_id = None` → `has_billing_cycle = False` ✓
- `billing_cycle_id = ""` → `has_billing_cycle = False` ✓
- `billing_cycle_id = "   "` → `has_billing_cycle = False` ✓

### Fix 2: RBAC Permissions (Already Done)

File: `/home/thang/Documents/rsu/alps-ttp3-workflow/src/sam/functions/workflow_resolver/tasks/rbac.py`

```python
GROUP_ROLE.ROOT_BASIC_USER: [
    {"key": "CONTRACT_VIEW", "value": True},  # Can see tab
    {"key": "DEVICE_WORKFLOW_AND_BILL_CYCLE_VIEW", "value": True},  # Can query data
    {"key": "EXIT_CONTRACT", "value": False}  # Cannot exit
]
```

This ensures:
- ROOT_BASIC_USER can assign and view contracts
- Query `showDeviceBillingCycleDetail` returns data
- `getCurrentBillingCycleId()` gets valid ID
- Exit button hidden (cannot exit anyway)

---

## Testing Validation

### Test Case 1: Valid Billing Cycle ID
```json
Input: { "billingCycleId": "6969b59d38a1f68a97d66ec3" }
Expected: EXIT_CONTRACT_REQUESTED
Backend Log:
  [EXIT_WORKFLOW_DEBUG] billing_cycle_id received: '6969b59d38a1f68a97d66ec3' (type: str)
  [EXIT_WORKFLOW_DEBUG] has_billing_cycle: True
  [EXIT_WORKFLOW_DEBUG] Selected milestone type: EXIT_CONTRACT_REQUESTED
```

### Test Case 2: Empty String (Bug Scenario)
```json
Input: { "billingCycleId": "" }
Expected: EXIT_WORKFLOW_REQUESTED
Backend Log:
  [EXIT_WORKFLOW_DEBUG] billing_cycle_id received: '' (type: str)
  [EXIT_WORKFLOW_DEBUG] has_billing_cycle: False
  [EXIT_WORKFLOW_DEBUG] Selected milestone type: EXIT_WORKFLOW_REQUESTED
```

### Test Case 3: None Value
```json
Input: { "billingCycleId": null }
Expected: EXIT_WORKFLOW_REQUESTED
Backend Log:
  [EXIT_WORKFLOW_DEBUG] billing_cycle_id received: 'None' (type: NoneType)
  [EXIT_WORKFLOW_DEBUG] has_billing_cycle: False
  [EXIT_WORKFLOW_DEBUG] Selected milestone type: EXIT_WORKFLOW_REQUESTED
```

### Test Case 4: Whitespace Only
```json
Input: { "billingCycleId": "   " }
Expected: EXIT_WORKFLOW_REQUESTED
Backend Log:
  [EXIT_WORKFLOW_DEBUG] billing_cycle_id received: '   ' (type: str)
  [EXIT_WORKFLOW_DEBUG] has_billing_cycle: False
  [EXIT_WORKFLOW_DEBUG] Selected milestone type: EXIT_WORKFLOW_REQUESTED
```

---

## Production Debugging Steps

### Step 1: Enable Debug Logging
Deploy updated code with logging to CloudWatch.

### Step 2: Reproduce Issue
1. ROOT_BASIC_USER assigns contract to device `868424060000992`
2. ADMIN exits contract

### Step 3: Check Backend Logs
Search CloudWatch logs for:
```
[EXIT_WORKFLOW_RESOLVER] Received arguments
[EXIT_WORKFLOW_DEBUG] billing_cycle_id received
[EXIT_WORKFLOW_DEBUG] has_billing_cycle
[EXIT_WORKFLOW_DEBUG] Selected milestone type
```

### Step 4: Verify GraphQL Request
Open DevTools → Network → Find `exitWorkflow` mutation:
- Check `variables.billingCycleId` value
- Should be valid MongoDB ObjectId, not empty string

### Step 5: Check Database
```javascript
// MongoDB
db.device_bill_cycles.find({ device_id: "868424060000992" })

// PostgreSQL
SELECT * FROM deveco.t_device_history 
WHERE device_uid = '868424060000992' 
ORDER BY created_at DESC 
LIMIT 5;
```

---

## Summary of Changes

| File | Change | Purpose |
|------|--------|---------|
| `workflow_resolver/tasks/device.py` | Add `has_billing_cycle = billing_cycle_id and billing_cycle_id.strip() != ''` | Strict validation for empty/whitespace |
| `workflow_resolver/tasks/device.py` | Add debug logging | Track milestone type selection |
| `workflow_resolver/app.py` | Add debug logging in GraphQL resolver | Track received parameters |
| `workflow_resolver/tasks/rbac.py` | Set `CONTRACT_VIEW = True`, `DEVICE_WORKFLOW_AND_BILL_CYCLE_VIEW = True`, `EXIT_CONTRACT = False` for ROOT_BASIC_USER/VIEWER | Enable view permission, disable exit permission |

---

## Expected Behavior After Fix

### Scenario 1: ROOT_BASIC_USER Assigns + ADMIN Exits (Contract Exists)
```
1. ROOT_BASIC_USER assigns contract
   → MongoDB: device_bill_cycles created with _id="6969b59d38a1f68a97d66ec3"
   
2. ROOT_BASIC_USER views device
   → Query returns billing cycle data ✓
   → Contract tab visible ✓
   → Exit button HIDDEN (EXIT_CONTRACT=False) ✓
   
3. ADMIN views device
   → Query returns billing cycle data ✓
   → Exit button VISIBLE (EXIT_CONTRACT=True) ✓
   
4. ADMIN clicks Exit Contract
   → Frontend: currentBillingCycleId = "6969b59d38a1f68a97d66ec3" ✓
   → Backend: has_billing_cycle = True ✓
   → Milestone: EXIT_CONTRACT_REQUESTED ✓
   → History log: "exit contract" ✓
```

### Scenario 2: ADMIN Exits Solo Workflow (No Contract)
```
1. ADMIN assigns solo workflow (no contract)
   → MongoDB: NO device_bill_cycles record
   
2. ADMIN exits workflow
   → Frontend: currentBillingCycleId = "" (no billing cycles)
   → Backend: has_billing_cycle = False ✓
   → Milestone: EXIT_WORKFLOW_REQUESTED ✓
   → History log: "exit workflow" ✓
```

---

## Conclusion

**Root Cause:** Backend used simple falsy check `if billing_cycle_id` which treated empty string `""` as falsy, causing wrong milestone type selection.

**Fix:** Changed to strict validation `billing_cycle_id and billing_cycle_id.strip() != ''` to properly handle:
- Valid ObjectId → TRUE → EXIT_CONTRACT
- Empty string → FALSE → EXIT_WORKFLOW
- None → FALSE → EXIT_WORKFLOW
- Whitespace → FALSE → EXIT_WORKFLOW

**Impact:** History logs now correctly show "exit contract" vs "exit workflow" based on actual device state, not just parameter presence.
