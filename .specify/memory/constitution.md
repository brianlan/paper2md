<!--
Sync Impact Report
Version: 0.0.0 → 1.0.0
Modified Principles:
- PRINCIPLE_1_NAME → I. Readability Over Cleverness
- PRINCIPLE_2_NAME → II. Test-Driven Delivery
- PRINCIPLE_3_NAME → III. Relentless Automation
- PRINCIPLE_4_NAME → IV. Intentional Simplicity
- PRINCIPLE_5_NAME → V. Explain the Why
Added Sections:
- Delivery Workflow Standards
- Review & Quality Gates
Removed Sections:
- None
Templates requiring updates:
- ✅ .specify/templates/plan-template.md
- ✅ .specify/templates/spec-template.md
- ✅ .specify/templates/tasks-template.md
Follow-up TODOs:
- None
-->

# paper2md Constitution

## Core Principles

### I. Readability Over Cleverness
Code exists for humans first, so every contribution MUST optimize clarity and future comprehension.
- Prefer explicit control flow, descriptive naming, and straightforward data shapes over "magic".
- Reject clever shortcuts that hide intent or couple unrelated concerns.
- Document non-obvious constraints adjacent to the code so reviewers never guess.
Rationale: Clear code shortens onboarding time, reduces bug risk, and enables confident refactors.

### II. Test-Driven Delivery
Every feature begins with a failing test that captures the desired behavior before implementation.
- Follow Red → Green → Refactor with no skipping; untested code cannot merge.
- Use mocks/stubs for external services so tests run locally and deterministically.
- Keep tests descriptive enough that they double as executable documentation of intent.
Rationale: TDD provides proof of correctness, allows safe change, and anchors discussions on behavior.

### III. Relentless Automation
Anything repeatable MUST be automated so engineers spend time on solving problems, not running checklists.
- Linting, formatting, unit/integration suites, and documentation checks run in CI for every PR.
- Local scripts mirror CI to avoid "works on my machine" drift.
- Breaking the build blocks merges until automation is green; manual overrides are prohibited without
  written approval in the PR.
Rationale: Automation enforces consistency, accelerates feedback, and prevents regressions.

### IV. Intentional Simplicity
Solutions stay as small as possible while honoring KISS, DRY, SOLID, and YAGNI.
- Build only what the current user story requires; defer speculative hooks.
- Extract shared behavior once it is duplicated twice, not preemptively.
- Design APIs and modules with single responsibilities and clear extension seams.
Rationale: Simplicity reduces maintenance cost and keeps the codebase malleable.

### V. Explain the Why
Comments, docs, and commit messages MUST capture reasoning, not narration.
- Write comments only where intent or tradeoffs would be unclear without context.
- Reference decisions, constraints, or rejected alternatives with links to specs or tickets.
- Update comments whenever behavior changes so they never drift from reality.
Rationale: Understanding the "why" accelerates reviews, debugging, and onboarding.

## Delivery Workflow Standards
- Start every change by writing or updating failing tests that describe the behavior and mock all
  external touchpoints.
- Push small, reviewable commits that each include both the tests and the implementation that makes
  them pass.
- Keep automation scripts (lint, test, format, docs) in the repo and reference them from plans and
  tasks so every engineer runs the same commands.
- Capture rationale for deviations (e.g., unavoidable coupling) directly in the PR description and
  link to the relevant comment blocks in code.

## Review & Quality Gates
- Code review blocks merge until readability, TDD evidence, and automation results are all verified.
- Reviewers confirm every change references the tests that cover it and that mocks/stubs keep tests
  hermetic.
- CI must prove lint, unit, integration, and docs checks succeed; failures are never ignored.
- Review checklists call out readability, simplicity, and rationale to ensure the constitution stays
  enforceable.

## Governance
- This constitution supersedes informal practices; feature plans, specs, and tasks MUST cite how they
  satisfy each principle.
- Amendments require: (1) written proposal in the repo, (2) approval from at least two maintainers,
  (3) documented migration steps if behavior changes.
- Versioning follows SemVer: MAJOR for principle changes/removals, MINOR for new sections/principles,
  PATCH for clarifications.
- Compliance is reviewed during quarterly retros; gaps result in tracked remediation tasks within the
  backlog.

**Version**: 1.0.0 | **Ratified**: 2025-11-07 | **Last Amended**: 2025-11-07
