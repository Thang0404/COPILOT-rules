# Workflow Development Standards - ALPS TTP3

## GitHub Workflow Rules

### Core Principles
1. Never suggest git commands in chat unless explicitly requested
2. User manages git operations manually
3. Focus on code changes, not version control
4. Deployment is separate from code development

### Communication Rules
- Explain code changes clearly
- Provide file-by-file summaries
- List what was modified, added, deleted
- Don't mention commits, pushes, branches unless asked

### Code Review Pattern
- Summarize changes after completing work
- Group changes by feature/fix
- Explain impact of each change
- Provide testing recommendations

---

## Development Workflow

### Feature Implementation Flow
1. Understand requirement
2. Identify files to change
3. Make code changes
4. Test locally if possible
5. Summarize changes
6. Wait for deployment (don't suggest it)

### Multi-File Changes
- Update frontend and backend together
- Keep changes synchronized
- Update GraphQL schema first
- Then update resolvers
- Finally update UI components

### Change Documentation
- List all modified files
- Explain purpose of each change
- Note dependencies between changes
- Highlight breaking changes

---

## Testing Approach

### Local Testing
- Create debug scripts when possible
- Use mock data for AWS services
- Test validation logic in isolation
- Verify parameter passing

### Integration Testing
- Wait for deployment to test environment
- Check CloudWatch logs for errors
- Verify end-to-end flow
- Test with real data

### Test Documentation
- List test scenarios
- Note expected behavior
- Document actual results
- Identify gaps in testing

---

## Communication Best Practices

### Explaining Changes
- Use clear, concise language
- Avoid technical jargon when possible
- Provide context for decisions
- Explain alternatives considered

### Problem Solving
- Break down complex issues
- Identify root causes
- Propose solutions with trade-offs
- Explain implementation steps

### Progress Updates
- Summarize completed work
- List remaining tasks
- Note blockers or dependencies
- Provide next steps

---

## Code Standards

### Consistency
- Follow existing patterns in codebase
- Match naming conventions
- Use same code style
- Preserve existing logic when adding features

### Preservation
- Don't delete working code unnecessarily
- Add new features without breaking old ones
- Keep backward compatibility
- Document breaking changes clearly

### Organization
- Group related changes together
- Keep commits focused on single feature
- Separate refactoring from features
- Update documentation with code

---

Last Updated: November 12, 2025
Project: ALPS TTP3 Workflow
Focus: Development process, not git operations
