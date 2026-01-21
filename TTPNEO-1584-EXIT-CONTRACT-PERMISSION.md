# TTPNEO-1584: Fix Exit Contract Button Permission

## Overview

Fix permission logic cho Exit Contract button để ROOT_BASIC_USER và ROOT_BASIC_VIEWER có thể xem Contract tab nhưng KHÔNG thể exit contract.

---

## Problem

Khi thêm permission cho ROOT_BASIC_USER/VIEWER để view Contract/Workflow tabs (TTPNEO-1584), phát hiện Exit button visibility bị couple với view permission, gây ra conflict:
- Dùng `canUpdate(Resource.Contract)` để control Exit button
- ROOT_BASIC_USER/VIEWER cần xem Contract tab nhưng KHÔNG được phép exit

---

## Solution Architecture

### Backend Flow

```
User Login
  ↓
Get user roles from Cognito groups
  ↓
rbac.py get_workflow_user_accessible_lists()
  ↓
role_mapped_permissions[highest_role]
  ↓
Return EXIT_CONTRACT permission
  ↓
GraphQL Response: {"key": "EXIT_CONTRACT", "value": "true"/"false"}
```

### Frontend Flow

```
AdditionalFeatureProvider (Context)
  ↓
Query: getWorkflowUserAccessibleList
  ↓
Parse response: EXIT_CONTRACT → boolean
  ↓
Store in userWorkflowAccessibles state
  ↓
Device Detail Component
  ↓
canExitContract = canReadWorkflowFeature(WorkflowAccessibles.EXIT_CONTRACT)
  ↓
Pass to BillCycleTab
  ↓
<BillCycleTab canExitContract={canExitContract} />
  ↓
Exit Button: {canExitContract && <Button>Exit Contract</Button>}
```

---

## Implementation Details

### 1. Backend Changes

#### [rbac.py](file:///home/thang/Documents/rsu/alps-ttp3-workflow/src/sam/functions/workflow_resolver/tasks/rbac.py#L20-L73)

Added EXIT_CONTRACT key to all roles in role_mapped_permissions:

```python
role_mapped_permissions = {
    GROUP_ROLE.ROOT_BASIC_USER: [
        {"key": "EXIT_CONTRACT", "value": False}  # Cannot exit
    ],
    GROUP_ROLE.ROOT_BASIC_VIEWER: [
        {"key": "EXIT_CONTRACT", "value": False}  # Cannot exit
    ],
    GROUP_ROLE.ADMIN: [
        {"key": "EXIT_CONTRACT", "value": True}   # Can exit
    ],
    # OPERATOR, SUPPORT, DELIVERY, STANDARD_USER: True
}
```

#### [Return string format](file:///home/thang/Documents/rsu/alps-ttp3-workflow/src/sam/functions/workflow_resolver/tasks/rbac.py#L149)

Convert boolean to string "true"/"false" for GraphQL compatibility:

```python
resource_list = [
    {"key": item["key"], "value": str(item["value"]).lower()} 
    for item in role_mapped_permissions[highest_role]
]
```

### 2. Frontend Changes

#### [AdditionalFeature Context](file:///home/thang/Documents/rsu/alps-ttp3-frontend/src/contexts/AdditionalFeature.tsx#L47)

Added EXIT_CONTRACT to WorkflowAccessibles enum:

```typescript
export enum WorkflowAccessibles {
  EXIT_CONTRACT = 'EXIT_CONTRACT',
  // ... other keys
}
```

#### [Parse permission response](file:///home/thang/Documents/rsu/alps-ttp3-frontend/src/contexts/AdditionalFeature.tsx#L88-L97)

Parse GraphQL response and store in context:

```typescript
const workflowAccessable = data.getWorkflowUserAccessibleList.resource.reduce(
  (result, workflowFeature) => ({
    ...result,
    [workflowFeature?.key as WorkflowAccessibles]: 
      workflowFeature?.value === 'true' || workflowFeature?.value === 'True',
  }),
  {},
);
setUserWorkflowAccessibles(workflowAccessable);
```

#### [Device Detail Component](file:///home/thang/Documents/rsu/alps-ttp3-frontend/src/components/Device/Detail/index.tsx#L185)

Get permission from context:

```typescript
const canExitContract = canReadWorkflowFeature(WorkflowAccessibles.EXIT_CONTRACT);
```

#### [Pass to BillCycleTab](file:///home/thang/Documents/rsu/alps-ttp3-frontend/src/components/Device/Detail/index.tsx#L816)

```typescript
<BillCycleTab
  // ... other props
  canExitContract={canExitContract}
/>
```

#### [BillCycleTab Component](file:///home/thang/Documents/rsu/alps-ttp3-frontend/src/components/Device/Detail/BillCycleTab/index.tsx#L61)

Updated component interface and usage:

```typescript
export type BillCycleProps = {
  canExitContract?: boolean;
  // ... other props
};

function BillCycleTab({ 
  canExitContract = false,  // Default to false for safety
  ...
}: BillCycleProps) {
  // Exit button conditional rendering
  canExitContract && (
    <Button onClick={() => setIsOpenModalExitContract(true)}>
      {t('exitContract')}
    </Button>
  )
}
```

---

## Permission Matrix

| Role | CONTRACT_VIEW | EXIT_CONTRACT | Can See Tab? | Can Exit? |
|------|---------------|---------------|--------------|-----------|
| ADMIN | True | True | ✅ | ✅ |
| ROOT_STANDARD_USER | True | True | ✅ | ✅ |
| ROOT_BASIC_OPERATOR | True | True | ✅ | ✅ |
| ROOT_BASIC_SUPPORT | True | True | ✅ | ✅ |
| ROOT_BASIC_DELIVERY | True | True | ✅ | ✅ |
| ROOT_BASIC_USER | True | **False** | ✅ | ❌ |
| ROOT_BASIC_VIEWER | True | **False** | ✅ | ❌ |

