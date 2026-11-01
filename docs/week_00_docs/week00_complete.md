### EduPulse — Week 00: Complete Execution Plan
7-day foundation sprint
1–7 November 2026

Let's establish the complete Week 00 plan properly.

I would use Week 00 as our Project Foundation & Desktop Transition Sprint. Its purpose is to prepare the new desktop project, understand the existing web implementation, validate the architecture, and establish the development process.

We are not trying to implement all 259 modules this week. We are preparing the foundation on which those modules will be developed.
Week 00 — Final objectives
1. Understand the existing EduPulse web application.
2. Establish a clean, independent desktop workspace.
3. Select and initialize the desktop framework.
4. Integrate the existing frontend into the desktop shell.
5. Establish desktop-to-backend communication.
6. Audit all 259 planned modules across 23 domains.
7. Finalize the architecture, documentation, and Week 01 backlog.

Day 01 — Existing system reconnaissance
1 November · Start today
Objective: Understand exactly what we already have.
Tasks
ID	Task	Expected outcome
D01.1	Inspect parent workspace and folder structure	Understand current organization
D01.2	Inspect complete web frontend	Identify authoritative UI and architecture
D01.3	Inspect backend architecture	Understand controllers, services, repositories
D01.4	Inspect database schema and Flyway migrations	Understand persistence
D01.5	Inspect authentication and authorization	Understand login, JWT, RBAC
D01.6	Inspect existing integrations	Identify API dependencies
D01.7	Inspect Git history, working tree and CI	Establish source-control baseline
D01.8	Identify existing technical debt	Create known-issues register

Deliverables
- Existing architecture report.
- Repository structure report.
- Frontend and backend dependency report.
- Database baseline.
- Authentication flow documentation.
- Existing bugs and technical debt list.
- Verified build/test baseline.
Restriction: No unnecessary changes to the old web project.
Day 02 — Desktop architecture and technology decisions
Objective: Decide how the new desktop application will work.
Tasks
1. Evaluate Tauri vs Electron.
2. Evaluate the existing React + TypeScript + Vite frontend for reuse.
3. Define desktop-to-backend communication.
4. Define application configuration and environment management.
5. Design authentication and secure credential storage.
6. Define session expiry and refresh-token behaviour.
7. Define desktop application updates and versioning.
8. Define cloud-hosted vs institution-hosted deployment architecture.
9. Define the initial offline read-only strategy.
Deliverables
- Architecture Decision Record (ADR).
- Desktop technology selection.
- High-level system architecture diagram.
- Authentication and security design.
- Deployment architecture.
- Offline capability specification.
Important: Tauri should be the initial candidate, not an unquestioned decision. We should make the final selection after checking the actual frontend compatibility and proof of concept.
Day 03 — Desktop project initialization
Objective: Create the new desktop application independently.
Tasks
1. Establish the final directory structure.
2. Initialize the selected desktop framework.
3. Configure the Windows development environment.
4. Configure the desktop application entry point.
5. Configure frontend build integration.
6. Configure application metadata and versioning.
7. Configure development and production build modes.
8. Create the first Windows desktop build.
9. Verify the application launches independently.
Deliverables
- Independent desktop project.
- Working desktop shell.
- Windows development build.
- Initial project README.
- Reproducible setup instructions.
The old web version remains a reference and must not be moved into the new application's source directories.
Day 04 — Frontend integration and desktop UI
Objective: Bring the existing EduPulse interface into the desktop environment.
Tasks
1. Integrate the existing frontend-v2/ code where feasible.
2. Verify routing and navigation.
3. Verify assets, icons, fonts and styling.
4. Verify responsive behaviour at desktop window sizes.
5. Establish minimum and maximum window dimensions.
6. Test application loading and error states.
7. Test page refresh and application restart behaviour.
8. Identify browser-dependent APIs that require desktop alternatives.
9. Identify desktop-specific UI improvements.
Deliverables
- Existing EduPulse UI running inside desktop.
- Navigation working.
- Asset loading verified.
- Desktop UI compatibility report.
- List of required UI adaptations.
Scope boundary: Do not redesign every screen this day. First achieve stable integration.
Day 05 — Backend integration and security foundation
Objective: Connect the desktop application to the existing backend.
Tasks
1. Configure API base URL management.
2. Establish desktop-to-Spring Boot communication.
3. Verify health-check endpoint.
4. Test authentication and login flow.
5. Test protected API access.
6. Verify token refresh and logout behaviour.
7. Design secure credential storage.
8. Handle network failures and expired sessions.
9. Verify CORS and desktop networking requirements.
10. Document security limitations and required fixes.
Deliverables
- Successful API communication.
- Authentication integration proof of concept.
- Secure token-storage implementation plan.
- Network error handling.
- Backend integration report.
Do not expose production credentials or embed secrets inside the desktop application. Production infrastructure and the production database remain untouched.
Day 06 — Complete ERP module audit
Objective: Map the original 259-module product scope against the existing implementation.

This is where we use your original Notion plan as the authoritative functional catalog.
Audit allocation
Section	Domains	Modules
Foundation Platform	5	46
Academic Domain	8	106
Enterprise Operations	10	107
Total	23	259

For every module, record
- Original module ID and name.
- Existing frontend implementation.
- Existing backend implementation.
- Database support.
- API availability.
- Authentication and permission coverage.
- Test coverage.
- Current implementation status.
- Missing functionality.
- Desktop-specific requirements.
Use these statuses:
Status	Meaning
Implemented	Complete workflow verified
Partial	Some functionality exists
Missing	Not implemented
Unverified	Evidence insufficient
Desktop adaptation	Existing feature needs desktop changes

Deliverables
- Complete 259-module matrix.
- Domain-wise implementation status.
- Module dependency map.
- Existing functionality inventory.
- Missing functionality inventory.
- Desktop adaptation requirements.
Do not mark a module complete simply because a corresponding screen or API exists.
Day 07 — System validation and Week 01 preparation
Objective: Consolidate the work and make the desktop project ready for sustained development.
Tasks
1. Review the desktop architecture.
2. Verify desktop application startup.
3. Verify frontend integration.
4. Verify backend connectivity.
5. Run relevant frontend and backend tests.
6. Review security and authentication decisions.
7. Review the 259-module matrix for missing entries.
8. Identify critical architectural dependencies.
9. Create the prioritized development backlog.
10. Document unresolved decisions and risks.
11. Create the Week 01 execution plan.
Deliverables
- Week 00 completion report.
- Verified desktop proof of concept.
- Architecture documentation.
- Module implementation matrix.
- Technical risk register.
- Prioritized backlog.
- Week 01 development plan.