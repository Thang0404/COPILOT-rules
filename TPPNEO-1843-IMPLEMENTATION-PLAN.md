# TPPNEO-1843: Implementation Plan - Block Fixed Contract Update Expiration in API

## Overview

Add validation to North Bound API `update_device_expiration` to block updates when device has fixed contract assigned.

---

## Architecture Decision

### Option Selected: Call Workflow Service API

**Rationale:**
- North Bound API doesn't have MongoDB connection
- Workflow service owns billing_cycle data
- Existing pattern: call external services via Lambda (e.g., customer_properties calls DEVICE_IDENTITY_SERVICE)

**Pattern Found:**
File: [customer_properties.py line 5](file:///home/thang/Documents/rsu/alps-ttp3-backend/modules/north_bound_api/north_bound_proxy_v2/src/handlers/customer_properties.py#L5)
```python
import requests as http_requests
```

**Config Available:**
- `WORKFLOW_SERVICE_ARN` - Lambda ARN for workflow service
- `WORKFLOW_API_PREFIX = "/api/v1"` - API prefix

---

## Implementation Steps

### Step 1: Add Error Code

**File:** `/modules/north_bound_api/north_bound_proxy_v2/src/validator/error.py`

Add new error code constant:
```python
UPDATE_EXPIRATION_NOT_ALLOWED_FOR_FIXED_CONTRACT = (
    "UPDATE_EXPIRATION_NOT_ALLOWED_FOR_FIXED_CONTRACT",
    "Cannot update expiration for device with fixed contract assigned"
)
```

---

### Step 2: Create Workflow Service Helper

**File (NEW):** `/modules/north_bound_api/north_bound_proxy_v2/src/utils/workflow_helper.py`

```python
import json
import boto3
from typing import Optional
from config import logger, WORKFLOW_SERVICE_ARN


def invoke_workflow_service(path: str, payload: dict) -> dict:
    """
    Invoke workflow service Lambda via ARN
    
    Args:
        path: API path (e.g., "/api/v1/device/<device_uid>/billing-cycle")
        payload: Request payload
    
    Returns:
        Response dict from workflow service
    """
    lambda_client = boto3.client('lambda')
    
    try:
        response = lambda_client.invoke(
            FunctionName=WORKFLOW_SERVICE_ARN,
            InvocationType='RequestResponse',
            Payload=json.dumps({
                "path": path,
                "httpMethod": "GET",
                "body": json.dumps(payload) if payload else None
            })
        )
        
        response_payload = json.loads(response['Payload'].read())
        
        if response.get('StatusCode') == 200 and response_payload.get('statusCode') == 200:
            return json.loads(response_payload.get('body', '{}'))
        else:
            logger.error(f"Workflow service error: {response_payload}")
            return None
            
    except Exception as e:
        logger.error(f"Failed to invoke workflow service: {e}")
        return None


def get_device_billing_cycle(tenant_id: str, device_uid: str) -> Optional[dict]:
    """
    Get device billing cycle from workflow service
    
    Returns:
        {
            "contract_mode": "fixed" | "variable",
            "is_activating_contract": bool,
            ...
        }
        or None if not found/error
    """
    path = f"/api/v1/device/{device_uid}/billing-cycle"
    payload = {"tenant_id": tenant_id}
    
    result = invoke_workflow_service(path, payload)
    
    if result and result.get('billing_cycle'):
        return result['billing_cycle']
    
    return None
```

---

### Step 3: Add Validation in update_device_expiration

**File:** `/modules/north_bound_api/north_bound_proxy_v2/src/handlers/update_device_expiration.py`

**Location:** After line 194 (service type check)

```python
# Import at top
from utils.workflow_helper import get_device_billing_cycle
from validator.error import UPDATE_EXPIRATION_NOT_ALLOWED_FOR_FIXED_CONTRACT

# Add validation after service type check (line ~195)
# Check if device has fixed contract assigned
device_billing_cycle = get_device_billing_cycle(
    tenant_id=tenant_id,
    device_uid=imei
)

if device_billing_cycle:
    contract_mode = device_billing_cycle.get('contract_mode')
    is_activating = device_billing_cycle.get('is_activating_contract', False)
    
    if contract_mode == 'fixed' and is_activating:
        raise SystemException(
            message=UPDATE_EXPIRATION_NOT_ALLOWED_FOR_FIXED_CONTRACT,
            properties={
                "device_uid": imei,
                "contract_mode": "fixed"
            }
        )
```

---

### Step 4: Add Test Cases

**File:** `/modules/north_bound_api/north_bound_proxy_v2/src/tests/test_update_device_expiration.py`

```python
@patch('handlers.update_device_expiration.get_device_billing_cycle')
def test_fixed_contract_blocks_update_expiration(self, mock_get_billing_cycle):
    """Test that fixed contract blocks expiration update"""
    mock_get_billing_cycle.return_value = {
        'contract_mode': 'fixed',
        'is_activating_contract': True
    }
    
    requests = [self.create_valid_request()]
    
    result = self.update_device_expiration_func(
        requests=requests,
        **self.base_params
    )
    
    self.assertEqual(
        result.update_expiration_response_list[0].result_code,
        'UPDATE_EXPIRATION_NOT_ALLOWED_FOR_FIXED_CONTRACT'
    )


@patch('handlers.update_device_expiration.get_device_billing_cycle')
def test_variable_contract_allows_update_expiration(self, mock_get_billing_cycle):
    """Test that variable contract allows expiration update"""
    mock_get_billing_cycle.return_value = {
        'contract_mode': 'variable',
        'is_activating_contract': True
    }
    
    requests = [self.create_valid_request()]
    
    result = self.update_device_expiration_func(
        requests=requests,
        **self.base_params
    )
    
    self.assertEqual(
        result.update_expiration_response_list[0].result_code,
        'REQUEST_SUCCESS'
    )


def test_no_contract_allows_update_expiration(self, mock_get_billing_cycle):
    """Test that no contract allows expiration update"""
    mock_get_billing_cycle.return_value = None
    
    requests = [self.create_valid_request()]
    
    result = self.update_device_expiration_func(
        requests=requests,
        **self.base_params
    )
    
    self.assertEqual(
        result.update_expiration_response_list[0].result_code,
        'REQUEST_SUCCESS'
    )
```

---

## Alternative: Direct MongoDB Query (Not Recommended)

### Why Not MongoDB?

**Pros:**
- Direct data access
- No Lambda invocation overhead

**Cons:**
- Requires adding pymongo dependency
- Need MongoDB connection string in config
- Duplicates workflow service logic
- Breaks service boundaries
- Security: need MongoDB credentials in North Bound API

**Decision:** Use Lambda invocation to respect service boundaries

---

## Files to Modify

| File | Action | Lines |
|------|--------|-------|
| `validator/error.py` | Add error code | ~End of file |
| `utils/workflow_helper.py` | Create new file | New |
| `handlers/update_device_expiration.py` | Add import | Top |
| `handlers/update_device_expiration.py` | Add validation | ~Line 195 |
| `tests/test_update_device_expiration.py` | Add tests | ~End of file |

---

## Testing Strategy

### Unit Tests
1. Mock `get_device_billing_cycle` to return fixed contract → Expect rejection
2. Mock to return variable contract → Expect success
3. Mock to return None (no contract) → Expect success
4. Mock workflow service error → Expect allows (fail-open for availability)

### Integration Tests
1. Real workflow service call with fixed contract device
2. Real workflow service call with variable contract device
3. Real workflow service call with no contract device

---

## Rollback Plan

If issues occur:
1. Comment out validation block (lines ~195-205)
2. Keep error code and helper (no side effects)
3. Deploy hotfix
4. Investigate issue

---

## Deployment Notes

### Environment Variables
Ensure `WORKFLOW_SERVICE_ARN` is set in Lambda environment:
```bash
WORKFLOW_SERVICE_ARN=arn:aws:lambda:region:account:function:workflow-service
```

### Lambda Permissions
North Bound API Lambda needs permission to invoke Workflow Service Lambda:
```json
{
  "Effect": "Allow",
  "Action": "lambda:InvokeFunction",
  "Resource": "arn:aws:lambda:*:*:function:workflow-service"
}
```

---

## Next: Implementation

Ready to implement? Files chỉnh sửa:
1. ✓ error.py - Add error code
2. ✓ workflow_helper.py - Create helper
3. ✓ update_device_expiration.py - Add validation
4. ✓ test_update_device_expiration.py - Add tests
