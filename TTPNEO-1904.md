During the device upload API with 10K, Getting error "504 Gateway Timeout"

## ⚠️ LƯU Ý QUAN TRỌNG

API `/api/v2/inventory/upload` **KHÔNG TỒN TẠI** trong codebase hiện tại!

Flow thực tế là **GraphQL API qua AppSync** thông qua 2 mutations:
1. `prepareBatchUpload` - Upload CSV lên S3
2. `finalizeBatchUpload` - Trigger processing

---

## 🎯 FLOW THỰC TẾ

### **Architecture Overview**

```
Frontend/GraphQL Client
    ↓ (prepareBatchUpload mutation)
upload_resolver
    ↓ (Upload to S3 + Insert t_upload)
Frontend
    ↓ (finalizeBatchUpload mutation)
upload_resolver
    ↓ (Send to BATCH_ACTION_DISPATCHER_SQS)
batch_action_dispatcher
    ↓ (Download CSV, Validate, Route per device)
batch_action_processor
    ↓ (Process device registration)
action_trigger
    ↓ (Execute action + Assign workflow)
Workflow Service
```

---

## 📝 PHASE 1: Prepare Upload (Frontend → upload_resolver)

### File: `alps-ttp3-backend/modules/upload_resolver/src/main.py`

### Function: `prepareBatchUpload()` (Line 199-253)

**GraphQL Mutation**:
```graphql
mutation prepareBatchUpload(
  $tenantId: String!
  $assignActionId: String!
  $rowCount: Int!
  $deviceType: Int!
  $workflowId: String
  $brandId: Int
  $provisionType: String
  $targetWorkspaceId: String
) {
  prepareBatchUpload(
    tenantId: $tenantId
    assignActionId: $assignActionId
    rowCount: $rowCount
    deviceType: $deviceType
    workflowId: $workflowId
    brandId: $brandId
    provisionType: $provisionType
    targetWorkspaceId: $targetWorkspaceId
  ) {
    uploadId
    fileUpload {
      url
      fields
    }
  }
}
```

**Python Code**:
```python
@tracer.capture_method
@event_logger
@permission_required(permission=Permission.UPDATE)
def prepareBatchUpload(**kwargs):
    tenant_id = kwargs.get("tenant_id")
    user_pool = kwargs.get("user_pool")
    assign_action_id = kwargs.get("assign_action_id")
    row_count = kwargs.get("row_count")
    device_type = kwargs.get("device_type")
    brand_id = kwargs.get("brand_id")
    provision_type = kwargs.get("provision_type")
    target_workspace_id = kwargs.get("target_workspace_id") or tenant_id
    workflow_id = kwargs.get("workflow_id")  # ✅ Workflow ID for assignment

    username, cognito_pool_id = user_pool.get("username"), user_pool.get("cognito_pool_id")
    conn = db.connect()
    
    # Validate target workspace
    tenant_ids = utils.pluck_by_key(elements=db.get_tenants(conn=conn, root_tenant_id=tenant_id), key='id')
    if target_workspace_id not in tenant_ids:
        return error_response(ErrorCode.INVALID_TENANT, i18n.t(ErrorCode.INVALID_TENANT))

    schema_name = db.get_schema_name(conn, target_workspace_id)
    target_workspace_name = db.retrieve_tenant_by_id(conn, target_workspace_id)
    user = db.get_user(conn, username, cognito_pool_id)
    
    # 🔑 Build ext_fields với workflow_id
    ext_fields = json.dumps({
        "assign_action_id": assign_action_id,
        'count': row_count,
        "dispatched": False,
        "brand_id": brand_id,
        "provision_type": provision_type,
        "device_type": device_type,
        "target_workspace_id": target_workspace_id,
        "target_workspace_name": target_workspace_name,
        "workflow_id": workflow_id  # ✅ Saved for later
    })

    try:
        # Generate S3 presigned URL
        object_name = S3_UPLOAD_BUCKET.format(
            tenant_base64=base64.b16encode(tenant_id.encode('utf-8')).decode('utf-8'),
            uuid=uuid.uuid4().hex
        )
        file_upload = s3_upload.file_upload(S3_STORAGE_BUCKET, object_name)
        
        # Insert to t_upload table
        upload = db.insert_upload(
            conn,
            schema_name,
            target_workspace_id,
            user.id,
            ext_fields,  # ✅ Contains workflow_id
            device_type,
            upload_channel=UploadChannels.PORTAL,
            file_url=file_upload['fields']['key']
        )
        conn.commit()
        
        return {
            "uploadId": upload.id,
            "fileUpload": {
                **file_upload, 
                "fields": json.dumps(file_upload['fields'])
            }
        }
    except Exception as error:
        conn.rollback()
        logger.exception("batch upload failed")
        return error_response(ErrorCode.UNEXPECTED_EXCEPTION, i18n.t(ErrorCode.UNEXPECTED_EXCEPTION))
```

