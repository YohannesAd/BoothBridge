# BoothBridge

A mobile-first conference networking and career opportunity tracker, inspired by attending AfroTech 2026.

BoothBridge is planned to bring event information, company QR links, recruiter and networking contacts, job opportunities, notes, and follow-up tasks into one private workspace. Attendees can review a scanned URL, save it under a company, and explicitly open it when ready.

## Current status

**Documentation foundation only.** There is no application implementation, dependency manifest, automated test suite, CI workflow, or deployment in this repository. All behavior described in the design documents is planned. No setup or run commands are available yet.

## MVP scope

- Private events and event-specific company entries.
- Multiple labeled links, contacts, and opportunities per company; event-level networking contacts are also supported.
- Camera QR scanning with manual URL entry as a fallback; review → save → explicitly open the destination.
- Independent booth check-in and job application tracking, plain-text notes, and follow-up tasks.
- Authentication, owner-based authorization, and a usable mobile interface.

AI assistance through Spring AI/OpenAI is deferred until after the core MVP. Sharing, automated applications, and offline synchronization are outside the MVP.

## Planned technology

| Area | Baseline |
| --- | --- |
| Web | Next.js 16, React, TypeScript; Tailwind CSS 4 and shadcn/ui |
| Backend | Java 21, Spring Boot 4.1, Spring Security |
| Persistence | MariaDB 12.3 LTS, InnoDB, Spring Data JPA, Flyway |
| Verification | JUnit, Mockito, Testcontainers, Vitest, Playwright |
| Automation | GitHub Actions; no workflow exists yet |

These are selected release lines, not installed dependencies. Pin compatible patch versions, the Node.js runtime, and the Java build tool during the first implementation milestone. Check the [Spring Boot requirements](https://docs.spring.io/spring-boot/system-requirements.html) and [MariaDB 12.3 LTS announcement](https://mariadb.org/mariadb-server-12-3-lts-released/) when doing so.

## Documentation

- [Product requirements and acceptance criteria](docs/product-requirements.md)
- [Architecture and planned repository structure](docs/architecture.md)
- [Database design and conceptual ERD](docs/database-design.md)
- [Initial REST API contracts](docs/api-contracts.md)
- [Testing strategy](docs/testing-strategy.md)
- [Security design](docs/security.md)
- [ADR 0001: Modular monolith](docs/decisions/0001-modular-monolith.md)
- [ADR 0002: MariaDB selection](docs/decisions/0002-mariadb-selection.md)
- [Contributor and agent conventions](AGENTS.md)

## First implementation milestone

Build one authenticated vertical slice: sign in → create an event → add a company → manually save and reopen an HTTPS link. Prove ownership isolation, database migrations, and persistence through a reload. Add meaningful unit, MariaDB integration, and browser checks to GitHub Actions. Camera scanning follows this foundation; see the [milestone acceptance criteria](docs/product-requirements.md#first-implementation-milestone).
