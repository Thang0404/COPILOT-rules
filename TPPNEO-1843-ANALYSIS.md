# TPPNEO-1843: Update Expiration Inconsistency - Prepaid Fixed Contract

## Problem Summary

**Inconsistent behavior** khi update expiration date với Prepaid fixed contract assigned:
- UI: Block (không cho phép) update expiration
- API: Allow (cho phép) update expiration successfully
- Bulk: Allow (cho phép) update expiration successfully

## Root Cause Analysis

### Current Implementation

#### 1. Frontend - Block Logic (PropertyTab)

**File:** [PropertyTab/index.tsx line 677](file:///home/thang/Documents/rsu/alps-ttp3-frontend/src/components/Device/Detail/PropertyTab/index.tsx#L677)

```
Edit button disabled = contractMode === CONTRACT_MODE.FIX
```

**Contract Modes:**
- `FIX` (Fixed): Update expiration BLOCKED ❌
- `VARIABLE` (Variable): Update expiration ALLOWED with constraints
  - Can only set future date (not past)
  - Time picker disabled for hours < current hour

**Logic flow:**
1. Device Detail loads `contractMode` from contract assignment
2. When `contractMode === 'FIX'`, Edit button is disabled
3. User cannot click edit to update expiration

#### 2. Backend - API Validation

**File:** [north_bound_api/update_device_expiration](file:///home/thang/Documents/rsu/alps-ttp3-backend/modules/north_bound_api/north_bound_proxy_v2/src/handlers/__init__.py#L27)

Current behavior:
- Accepts update expiration requests for fixed contracts
- No validation blocking fixed contracts
- Returns success

#### 3. Backend - Bulk Action Validation

**File:** Bulk update handler (action_trigger or batch processor)

Current behavior:
- Processes update expiration via bulk pipeline
- No validation blocking fixed contracts
- Returns success

---

## Implementation Points

### Phase 1: Understand Business Rules (Per Attachment)

From copilot-rules attachment:
- **Fixed Contract** (`isFixed = true`): `không cho update expiration` = NOT ALLOWED
- **Variable Contract** (`isFixed = false`): `cho update nhưng không được là quá khứ` = ALLOWED but no past dates

---

## Flow Diagrams

### Current State - 3 Different Behaviors

```
Prepaid Device with Fixed Contract Assigned
    ↓
┌─────────────────────────────┬──────────────────────────┬─────────────────────────┐
│         UI Flow             │      API Flow            │     Bulk Flow           │
├─────────────────────────────┼──────────────────────────┼─────────────────────────┤
│ 1. Device Detail            │ 1. updateDeviceProperties│ 1. prepareBatchAction   │
│    loaded                   │    mutation              │                         │
│    ↓                        │    ↓                     │    ↓                    │
│ 2. contractMode             │ 2. Backend resolver      │ 2. Dispatcher service   │
│    = 'FIX'                  │    receives request      │                         │
│    ↓                        │    ↓                     │    ↓                    │
│ 3. Edit button              │ 3. NO validation block   │ 3. NO validation block  │
│    DISABLED ✓               │    ↓                     │    ↓                    │
│    ↓                        │ 4. Updates database      │ 4. Updates database     │
│ 4. Cannot edit              │    ↓                     │    ↓                    │
│    Block ✓                  │ 5. Returns SUCCESS ✗    │ 5. Returns SUCCESS ✗    │
│                             │    (Should block)        │    (Should block)       │
└─────────────────────────────┴──────────────────────────┴─────────────────────────┘

INCONSISTENCY: UI blocks but API/Bulk allow
```

### Expected State - Option 1: Block Everywhere (Recommended)

```
Prepaid Device with Fixed Contract Assigned
    ↓
ALL paths: Update Expiration BLOCKED ✓
    ↓
┌─────────────────────────────┬──────────────────────────┬─────────────────────────┐
│         UI Flow             │      API Flow            │     Bulk Flow           │
├─────────────────────────────┼──────────────────────────┼─────────────────────────┤
│ Edit button DISABLED        │ Validation: is_fixed     │ Validation: is_fixed    │
│ (Check contractMode='FIX')  │ = true → Reject          │ = true → Reject         │
│                             │ Return ERROR             │ Return ERROR            │
└─────────────────────────────┴──────────────────────────┴─────────────────────────┘
```

### Expected State - Option 2: Allow Everywhere

```
Prepaid Device with Fixed Contract Assigned
    ↓
ALL paths: Update Expiration ALLOWED ✓
    ↓
┌─────────────────────────────┬──────────────────────────┬─────────────────────────┐
│         UI Flow             │      API Flow            │     Bulk Flow           │
├─────────────────────────────┼──────────────────────────┼─────────────────────────┤
│ Edit button ENABLED         │ Validation: None         │ Validation: None        │
│ (Remove contractMode check) │ Return SUCCESS           │ Return SUCCESS          │
└─────────────────────────────┴──────────────────────────┴─────────────────────────┘
```

---

## Evidence & References

**UI Code:**
- [PropertyTab/index.tsx line 677: disabled button](file:///home/thang/Documents/rsu/alps-ttp3-frontend/src/components/Device/Detail/PropertyTab/index.tsx#L677)
- [PropertyTab/index.tsx line 547: contractMode logic](file:///home/thang/Documents/rsu/alps-ttp3-frontend/src/components/Device/Detail/PropertyTab/index.tsx#L547)

**Backend Constants:**
- [device_resolver/const.py: UPDATE_EXPIRATION_REQ milestone](file:///home/thang/Documents/rsu/alps-ttp3-backend/modules/device_resolver/src/const.py#L131)
- [device_resolver/helpers.py: trigger_update_expiration_time](file:///home/thang/Documents/rsu/alps-ttp3-backend/modules/device_resolver/src/helpers.py#L229)

**Workflow Service:**
- [assign_billing_cycle.py: Prepaid contract assignment logic](file:///home/thang/Documents/rsu/alps-ttp3-workflow/src/apiservice/controller/assign_billing_cycle.py#L1135)

---

## Implementation Strategy

### Option A: BLOCK Everywhere (Most Secure - Recommended)

**Rationale:**
- Fixed contracts have locked expiration by definition
- Prevents accidental updates via APIs
- Consistent with business rule: "fixed contract: không cho update expiration"

**Changes needed:**
1. Frontend: Keep existing block (✓ Already works)
2. Backend API: Add validation to reject fixed contracts
3. Backend Bulk: Add validation to reject fixed contracts

### Option B: ALLOW Everywhere

**Rationale:**
- More flexible for edge cases
- API/Bulk users have more control
- Allows manual overrides if needed

**Changes needed:**
1. Frontend: Remove contractMode === 'FIX' check
2. Backend API: No changes needed (already allows)
3. Backend Bulk: No changes needed (already allows)

---

## Decision Required

**Choose one:**
1. Option A: Block update expiration for fixed contracts in API & Bulk
2. Option B: Allow update expiration for fixed contracts in UI

**Recommendation:** Option A (Block everywhere) - aligns with business rule and security principle