**Database**: `t_upload`
```sql
INSERT INTO t_upload (
    id,                -- Auto-generated
    tenant_id,         -- target_workspace_id
    created_by,        -- user.id
    ext_fields,        -- JSON with workflow_id
    device_type,       -- 1 (SMARTPHONE) or 2 (TABLET)
    upload_channel,    -- "PORTAL"
    file_url,          -- S3 key path
    status,            -- NULL (pending)
    created_at         -- Now
)
VALUES (
    123,
    'tenant-uuid',
    456,
    '{
        "assign_action_id": "action-uuid",
        "count": 150,
        "dispatched": false,
        "device_type": 1,
        "workflow_id": "68a5ae7a4cb9e87b88b7d344"  -- ✅ Saved here
    }',
    1,
    'PORTAL',
    'uploads/ABC123/def456.csv',
    NULL,
    '2026-02-04 10:00:00'
)
```

**Response**:
```json
{
    "uploadId": "123",
    "fileUpload": {
        "url": "https://s3.amazonaws.com/...",
        "fields": "{\"key\": \"uploads/...\", \"policy\": \"...\"}"
    }
}
```

**Frontend Action**: Upload CSV to S3 using presigned URL

---

## 📝 PHASE 2: Finalize Upload (Frontend → upload_resolver)

### File: `alps-ttp3-backend/modules/upload_resolver/src/main.py`

### Function: `finalizeBatchUpload()` (Line 259-316)

**GraphQL Mutation**:
```graphql
mutation finalizeBatchUpload(
  $tenantId: String!
  $uploadId: String!
  $workflowId: String
  $simControlName: String
  $simControlId: String
  $billingDetail: AWSJSON
) {
  finalizeBatchUpload(
    tenantId: $tenantId
    uploadId: $uploadId
    workflowId: $workflowId
    simControlName: $simControlName
    simControlId: $simControlId
    billingDetail: $billingDetail
  )
}
```

**Python Code**:
```python
@tracer.capture_method
@event_logger
@permission_required(permission=Permission.UPDATE)
def finalizeBatchUpload(**kwargs):
    user_pool = kwargs.get("user_pool")
    tenant_id = kwargs.get("tenant_id")
    upload_id = kwargs.get("upload_id")
    workflow_id = kwargs.get("workflow_id")  # ✅ Can override from frontend
    
    # Optional: SIM control config
    sim_control = {}
    if kwargs.get("sim_control_name") and kwargs.get("sim_control_id"):
        sim_control = {
            "sim_control_name": kwargs.get("sim_control_name"),
            "sim_control_id": kwargs.get("sim_control_id")
        }
    billing_detail = kwargs.get("billing_detail")

    username, cognito_pool_id = user_pool.get("username"), user_pool.get("cognito_pool_id")
    
    conn = db.connect()
    schema_name = db.get_schema_name(conn, tenant_id)
    user = db.get_user(conn, username, cognito_pool_id)
    
    # Get upload record
    upload = db.get_upload(conn, schema_name, upload_id)
    if upload is None:
        return error_response("DBerror", "upload id not found!")

    ext_fields = upload.ext_fields
    
    # Check if already dispatched
    if ext_fields.get("dispatched"):
        logger.info("Dispatched already submited, So nothing todo")
        return

    try:
        # Mark as dispatched
        modified_ext_fields = {**ext_fields, 'dispatched': True}
        db.update_upload(conn, schema_name, upload_id, json.dumps(modified_ext_fields))
        conn.commit()
        conn.close()

        # 🔥 Build payload for dispatcher
        batch_action_dispatcher_payload = {
            "user_id": user.id,
            "tenant_id": tenant_id,
            "upload_id": upload_id,
            "batch_upload_type": "UPLOAD",  # ✅ Type = UPLOAD (not WORKFLOW_ACTION)
            "billing_detail": billing_detail,
            "sim_control": sim_control,
            "workflow_id": workflow_id  # ✅ Workflow ID passed to dispatcher
        }
        
        logger.info(f"Send SQS to {BATCH_ACTION_DISPATCHER_SQS} with payload {batch_action_dispatcher_payload}")
        
        # Send to BATCH_ACTION_DISPATCHER_SQS
        send_message(batch_action_dispatcher_payload, queue_url=BATCH_ACTION_DISPATCHER_SQS)
        
        return
    except Exception as error:
        conn.close()
        logger.exception(f"Mark upload record submited error: {error}")
        return error_response(ErrorCode.UNEXPECTED_EXCEPTION, i18n.t(ErrorCode.UNEXPECTED_EXCEPTION))
```

