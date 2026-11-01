### EduPulse — Week 00, Day 01
1 November 2026
Repository reconnaissance

Objective: Understand the existing EduPulse web application completely before making any desktop-related changes.

Today's work is strictly an inspection and documentation task. Antigravity must not modify the project.
1. Workspace and repository inspection
- [ ] Inspect the complete Edupulse ERP desktop parent folder.
- [ ] Identify the old web project folder.
- [ ] Identify whether a separate desktop project folder already exists.
- [ ] Inspect Git repositories, branches, working trees and latest commits.
- [ ] Identify duplicate, legacy or unused project directories.
- [ ] Understand the relationship between the old web project and the new workspace.
Output: Workspace structure report.
2. Frontend architecture audit
Inspect the authoritative frontend-v2/ application.
- [ ] React and TypeScript architecture.
- [ ] Vite configuration and build scripts.
- [ ] Routing and navigation.
- [ ] Authentication and session management.
- [ ] API client and backend integration.
- [ ] State management and TanStack Query.
- [ ] UI components, styling and design system.
- [ ] Environment variables and configuration.
- [ ] Browser-specific APIs and dependencies.
- [ ] Existing frontend build status.
Output: Frontend architecture and compatibility report.
3. Backend architecture audit
Inspect the backend/ application.
- [ ] Spring Boot application structure.
- [ ] Controllers and REST endpoints.
- [ ] Services and business logic.
- [ ] Repositories and entity relationships.
- [ ] DTOs and validation.
- [ ] Spring Security and JWT authentication.
- [ ] Role-based authorization.
- [ ] Exception handling and logging.
- [ ] Background jobs and integrations.
- [ ] Backend build and test status.
Output: Backend architecture report.
4. Database audit
- [ ] Inspect PostgreSQL configuration.
- [ ] Inspect all Flyway migration files.
- [ ] Identify existing database tables and relationships.
- [ ] Understand entity-to-table mappings.
- [ ] Identify indexes and constraints.
- [ ] Identify migration inconsistencies or risks.
- [ ] Document database configuration without exposing credentials.
Output: Database architecture and migration report.

Do not connect to or modify the production Neon database.
5. Authentication and authorization audit
Trace the complete authentication lifecycle:
Login
  ↓
Credential Validation
  ↓
JWT Generation
  ↓
Token Storage
  ↓
Protected API Request
  ↓
Token Validation
  ↓
Role / Permission Check
  ↓
Response
  ↓
Token Refresh / Logout


Inspect:
- Access-token and refresh-token handling.
- Frontend token storage.
- Backend token validation.
- Role and permission enforcement.
- Session expiration.
- Logout behaviour.
- Protected routes and APIs.
Output: Authentication flow diagram and security observations.
6. Testing and deployment baseline
- [ ] Inspect GitHub Actions workflows.
- [ ] Identify frontend CI checks.
- [ ] Identify backend H2 and PostgreSQL integration tests.
- [ ] Inspect test configuration.
- [ ] Verify local frontend build.
- [ ] Verify backend compilation and tests where safe.
- [ ] Document existing CI failures.
- [ ] Inspect Vercel, Render and Neon deployment configuration without changing anything.
Output: Build, testing and deployment baseline.
7. Technical debt and desktop migration risks
Identify:
- Browser-dependent functionality.
- Hardcoded API URLs.
- Authentication storage limitations.
- File upload/download dependencies.
- Desktop notification requirements.
- Native Windows integration requirements.
- Potential Tauri compatibility issues.
- Existing bugs and architectural weaknesses.
Output: Technical debt and desktop migration risk register.

Final deliverables for Day 01
Expected documentation
1. 01-workspace-audit.md
2. 02-frontend-architecture.md
3. 03-backend-architecture.md
4. 04-database-audit.md
5. 05-authentication-security-audit.md
6. 06-build-test-deployment-baseline.md
7. 07-technical-debt-and-risks.md
   
   Day 01 completion criteria
   We understand the existing codebase, its current health, its architecture and the major obstacles to desktop migration.