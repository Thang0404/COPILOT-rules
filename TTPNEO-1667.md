# TTPNEO-1667: Contract Assign Success Message Wrong + Missing History Milestone

## Problem

Issue: Assign Contract success toast and history
- Toast shows "Workflow assigned successfully" instead of "Contract assigned successfully"
- Device History tab empty after assign contract (no milestone)
- Exit Contract already working correctly (has milestone in History tab)
- Assign Solo Workflow already working correctly (has milestone in History tab)


## Flow Analysis - 3 Separate Flows

### Flow 1: Exit Contract (WORKING - Has Milestone)

```
User clicks Exit Contract
  ↓
[Frontend calls exitWorkflowAndContractFromDevice](file:///home/thang/Documents/rsu/alps-ttp3-workflow/src/apiservice/controller/device.py)
  ↓ REST API to Workflow Service
[remove_workflow_and_contract_from_device](file:///home/thang/Documents/rsu/alps-ttp3-workflow/src/apiservice/controller/device.py)
  ↓ Deletes MongoDB records
  ↓ Calls Backend API to write milestone
[Backend writes EXIT_WORKFLOW_REQUESTED milestone](file:///home/thang/Documents/rsu/alps-ttp3-backend/modules/action_trigger/src/workflow_handler.py#L244)
  - milestone_type = EXIT_WORKFLOW_REQUESTED
  - ext_fields = {"exitWorkflowReason": reason}
  ↓
Device History tab shows: "Contract schema removed" with reason ✓ WORKING
```

### Flow 2: Assign Solo Workflow (WORKING - Has Milestone)

### Flow 2: Assign Solo Workflow (WORKING - Has Milestone)

```
User clicks Assign Workflow
  ↓
[Frontend handleAssignWorkflow](file:///home/thang/Documents/rsu/alps-ttp3-frontend/src/components/Device/Detail/index.tsx#L441)
  ↓ mutation assignWorkflowToDevice
[Workflow API assign_workflow_to_device](file:///home/thang/Documents/rsu/alps-ttp3-workflow/src/apiservice/api/api_v1/endpoints/device.py#L221)
  ↓ Creates DeviceWorkflowStepDB in MongoDB
  ↓ Returns success
Frontend shows toast: "Workflow assigned successfully" ✓ WORKING
  ↓
Later: Backend Action Trigger handles pending workflow after device registers
[assign_solo_workflow_to_device](file:///home/thang/Documents/rsu/alps-ttp3-backend/modules/action_trigger/src/workflow_handler.py#L596)
  - milestone_handler.add_custom_milestone()
  - milestone_type = ASSIGN_WORKFLOW_REQUESTED
  - ext_fields = {"workflowName": name}
  ↓
Device History tab shows: Workflow assignment milestone ✓ WORKING
```

### Flow 3: Assign Contract (BROKEN - No Milestone)