**SQS Message to Dispatcher**:
```json
{
    "user_id": 456,
    "tenant_id": "tenant-uuid",
    "upload_id": "123",
    "batch_upload_type": "UPLOAD",
    "billing_detail": null,
    "sim_control": {},
    "workflow_id": "68a5ae7a4cb9e87b88b7d344"
}
```

---

## 📝 PHASE 3: Dispatcher Processing

### File: `alps-ttp3-backend/modules/batch_action_dispatcher/src/main.py`

### Function: `batch_action_upload()` (Around line 150-500)

**Nhận message từ BATCH_ACTION_DISPATCHER_SQS**:
```python
def batch_action_upload(**kwargs):
    """
    Main dispatcher function - routes based on batch_upload_type
    """
    batch_upload_type = kwargs.get("batch_upload_type")
    upload_id = kwargs.get("upload_id")
    workflow_id = kwargs.get("workflow_id")
    
    # Query upload record từ DB
    conn = db.connect()
    schema_name = db.get_schema_name(conn, tenant_id)
    upload = db.get_upload(conn, schema_name, upload_id)
    ext_fields = json.loads(upload.ext_fields)
    
    # Get workflow_id from ext_fields if not in payload
    if not workflow_id:
        workflow_id = ext_fields.get("workflow_id")
    
    # Get action_id từ ext_fields
    assign_action_id = ext_fields.get("assign_action_id")
    device_type = ext_fields.get("device_type", DeviceType.SMARTPHONE)
    
    # Download CSV from S3
    file_url = upload.file_url
    df = download_and_process_csv(file_url, S3_STORAGE_BUCKET)
    
    # Get device UIDs from CSV
    device_uids = df['deviceUid'].tolist()  # hoặc IMEI column
    
    # Query devices từ database
    devices = db.get_devices_by_imeis(
        schema_name=schema_name,
        imeis=device_uids,
        is_include_imei2=True
    )
    
    # ⚠️ CRITICAL: Check batch_upload_type
    if batch_upload_type == "UPLOAD":
        # This is REGISTER flow (devices in IDLE state)
        # Will be processed by upload_processor
        process_upload_devices(devices, workflow_id, assign_action_id)
    elif batch_upload_type == "WORKFLOW_ACTION":
        # This is direct workflow assignment (devices already registered)
        # Requires state [2,4,5] and validates service type
        process_workflow_action(devices, workflow_id)
    
    return
```

**For UPLOAD type** (REGISTER devices):
```python
def process_upload_devices(devices, workflow_id, assign_action_id):
    """
    Process devices for REGISTER action
    Devices must be in IDLE state (1)
    """
    for device_uid in device_uids:
        # Build payload
        payload = {
            "device_uid": device_uid,
            "user_id": user_id,
            "tenant_id": tenant_id,
            "upload_id": upload_id,
            "assign_action_id": assign_action_id,  # REGISTER action
            "workflow_id": workflow_id,  # ✅ Pass workflow_id
            "device_type": device_type
        }
        
        # Send to UPLOAD_PROCESSOR_SQS
        send_message(payload, queue_url=UPLOAD_PROCESSOR_SQS)
```

**SQS Message to upload_processor**:
```json
{
    "device_uid": "350377187371347",
    "user_id": 456,
    "tenant_id": "tenant-uuid",
    "upload_id": "123",
    "assign_action_id": "action-uuid",
    "workflow_id": "68a5ae7a4cb9e87b88b7d344",
    "device_type": 1
}
```

---

## 📝 PHASE 4: Upload Processor

### File: `alps-ttp3-backend/modules/upload_processor/src/event_handler.py`

### Function: `handler()` (Line 1-17)

