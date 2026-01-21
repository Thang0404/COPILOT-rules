# TTPNEO-1584: Root Cause Analysis - Exit Contract History Log Bug

## Problem Description

**Scenario:**
1. ROOT_BASIC_USER assigns contract to device
2. ADMIN exits contract from the same device
3. History tab shows "exit workflow" instead of "exit contract"

**Evidence from screenshot:**
```
18:50:24 - Workflow completed
18:50:23 - manh.pham@trustonic.com have requested to exit workflow manually with reason: caccaca
18:44:38 - sinh.vo@trustonic.com has assigned the contract brian-contract-simcontrolrule
```

The log says "exit workflow" but should say "exit contract".

---

## Root Cause Analysis

### Backend Logic (CORRECT)

File: `/home/thang/Documents/rsu/alps-ttp3-workflow/src/sam/functions/workflow_resolver/tasks/device.py`

```python
def exit_device_workflow(
    tenant_id: str, device_id: str, user_id: str, user_group_roles: List, 
    reason: Optional[str]=None, billing_cycle_id: Optional[str]=None, root_tenant_id: Optional[str]=None
):
    # Line 334-336: Milestone type selection based on billing_cycle_id
    milestone_type_requested = ActionTypeName.EXIT_CONTRACT_REQUESTED if billing_cycle_id else ActionTypeName.EXIT_WORKFLOW_REQUESTED
    milestone_type_rejected = ActionTypeName.EXIT_CONTRACT_REJECTED if billing_cycle_id else ActionTypeName.EXIT_WORKFLOW_REJECTED
    milestone_type_granted = ActionTypeName.EXIT_CONTRACT_GRANTED if billing_cycle_id else ActionTypeName.EXIT_WORKFLOW_GRANTED
```

✅ **Backend logic is CORRECT:**
- If `billing_cycle_id` exists → writes `EXIT_CONTRACT_REQUESTED` milestone
- If `billing_cycle_id` is None → writes `EXIT_WORKFLOW_REQUESTED` milestone

### Frontend Logic (CORRECT)

File: `/home/thang/Documents/rsu/alps-ttp3-frontend/src/components/Device/Detail/BillCycleTab/index.tsx`

```typescript
const onExitContract = async (values: any) => {
  const currentBillingCycleId = getCurrentBillingCycleId(data);  // Line 293
  const reason = ...;
  const res = await exitContractMutate({
    reason,
    billingCycleId: currentBillingCycleId,  // Line 300 - Passes billingCycleId
  });
```

```typescript
export const getCurrentBillingCycleId = (deviceBillingCycleData) => {
  const data = deviceBillingCycleData?.showDeviceBillingCycleDetail?.items;
  
  if (data?.length) {
    // Returns current billing cycle ID
    return sortedData[0]?._id;
  }
  return '';  // ❌ Returns empty string if no data!
};
```

✅ **Frontend logic is CORRECT** - passes `billingCycleId` to mutation

### The Real Problem: RBAC Permissions (BROKEN)

File: `/home/thang/Documents/rsu/alps-ttp3-workflow/src/sam/functions/workflow_resolver/tasks/rbac.py`

**BEFORE FIX (BROKEN):**
```python
GROUP_ROLE.ROOT_BASIC_USER: [
    {"key": "CONTRACT_VIEW", "value": False},  # ❌ Cannot see Contract tab
    {"key": "DEVICE_WORKFLOW_AND_BILL_CYCLE_VIEW", "value": False},  # ❌ Cannot query billing cycle data
]
```

**Impact Flow:**

