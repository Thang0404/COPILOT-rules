# Debug File Refactored - Single Device Test

## Simplified Structure ✅

Refactored to test only **ONE real device** from your curl command.

---

## Configuration

### Lines 11-25: Mock Function (Simplified)

```python
def mock_get_contract_info(tenant_id: str, device_uid: str):
    # Change this to test different scenarios:
    # - {"contract_mode": "fixed", "is_activating": True}  → BLOCKED
    # - {"contract_mode": "variable", "is_activating": True} → Check past date
    # - None → ALLOWED (no contract)
    
    contract_info = {"contract_mode": "variable", "is_activating": True}
    return contract_info
```

**To test different scenarios, just change the `contract_info` value.**

---

### Lines 72-79: Test Configuration

```python
# Your real device from curl
DEVICE_UID = "359445338655677"
EXPIRATION = 1769504123  # Change to test past date: int(time.time()) - 86400

print(f"Device UID: {DEVICE_UID}")
print(f"Expiration: {EXPIRATION}")
print(f"Tenant: /qa1")
```

**To test different devices or dates, just change these 2 variables.**

---

## How to Test Different Scenarios

### Scenario 1: Fixed Contract (BLOCKED)

**File:** `debug_nb.py` line ~57
```python
contract_info = {"contract_mode": "fixed", "is_activating": True}
```

**Expected:**
```json
{
  "resultCode": "UPDATE_EXPIRATION_NOT_ALLOWED_FOR_FIXED_CONTRACT",
  "resultMessage": "Cannot update expiration for device with fixed contract assigned"
}
```

---

### Scenario 2: Variable Contract + Future Date (ALLOWED)

**File:** `debug_nb.py` line ~57
```python
contract_info = {"contract_mode": "variable", "is_activating": True}
```

**File:** `debug_nb.py` line ~74
```python
EXPIRATION = 1769504123  # Future date
```

**Expected:**
```json
{
  "resultCode": "REQUEST_SUCCESS",
  "resultMessage": "Request is successful"
}
```

---

### Scenario 3: Variable Contract + Past Date (BLOCKED)

**File:** `debug_nb.py` line ~57
```python
contract_info = {"contract_mode": "variable", "is_activating": True}
```

**File:** `debug_nb.py` line ~74
```python
EXPIRATION = int(time.time()) - 86400  # Yesterday
```

**Expected:**
```json
{
  "resultCode": "EXPIRATION_CANNOT_BE_IN_PAST",
  "resultMessage": "Expiration date cannot be in the past for device with variable contract"
}
```

---

### Scenario 4: No Contract (ALLOWED)

**File:** `debug_nb.py` line ~57
```python
contract_info = None  # No contract
```

**Expected:**
```json
{
  "resultCode": "REQUEST_SUCCESS",
  "resultMessage": "Request is successful"
}
```

---

## Test Different Devices

Just change the `DEVICE_UID` variable:

```python
DEVICE_UID = "359445338655677"  # Your device from curl
# or
DEVICE_UID = "R8YW20K0YRT"      # Different device
```

---

## Run Debug

```bash
cd /home/thang/Documents/rsu/alps-ttp3-backend/modules/north_bound_api/north_bound_proxy_v2/src
python debug_nb.py
```

---

## Expected Output Format

```
================================================================================
Testing Update Expiration with Contract Validation
================================================================================

Device UID: 359445338655677
Expiration: 1769504123
Tenant: /qa1

================================================================================

[MOCK] get_contract_info called with tenant_id=/qa1, device_uid=359445338655677
[MOCK] Contract info for 359445338655677: {'contract_mode': 'variable', 'is_activating': True}

================================================================================
RESULT
================================================================================
{
  "statusCode": 200,
  "body": {
    "updateExpirationResponseList": [
      {
        "deviceUid": "359445338655677",
        "resultCode": "REQUEST_SUCCESS",
        "resultMessage": "Request is successful"
      }
    ]
  }
}
================================================================================
```

---

## Quick Test Matrix

| Change `contract_info` to | Change `EXPIRATION` to | Expected Result |
|---------------------------|------------------------|-----------------|
| `{"contract_mode": "fixed", ...}` | Any | ❌ BLOCKED (Fixed) |
| `{"contract_mode": "variable", ...}` | Future | ✅ ALLOWED |
| `{"contract_mode": "variable", ...}` | Past | ❌ BLOCKED (Past) |
| `None` | Any | ✅ ALLOWED |

---

## Summary

✅ **Single device test** - Easy to understand  
✅ **Two variables to change** - `DEVICE_UID` and `EXPIRATION`  
✅ **One mock to modify** - `contract_info` in `mock_get_contract_info()`  
✅ **Clean output** - Shows device info, mock calls, and result  
✅ **No multiple test cases** - Just run once per scenario

Much simpler than before!
