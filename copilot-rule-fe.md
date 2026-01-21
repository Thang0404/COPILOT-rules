# Frontend Development Standards - ALPS TTP3

## GraphQL Changes

### CRITICAL RULE: Never Edit Auto-Generated Files
- NEVER edit: GraphQL.type.ts, mutations.ts, queries.ts
- ALWAYS: Edit appsync/main.graphql → Run npm run codegen

### Workflow
1. Edit backend schema (appsync/main.graphql)
2. Run: npm run codegen
3. Import generated types in components

---

## Form Validation

### Use Form.useWatch for Real-time Validation
```typescript
// WRONG - Stale values
const values = form.getFieldsValue();
if (values.assignWorkflowId) { setDisableSubmit(false); }

// CORRECT - Reactive
const assignWorkflowId = Form.useWatch('assignWorkflowId', form);
if (assignWorkflowId) { setDisableSubmit(false); }
```

### Validation Pattern
- Use early returns for each action type
- Keep validation logic separated
- Check watched values, not getFieldsValue

---

## State Management

### Enums & Constants
- Define enums in one place
- Map frontend enums to backend values in constants/device.ts
- Use initial state objects for reset functionality

### State Initialization
```typescript
const initialWorkflow = { id: '', name: '' };
const [selectedWorkflow, setSelectedWorkflow] = useState(initialWorkflow);
```

---

## Code Generation

### Commands
```bash
npm run codegen  # Generate GraphQL types
npm run format   # Format code
npm run lint     # Check linting
```

### What Codegen Does
1. Reads appsync/main.graphql
2. Generates types, mutations, queries
3. Ensures type safety

---

## Best Practices

### 1. Code Preservation
- Add new code without removing existing logic
- Keep backward compatibility
- Don't delete working code

### 2. Type Safety
- Use proper TypeScript types
- Avoid any
- Leverage auto-generated types

### 3. Form Fields
- Use consistent naming matching GraphQL
- Clear, descriptive names
- Match schema variable names

### 4. Conditional Rendering
- Use clear conditions
- Avoid nested ternaries
- Early returns for clarity

---

## Common Mistakes

### 1. Editing Auto-Generated Files
- Don't edit mutations.ts manually
- Edit schema, then run codegen

### 2. Not Using Form.useWatch
- getFieldsValue gives stale values
- Use Form.useWatch for reactive validation

### 3. Incorrect Enum Mapping
- Map all enums explicitly
- Don't rely on fallback values
- Match frontend/backend enum values

---

## Feature Implementation Checklist

### Backend
- [ ] Update appsync/main.graphql
- [ ] Update Lambda resolvers
- [ ] Deploy changes
- [ ] Verify in CloudWatch

### Frontend
- [ ] Run npm run codegen
- [ ] Add enums if needed
- [ ] Add Form.useWatch
- [ ] Implement validation
- [ ] Add UI fields
- [ ] Map batchUploadType
- [ ] Pass parameters
- [ ] Test submission

---

## Debugging Tips

### Form Issues
- Check Form.useWatch usage
- Console.log watched values
- Verify early returns

### GraphQL Issues
- Run codegen after schema changes
- Check import paths
- Verify parameter names

### Backend Issues
- Check network tab payload
- Review CloudWatch logs
- Verify parameter propagation

---

Last Updated: November 12, 2025
Project: ALPS TTP3 Frontend
Tech Stack: React, TypeScript, Ant Design, GraphQL