```
ROOT_BASIC_USER assigns contract (via Assign Contract button)
  ↓
Contract assigned successfully in MongoDB
  ↓
ROOT_BASIC_USER tries to view Contract tab
  ↓
Frontend checks: canReadContract && canReadWorkFlowAndBillCycleview
  ↓
BOTH are FALSE → Contract tab HIDDEN ❌
  ↓
ROOT_BASIC_USER cannot see what they just assigned!
  ↓
---
Later: ADMIN tries to exit contract
  ↓
Frontend calls: getCurrentBillingCycleId(data)
  ↓
Query showDeviceBillingCycleDetail
  ↓
ADMIN has DEVICE_WORKFLOW_AND_BILL_CYCLE_VIEW = True
  ↓
BUT device was assigned by ROOT_BASIC_USER who had FALSE permission
  ↓
Billing cycle data exists in MongoDB but...
  ↓
GraphQL query returns data?.showDeviceBillingCycleDetail?.items = [] or undefined
  ↓
getCurrentBillingCycleId() returns '' (empty string)
  ↓
exitContractMutate({ billingCycleId: '' })  // Empty string!
  ↓
Backend receives billing_cycle_id = '' (falsy value)
  ↓
milestone_type = EXIT_WORKFLOW_REQUESTED (because not billing_cycle_id)
  ↓
History log shows "exit workflow" instead of "exit contract" ❌
```

---

## Solution

### Fix: Update RBAC Permissions

File: `/home/thang/Documents/rsu/alps-ttp3-workflow/src/sam/functions/workflow_resolver/tasks/rbac.py`

**AFTER FIX (CORRECT):**
```python
GROUP_ROLE.ROOT_BASIC_USER: [
    {"key": "CONTRACT_VIEW", "value": True},  # ✅ Can see Contract tab
    {"key": "DEVICE_WORKFLOW_AND_BILL_CYCLE_VIEW", "value": True},  # ✅ Can query billing cycle data
    {"key": "EXIT_CONTRACT", "value": False}  # ✅ Cannot exit contract (button hidden)
]

GROUP_ROLE.ROOT_BASIC_VIEWER: [
    {"key": "CONTRACT_VIEW", "value": True},  # ✅ Can see Contract tab
    {"key": "DEVICE_WORKFLOW_AND_BILL_CYCLE_VIEW", "value": True},  # ✅ Can query billing cycle data
    {"key": "EXIT_CONTRACT", "value": False}  # ✅ Cannot exit contract (button hidden)
]

GROUP_ROLE.ADMIN: [
    {"key": "CONTRACT_VIEW", "value": True},
    {"key": "DEVICE_WORKFLOW_AND_BILL_CYCLE_VIEW", "value": True},
    {"key": "EXIT_CONTRACT", "value": True}  # ✅ Can exit contract
]
```

### Fixed Flow:

```
ROOT_BASIC_USER assigns contract
  ↓
Contract assigned in MongoDB ✅
  ↓
ROOT_BASIC_USER views Device Detail
  ↓
CONTRACT_VIEW = True + DEVICE_WORKFLOW_AND_BILL_CYCLE_VIEW = True
  ↓
Contract tab VISIBLE ✅
  ↓
Query showDeviceBillingCycleDetail returns billing cycle data ✅
  ↓
ROOT_BASIC_USER can see assigned contract ✅
  ↓
EXIT_CONTRACT = False → Exit button HIDDEN ✅
  ↓
---
Later: ADMIN exits contract
  ↓
ADMIN has EXIT_CONTRACT = True → Exit button VISIBLE ✅
  ↓
getCurrentBillingCycleId(data) returns actual billing cycle ID ✅
  ↓
exitContractMutate({ billingCycleId: '507f1f77bcf86cd799439011' })
  ↓
Backend receives billing_cycle_id with valid MongoDB ObjectId
  ↓
milestone_type = EXIT_CONTRACT_REQUESTED ✅
  ↓
History log shows "exit contract" correctly ✅
```

---

## Permissions Matrix

| Role | CONTRACT_VIEW | DEVICE_WORKFLOW_AND_BILL_CYCLE_VIEW | EXIT_CONTRACT | Can See Tab? | Can Query Data? | Can Exit? |
|------|---------------|-------------------------------------|---------------|--------------|-----------------|-----------|
| ADMIN | True | True | True | ✅ | ✅ | ✅ |
| ROOT_STANDARD_USER | True | True | True | ✅ | ✅ | ✅ |
| ROOT_BASIC_OPERATOR | True | True | True | ✅ | ✅ | ✅ |
| ROOT_BASIC_SUPPORT | True | True | True | ✅ | ✅ | ✅ |
| ROOT_BASIC_DELIVERY | True | True | True | ✅ | ✅ | ✅ |
| **ROOT_BASIC_USER** | **True** | **True** | **False** | ✅ | ✅ | ❌ |
| **ROOT_BASIC_VIEWER** | **True** | **True** | **False** | ✅ | ✅ | ❌ |

