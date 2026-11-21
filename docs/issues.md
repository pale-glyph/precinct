# Specification issues

This document tracks outstanding decisions and gaps in the [functional specification](functional-specification.md) and [architecture specification](architecture-specification.md). All issues below are open. When an issue is resolved, record the decision in the relevant specification and update this document.

**Priority:** Foundation issues need decisions before implementing the affected data model or behavior. API issues need decisions before the API and integration contracts can be completed. Production issues need decisions before production deployment. These categories indicate when a decision is needed, not a fixed implementation order.

The specifications now use PostgreSQL records, optimistic version checks, a single deployment-configured API key, and a transactional PostgreSQL audit table. Event sourcing and its associated projection and replay decisions are no longer in scope. Issue numbers are retained for continuity.

## Foundation

### ISS-01: Erasure and retained copies

Organisation sweeps erase child data while preserving shared accounts. Define which Precinct-owned records and references a sweep removes, and how audit entries and webhook deduplication data are treated. Metadata is opaque and client-owned, so decide whether Precinct's erasure guarantee covers references or personal data embedded in other records' metadata, or whether the client must remove them. Define backup retention, restoration safeguards, and when erasure is considered complete, including interrupted sweeps and restored copies.

### ISS-02: Transactions and cross-record invariants

Specify transaction scope, database constraints, and concurrency control for membership and Clerk user identifier uniqueness, valid references, ownership transfers, and operations racing with deletion or sweep. Define which changes advance which record versions, especially membership and account metadata changes. Version conflict and retry behavior is tracked in ISS-05.

### ISS-03: Clerk account mirror provisioning and recovery

Clerk owns accounts; Precinct creates mirrors identified by Clerk user ID and does not create or link independent accounts. Define how existing Clerk users are initially mirrored, what happens when an update or deletion arrives before creation, and how duplicate creations or events arriving after a sweep are handled. Define how clients handle an account that has not yet been mirrored. Future provider support is outside the first release.

### ISS-04: Lifecycle, ownership, and sweep coordination

Define the grace period and its configuration, when the period starts, whether repeated deletion requests reset it, and what happens if the sweep runs late. Define allowed operations on pending records and whether and when a Clerk-deleted account mirror can be restored or its deletion cancelled. Decide whether an owner must be a member, which account mirrors may own an organisation, and what happens when a mirror with owned organisations or memberships is swept. Specify membership removal, sweep scheduling, and concurrency with cancellation and other commands.

### ISS-05: Version checks and request idempotency

Define version representation and initial values, which API operations require an expected version, and conflict responses. Define whether no-op updates advance a version, and how clients retry a successful update whose response was lost without mistaking the resulting conflict for a competing change. Webhook ordering and reconciliation are covered separately in ISS-09.

## API and integration

### ISS-07: API contracts and record visibility

Define endpoints and operations, identifiers for memberships, account mirrors and Clerk users, request and response schemas, pagination, filtering, errors, and API versioning. Specify update semantics for omitted and null fields, lifecycle commands, and whether pending-deletion records appear in reads and lists. Define creation and retry behavior, including how clients obtain stable identifiers. Specify request-size limits and the meaning of a valid email and Unicode character for the existing field limits.

### ISS-09: Clerk webhook delivery and reconciliation

Define verification configuration, event identity and deduplication retention, atomic processing, acknowledgement timing, retry responses, durable handling of failures, and operational recovery. Specify how missing, duplicate, delayed, and out-of-order events are reconciled with Clerk and deletion (ISS-03 and ISS-05), and how reconciliation can access authoritative Clerk data. Decide how invalid mapped fields, such as values beyond Precinct's limits, are handled without endless retries or silent data loss. Preserve application-owned metadata when applying Clerk profile updates.

## Production readiness

### ISS-10: Audit table content and retention

Define the minimal fields needed to identify the action, affected record, source, and time without storing change details or secrets. Clarify whether membership and identity changes, failed or ignored requests, and each child removed by a sweep need entries. Define access controls and retention cleanup, and ensure these are compatible with erasure (ISS-01). The 90-day default and atomic commit with database changes are already specified.

### ISS-11: API key administration and deployment security

Define secret delivery, behavior when the key is missing at startup, and a coordinated procedure for replacing the single configured key in both service and client. Specify how configuration changes take effect, TLS termination, secret management, authentication failure handling and practical rate limiting. Key replacement may interrupt requests; multiple simultaneous keys are not part of the current design.

### ISS-12: Deployment, recovery, and operational targets

Define PostgreSQL schema and migration ownership, service startup and database readiness, health checks, upgrades, and backup and restore procedures. Set availability, recovery, observability, capacity, and performance targets. Include erasure-aware restoration procedures.

### ISS-13: Acceptance and verification criteria

Turn the expected outcomes and resolved decisions into testable scenarios. Cover concurrent uniqueness and references, ownership and lifecycle transitions, version conflicts and retries, webhook verification and ordering, erasure and restoration, transactional auditing, and API key replacement. Define which checks gate a release.
