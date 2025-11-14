# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This is a **documentation repository** that defines patterns, conventions, and best practices for Verji's backend microservices. It does NOT contain runnable code - it contains architectural guidance that developers reference when building and maintaining backend services.

**Key point**: Changes here affect how developers across multiple teams implement backend features. Documentation should be clear, comprehensive, and include complete examples.

## Repository Structure

```
verji-backend-conventions/
├── NSERVICEBUS.md    # NServiceBus patterns for event-driven architecture
└── CLAUDE.md         # This file
```

Currently contains one primary document:
- **NSERVICEBUS.md**: Comprehensive guide for implementing NServiceBus message handlers and sagas, including CQRS patterns, event-driven architecture, saga orchestration, and transactional consistency patterns

## How to Maintain This Repository

### Adding New Conventions Documents

When adding new convention documents (e.g., API conventions, database patterns, deployment practices):

1. Create a new markdown file with an ALL-CAPS name (following NSERVICEBUS.md convention)
2. Structure with clear hierarchy: Overview → Core Concepts → Patterns → Examples → Checklist
3. Include complete, runnable code examples with explanatory comments
4. Reference actual implementation files from production repos where applicable
5. Add anti-patterns ("DO NOT" examples) alongside correct patterns
6. Ensure examples show the full context (imports, class structure, error handling)

### Updating Existing Documentation

When updating NSERVICEBUS.md or other documents:

1. **Preserve structure**: Keep existing section hierarchy unless reorganization is necessary
2. **Add examples**: When explaining new patterns, include complete code examples
3. **Show the "why"**: Explain the reasoning behind patterns, not just the "how"
4. **Cross-reference**: Link related patterns within the document
5. **Update reference implementations**: Keep the list of reference implementations current
6. **Maintain consistency**: Use the same code style and explanation format as existing content

### Documentation Standards

All convention documents should include:

- **Overview section**: High-level explanation of what this enables
- **Pattern sections**: Specific, reusable patterns with code examples
- **Complete examples**: Full implementations showing patterns in context
- **Anti-patterns**: Clear examples of what NOT to do and why
- **Checklist**: Summary checklist for developers to verify correct implementation
- **Reference implementations**: Pointers to actual production code examples

### Code Examples Format

Examples should follow this structure:
```csharp
// Context comment explaining the scenario
public class ExampleHandler : IHandleMessages<ExampleCommand>
{
    // Show complete implementation including:
    // - Constructor with dependencies
    // - Logging setup
    // - Error handling
    // - All required patterns
}
```

### When to Update

Update this documentation when:
- New architectural patterns are established across services
- Existing patterns evolve based on lessons learned
- Common mistakes are identified that need clarification
- New NServiceBus features are adopted organization-wide
- Cross-cutting concerns (logging, error handling) change

### Git Workflow

- Commit messages should clearly describe what convention was added/changed
- Keep commits focused on single topics (one pattern update per commit)
- This repo is referenced by developers, so maintain a clean commit history

## No Build/Test Commands

This repository contains only markdown documentation. There are:
- No build commands
- No test commands
- No compilation steps
- No dependencies to install

All work here is documentation editing and maintenance.
