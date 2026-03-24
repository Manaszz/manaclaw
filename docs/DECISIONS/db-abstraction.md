# ADR: Database Abstraction Layer

**Status:** Accepted

**Date:** 2026-03-24

## Context

NanoClaw uses SQLite directly via `better-sqlite3`. That suits single-node and local setups but does not meet typical corporate requirements for managed PostgreSQL, HA, and operational tooling.

## Decision

Introduce a **DatabaseAdapter** interface with two implementations:

- **SQLiteAdapter** — default for local development and simple deployments.
- **PostgresAdapter** — optional for enterprise and production PostgreSQL.

Selection is controlled via `config.toml` (or equivalent configuration), not compile-time flags.

## Consequences

**Positive**

- Zero-config local development with SQLite by default.
- Enterprise readiness when PostgreSQL is required for scale, backup, and DBA practices.

**Negative**

- Additional code paths and tests to maintain for two adapters.
- Schema migrations must stay synchronized and validated across both backends.

**Neutral**

- Application code targets the adapter contract; SQL dialect differences are isolated in adapter implementations.
