# Verji Backend Conventions

Shared development conventions, patterns, and best practices for Verji backend microservices.

## Purpose

This repository serves as the **single source of truth** for organization-wide backend development standards. It contains detailed guidance on architectural patterns, frameworks, and conventions used across all Verji backend services.

### Goals

- **Consistency** - Ensure all backend services follow the same patterns and conventions
- **Onboarding** - Accelerate new developer productivity with comprehensive documentation
- **Knowledge Sharing** - Capture and distribute institutional knowledge about our architecture
- **AI Assistance** - Provide context for AI coding assistants (Claude Code, GitHub Copilot, etc.)
- **Quality** - Maintain high code quality through documented best practices

## Structure

This repository is organized by technology and architectural concern:

### Current Documentation

- **[NSERVICEBUS.md](./NSERVICEBUS.md)** - NServiceBus patterns for message handlers and sagas
  - Message handler conventions
  - Saga orchestration patterns
  - Dual state management (SagaData vs Domain Objects)
  - OutBox pattern with Marten sessions
  - Internal vs public commands
  - Delegation patterns
  - Timeout handling and fallback strategies

### Planned Documentation

- **MARTEN.md** - Marten document database patterns
  - Multi-tenancy with tenant resolution
  - Session lifecycle and OutBox integration
  - Patch operations and bulk updates
  - Event store patterns

- **LOGGING.md** - Structured logging conventions
  - Serilog configuration
  - Log level guidelines
  - CorrelationId propagation
  - OpenTelemetry integration

- **TESTING.md** - Testing patterns and practices
  - xUnit conventions
  - Moq and NFluent usage
  - Integration test patterns
  - Test data management

- **CASBIN.md** - Access control patterns
  - Casbin RBAC/ABAC configuration
  - Policy management
  - Custom group patterns

## Usage

### For Developers

**When starting work in a Verji backend repository:**

1. Check the repository's `CLAUDE.md` file for project-specific conventions
2. Follow links to relevant sections in this conventions repository
3. Apply the documented patterns to your implementation

**When you encounter a new pattern:**

1. Check if it's documented here first
2. If missing, document it and submit a PR
3. Update project `CLAUDE.md` files to reference the new pattern

### For Project CLAUDE.md Files

Each backend project should include a `CLAUDE.md` file that references these conventions. Example structure:

```markdown
# CLAUDE.md

## Overview
Brief description of this specific project.

## NServiceBus Patterns

This project follows Verji's standard NServiceBus patterns.

**For detailed guidance, see:**
- [NServiceBus Handlers & Sagas](https://github.com/verji/verji-backend-conventions/blob/main/NSERVICEBUS.md)

**Project-specific details:**
- Main handler location: `YourProject.Esb.Handlers/`
- Example saga: `YourSpecificSaga.cs`

## Marten Data Access

**For detailed guidance, see:**
- [Marten Patterns](https://github.com/verji/verji-backend-conventions/blob/main/MARTEN.md)

**Project-specific details:**
- Database: PostgreSQL 14
- Tenant resolution: `IResolveTenants` service
```

### For AI Coding Assistants

When using Claude Code, GitHub Copilot, or similar tools:

1. The tool will read your project's `CLAUDE.md` file
2. `CLAUDE.md` references this conventions repository
3. For comprehensive patterns, prompt the AI to read specific convention files:
   - "Read the NServiceBus saga patterns from verji-backend-conventions"
   - "Check the Marten conventions before implementing this repository"

## Contributing

### Adding New Conventions

1. **Create a new markdown file** for the topic (e.g., `AUTHENTICATION.md`)
2. **Follow the established structure:**
   - Overview section explaining purpose
   - Pattern sections with clear headings
   - Code examples from actual implementations
   - Reference implementations section linking to real code
3. **Include practical examples** from existing codebases
4. **Submit a PR** with clear description of what's being documented
5. **Update this README** to list the new convention file

### Updating Existing Conventions

1. **Ensure accuracy** - Verify patterns against current implementations
2. **Maintain examples** - Keep code examples up-to-date
3. **Version carefully** - Note when patterns change to avoid breaking existing code
4. **Communicate changes** - Announce significant updates to the team

### Documentation Guidelines

**DO:**
- ✅ Extract patterns from actual working code
- ✅ Provide complete, runnable examples
- ✅ Explain the "why" behind patterns, not just the "how"
- ✅ Reference specific files in existing repos as examples
- ✅ Include both correct and incorrect patterns (with clear warnings)
- ✅ Use code examples from multiple repos to show consistency

**DON'T:**
- ❌ Document generic framework features (link to official docs instead)
- ❌ Include obvious best practices ("write good code", "add comments")
- ❌ Make up hypothetical examples when real ones exist
- ❌ Document project-specific details (those belong in project CLAUDE.md files)
- ❌ Duplicate information that's better maintained elsewhere

## Relationship to Project Documentation

```
┌─────────────────────────────────────────────┐
│  verji-backend-conventions (This Repo)      │
│  Organization-wide patterns and standards   │
│                                             │
│  - NSERVICEBUS.md                           │
│  - MARTEN.md                                │
│  - LOGGING.md                               │
│  └─ Reusable across ALL projects           │
└─────────────────────────────────────────────┘
                    ▲
                    │ Referenced by
                    │
┌─────────────────────────────────────────────┐
│  Individual Project Repositories            │
│                                             │
│  verji-itops/                               │
│  ├─ CLAUDE.md ──┐                           │
│  └─ Project-specific details                │
│                                             │
│  verji-account/                             │
│  ├─ CLAUDE.md ──┤                           │
│  └─ Project-specific details                │
│                                             │
│  verji-billing/                             │
│  ├─ CLAUDE.md ──┘                           │
│  └─ Project-specific details                │
└─────────────────────────────────────────────┘
```

### What Goes Where?

**In verji-backend-conventions (this repo):**
- Architectural patterns used across multiple projects
- Framework-specific conventions (NServiceBus, Marten, etc.)
- Organization-wide coding standards
- Multi-tenancy patterns
- Logging and observability conventions

**In project CLAUDE.md files:**
- Project-specific architecture overview
- Build and test commands for that project
- Links to this conventions repo
- Project-specific examples and edge cases
- Service-specific configuration details

## Maintenance

This repository is maintained by the Verji backend team.

### Review Cycle

- **Quarterly** - Review all documentation for accuracy
- **As Needed** - Update when patterns evolve
- **On Request** - Add new topics when gaps identified

### Keeping It Current

When you notice a pattern has evolved:

1. Update the documentation in this repo
2. Add a note about what changed and why
3. Notify the team if it's a breaking change
4. Update examples in project repos if needed

## Questions or Suggestions?

- Open an issue in this repository
- Discuss in #backend-architecture Slack channel
- Submit a PR with proposed changes

---

**Repository:** https://github.com/verji/verji-backend-conventions
**License:** Internal use only - Verji AS
**Maintained by:** Verji Backend Team
