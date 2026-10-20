# NotifyX

**A reusable, multi-tenant notification infrastructure platform** built with TypeScript, NestJS, PostgreSQL, Redis, and BullMQ.

NotifyX is designed to let multiple applications send notifications through a shared service instead of reimplementing notification delivery in every project.

## Goals

- Multi-tenant organization and application model with strict application-level isolation.
- Email and in-app notifications in the initial release.
- Combined channel routing based on request intent, application defaults, recipient preferences, and applicable policies.
- Durable notification records in PostgreSQL.
- Transactional outbox for reliable publication of background work.
- Redis and BullMQ for asynchronous processing.
- Bounded retries with exponential backoff, jitter, and error classification.
- Idempotent requests and independently tracked channel deliveries.
- Authorized, auditable recovery for exhausted or uncertain deliveries.
- Reusable TypeScript SDK for integrating current and future applications.

## Architecture (planned)

```text
Applications
    |
    v
REST API / TypeScript SDK
    |
    v
Authentication + tenant-scoped authorization
    |
    v
Notification core
    |-- routing, preferences, templates
    |-- PostgreSQL transaction:
    |     notification + deliveries + outbox events
    v
Outbox dispatcher -----> Redis / BullMQ
                              |
                    +---------+---------+
                    v                   v
              Email worker         In-app worker
                    |                   |
              Provider adapter     Persist notification
                    |                   |
             Local mail sink       Socket.IO (real-time)
```

This is the target design, not an assertion that these components are already implemented. The first deployment will be a **modular monolith**; modules will only be separated into services if measured requirements justify it.

## Technology choices

- **Runtime / language:** Node.js and TypeScript
- **Backend:** NestJS
- **Database:** PostgreSQL
- **Database access:** Prisma
- **Queue:** BullMQ backed by Redis
- **In-app real time:** Socket.IO
- **Validation:** NestJS DTO validation at HTTP boundaries; use consistent typed contracts internally
- **Testing:** Jest and integration tests
- **Local environment:** Docker Compose
- **Package manager:** pnpm workspaces

## Core domain concepts

- **Tenant:** An organization using NotifyX.
- **Application:** A product registered under one tenant, with its own credentials and configuration.
- **Recipient:** An application-scoped mapping to a user owned by that application.
- **Notification:** One logical request.
- **Delivery:** One channel-specific delivery for a notification.
- **Delivery attempt:** A recorded attempt to process a delivery.
- **Outbox event:** Durable work committed in the same PostgreSQL transaction as the notification records.

Applications remain the source of truth for their user accounts. NotifyX stores only the notification-specific recipient information required to deliver and manage notifications.

## Reliability principles

1. PostgreSQL is the durable source of truth; Redis is used for background work, not permanent notification history.
2. Notification, channel-delivery, and outbox records are committed atomically.
3. Outbox publication and queue processing are designed for at-least-once execution; duplicate jobs must be safe.
4. Idempotency keys protect logical request creation. Reusing a key with different request content must return a conflict.
5. Email and in-app delivery outcomes are tracked independently.
6. Retries use error classification, exponential backoff with jitter, and bounded attempts.
7. An uncertain external-provider outcome is not blindly treated as a definite failure.
8. In-app notifications are persisted before real-time socket delivery.
9. Tenant/application scope is derived from authenticated credentials, never trusted from a client-provided tenant ID.
10. Secrets and unnecessary personal information must not be written to logs.

**Delivery semantics note:** We aim for durable, at-least-once processing with idempotent state transitions. Exactly-once delivery to an external email provider cannot be assumed.

## Development roadmap

- [ ] **Milestone 1 — Foundation:** repository structure, pnpm workspace, NestJS application, Docker Compose, PostgreSQL, Redis, environment configuration.
- [ ] **Milestone 2 — Data layer:** Prisma schema, tenant-safe relationships, constraints, indexes, migrations, database tests.
- [ ] **Milestone 3 — Security:** API keys, overlapping key rotation, scopes, tenant isolation, audit logging.
- [ ] **Milestone 4 — Pipeline:** idempotent notification API, transactional outbox, dispatcher, BullMQ workers, retries, recovery.
- [ ] **Milestone 5 — Channels:** local mail sink, provider adapter, in-app notification storage, Socket.IO delivery.
- [ ] **Milestone 6 — Integration:** templates, preferences, SDK, integration tests, observability, deployment hardening.

## Local development

Local development instructions will be added as Milestone 1 establishes the workspace and Docker environment. Never commit real credentials or production recipient data. Use `.env.example` for variable names and keep local `.env` files out of version control.

## Security status

NotifyX is under development. Do not use it for production notifications until authentication, authorization, tenant-isolation tests, provider webhook verification, retention, backups, rate limits, and operational recovery have been implemented and reviewed.

## License

No license has been selected yet.
