# Platform Architecture Blueprint

This document outlines the service topology, interfaces, data stores, and operational practices for the trading platform.

## Core Services

### Auth & Billing
- **Responsibilities:** user authentication (SSO/OAuth), authorization/entitlements, subscription management, payment processing, invoice generation, and webhooks for billing events.
- **Dependencies:** PostgreSQL for users, plans, invoices; payment provider webhooks; Redis for short-lived tokens and rate limits.
- **Interfaces:** REST for signup/login/subscription flows; webhooks for payment events; publishes entitlement updates to Redis Streams or RabbitMQ.

### Solver & Orchestration
- **Responsibilities:** manage solver workloads (range sims, equities, EV diffing), GPU scheduling, and result collation.
- **Dependencies:** GPU node pools in Kubernetes, object storage for artifacts (sim outputs, checkpoints), job queue for async runs, Redis cache for hot results.
- **Interfaces:** gRPC for low-latency solver RPCs; async job submission via REST/GraphQL; progress/events emitted to WebSocket channels.

### Content & Range Catalog
- **Responsibilities:** manage curated ranges, playbooks, drills, and editorial content; handle versioning and ownership.
- **Dependencies:** PostgreSQL for catalog metadata; object storage for media/range files; Redis for catalog caching and feature-flagged rollouts.
- **Interfaces:** REST/GraphQL for product surfaces; signed URL generation for media access.

### Trainer Session Service
- **Responsibilities:** coordinates training sessions (live or async), tracks steps, records feedback, and streams analytics to clients.
- **Dependencies:** PostgreSQL for session state; Redis for presence and real-time session coordination; WebSocket for streaming; object storage for session artifacts (replays, notes).
- **Interfaces:** REST/GraphQL for session lifecycle; WebSocket for trainer/live analysis streaming; emits events to observability stack.

### Hand-History Ingestion
- **Responsibilities:** parse, normalize, and enrich uploaded hand histories; detect formats and extract stats.
- **Dependencies:** object storage for raw uploads; PostgreSQL for parsed records; job queue for CPU-heavy parsing; Redis for deduplication and rate limits.
- **Interfaces:** REST upload endpoints; async parse jobs with status callbacks; publishes enriched hands to downstream analytics (Prometheus/OpenTelemetry events).

### Notification & Ad Service
- **Responsibilities:** deliver email/push/in-app notifications, and manage ad/placement experiments.
- **Dependencies:** PostgreSQL for campaigns, user preferences, and experiments; Redis for rate limits; feature-flag/A-B framework for rollouts; third-party channels for delivery.
- **Interfaces:** REST for campaign management; message queue for async sends; Webhook/WebSocket hooks for in-app delivery.

## API Surface
- **gRPC:** solver calls and internal low-latency control paths.
- **REST/GraphQL:** product experiences (catalog, sessions, billing, notifications) and administrative tooling.
- **WebSocket:** trainer/live analysis streaming, solver progress, and in-app notification delivery.
- **Webhook:** payment provider events, notification callbacks.

## Persistence Strategy
- **PostgreSQL:** source of truth for users, payments, sessions, catalog metadata, campaigns/experiments, and parsed hand histories.
- **Object Storage (S3/GCS):** artifacts (solver outputs, range files, media, session recordings, raw uploads).
- **Redis:** cache, rate limiting, presence, short-lived tokens, and fast feature-flag evaluations.

## Orchestration & Runtime
- **Kubernetes:** multi-pool clusters with GPU node pools dedicated to solver workloads; general pools for control-plane and stateless services.
- **Autoscaling:** Horizontal Pod Autoscaling on request/concurrency metrics; cluster autoscaler tuned for bursty solver demand; queue depth-based scaling for workers.
- **Job Queue:** RabbitMQ or Redis Streams for async simulations, parsing, and notification fan-out; worker roles separated by GPU/CPU needs.
- **Configuration:** environment-based configuration with secrets in vault/KMS; per-namespace isolation for prod/stage/dev.

## Observability & Safety
- **Tracing:** OpenTelemetry instrumentation across gRPC/HTTP/WebSocket pathways; baggage/propagation enabled for solver jobs.
- **Metrics:** Prometheus (with exemplars) for SLIs on latency, error rate, queue depth, and GPU utilization; dashboards per service.
- **Logging:** structured JSON logs shipped to centralized storage; correlation IDs propagated from edge.
- **Feature Flags & A/B:** centralized flag service controlling catalog rollouts, notification treatments, and solver betas; experiment assignments recorded in PostgreSQL.
- **Operational Runbooks:** alerting for queue backlog, GPU pool saturation, billing webhook failures, and cache pressure; chaos drills for failover scenarios.