---

## Why This Bug Only Happened with ROOT_BASIC_USER

1. **ROOT_BASIC_USER assigns contract** → Works (Assign Contract button bypasses RBAC for assignment)
2. **ROOT_BASIC_USER has no view permission** → Cannot see Contract tab or query billing cycle data
3. **Data exists in MongoDB** but ROOT_BASIC_USER cannot query it
4. **ADMIN tries to exit** → Frontend cannot get `billingCycleId` because data query returns empty
5. **Empty `billingCycleId`** → Backend treats as "exit workflow" not "exit contract"
6. **Wrong milestone type** → History log shows "exit workflow"

---

## Related Issues

### TTPNEO-1667: Exit Contract Working Correctly
- When user with ADMIN role both assigns AND exits → works correctly
- Because ADMIN has `DEVICE_WORKFLOW_AND_BILL_CYCLE_VIEW = True`
- Can query billing cycle data → `billingCycleId` is valid → correct milestone type

### TTPNEO-1584: Original Ticket
- Goal: ROOT_BASIC_USER/VIEWER can VIEW Contract tab but cannot EXIT
- Solution: 
  - Set `CONTRACT_VIEW = True` (show tab)
  - Set `DEVICE_WORKFLOW_AND_BILL_CYCLE_VIEW = True` (query data)
  - Set `EXIT_CONTRACT = False` (hide button)

---

## Testing Checklist

### Scenario 1: ROOT_BASIC_USER Assigns + ADMIN Exits
- [ ] ROOT_BASIC_USER assigns contract → Success
- [ ] ROOT_BASIC_USER sees Contract tab → ✅ Visible
- [ ] ROOT_BASIC_USER does NOT see Exit button → ✅ Hidden
- [ ] ROOT_BASIC_USER can see assigned contract details → ✅ Data loads
- [ ] ADMIN sees Exit button → ✅ Visible
- [ ] ADMIN exits contract → Success
- [ ] History log shows "exit contract" (not "exit workflow") → ✅ Correct

### Scenario 2: ADMIN Assigns + ADMIN Exits
- [ ] ADMIN assigns contract → Success
- [ ] ADMIN sees Contract tab + Exit button → ✅ Both visible
- [ ] ADMIN exits contract → Success
- [ ] History log shows "exit contract" → ✅ Correct

### Scenario 3: ROOT_BASIC_USER Assigns Solo Workflow
- [ ] ROOT_BASIC_USER should NOT see Workflow tab (WORKFLOW_VIEW = False)
- [ ] Verify permission doesn't break workflow assignment flow

---

## Summary

**Bug:** History log showed "exit workflow" instead of "exit contract" when ADMIN exited a contract that was assigned by ROOT_BASIC_USER.

**Root Cause:** ROOT_BASIC_USER had `DEVICE_WORKFLOW_AND_BILL_CYCLE_VIEW = False`, so billing cycle data couldn't be queried. Frontend sent empty `billingCycleId` to backend, causing wrong milestone type.

**Fix:** Set `DEVICE_WORKFLOW_AND_BILL_CYCLE_VIEW = True` for ROOT_BASIC_USER/VIEWER to allow querying billing cycle data while keeping `EXIT_CONTRACT = False` to hide exit button.

**Impact:** 
- ✅ ROOT_BASIC_USER can now see Contract tab
- ✅ ROOT_BASIC_USER can view assigned contracts
- ✅ ROOT_BASIC_USER cannot exit contracts (button hidden)
- ✅ ADMIN can exit contracts assigned by ROOT_BASIC_USER
- ✅ History log shows correct milestone type ("exit contract")
