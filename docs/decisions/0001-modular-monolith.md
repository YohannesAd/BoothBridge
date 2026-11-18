# ADR 0001: Use a modular monolith for the backend

- Status: accepted design direction; implementation pending
- Date: 2026-11-18
- Scope: backend application boundaries and deployment model

## Context

BoothBridge needs a small set of related event, company, link, contact, opportunity, note, and task workflows. These share ownership rules and relational references. The current repository contains documentation only. The MVP needs a manageable development and operational model while preserving clear feature boundaries.

## Decision

Build one Java 21/Spring Boot 4.1 backend, organized by feature with thin controllers, application services, explicit DTOs, and feature-owned persistence. Use one MariaDB schema with database constraints and local transactions. Maintain a separate Next.js frontend that calls the backend API; the modular monolith decision applies to the backend, not the number of runtime processes in the system.

Expose small application interfaces across feature boundaries and centralize only technical concerns such as error envelopes and principal access. Do not introduce independently deployed domain services, asynchronous messaging, or distributed data ownership for the MVP. See [architecture](../architecture.md) for responsibilities and dependencies.

## Alternatives considered

| Alternative | Assessment |
| --- | --- |
| Microservices by feature | Independent scaling/deployment may help at larger scale, but introduces network failures, distributed authorization, service operations, and cross-service consistency before a demonstrated need. |
| Unstructured layered monolith | Simple initially, but a single global controller/service/repository layout makes feature boundaries and ownership of rules harder to maintain. |
| Next.js-only full stack | Reduces runtime count, but does not meet the chosen Java/Spring backend direction and would place persistence and domain services in the web runtime. |

## Consequences and revisit conditions

- Local transactions, a single backend release, and direct debugging make the MVP easier to reason about and test.
- Package boundaries require discipline; a shared schema and runtime allow changes in one feature to affect others. Integration tests and narrow interfaces are required safeguards.
- The backend scales as a unit. In-memory sessions initially constrain backend replication and expire on restart; a replication/session design is required before scaling out.
- Extract a service only when measured scaling, fault isolation, deployment cadence, or team ownership justifies its operational and consistency cost. Supersede this ADR with an explicit data and authorization migration plan if that happens.