### Flow 3: Assign Contract (BROKEN - No Milestone)

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. User Assigns Contract from Device Detail Page               │
│    [handleAssignContractSchema](file:///home/thang/Documents/rsu/alps-ttp3-frontend/src/components/Device/Detail/BillCycleTab/components/AssignContractButton.tsx#L218)                                       │
│    - Calls mutation reassignBillingCycle                        │
└──────────────────┬──────────────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. Workflow Service Processes Request                          │
│    [assign_billing_cycle_to_device](file:///home/thang/Documents/rsu/alps-ttp3-workflow/src/apiservice/controller/assign_billing_cycle.py#L482)                          │
│    - Creates DeviceBillCycleDB in MongoDB                       │
│    - Creates DeviceWorkflowStepDB in MongoDB                    │
│    - Updates DeviceDB.is_activating_contract = True            │
│    NO Backend API call to write milestone                      │
└──────────────────┬──────────────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. Frontend Shows Wrong Toast                                  │
│    [Line 248: t('assignSuccess')](file:///home/thang/Documents/rsu/alps-ttp3-frontend/src/components/Device/Detail/BillCycleTab/components/AssignContractButton.tsx#L248)                                │
│    BUG 1: Shows "Workflow assigned successfully"               │
│    Should show "Contract assigned successfully"                │
└──────────────────┬──────────────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. Device History Tab Empty                                    │
│    BUG 2: No milestone in PostgreSQL t_device_history          │
│    History tab shows nothing for assign contract action        │
└─────────────────────────────────────────────────────────────────┘
```


## Root Cause

Comparison with working flows:

Exit Contract (WORKING):
- [Calls Backend API](file:///home/thang/Documents/rsu/alps-ttp3-backend/modules/action_trigger/src/workflow_handler.py#L244) to write EXIT_WORKFLOW_REQUESTED milestone
- History shows "Contract schema removed"

Assign Workflow (WORKING):
- Backend writes ASSIGN_WORKFLOW_REQUESTED milestone after device registration
- History shows workflow assignment

Assign Contract (BROKEN):
- [Workflow Service](file:///home/thang/Documents/rsu/alps-ttp3-workflow/src/apiservice/controller/assign_billing_cycle.py#L482) only writes MongoDB
- NO call to Backend API to write milestone
- [Wrong toast key](file:///home/thang/Documents/rsu/alps-ttp3-frontend/src/components/Device/Detail/BillCycleTab/components/AssignContractButton.tsx#L248) uses `assignSuccess` instead of `assignContractSuccess`


## Implementation Plan

### Phase 1: Fix Toast Message

Files to edit:
1. [AssignContractButton.tsx line 248](file:///home/thang/Documents/rsu/alps-ttp3-frontend/src/components/Device/Detail/BillCycleTab/components/AssignContractButton.tsx#L248)
   - Change `t('assignSuccess')` to `t('assignContractSuccess')`

2. [en.json translations](file:///home/thang/Documents/rsu/alps-ttp3-frontend/src/i18n/locales/en.json)
   - Add key: `"assignContractSuccess": "Contract assigned successfully"`
   - Add key: `"assignContractFailed": "Assign contract failed"`

3. [es.json translations](file:///home/thang/Documents/rsu/alps-ttp3-frontend/src/i18n/locales/es.json)
   - Add key: `"assignContractSuccess": "Contrato asignado exitosamente"`
   - Add key: `"assignContractFailed": "Error al asignar contrato"`


### Phase 2: Add Milestone Constants

Files to edit:
1. [Backend const.py line 123](file:///home/thang/Documents/rsu/alps-ttp3-backend/modules/device_resolver/src/const.py#L123)
   - Add constant: `ASSIGN_CONTRACT_REQUESTED = "ASSIGN_CONTRACT_REQUESTED"`

2. [Backend const.py NON_STATE_PRE_MILESTONE](file:///home/thang/Documents/rsu/alps-ttp3-backend/modules/device_resolver/src/const.py#L152)
   - Add `ASSIGN_CONTRACT_REQUESTED` to list

3. [Backend const.py NON_STATE_MILESTONE](file:///home/thang/Documents/rsu/alps-ttp3-backend/modules/device_resolver/src/const.py#L156)
   - Add `ASSIGN_CONTRACT_REQUESTED` to list

4. [Action Trigger constant.py](file:///home/thang/Documents/rsu/alps-ttp3-backend/modules/action_trigger/src/constant.py)
   - Add to MileStoneType class: `ASSIGN_CONTRACT_REQUESTED = "ASSIGN_CONTRACT_REQUESTED"`


### Phase 3: Create Backend Milestone API

Files to edit:
1. [north_bound_proxy_v2 handlers.py](file:///home/thang/Documents/rsu/alps-ttp3-backend/modules/north_bound_api/north_bound_proxy_v2/src/handlers.py)
   - Add function: `add_device_milestone(event, context)`
   - Get device_uid from path params
   - Get milestone_type and ext_fields from body
   - Call milestone_handler.add_custom_milestone()

2. [north_bound_proxy_v2 serverless.yml](file:///home/thang/Documents/rsu/alps-ttp3-backend/modules/north_bound_api/north_bound_proxy_v2/serverless.yml)
   - Add endpoint: `POST /api/v2/device/{device_uid}/milestone`


### Phase 4: Call Backend API from Workflow Service

Files to edit:
1. [Workflow assign_billing_cycle.py after line 638](file:///home/thang/Documents/rsu/alps-ttp3-workflow/src/apiservice/controller/assign_billing_cycle.py#L638)
   - Add helper function: `write_contract_milestone()`
   - Use httpx.AsyncClient to POST to Backend API
   - Call after device.save(session=session)
   - Pass contract_name in ext_fields

2. [Workflow settings.py](file:///home/thang/Documents/rsu/alps-ttp3-workflow/src/apiservice/settings.py)
   - Add config: `BACKEND_API_URL = os.getenv("BACKEND_API_URL", "https://api.dev.ttp.rsu.com")`


## Testing

Phase 1:
- Assign contract from UI shows "Contract assigned successfully" (EN)
- Shows "Contrato asignado exitosamente" (ES)

Phase 2 + 3 + 4:
- Assign contract creates milestone in t_device_history table
- Device History tab shows milestone with contract name
- Milestone type = ASSIGN_CONTRACT_REQUESTED




### chỗ này tao tự note, không được sửa, không được xoá
- hiện tại nút exit contract trong tab contract và nút exit workflow trong tab workflow đều gọi graghql exitWorkflow với tham số giống nhau
- backend sẽ phân biệt exit contract hay exit workflow dựa vào việc có truyền billingCycleVersionId hoặc billingCycleId hay không
- nếu có thì là exit contract, không có thì là exit workflow
- tương tự khi viết milestone, backend cũng phân biệt 2 trường hợp này để viết milestone tương ứng: EXIT_CONTRACT_REQUESTED hoặc EXIT_WORKFLOW_REQUESTED 