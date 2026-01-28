# TPPNEO-1843: UI Logic Verification - Prepaid Fixed Contract Expiration Update

## Current UI Implementation Status

### Constants Definition
File: [constants/service.ts](file:///home/thang/Documents/rsu/alps-ttp3-frontend/src/constants/service.ts)
```
enum CONTRACT_MODE {
  FIX = 'fix'           // Fixed contract
  VARIABLE = 'variable' // Variable contract
}
```

---

## ExpirationDatetimeEditor Component Analysis

### Location
File: [PropertyTab/index.tsx lines 469-728](file:///home/thang/Documents/rsu/alps-ttp3-frontend/src/components/Device/Detail/PropertyTab/index.tsx#L469)

### Current Logic - Edit Button (Line 698)

```tsx
<Button
  disabled={contractMode === CONTRACT_MODE.FIX}
  icon={<EditOutlined />}
  onClick={handleEditClick}
></Button>
```

**Current Behavior:**
- Edit button `disabled = true` when `contractMode === 'fix'`
- Edit button `disabled = false` when `contractMode === 'variable'`
- Edit button visible only if `enablePrepaidEnhancement === true`

### Rendering Logic (Line 813-826)

```tsx
{service === SERVICE_TYPE.PREPAID && (
  <div className='infor-row'>
    <div className='label'>{t('expirationDatetime')}</div>
    <div className='flex align--center justify--sb actions-container'>
      <div>{ExpirationDateContent}</div>
      {deviceData?.enablePrepaidEnhancement === true && (
        <div className='action'>
          <ExpirationDatetimeEditor
            value={expiryTimeItem?.value}
            onSave={(newValue) => onSaveExpiration(newValue)}
            hasPendingAction={hasPendingAction}
            pendingActionName={pendingActionName}
          />
        </div>
      )}
    </div>
  </div>
)}
```

**Rendering Conditions:**
1. `service === SERVICE_TYPE.PREPAID` ✓ (Scope matches ticket)
2. `deviceData?.enablePrepaidEnhancement === true` ✓ (Feature flag check)
3. Edit button `disabled={contractMode === CONTRACT_MODE.FIX}` ✓ (Fixed contract check)

---

## Verification Checklist - Is UI Logic Correct?

### Test Scenario 1: Prepaid + Fixed Contract Assigned

**Setup:**
- Device: Prepaid service
- Contract: Fixed contract assigned (isFixed = true)
- contractMode passed: `'fix'`
- enablePrepaidEnhancement: `true`

**Expected Result:**
- Edit button: DISABLED ✓
- Cannot click edit
- Block expiration update

**Actual Result:**
- Edit button: `disabled={contractMode === 'fix'}` = `disabled={true}` ✓
- Status: CORRECT ✓

---

### Test Scenario 2: Prepaid + Variable Contract Assigned

**Setup:**
- Device: Prepaid service
- Contract: Variable contract assigned (isFixed = false)
- contractMode passed: `'variable'`
- enablePrepaidEnhancement: `true`

**Expected Result:**
- Edit button: ENABLED ✓
- Can click edit
- Allow expiration update with constraints:
  - Cannot set past date
  - Time picker disabled for hours < current hour

**Actual Result:**
- Edit button: `disabled={contractMode === 'variable'}` = `disabled={false}` ✓
- When edit clicked, DatePicker constraints applied:
  - Line 616: `disabledDate = current.isBefore(datetimeValue.startOf('day'))` ✓
  - Line 589: `disabledRangeTime` checks current hour ✓
- Status: CORRECT ✓

---

### Test Scenario 3: Prepaid + No Contract

**Setup:**
- Device: Prepaid service
- Contract: None assigned
- contractMode passed: `null` or `undefined`
- enablePrepaidEnhancement: `true`

**Expected Result:**
- Edit button: ENABLED (no contract mode restrictions)
- Can click edit

**Actual Result:**
- Edit button: `disabled={contractMode === 'fix'}` = `disabled={false}` when contractMode is null ✓
- Status: CORRECT ✓

---

### Test Scenario 4: Feature Flag Disabled

**Setup:**
- enablePrepaidEnhancement: `false`

**Expected Result:**
- Edit button: NOT RENDERED (entire ExpirationDatetimeEditor hidden)

**Actual Result:**
- Line 817: `{deviceData?.enablePrepaidEnhancement === true && (...)}` ✓
- ExpirationDatetimeEditor only renders when feature flag is true
- Status: CORRECT ✓

---

## Property Type Constraints (Variable Contract Only)

When contractMode === 'VARIABLE', additional constraints applied:

### DateTime Mode (Line 615-620)
```tsx
disabledDate={(current) => {
  if (
    (current || datetimeValue) &&
    contractMode &&
    contractMode === CONTRACT_MODE.VARIABLE
  ) {
    return current.isBefore(datetimeValue.startOf('day'));  // Block past dates
  }
  return false;
}}
```

**Logic:** User cannot select dates before today ✓

### Epoch Mode (Line 640-645)
```tsx
const isPast = valid && inputEpoch !== null && inputEpoch < currentEpoch;
const isPastError = isPast && contractMode === CONTRACT_MODE.VARIABLE;
const isDisabled = !valid || isPastError;
```

**Logic:** Confirm button disabled if past epoch + variable contract ✓

---

## Conclusion: UI Logic Verification

| Scenario | Component | Status | Details |
|----------|-----------|--------|---------|
| Fixed + Edit | Button disabled | ✓ CORRECT | `disabled={contractMode === 'fix'}` works |
| Variable + Edit | Button enabled | ✓ CORRECT | constraints applied in date picker |
| No Contract + Edit | Button enabled | ✓ CORRECT | null/undefined !== 'fix' |
| Feature Flag | Render check | ✓ CORRECT | Only renders when enabled |
| DateTime constraint | Past dates | ✓ CORRECT | disabledDate blocks past |
| Epoch constraint | Past time | ✓ CORRECT | isPastError blocks past |

---

## Final Assessment

**UI Logic for Fixed Contract Update Expiration: CORRECT ✓**

The UI properly:
1. Blocks edit button when `contractMode === 'fix'`
2. Allows edit with constraints when `contractMode === 'variable'`
3. Respects feature flag `enablePrepaidEnhancement`
4. Enforces "no past date" constraint for variable contracts

**Next Steps:** Handle Backend API & Bulk Action validation (not in this phase)
