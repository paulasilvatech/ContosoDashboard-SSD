<!--
  ============================================================================
  SYNC IMPACT REPORT
  ============================================================================
  Version Change: N/A (initial) → 1.0.0
  
  Modified Principles: None (initial constitution)
  
  Added Sections:
  - Core Principles (5 principles)
  - Architecture Standards
  - Development Workflow
  - Governance
  
  Removed Sections: None (initial constitution)
  
  Templates Requiring Updates:
  - ✅ .specify/templates/plan-template.md (Constitution Check section aligned)
  - ✅ .specify/templates/spec-template.md (Requirements align with security principle)
  - ✅ .specify/templates/tasks-template.md (Phase structure supports principles)
  - ✅ .specify/templates/agent-file-template.md (No changes needed)
  - ✅ .specify/templates/checklist-template.md (No changes needed)
  
  Follow-up TODOs: None
  ============================================================================
-->

# ContosoDashboard Constitution

## Core Principles

### I. Training-First Design

All features MUST be designed for educational clarity over production optimization. Code MUST demonstrate industry-standard patterns in a simplified, learnable form. Every architectural decision MUST include inline documentation explaining the "why" behind the pattern. Features MUST work offline without cloud dependencies to maximize training availability.

**Rationale**: This is a training application for Spec-Driven Development. Learners need clear, well-documented examples they can study and understand without requiring Azure subscriptions or external services.

### II. Defense in Depth Security

Security MUST be enforced at multiple layers: middleware, page attributes (`[Authorize]`), and service-level checks. All protected pages MUST use the `[Authorize]` attribute. Services MUST validate user authorization before returning data (IDOR protection). Role-based access control (RBAC) MUST be applied consistently across all features.

**Rationale**: Even in a training context with mock authentication, demonstrating proper security patterns is essential. Defense in depth ensures that security failures at one layer are caught by another, and students learn production-ready security practices.

### III. Abstraction for Migration

All infrastructure dependencies (database, file storage, authentication) MUST use interface abstractions. Implementations MUST be swappable via dependency injection without changing business logic. Connection strings and configuration MUST be externalized to `appsettings.json`. Each abstraction MUST have a documented migration path to Azure services.

**Rationale**: The offline-first architecture with abstraction layers enables seamless migration to Azure services (Azure SQL, Blob Storage, Entra ID) when students move to production. This teaches proper separation of concerns and dependency injection patterns.

### IV. Clean Service Architecture

Business logic MUST reside in Service classes, not in Razor pages or components. All services MUST follow the async/await pattern for non-blocking operations. Entity Framework queries MUST use eager loading (`.Include()`) to prevent N+1 query problems. Database models MUST include appropriate indexes for frequently queried fields.

**Rationale**: Clean separation of concerns (Models, Services, Data, Pages) enables testability, maintainability, and teaches students proper layered architecture. Async patterns and query optimization demonstrate production-ready performance practices.

### V. Specification-Driven Development

All features MUST begin with a feature specification before implementation. User stories MUST be prioritized and independently testable. Implementation plans MUST include a Constitution Check gate before research begins. Tasks MUST be organized by user story to enable incremental delivery.

**Rationale**: The GitHub Spec Kit workflow enforces thoughtful design before coding. By requiring specifications, we ensure features are well-understood, properly scoped, and aligned with project principles before any code is written.

## Architecture Standards

**Technology Stack**:
- **Framework**: ASP.NET Core 8.0+ with Blazor Server
- **Database**: SQL Server LocalDB (development), Azure SQL (production path)
- **Authentication**: Cookie-based with claims-based identity
- **Authorization**: Role-based access control (Employee, TeamLead, ProjectManager, Administrator)
- **Styling**: Bootstrap 5.3 with Bootstrap Icons

**Project Structure**:
```plaintext
ContosoDashboard/
├── Data/           # EF Core DbContext and database configuration
├── Models/         # Entity classes with relationships
├── Services/       # Business logic with interface abstractions
├── Pages/          # Blazor components and Razor pages
├── Shared/         # Shared layouts and navigation
└── wwwroot/        # Static assets (CSS, images)
```

**Coding Standards**:
- Use async/await throughout for all I/O operations
- Apply `[Authorize]` attribute on all protected pages
- Implement service-level authorization checks for data access
- Include XML documentation comments on public APIs
- Follow C# naming conventions (PascalCase for public members, camelCase for private)

## Development Workflow

**Feature Implementation Process**:
1. **Spec**: Create feature specification using `/speckit.spec` command
2. **Plan**: Generate implementation plan using `/speckit.plan` command
3. **Constitution Check**: Verify alignment with principles before proceeding
4. **Tasks**: Break down into tasks using `/speckit.tasks` command
5. **Implement**: Complete tasks organized by user story
6. **Review**: Verify security, architecture, and documentation requirements

**Code Review Requirements**:
- All PRs MUST pass Constitution Check verification
- Security patterns MUST be validated (authorization, IDOR protection)
- Service abstractions MUST be properly implemented
- Async patterns MUST be consistently applied

**Testing Guidance**:
- Tests are OPTIONAL unless explicitly requested in the specification
- When included, tests SHOULD cover: authentication flows, authorization enforcement, service-level security, and user isolation

## Governance

This constitution supersedes all other development practices for the ContosoDashboard project. All feature implementations MUST comply with these principles.

**Amendment Process**:
1. Propose changes with rationale and impact analysis
2. Document the version change type (MAJOR/MINOR/PATCH)
3. Update dependent templates if principles change
4. Record amendment in the Sync Impact Report

**Versioning Policy**:
- MAJOR: Backward-incompatible principle removals or redefinitions
- MINOR: New principle/section added or materially expanded guidance
- PATCH: Clarifications, wording improvements, non-semantic refinements

**Compliance Review**:
- All PRs MUST include Constitution Check verification
- Complexity deviations MUST be justified in the implementation plan
- Security exceptions MUST be documented with compensating controls

**Version**: 1.0.0 | **Ratified**: 2026-01-28 | **Last Amended**: 2026-01-28
