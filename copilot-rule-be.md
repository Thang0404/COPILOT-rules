# Backend Development Standards - ALPS TTP3

## Python Lambda Functions

### Core Rules
1. Always propagate new parameters through entire data flow
2. Extract parameters from kwargs/event using `.get()`
3. Pass parameters to SQS messages, database calls, and downstream functions
4. Never assume parameter exists - use `.get()` with None fallback
5. Test parameter flow with debug scripts using mock boto3

### Parameter Propagation Pattern
```
GraphQL Resolver
  -> Extract from kwargs.get("param_name")
  -> Add to SQS payload
  -> Send to next Lambda

Next Lambda
  -> Extract from event.get("param_name")
  -> Add to next SQS/DB call
  -> Continue propagation
```

### SQS Message Pattern
- Always include new parameters in message payload
- Use descriptive keys matching GraphQL schema
- Include tenant_id, user_id for tracing
- Add timestamp for debugging

### Common Mistakes
- Forgetting to propagate through intermediate Lambdas
- Using wrong key names between services
- Not adding to chunked/split operations
- Missing in error handling paths

### Deployment
- Code changes must be deployed to Lambda
- Local fixes don't affect deployed environment
- Test on deployed environment, not just local
- Check CloudWatch logs to verify parameter received

---

## GraphQL Schema (AppSync)

### Schema Definition Rules
1. Add new parameters to mutation/query signature
2. Use correct GraphQL types (String, Int, Boolean, AWSJSON)
3. Add to both type definition AND resolver mapping
4. Keep parameter names consistent across schema

### Resolver Mapping
- Update request.vtl if using VTL templates
- Pass parameters from GraphQL to Lambda event
- Map GraphQL names to Lambda parameter names

### Common Types
- String: for IDs, names, text
- Int: for counts, durations, numbers
- Boolean: for flags
- AWSJSON: for complex objects
- Custom Input Types: for nested structures

---

## DynamoDB Operations

### Parameter in DB Operations
- Add new fields to put_item/update_item
- Include in query/scan filters if searchable
- Add to GSI/LSI if needed for queries
- Update data models/schemas

### Best Practices
- Use consistent attribute naming
- Add created_at, updated_at timestamps
- Include tenant_id for multi-tenancy
- Use meaningful key names

---

## Testing & Debugging

### Local Testing
- Create event JSON files with sample data
- Mock boto3 clients (SQS, DynamoDB, S3)
- Print/log parameter values at each step
- Test error cases and None values

### Debug Script Pattern
- Mock AWS services
- Load event from JSON file
- Call handler function
- Print outputs and verify flow

### CloudWatch Logs
- Check logs after deployment
- Search for parameter name
- Verify parameter in payload
- Trace through entire flow

---

## Code Organization

### File Structure
- Keep related functions in same file
- Use descriptive function names
- Add docstrings for complex logic
- Group imports by standard/external/local

### Function Pattern
- Extract parameters at function start
- Validate required parameters
- Process business logic
- Return or send to next service

---

Last Updated: November 12, 2025
Project: ALPS TTP3 Backend
Tech Stack: Python, AWS Lambda, DynamoDB, SQS, AppSync
