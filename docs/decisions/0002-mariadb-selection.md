# ADR 0002: Use MariaDB 12.3 LTS with InnoDB

- Status: accepted design direction; implementation pending
- Date: 2026-11-18
- Scope: primary transactional store and schema evolution

## Context

Attendee-owned events contain related company entries, links, contacts, opportunities, notes, and tasks. The MVP needs reliable writes, foreign-key integrity, transactional reference checks, and ordinary filtered lists. There is no implemented database, migration, test container, or hosting choice yet.

## Decision

Use the MariaDB 12.3 LTS release line with InnoDB, Spring Data JPA, and versioned Flyway migrations. MariaDB identifies 12.3 as an LTS release in its [release announcement](https://mariadb.org/mariadb-server-12-3-lts-released/). Select and pin a supported patch version during implementation; do not rely on a floating `latest` image.

Use UTF-8 user text, explicit relational constraints, UTC instants, and event-local calendar dates/time zones. Flyway owns schema evolution; JPA validates mappings rather than modifying the schema automatically. Integration tests use the same MariaDB release line through Testcontainers. See [database design](../database-design.md) for relationships and constraints.

## Alternatives considered

| Alternative | Assessment |
| --- | --- |
| PostgreSQL | Strong relational alternative with a different feature ecosystem; no demonstrated MVP requirement outweighs the selected MariaDB direction. This decision does not claim MariaDB is universally faster or better. |
| MySQL | Similar relational fit, but it is a different product with compatibility differences; selecting it would not eliminate version-specific validation. |
| SQLite | Useful for embedded/local applications, but would require different concurrency and deployment assumptions from the intended multi-user server database. |
| Document database | Flexible records, but the ownership hierarchy and same-event/company constraints fit relational foreign keys and transactions without duplicating related data. |
| H2 as an integration substitute | May accelerate some isolated checks, but cannot prove MariaDB SQL, migrations, collation, foreign keys, or transaction behavior. It is not the integration-test database. |

## Consequences and revisit conditions

- Relational constraints and local transactions protect relationships while keeping the application model straightforward.
- MariaDB requires operations work: credentials, patching, capacity, backups, recovery tests, and deployment migration coordination. An LTS label does not perform those tasks or guarantee compatibility.
- Verify the chosen Boot-managed JPA/Hibernate versions, MariaDB JDBC driver, Flyway database support, and Testcontainers image together before declaring the baseline working. Avoid unnecessary dependency overrides.
- Test the selected release's actual isolation behavior; the [12.3 announcement](https://mariadb.org/mariadb-server-12-3-lts-released/) calls out a changed InnoDB snapshot-isolation default. Do not assume behavior from a different database or older MariaDB release.
- Revisit if hosting availability, measured query/concurrency behavior, required database capabilities, or team operations costs justify a migration. A change requires a new ADR and migration/recovery plan, not just a driver swap.
