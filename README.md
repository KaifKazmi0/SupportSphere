# SupportSphere — Multi-Tenant Customer Support SaaS

> Project blueprint, architecture guide, feature specification, and learning roadmap.

## Product
SupportSphere lets multiple companies run customer-support operations on shared infrastructure while keeping tenant data isolated. Customers create tickets; agents respond; managers track SLAs and analytics.

## User roles
- **Platform administrator:** tenants, plans, platform health.
- **Tenant owner/admin:** workspace, membership, roles, settings.
- **Support manager:** queues, assignment, SLA and team reports.
- **Support agent:** ticket replies, internal notes, status changes.
- **Customer:** create tickets, attach files, view permitted history.

## Recommended stack
- **Backend:** Java 21 LTS, Spring Boot, Spring Security, Spring Data JPA.
- **Frontend:** React, TypeScript, Vite.
- **Database:** PostgreSQL.
- **Cache/ephemeral state:** Redis.
- **Real-time UI:** Spring WebSockets.
- **Background work:** RabbitMQ.
- **Event streaming:** Kafka.
- **Push notifications:** Firebase Cloud Messaging (FCM).
- **Optional attachment storage:** Firebase Storage or S3-compatible object storage.
- **Deployment:** Docker Compose, Nginx, GitHub Actions; Kubernetes later.
- **Testing/observability:** JUnit, Mockito, Testcontainers, Prometheus, Grafana, OpenTelemetry.

## What “streaming” means here
| Need | Technology | Reason |
|---|---|---|
| Live ticket/chat updates | WebSockets | Push changes to connected clients |
| Durable events for analytics/search/audit | Kafka | Replayable event streams and independent consumers |
| Background notification jobs | RabbitMQ | Work queues, acknowledgements, retries, dead-letter handling |
| Push to an inactive browser/phone | Firebase Cloud Messaging | Push delivery to registered clients |
| File attachments | Object storage | Files are not suitable for relational rows or Kafka payloads |

This project does **not** need video streaming initially. For future video calls, evaluate WebRTC and an SFU service separately.

## Firebase boundaries
- Use FCM for push notifications like “Agent replied” or “SLA deadline approaching.”
- Keep ticket data and authorization in the Spring Boot/PostgreSQL application.
- Store device tokens with user/device metadata and revoke invalid tokens.
- Avoid putting sensitive ticket text in push payloads; fetch ticket details after authentication.
- Firebase Authentication is optional. For this learning project, use one clear identity authority—Spring Security is the recommended starting point—rather than duplicating authentication systems.
- Firebase Storage is optional for attachments; validate tenant access on every upload/download.

## Architecture
```text
React client
    |
  Nginx (TLS / reverse proxy)
    |
Spring Boot modular monolith
  |        |         |
Postgres  Redis   WebSockets
  |
Transactional outbox
  |
Publisher -> Kafka -> analytics/search/audit consumers
          -> RabbitMQ -> notification worker -> Email / Firebase FCM
```

Use the **transactional outbox** for important events: commit the business change and outbox record in one database transaction, then publish asynchronously. Consumers must tolerate duplicate delivery through idempotency.

## Suggested modules
```text
backend/src/main/java/.../
  identity/        # users, login, tokens
  tenancy/         # tenant context and membership
  tickets/         # ticket lifecycle and assignment
  conversations/   # messages and internal notes
  notifications/   # preferences, FCM/email dispatch
  sla/             # deadlines and escalation
  analytics/       # reports/read models
  audit/           # business/security audit trail
  integrations/    # broker adapters
  shared/          # common errors and utilities
```

## Data model sketch
```text
tenants(id, name, plan, status, created_at)
users(id, email, password_hash, display_name, created_at)
tenant_memberships(tenant_id, user_id, role, status)
tickets(id, tenant_id, customer_id, assignee_id, subject,
        description, status, priority, created_at, updated_at)
ticket_messages(id, tenant_id, ticket_id, sender_id,
                message_type, body, created_at)
ticket_attachments(id, tenant_id, ticket_id, object_key,
                   filename, content_type, size_bytes)
ticket_events(id, tenant_id, ticket_id, event_type,
              actor_id, payload, created_at)
notification_preferences(user_id, channel, enabled)
device_tokens(id, user_id, platform, token, last_seen_at)
outbox_events(id, aggregate_type, aggregate_id, event_type,
              payload, created_at, published_at)
```

Tenant isolation rules:
1. Derive tenant context from authenticated membership, not an untrusted client field.
2. Scope every tenant-owned query.
3. Check ticket/object permissions, not only login status.
4. Use foreign keys, composite constraints, and indexes.
5. Consider PostgreSQL Row-Level Security as defense in depth.
6. Test cross-tenant access across REST, search, attachments, WebSockets, and exports.

