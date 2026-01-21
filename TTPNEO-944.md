# TTPNEO-944: Independent Workflow Schedule Flow Optimization

## 📋 Overview
Optimize workflow event processor schedule flow to improve reliability, scalability, and support repetition feature.


không cần quan tâm đến time window nữa, đó lầ thiết kế có chủ đích, không đề xuất giải phap chỉnh sửa nữa
## Implementation Phases

### **Phase 1: Quick Wins
**Scope**: Optimize existing logic without architectural changes

#### 1.1 Expand Time Window
- **File**: `workflow_processor/db.py`
- **Change**: `timedelta(minutes=1)` → `timedelta(minutes=5)`
- **Benefit**: Reduce missed schedules by 80%

#### 1.2 Add Validation Cache
- **File**: `workflow_processor/tasks/ws_processor.py`
- **Add**: Device validation result caching
- **Benefit**: Reduce API calls by 50-70%

#### 1.3 Implement Retry Mechanism
- **File**: `workflow_processor/tasks/ws_processor.py`
- **Function**: `push_trigger_payload_to_sqs()`
- **Add**: 3 retries with exponential backoff
- **Benefit**: Reduce step loss from 5% → <0.1%

**Deliverables**:
- ✅ Modified `db.py` with 5-minute window
- ✅ Cache layer in `ws_processor.py`
- ✅ Retry logic with status tracking
- ✅ Unit tests for new logic

---

### **Phase 2: DynamoDB Integration
**Scope**: Add execution tracking and state management

#### 2.1 DynamoDB Table Design
```python
# Table: workflow_execution_schedule
{
    "PK": "TENANT#{tenant_id}#DATE#{date}",
    "SK": "TIME#{scheduled_time}#STEP#{step_id}",
    "device_workflow_step_id": "...",
    "scheduled_time": "2024-01-15T10:30:00Z",
    "status": "PENDING|QUEUED|EXECUTED|FAILED",
    "validation_result": "VALID|INVALID",
    "retry_count": 0,
    "ttl": 1705334400
}

# GSI: status-scheduled_time-index
```

#### 2.2 Daily Scanner v2
- **File**: `functions/daily_scanner_v2/daily_scanner_v2.py` (NEW)
- **Schedule**: Daily at 00:00 UTC
- **Function**: 
  - Scan next 24h workflow steps
  - Pre-validate steps
  - Store in DynamoDB

#### 2.3 Update Existing Processor
- **File**: `workflow_processor/tasks/ws_scanner.py`
- **Add**: DynamoDB write operations
- **Parallel run** with existing S3 flow

**Deliverables**:
- ✅ DynamoDB table via SAM template
- ✅ Daily scanner Lambda function
- ✅ Integration with existing flow
- ✅ Monitoring & CloudWatch dashboards

---

### **Phase 3: Event-Driven Scheduler
**Scope**: Replace Step Function with EventBridge + Lambda

#### 3.1 Scheduler Lambda
- **File**: `functions/scheduler_v2/scheduler_v2.py` (NEW)
- **Trigger**: EventBridge (every 5 minutes)
- **Function**:
  - Query DynamoDB for next 5-min window
  - Push to SQS Execution Queue
  - Update status to QUEUED

#### 3.2 Executor Lambda
- **File**: `functions/executor_v2/executor_v2.py` (NEW)
- **Trigger**: SQS Execution Queue
- **Function**:
  - Re-validate (lightweight)
  - Send to Action Trigger SQS
  - Update status to EXECUTED
  - Handle retries (3 attempts)

#### 3.3 SQS Queues
```yaml
ExecutionQueue:
  Type: AWS::SQS::Queue
  Properties:
    VisibilityTimeout: 180
    MessageRetentionPeriod: 86400
    RedrivePolicy:
      deadLetterTargetArn: !GetAtt ExecutionDLQ.Arn
      maxReceiveCount: 3

ExecutionDLQ:
  Type: AWS::SQS::Queue
  Properties:
    MessageRetentionPeriod: 1209600  # 14 days
```

**Deliverables**:
- ✅ Scheduler Lambda with EventBridge trigger
- ✅ Executor Lambda with SQS integration
- ✅ DLQ for failed messages
- ✅ CloudWatch alarms for DLQ depth

---

### **Phase 4: Repetition Optimization

### **Phase 5: Migration & Deprecation

