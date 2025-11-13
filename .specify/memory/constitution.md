<!--
Sync Impact Report:
Version: 1.1.0 (Added documentation language requirement)
Changes:
  - v1.0.0 (2025-11-14): Initial creation with 6 core principles
  - v1.1.0 (2025-11-14): Added Principle VII - Documentation Language (NON-NEGOTIABLE)

Modified Principles:
  - NEW: Principle VII - Documentation Language (zh-TW requirement)

Added Sections:
  - Documentation language requirements for all user-facing and technical docs
  - Exceptions list for code and technical terms

Templates Impact:
  - ⚠️ spec-template.md: Should be translated to zh-TW or include zh-TW instructions
  - ⚠️ plan-template.md: Should be translated to zh-TW or include zh-TW instructions
  - ⚠️ tasks-template.md: Should be translated to zh-TW or include zh-TW instructions
  - ✅ Constitution compliance checklist updated with documentation language requirement

Follow-up TODOs:
  - Consider translating template files to zh-TW
  - Update /speckit commands to enforce zh-TW documentation generation
-->

# LINE Bot Autopilot Constitution

## Core Principles

### I. Security First (NON-NEGOTIABLE)

Remote automation poses significant security risks. Every feature MUST:

- Require explicit user authorization before executing system commands
- Validate and sanitize all user inputs to prevent command injection
- Implement secure credential storage (never hardcode passwords, API keys, or meeting credentials)
- Use secure communication channels (HTTPS, encrypted webhooks)
- Log all security-relevant operations with timestamps and user identifiers
- Fail securely: On error, deny access rather than grant it

**Rationale**: Remote system control via messaging creates attack vectors. Security cannot be retrofitted—it must be designed in from the start.

### II. User Confirmation for Critical Actions

Users interacting via mobile devices can accidentally trigger commands. The system MUST:

- Require explicit confirmation for destructive or irreversible actions (e.g., stopping recording, leaving meetings)
- Present clear, actionable confirmation dialogs with Yes/No options
- Timeout confirmation requests after reasonable period (default: 60 seconds)
- Cancel operations when confirmation is denied or times out
- Provide clear status feedback after every action

**Rationale**: Prevents accidental execution of critical operations on mobile devices where touch errors are common.

### III. Robust Error Handling

Remote automation failures must be gracefully communicated to users. Every operation MUST:

- Catch and handle all exceptions without crashing the bot
- Provide meaningful error messages to users (avoid technical jargon)
- Log detailed error context for debugging (stack traces, system state)
- Implement retry logic with exponential backoff for transient failures
- Define fallback behavior for each failure scenario
- Never leave the system in an inconsistent state

**Rationale**: Users are remote and cannot directly observe system failures. Clear error communication and automatic recovery are essential.

### IV. Status Visibility

Remote users cannot see the controlled computer. The system MUST:

- Provide current status on demand (meeting state, recording state, screen snapshot)
- Update users proactively when state changes (meeting joined, recording started/stopped)
- Include timestamps in all status reports
- Support screenshot capture for visual confirmation
- Maintain audit log of all state transitions

**Rationale**: Remote operation requires trust. Status visibility builds confidence and enables users to verify system behavior.

### V. Testability and Integration Testing

Remote automation requires high reliability. Development MUST follow:

- Test-driven development: Write tests before implementation
- Integration tests for all critical paths (Zoom joining, screen recording, LINE messaging)
- Contract tests for all external integrations (LINE Bot API, Zoom, screen recording tools)
- End-to-end tests simulating complete user journeys
- Mock external dependencies in unit tests
- Automated testing in CI/CD pipeline

**Rationale**: Remote automation failures impact user trust and productivity. Comprehensive testing ensures reliability.

### VI. Platform-Specific Reliability

macOS automation has platform-specific constraints. Implementation MUST:

- Use stable, documented APIs (prefer native frameworks over scripting when possible)
- Handle macOS permission requirements (screen recording, accessibility, automation)
- Gracefully degrade when permissions are missing (inform user with instructions)
- Test across supported macOS versions
- Document system requirements and setup procedures
- Handle macOS-specific edge cases (window focus, full-screen mode, multi-display setups)

