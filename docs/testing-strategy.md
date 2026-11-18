# Testing strategy

Status: planned verification. There are no test files, dependencies, commands, CI workflows, coverage reports, or recorded passing checks in this repository yet. Introduce tests with each implementation slice rather than building the entire application first.

## Test layers

| Layer | Tools | What to prove |
| --- | --- | --- |
| Backend unit | JUnit and Mockito | Application rules, ownership decisions, independent workflow states, date handling, reference validation, and mapping. Mock external collaborators; test pure rules without starting Spring. |
| Frontend unit/component | Vitest and a compatible component testing library selected during setup | Form validation, pending/error/success states, accessible feedback, QR result review, draft handling, and explicit destination confirmation. Mock the API boundary, not component internals. |
| Backend integration | JUnit, Spring Boot test support, Testcontainers with MariaDB 12.3 | Flyway migrations, JPA mappings, real foreign keys/checks, transactions, optimistic locking, scoped queries, pagination, and HTTP/security behavior. Use the selected MariaDB patch image, not H2 as a substitute. |
| End-to-end | Playwright against the web app and backend | Attendee journeys across UI, session, API, and persistence, including failure cases. Use synthetic identities and records and a controlled test identity provider. |
| Device/exploratory | Real iOS Safari and Android Chrome | Camera permissions, decoder behavior, tap-to-open behavior, narrow layouts, keyboard/focus, and conference-like network interruption. Browser emulation is insufficient evidence for real camera support. |

## Required scenarios

| Area | High-value checks |
| --- | --- |
| Ownership | Two synthetic accounts; prove the second cannot list, read, patch, or delete the first's event or children. Test every scoped reference and ID filter, including a valid ID from another event owned by the same user. |
| Identity/security | OIDC issuer/subject mapping, session renewal and logout, signed-out `401`, valid-session CSRF rejection, allowed callback/return paths, no private caching, and safe error bodies. Test actual security filters rather than bypassing them for protected tests. |
| URL policy | Valid HTTPS URLs and preserved query/fragment; reject unsupported schemes, relative URLs, user-info credentials, literal IPs, local/internal hosts, unusual ports, embedded whitespace/control characters, and backslashes. Include IDN/punycode and parser edge cases in shared fixtures. Verify saves do not trigger server destination requests. |
| QR flow | One reviewed result per scan session; no navigation on decode; manual fallback after denied/unavailable camera; invalid payload feedback; camera cleanup; pending submission disabled. Save failure retains the active draft; success offers an explicit open action. |
| Separate states | Scanning, saving, opening, or changing link category never changes check-in or application state. An opportunity status change leaves company check-in and sibling opportunities untouched. Check explicit reset/correction behavior and timestamps. |
| Database | Blank-schema Flyway migration and JPA validation; invalid cross-event/company references rejected by the database; transaction rollback; duplicate URLs allowed; restricted parent/opportunity deletion. Add upgrade tests once a released prior migration baseline exists. |
| Concurrency | Two writers with one ETag: one succeeds and the other receives `412`; no silent lost update. Include conditional deletion and simultaneous dependency creation/deletion. |
| Data validation | Blank/overlong fields, unknown/null fields, malformed JSON/UUIDs, invalid enums, date range, time zone, future application date, UTC date boundaries, and invalid pagination/filter parameters. |
| Notes/tasks | Plain-text rendering escapes HTML; notes retain line breaks. Complete/reopen timestamps, due-time display across daylight-saving changes, task filters, and event-only records behave as documented. |
| Error recovery | API validation errors, expired sessions, disconnected saves, dependency conflicts, uncertain POST results, and stale forms produce actionable feedback without false success or automatic duplicate submission. |

Use an injected clock for application date and timestamp rules. Parameterize status and URL cases where useful. Test outcomes and invariants rather than private methods or mocks mirroring every implementation line.

## Core E2E journeys

1. **First milestone:** sign in → create event → add company → manually enter/review/save an HTTPS link → reload → inspect and explicitly open the saved destination. Verify another account cannot access it.
2. **QR capture:** use a deterministic synthetic QR payload → review → label → save → explicitly open. Separately verify denied-camera fallback and a manual real-device scan.
3. **Conference follow-through:** create multiple links, contacts, and opportunities under one company; mark the booth checked in and one opportunity applied; confirm siblings remain unchanged; save a note and complete/reopen a follow-up task.
4. **Recovery and deletion:** reject unsafe URLs, retain a draft after failure, resolve a stale edit, unlink dependent records, then delete in the documented order.

Intercept or stub external destination navigation in browser tests; do not submit applications, contact people, depend on live career pages, or claim completion on another website. Test OIDC through a local controlled provider in CI, with a separate manual smoke check against the chosen real provider when configured. Do not use production accounts or weaken production security for tests.

## Test data and CI plan

- Use reserved example domains, fabricated company names, and generated UUIDs. QR fixtures must have no private tokens or real personal details.
- Give each test an isolated user/event scope; use disposable MariaDB containers and deterministic cleanup. Never run against a production database.
- Redact URLs, query strings, cookies, tokens, contacts, and note text from logs. Limit CI screenshot/trace artifacts to synthetic data and define short artifact retention when configuring workflows.
- For implementation PRs, plan jobs for frontend lint/type/unit checks, backend compilation/unit tests, MariaDB integration tests, and a small critical Playwright suite. Run broader browser/device checks for scanner or navigation changes. CI needs a container-capable runner for Testcontainers.
- Documentation-only changes require links, consistency, examples, and diff checks; they do not imply application tests ran. Do not add a workflow with placeholder commands.
- A relevant required check should block merge once configured. Avoid unexplained retries masking flaky tests; retain actionable diagnostics and fix isolation problems.

## Release evidence and unresolved items

Do not set an arbitrary coverage percentage as a substitute for critical-path checks. A feature is ready for review when its acceptance criteria, negative paths, and applicable integration/browser checks have evidence. Introduce measured performance budgets with realistic event sizes before production; none exist yet.

Runtime versions, build commands, component testing helpers, CI job names, performance targets, and the test OIDC provider remain to be selected during implementation. Document exact runnable commands in the README once verified.
