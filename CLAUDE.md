# CLAUDE.md

Guidance for working in this repository.

## What this is

A personal media tracker for **books, movies/TV, and video games** - track what you're consuming, have consumed, or want to consume, with ratings, reviews, and consumption insights. Built as a multi-service Go system on Kubernetes, primarily as a vehicle for learning Go microservices, gRPC, Kafka, Pulumi, Kubernetes (CKA prep), and the observability stack.

## Current state

Greenfield. Only `services/media/go.mod` exists so far (module `github.com/usmanzaheer1995/home-media-tracker/media`, Go 1.26). The `proto/` and `infra/` directories are empty. Most of this file describes the intended target architecture, not yet-written code — treat it as the design contract to build toward.

## Architecture

Five Go services behind a gateway, communicating via gRPC (sync) and Kafka (async). Each service owns its own Postgres database — no shared schemas.

- **gateway** — only service speaking REST to the outside world; translates HTTP → gRPC (BFF / gRPC-gateway pattern).
- **media** — external API integration (TMDB, OpenLibrary, IGDB), caches canonical media records in Postgres, exposes search.
- **tracking** — CRUD for tracking entries + progress; Kafka producer of `media.started`, `media.finished`, `media.dropped`.
- **review** — ratings and text reviews; Kafka consumer of `media.finished`.
- **recommendation** — pulls rating history from review, queries external APIs for similar titles, returns ranked suggestions.
- **insights** — Kafka consumer of tracking/review events; aggregates consumption stats over time.

**Sync (gRPC):** gateway → any service; recommendation → media; tracking → media (resolve item on add).
**Async (Kafka):** tracking emits events; review and insights consume.

## Layout

```
proto/        # Shared protobuf contracts, versioned (media/v1, tracking/v1, ...)
services/     # One Go module per service: gateway, media, tracking, review, recommendation, insights
infra/        # Pulumi Go SDK, layered stacks:
              #   cluster/  → k3s / Hetzner VM provisioning
              #   platform/ → Kafka, Postgres, observability, ingress, ArgoCD
              #   apps/     → per-service Kubernetes manifests
```

Pulumi stack dependency order: `apps` → `platform` → `cluster`.

## Tech stack & conventions

- **Language:** Go. Module path prefix `github.com/usmanzaheer1995/home-media-tracker/<service>`.
- **Inter-service:** gRPC + protobuf. Contracts live in `proto/<service>/v1/`, generated to Go via `protoc`. Each service runs a gRPC server; callers use generated client stubs.
- **HTTP routing:** Chi — gateway only.
- **Database:** sqlc + pgx (type-safe SQL, no ORM). Postgres per service.
- **Messaging:** franz-go (pure-Go Kafka client), KRaft mode (no Zookeeper).
- **Config:** Viper.
- **Logging:** `slog` (stdlib), structured JSON. Inject trace IDs to correlate with traces.
- **Metrics:** `prometheus/client_golang`; expose `/metrics`. Build RED dashboards per service.
- **Tracing:** OpenTelemetry Go SDK → Tempo, including gRPC interceptors across service boundaries.
- **IaC:** Pulumi Go SDK.

Each service should expose a `/healthz` endpoint for Kubernetes liveness/readiness probes.

## Build order (intended)

1. media (TMDB integration + Postgres cache + search) — start here, immediately demoable.
2. tracking (add items, update status) — the core loop.
3. Wire tracking → media (resolve items) for an end-to-end flow.
4. review — introduce Kafka here (tracking emits, review consumes).
5. insights — subscribe to events, accumulate stats.
6. recommendation — last; depends on real data in the others.

## External APIs

- **TMDB** (movies/TV) — free, API key.
- **OpenLibrary** (books) — free, no key.
- **IGDB** (games) — free, requires a Twitch developer account.

## Notes for agents

- When adding a service, mirror the existing module/path conventions and keep its database private to it.
- Define or update the `.proto` contract before implementing a gRPC change, then regenerate.
- Events are the integration boundary for tracking → review/insights; prefer adding event types over new synchronous calls where it fits the async pattern.
