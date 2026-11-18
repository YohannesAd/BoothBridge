# Repository working agreements

These instructions apply to the entire repository and to human contributors and coding agents. Follow explicit task scope; a documentation request does not authorize application scaffolding, dependency installation, or Git publication.

## Current foundation

- This repository contains design documentation only. Treat documented application behavior and testing as planned until implementation and evidence exist.
- Preserve existing work and inspect `git status`, the active branch, and relevant files before editing. Do not discard unrelated changes.
- Keep documentation concise and consistent. Update affected requirements, API contracts, security guidance, and ADRs when changing a design.
- Never include real attendee or recruiter information, credentials, tokens, or private QR URLs. Use synthetic fixtures and reserved example domains.

## Architecture and implementation

- Use a modular monolith with feature-based organization; follow [architecture.md](docs/architecture.md).
- Keep Next.js routes and Java controllers thin. Separate request validation, application rules, persistence, and presentation.
- Spring Boot owns business rules and authorization. Do not duplicate them in Next.js API handlers or Server Actions.
- Use explicit request/response DTOs; do not expose JPA entities. Validate all client input server-side.
- Enforce event ownership on every scoped read and write, including nested references. IDs are not authorization.
- Keep booth check-in independent from application status. Scanning, saving, and opening a URL must not imply either status.
- Use Flyway for schema changes; do not rely on JPA schema auto-update. Preserve applied migration history and add a forward migration.
- Avoid microservices, generic frameworks, speculative abstractions, and AI features before the core MVP.
- Record significant decisions in `docs/decisions/NNNN-short-title.md`: context, decision, alternatives, consequences, status, and date. Supersede an ADR explicitly when reversing it.

## Git and GitHub conventions

- Use focused branches: `docs/<topic>`, `feat/<topic>`, `fix/<topic>`, `test/<topic>`, or `chore/<topic>`.
- This foundation task stays on `docs/project-foundation`. Do not switch branches or create commits, pushes, merges, or pull requests for this task.
- For future implementation work, identify the configured default branch before branching. Do not assume its name or change repository settings.
- When commits are requested, use short imperative Conventional Commit subjects, for example `docs: define QR link workflow` or `feat(links): save a company link`.
- Do not commit generated output, dependency caches, local environment files, or secrets. Commit lockfiles and build wrappers when introduced.
- Use the pull request template. Explain the problem, resulting behavior, scope, verification actually performed, and relevant risks or ADRs. Reference an issue when one exists.
- Require relevant checks and reviewer approval before merge as a proposed team policy; branch protection is not configured by this document. Prefer squash merges for focused PRs unless the team chooses otherwise.
- Never force-push, rewrite shared history, or perform destructive Git operations without explicit authorization. Do not publish changes merely because editing was authorized.

## Verification and reporting

- Discover commands from actual manifests and build files once they exist. Do not invent run commands, green checks, coverage figures, or deployment status.
- For documentation-only work, verify relative links, terminology, endpoint/schema consistency, examples, whitespace, and the diff. Do not install tooling just to check prose.
- For implementation work, run the relevant existing lint, type, unit, integration, and E2E checks; follow [testing-strategy.md](docs/testing-strategy.md). Explain checks that could not run and why.
- Test business rules and failure paths, especially cross-user access and unsafe URLs. Keep test data synthetic and tests deterministic.
- Keep security controls intact; do not disable authentication or CSRF to make a test pass.
- Before handoff, review the diff and report changes, validation results, limitations, and open decisions. Stop at the requested review boundary.