**Nhận message từ UPLOAD_PROCESSOR_SQS**:
```python
def handler(event, _, db):
    """
    Main upload processor handler
    """
    if event.get("action") == ActionUpload.UPLOAD_SIDE_IMEI:
        # Handle side IMEI upload
        return side_imeis_upload_handler.handler(db, event)
    
    elif event.get("action") == ActionUpload.NORTHBOUND_UPLOAD_SINGLE:
        # Handle single device upload from API
        device_type = event.get("device_type") or DeviceType.SMARTPHONE
        upload_processor_initializer = UploadProcessorInitializer(device_type)
        upload_processor = upload_processor_initializer(dao=db, **event)
        return upload_processor.upload()
    
    # Default: Handle batch upload
    device_uid = event.get("device_uid")
    workflow_id = event.get("workflow_id")
    assign_action_id = event.get("assign_action_id")
    
    # Process device registration
    result = register_device_and_assign_workflow(
        device_uid=device_uid,
        workflow_id=workflow_id,
        assign_action_id=assign_action_id,
        **event
    )
    
    return result
```

### File: `alps-ttp3-backend/modules/upload_processor/src/smartphone_upload_handler.py`

**Process device**:
```python
class SmartphoneUploadHandler:
    def upload(self):
        """
        1. Validate device (IMEI format, Luhn check)
        2. Insert/Update device in database
        3. Send to BATCH_ACTION_PROCESSOR_SQS
        """
        device_uid = self.event.get("device_uid")
        workflow_id = self.event.get("workflow_id")
        assign_action_id = self.event.get("assign_action_id")
        
        # Validate IMEI
        if not self.validate_imei(device_uid):
            return {"error": "Invalid IMEI"}
        
        # Insert device to database (state = IDLE)
        device = self.db.insert_device(
            imei=device_uid,
            tenant_id=self.tenant_id,
            state_id=StateId.IDLE,
            service_type_id=self.service_type_id
        )
        
        # Build payload for processor
        payload = {
            "device_uid": device_uid,
            "user_id": self.user_id,
            "tenant_id": self.tenant_id,
            "action_id": assign_action_id,  # REGISTER action
            "workflow_id": workflow_id,  # ✅ Pass workflow for pending assignment
            "device_id": device.id
        }
        
        # Send to BATCH_ACTION_PROCESSOR_SQS
        send_message(payload, queue_url=BATCH_ACTION_PROCESSOR_SQS)
        
        return {"success": True, "device_id": device.id}
```

**SQS Message to batch_action_processor**:
```json
{
    "device_uid": "350377187371347",
    "user_id": 456,
    "tenant_id": "tenant-uuid",
    "action_id": "register-action-uuid",
    "workflow_id": "68a5ae7a4cb9e87b88b7d344",
    "device_id": 12345
}
```

---

## 📝 PHASE 5: Batch Action Processor

### File: `alps-ttp3-backend/modules/batch_action_processor/src/devices/device.py`

**Forward to action_trigger**:
```python
def process_device_action(**message):
    """
    Processor forwards message to trigger with workflow_id
    """
    payload_dto = PayloadDTO(**message)
    
    # Build trigger payload
    trigger_payload = {
        "device_uid": payload_dto.device_uid,
        "tenant_id": payload_dto.tenant_id,
        "user_id": payload_dto.user_id,
        "action_id": payload_dto.action_id,  # REGISTER action
        "workflow_id": payload_dto.workflow_id  # ✅ Pass workflow_id
    }
    
    # Send to ACTION_TRIGGER_SQS
    send_message(trigger_payload, queue_url=ACTION_TRIGGER_SQS)
```

**SQS Message to action_trigger**:
```json
{
    "device_uid": "350377187371347",
    "tenant_id": "tenant-uuid",
    "user_id": 456,
    "action_id": "register-action-uuid",
    "workflow_id": "68a5ae7a4cb9e87b88b7d344"
}
```

---

## 📝 PHASE 6: Action Trigger (Execute REGISTER + Assign Workflow)

### File: `alps-ttp3-backend/modules/action_trigger/src/handlers/message_handler.py`

**Execute action and assign workflow**:
```python
def handle_message(conn, message, **kwargs):
    """
    1. Execute REGISTER action (move device IDLE → READY_FOR_USE)
    2. Check if workflow_id present
    3. Call workflow service to assign workflow
    """
    payload_dto = PayloadDTO(**message)
    
    # Get schema
    schema = base_dao.get_schema_by_tenant_id(payload_dto.tenant_id)
    
    # Get device
    device = base_dao.get_device_by_uid(
        schema_name=schema.name,
        device_uid=payload_dto.device_uid
    )
    
    # Get action from DB
    action = base_dao.get_action_by_id(
        schema_name=schema.name,
        action_id=payload_dto.action_id
    )
    
    # Execute REGISTER action
    # This moves device from IDLE (1) → READY_FOR_USE (2)
    result = execute_register_action(device, action, schema)
    
    # ✅ CRITICAL: After REGISTER, check for workflow_id
    workflow_id = payload_dto.workflow_id
    
    if workflow_id:
        logger.info(f"Pending workflow detected: {workflow_id}")
        
        # ⚠️ IMPORTANT: Refresh device state after REGISTER
        device = base_dao.get_device_by_uid(
            schema_name=schema.name,
            device_uid=payload_dto.device_uid
        )
        
        # Call workflow service to assign workflow
        workflow_result = workflow_handler.assign_solo_workflow_to_device(
            device_uid=payload_dto.device_uid,
            workflow_id=workflow_id,
            tenant_id=payload_dto.tenant_id,
            user_id=payload_dto.username,
            schema=schema
        )
        
        if workflow_result['success']:
            logger.info(f"Workflow {workflow_id} assigned successfully")
        else:
            logger.error(f"Failed to assign workflow: {workflow_result['error']}")
    
    return result
```