---

## Edge Cases Handled

### 1. Non-Root Tenant
Backend returns permission list WITHOUT EXIT_CONTRACT key:
```python
if not is_root_tenant:
    return {"resource": [
        {"key": "CONTRACT_VIEW", "value": contract_flag},
        # EXIT_CONTRACT not included
    ]}
```

Frontend handles missing key:
```typescript
canExitContract = false  // Default value when key missing
```

**Result:** Exit button hidden for all non-root tenant users ✅

### 2. Contract Not Activated
Similar to non-root tenant - EXIT_CONTRACT not in response:
```python
if not contract_flag:
    return {"resource": [
        {"key": "CONTRACT_VIEW", "value": False},
        # EXIT_CONTRACT not included
    ]}
```

**Result:** Exit button hidden when contract feature disabled ✅

### 3. Permission Format Compatibility
Backend converts Python boolean → string:
```python
str(True).lower()   # "true"
str(False).lower()  # "false"
```

Frontend handles both formats:
```typescript
value === 'true' || value === 'True'
```

**Result:** Compatible with different Python/GraphQL serialization ✅

---

## Testing Checklist

### Backend Tests
- [ ] ROOT_BASIC_USER returns EXIT_CONTRACT=false
- [ ] ROOT_BASIC_VIEWER returns EXIT_CONTRACT=false  
- [ ] ADMIN returns EXIT_CONTRACT=true
- [ ] Non-root tenant does NOT return EXIT_CONTRACT key
- [ ] Contract not activated does NOT return EXIT_CONTRACT key
- [ ] Value format is string "true"/"false" (lowercase)

### Frontend Tests
- [ ] ROOT_BASIC_USER sees Contract tab but NO exit button
- [ ] ROOT_BASIC_VIEWER sees Contract tab but NO exit button
- [ ] ADMIN sees Contract tab AND exit button
- [ ] Exit button click triggers modal (for allowed roles)
- [ ] Non-root tenant users never see exit button
- [ ] Missing EXIT_CONTRACT key defaults to false (button hidden)

### Integration Tests
- [ ] Click exit button → creates EXIT_CONTRACT_REQUESTED milestone
- [ ] History log shows "exit contract" (not "exit workflow")
- [ ] Billing cycle ID present → EXIT_CONTRACT milestone type
- [ ] Billing cycle ID missing → EXIT_WORKFLOW milestone type

---

## Debug Test File

Created [test_rbac_exit_contract.py](file:///home/thang/Documents/rsu/alps-ttp3-workflow/test_rbac_exit_contract.py) to simulate RBAC logic:

```bash
cd /home/thang/Documents/rsu/alps-ttp3-workflow
python3 test_rbac_exit_contract.py
```

Tests:
1. EXIT_CONTRACT permission for all 7 roles
2. Non-root tenant scenario (EXIT_CONTRACT missing)
3. Contract not activated (EXIT_CONTRACT missing)
4. Milestone type selection logic (EXIT_CONTRACT vs EXIT_WORKFLOW)

---

## Related Code References

### Backend
- [role_mapped_permissions definition](file:///home/thang/Documents/rsu/alps-ttp3-workflow/src/sam/functions/workflow_resolver/tasks/rbac.py#L20-L73)
- [get_workflow_user_accessible_lists logic](file:///home/thang/Documents/rsu/alps-ttp3-workflow/src/sam/functions/workflow_resolver/tasks/rbac.py#L109-L160)
- [EXIT_CONTRACT constants](file:///home/thang/Documents/rsu/alps-ttp3-workflow/src/sam/functions/workflow_resolver/const.py#L10-L12)
- [Milestone type selection](file:///home/thang/Documents/rsu/alps-ttp3-workflow/src/sam/functions/workflow_resolver/tasks/device.py#L334-L336)

### Frontend
- [WorkflowAccessibles enum](file:///home/thang/Documents/rsu/alps-ttp3-frontend/src/contexts/AdditionalFeature.tsx#L47-L53)
- [Permission parsing](file:///home/thang/Documents/rsu/alps-ttp3-frontend/src/contexts/AdditionalFeature.tsx#L88-L97)
- [Device Detail usage](file:///home/thang/Documents/rsu/alps-ttp3-frontend/src/components/Device/Detail/index.tsx#L185)
- [BillCycleTab prop](file:///home/thang/Documents/rsu/alps-ttp3-frontend/src/components/Device/Detail/BillCycleTab/index.tsx#L61)
- [Exit button conditional](file:///home/thang/Documents/rsu/alps-ttp3-frontend/src/components/Device/Detail/BillCycleTab/index.tsx#L417)

---

## Summary

✅ **Completed:**
- Added EXIT_CONTRACT permission key to backend RBAC
- Set EXIT_CONTRACT=False for ROOT_BASIC_USER and ROOT_BASIC_VIEWER
- Updated frontend context to handle EXIT_CONTRACT permission
- Modified Device Detail to fetch and pass permission to child component
- Updated BillCycleTab to use canExitContract prop instead of ResourcePermission
- Converted Python boolean to string for GraphQL compatibility
- Handled edge cases (non-root tenant, contract not activated)

✅ **Result:**
- ROOT_BASIC_USER/VIEWER can view Contract tab but cannot exit contract
- All other roles can view AND exit contract
- Separation of view and update permissions achieved
- No impact on non-root tenants or when contract feature is disabled