## Ticket lifecycle
Example: `NEW -> OPEN -> IN_PROGRESS -> WAITING_FOR_CUSTOMER -> RESOLVED -> CLOSED`, with policy-defined reopen transitions. Define who can perform each transition and which SLA/event side effects occur. Internal notes must never be exposed to customers.

## API sketch
| Route | Purpose |
|---|---|
| `POST /api/v1/tenants` | Create workspace |
| `POST /api/v1/auth/login` | Authenticate |
| `GET /api/v1/tickets` | Tenant-scoped list with filters and pagination |
| `POST /api/v1/tickets` | Create ticket |
| `GET /api/v1/tickets/{id}` | Authorized ticket detail |
| `POST /api/v1/tickets/{id}/messages` | Reply or internal note |
| `PATCH /api/v1/tickets/{id}/status` | Validated state transition |
| `POST /api/v1/devices` | Register push token |
| `GET /api/v1/analytics/summary` | Manager analytics |

Use DTOs, validation, consistent errors, pagination, request IDs and OpenAPI. Do not expose database entities directly.

## Security checklist
- Hash passwords; never store plaintext.
- Use a deliberate access-token/session and refresh strategy.
- Enforce tenant membership, role and object-level authorization on every request.
- Rate-limit login, signup and public endpoints.
- Validate attachment size/type and access; avoid predictable public object URLs.
- Configure TLS, CORS, secure headers, and CSRF protections appropriate to the auth model.
- Never log secrets, tokens or sensitive message contents.
- Authenticate WebSocket handshakes and authorize each subscription.
- Add dependency/secret scanning, backups and restore drills.

## Testing
- **Unit:** ticket transitions, SLA calculations, authorization decisions — JUnit/Mockito.
- **Integration:** DB constraints and tenant scoping — Spring Boot Test/Testcontainers.
- **Broker:** retries, duplicates, poison messages, dead letters.
- **Security:** cross-tenant access attempts.
- **Frontend:** React Testing Library/Vitest.
- **End-to-end:** Playwright.
- **Load:** k6 or equivalent.

Record the environment, dataset size, concurrency, p50/p95 latency, throughput and error rate. Never invent benchmark results.

## Observability
Track request rate/errors/p95 latency, DB pool saturation, queue depth, Kafka consumer lag, notification failures and WebSocket connections. Use structured logs, correlation IDs, metrics, traces, timeouts, bounded retries, backoff and jitter. Test backup restoration.

## Ten-week roadmap
1. Requirements, schema, repository, Compose and PostgreSQL.
2. Login, memberships, roles and tenant isolation.
3. Ticket lifecycle, validation, pagination.
4. Messages, assignment, audit trail and React workflows.
5. WebSocket live updates.
6. RabbitMQ worker, email and FCM push.
7. Transactional outbox and Kafka analytics/audit consumer.
8. SLA timers, dashboards, caching and search.
9. CI/CD, security/load tests and monitoring.
10. Cloud release; Kubernetes and optional AI summaries after the core system is stable.

## Two-person split
- **Developer A:** identity, tenancy, tickets, database, authorization, backend integration tests.
- **Developer B:** React screens, API integration, frontend tests, live-update UI.
- Both review PRs and alternate ownership of messaging, deployment, and observability.

## Interview questions to prepare
- Why modular monolith first?
- Why WebSockets for live UI but Kafka for durable streams?
- When is RabbitMQ a better fit than Kafka?
- How does the transactional outbox prevent lost events?
- How do consumers handle duplicates?
- How do tenant isolation tests prevent data leakage?
- What can go wrong with cache invalidation?
- What if FCM is unavailable or a token expires?
- How do you measure p95 latency?
- What evidence would justify extracting a service?

## Definition of done
- Reproducible demo and setup guide.
- Cross-tenant access tests pass.
- Ticket lifecycle, replies, assignment and audit work end to end.
- WebSocket subscriptions are authorized.
- Notification retries and duplicate event handling are tested.
- Secrets are not committed; sensitive data is not logged.
- CI tests and builds the app.
- Metrics and an honest load-test report are available.
- README has architecture, API docs, screenshots and known limitations.

## Official references
- Spring Boot: https://spring.io/projects/spring-boot
- Spring Security: https://spring.io/projects/spring-security
- PostgreSQL: https://www.postgresql.org/docs/
- Redis: https://redis.io/docs/
- Kafka: https://kafka.apache.org/documentation/
- RabbitMQ: https://www.rabbitmq.com/docs
- Firebase Cloud Messaging: https://firebase.google.com/docs/cloud-messaging
- Firebase Storage: https://firebase.google.com/docs/storage
- React: https://react.dev/
- Docker Compose: https://docs.docker.com/compose/
- Kubernetes: https://kubernetes.io/docs/
- OpenTelemetry: https://opentelemetry.io/docs/

**Principle:** use technology to solve a defined problem. A smaller secure system with tests, clear failure handling and measured performance is stronger than a large stack that barely works.
