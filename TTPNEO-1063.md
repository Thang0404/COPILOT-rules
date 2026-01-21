# TTPNEO-1063: [Independent Workflow][Bulk action][Multiservice]Missing Assign a Workflow modal when Using Bulk action change service

## Context
Hiện tại single device flow (Device Details) support change service action và assign workflow riêng biệt, nhưng KHÔNG cho phép assign workflow khi change service. Task này thêm UI để cho phép chọn solo workflow khi thực hiện change service action.

## Current State

### Change Service (Multi-Service Action)
- Support cho actions có flag `isMultiService: true`
- Flow: Service hiện tại + State hiện tại → Service mới + State mới
- Services hỗ trợ: TẤT CẢ services (Prepaid, Postpaid, SIM Control, Supply Chain, Inventory, Theft Defense, Subsidy Protection)
- States hỗ trợ: Tất cả states NGOẠI TRỪ IDLE, ENROLLED, RELEASED (Supply Chain cho phép RELEASED)
- Backend logic: `isAssignNewServiceType: true` trong action ext_fields
- Location: `ActionForm/index.tsx` line 900-940

### Solo Workflow
- Support cho Register actions và Change Service actions
- Services hỗ trợ: TẤT CẢ services có trong SERVICE_WORKFLOW_MAPPERS (Prepaid, Postpaid, SIM Control, Supply Chain, Inventory, Theft Defense, Subsidy Protection)
- Điều kiện query: `isSoloWorkflow: true`, `isActive: true` và device phải ở states valid (không phải IDLE, RELEASED)
- Constraint hiện tại (Register actions): Disabled khi đã chọn Contract (mutual exclusion)
- Location: `ActionForm/index.tsx` line 1152-1180

## Task Requirements

### Frontend Changes
1. Thêm workflow dropdown vào ActionForm khi action type là change service (isMultiService)
2. Remove constraint disabled giữa Workflow và Contract - chỉ cần cho phép chọn Workflow (không quan tâm Contract)
3. Fetch danh sách workflows dựa vào service type mới (target service sau khi change)
4. Pass workflowId vào assignAction mutation khi submit

### Key Notes
- Workflow dropdown chỉ show khi: action.isMultiService === true AND canReadWorkflow permission
- Service type để fetch workflows: extract từ action.serviceName (target service sau khi change)
- Validation: Workflow optional, không bắt buộc phải chọn
- Backend API assignAction đã support nhận workflowId parameter
- Tất cả services đều support solo workflow (không chỉ Prepaid/Postpaid/SIM Control)
- Không cần constraint với Contract vì change service action không có Contract dropdown

### Implementation Steps
1. Kiểm tra action.isMultiService để hiển thị workflow section
2. Extract target service name từ selectedAction.actionName 
3. Set selectedServiceType state để trigger fetch workflows
4. Add workflow Form.Item với dropdown (tương tự Register action hiện có)
5. Update onFinish để include workflowId khi có values.assignWorkflowId
6. Test với different service transitions (ví dụ: Prepaid/Protected → Postpaid/Locked)