**Rationale**: macOS platform has unique security models and automation constraints that must be explicitly addressed.

### VII. Documentation Language (NON-NEGOTIABLE)

All user-facing documentation and technical documentation MUST be written in Traditional Chinese (zh-TW). This includes:

- User manuals and guides
- README files
- Feature specifications (spec.md)
- Implementation plans (plan.md)
- Task lists (tasks.md)
- Code comments for complex logic
- Error messages displayed to users
- Status messages and notifications

**Exceptions allowed**:
- Code (variable names, function names, class names) should remain in English for maintainability
- Technical terms without common Chinese equivalents may remain in English with Chinese explanation
- Third-party library names and framework references
- Git commit messages (may be in English or Chinese)

**Rationale**: Primary users are Traditional Chinese speakers. Documentation in their native language reduces cognitive load, prevents misunderstandings, and improves user experience and developer efficiency.

## Security Requirements

### Credential Management

- **MUST** use environment variables or secure configuration files for API keys (LINE Bot token, Zoom API credentials)
- **MUST** encrypt stored credentials at rest
- **MUST** never log credentials in plaintext
- **MUST** rotate credentials according to security policy (minimum: annually)
- **MUST** implement credential validation on startup

### Access Control

- **MUST** authenticate LINE Bot webhook requests (validate signatures)
- **MUST** maintain whitelist of authorized LINE user IDs (configurable)
- **MUST** rate-limit requests to prevent abuse
- **MUST** implement session timeouts for confirmation flows
- **MUST** audit all access attempts (successful and failed)

### Data Privacy

- **MUST** minimize data collection (collect only what's necessary for functionality)
- **MUST** encrypt data in transit (HTTPS/TLS for all communications)
- **MUST** implement data retention policy for logs and recordings
- **MUST** provide mechanism to delete user data on request
- **MUST** comply with applicable privacy regulations

## Development Workflow

### Feature Development Lifecycle

1. **Specification** (`/speckit.specify`): Define user requirements with security considerations
2. **Clarification** (`/speckit.clarify`): Resolve ambiguities, especially around security and error handling
3. **Planning** (`/speckit.plan`): Design technical approach with security gates
4. **Task Generation** (`/speckit.tasks`): Break down into testable, secure implementation units
5. **Implementation** (`/speckit.implement`): Execute with test-first approach
6. **Review**: Verify constitution compliance before merge

### Constitution Compliance

Every feature MUST pass constitution check before implementation:

- [ ] Security First: Authorization, input validation, secure storage verified
- [ ] User Confirmation: Critical actions require explicit confirmation
- [ ] Error Handling: All failure modes handled with user-friendly messages
- [ ] Status Visibility: Users can query and receive proactive status updates
- [ ] Testability: Integration and contract tests defined
- [ ] Platform Reliability: macOS permissions and edge cases addressed
- [ ] Documentation Language: All documentation written in Traditional Chinese (zh-TW)

### Code Review Requirements

All pull requests MUST:

- Include test coverage for new functionality (minimum 80% line coverage)
- Pass automated security scanning (no hardcoded credentials, no injection vulnerabilities)
- Document security considerations in PR description
- Include integration test results showing actual Zoom/recording/LINE Bot interaction
- Verify error handling with failure scenario tests
- Update documentation for new features or changed behavior

## Governance

### Amendment Process

Constitution amendments require:

1. Documented justification (why current principles are insufficient)
2. Impact analysis on existing features
3. Review by project maintainers
4. Version bump following semantic versioning
5. Migration plan for affected code
6. Update to dependent templates and documentation

### Versioning Policy

- **MAJOR** (X.0.0): Backward-incompatible changes to core principles (e.g., removing security requirement)
- **MINOR** (0.X.0): New principles added or existing principles materially expanded
- **PATCH** (0.0.X): Clarifications, wording improvements, non-semantic changes

### Compliance Review

- Constitution compliance MUST be verified during code review
- Violations of NON-NEGOTIABLE principles result in automatic PR rejection
- Complexity requiring principle violations MUST be justified in plan.md Complexity Tracking section
- Quarterly audit of codebase for constitution adherence

**Version**: 1.1.0 | **Ratified**: 2025-11-14 | **Last Amended**: 2025-11-14