---

## 📝 PHASE 7: Workflow Service (Final Assignment)

### File: `alps-ttp3-workflow/src/apiservice/api/api_v1/endpoints/device.py`

**Assign workflow to device**:
```python
@router.post("/{device_uid}/workflow")
async def assign_workflow_to_device(
    device_uid: str,
    workflow_id: str,
    tenant_id: str,
    user_id: str
):
    """
    Assign workflow to device in MongoDB
    """
    # Get device from MongoDB
    device = await DeviceDB.find_one({
        "device_uid": device_uid,
        "tenant_id": tenant_id
    })
    
    if not device:
        raise HTTPException(status_code=404, detail="Device not found")
    
    # Get workflow
    workflow = await WorkflowVersioningDB.get(workflow_id)
    
    if not workflow:
        raise HTTPException(status_code=404, detail="Workflow not found")
    
    # ✅ Update device with workflow info
    device.is_activating_workflow = True
    device.workflow_version_id = workflow_id
    device.workflow = {
        "workflowId": workflow_id,
        "workflowName": workflow.name,
        "assignedAt": datetime.utcnow().isoformat(),
        "assignedBy": user_id
    }
    
    await device.save()
    
    return {
        "success": True,
        "message": "Workflow assigned successfully"
    }
```

---

## 🎯 SUMMARY: Complete Flow

### GraphQL APIs (NOT REST)
1. ✅ `prepareBatchUpload` - Create upload + S3 presigned URL
2. ✅ `finalizeBatchUpload` - Trigger processing

### Module Flow
```
upload_resolver (prepareBatchUpload)
    ↓ S3 + t_upload table
Frontend (Upload CSV to S3)
    ↓
upload_resolver (finalizeBatchUpload)
    ↓ BATCH_ACTION_DISPATCHER_SQS
batch_action_dispatcher
    ↓ UPLOAD_PROCESSOR_SQS (per device)
upload_processor (register device)
    ↓ BATCH_ACTION_PROCESSOR_SQS
batch_action_processor
    ↓ ACTION_TRIGGER_SQS
action_trigger (REGISTER + assign workflow)
    ↓ HTTP POST
workflow_service (MongoDB update)
```

### Key Differences from Expected
- ❌ NO REST API `/api/v2/inventory/upload`
- ✅ GraphQL mutations via AppSync
- ✅ `batch_upload_type = "UPLOAD"` (not "WORKFLOW_ACTION")
- ✅ Devices must be in IDLE state for REGISTER
- ✅ Workflow assignment happens AFTER REGISTER completes

### Database Tables
1. `t_upload` - Upload metadata + workflow_id
2. `t_device` - Device records
3. `t_assigned_action_history` - Action execution history
4. MongoDB `DeviceDB` - Workflow assignment

---

## 🔍 How to Test

### 1. Prepare Upload
```graphql
mutation {
  prepareBatchUpload(
    tenantId: "tenant-uuid"
    assignActionId: "register-action-id"
    rowCount: 100
    deviceType: 1
    workflowId: "68a5ae7a4cb9e87b88b7d344"
  ) {
    uploadId
    fileUpload {
      url
      fields
    }
  }
}
```

### 2. Upload CSV to S3
Use presigned URL from step 1

### 3. Finalize Upload
```graphql
mutation {
  finalizeBatchUpload(
    tenantId: "tenant-uuid"
    uploadId: "123"
    workflowId: "68a5ae7a4cb9e87b88b7d344"
  )
}
```

### 4. Monitor CloudWatch Logs
- `upload_resolver` logs
- `batch_action_dispatcher` logs
- `upload_processor` logs
- `batch_action_processor` logs
- `action_trigger` logs
- `workflow_service` logs
