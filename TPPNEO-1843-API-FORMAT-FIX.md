# Update Expiration API - Correct Format

## Root Cause

API schema expects `expiration` at **root level**, not inside `customProperty`.

## Schema Validation

From `validator/schemas/schema.py` line 643:

```python
UPDATE_EXPIRATION_SCHEMA = {
    "type": "object",
    "properties": {
        "deviceUid": {"type": "string", "title": "deviceUid", "is_valid_imei": True},
        "expiration": {"type": ["integer", "string"], "title": "expiration"},
        "customProperty": {"type": "object", "title": "customProperty", "allow_redundant_keys": True}
    },
    "required": ["deviceUid", "expiration"]
}
```

**Required fields:** `deviceUid`, `expiration`

---

## Wrong Format

```json
{
  "updateExpirationList": [
    {
      "deviceUid": "359445338655677",
      "customProperty": {
         "expiration": 1769504123
      }
    }
  ]
}
```

**Error:**
```json
{
  "updateExpirationResponseList": [{
    "deviceUid": "359445338655677",
    "resultCode": "REQUIRED_PROPERTIES",
    "resultMessage": "Please input expiration"
  }]
}
```

---

## Correct Format

```json
{
  "updateExpirationList": [
    {
      "deviceUid": "359445338655677",
      "expiration": 1769504123
    }
  ]
}
```

---

## CURL Command - Correct

```bash
curl -X 'POST' \
  'https://api.qa.trustonic-alps.com/api/v2/device/updateExpiration' \
  -H 'accept: application/json' \
  -H 'tenantId: /qa1' \
  -H 'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ0ZW5hbnRfaWQiOiIvcWExIiwidXNlcl9pZCI6MjUzLCJleHAiOjE3Njk0MjE1MDh9.c-wAFubzmzy7FSYHMzytRZ0F2xTac--WGj6icKIBdb4' \
  -H 'Content-Type: application/json' \
  -d '{
  "updateExpirationList": [
    {
      "deviceUid": "359445338655677",
      "expiration": 1769504123
    }
  ]
}'
```

---

## Expected Response - Fixed Contract

If device `359445338655677` has a fixed contract assigned:

```json
{
  "updateExpirationResponseList": [
    {
      "deviceUid": "359445338655677",
      "resultCode": "UPDATE_EXPIRATION_NOT_ALLOWED_FOR_FIXED_CONTRACT",
      "resultMessage": "Cannot update expiration for device with fixed contract assigned"
    }
  ]
}
```

---

## Expected Response - Variable Contract or No Contract

```json
{
  "updateExpirationResponseList": [
    {
      "deviceUid": "359445338655677",
      "resultCode": "REQUEST_SUCCESS",
      "resultMessage": "Request is successful"
    }
  ]
}
```

---

## Debug File Updated

`debug_nb.py` line 72-80 updated with:
- Correct device UID: `359445338655677`
- Correct expiration: `1769504123`
- **Fixed:** `expiration` at root level (not in `customProperty`)

---

## Summary

**Key Change:** Move `expiration` from `customProperty` to root level of each item

**Before:**
```json
{"deviceUid": "...", "customProperty": {"expiration": 123}}
```

**After:**
```json
{"deviceUid": "...", "expiration": 123}
```
