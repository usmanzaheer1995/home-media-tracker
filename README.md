# Home Media Tracker

## Concept

Track media you are **consuming**, **have consumed**, or **want to consume** across three types:

- Books
- Movies & TV Shows
- Video Games

The system tracks history, allows ratings and reviews, and surfaces insights about consumption habits and tastes over time.

---

## Context

The stack covers the following key skill gaps:

- Multi-service Go applications
- Kubernetes
- Pulumi for infrastructure as code (Go SDK)
- Kafka for async messaging
- gRPC for inter-service communication
- Observability: Prometheus, Grafana, Loki, Tempo, OpenTelemetry

---

## The Services

### 1. Media Service
The core catalog. Talks to external APIs to fetch and cache metadata. When you add something to your tracker, this service resolves it to a canonical record.

**Responsibilities:**
- External API integration and caching (TMDB, OpenLibrary, IGDB)
- Canonical media records (avoid hitting external APIs on every request)
- Search: e.g. "find me a book called Project Hail Mary"

### 2. Tracking Service
Owns your personal relationship with each piece of media - status, progress, and timestamps.

**Responsibilities:**
- CRUD for tracking entries
- Progress updates (page 140 of 400, episode 3 of 8, 12 hours into a game)
- Emits events when status changes: `media.started`, `media.finished`, `media.dropped`

### 3. Review Service
Ratings and written reviews, decoupled from tracking. Allows re-rating something years later without it being a tracking action.

**Responsibilities:**
- Store ratings and optional text reviews
- Listens for `media.finished` events to prompt a review
- Rating history over time

### 4. Recommendation Service
Looks at finished and rated media and suggests what to consume next.

**Responsibilities:**
- Pulls rating history from Review Service
- Queries external APIs for similar titles
- Returns ranked suggestions per media type

### 5. Insights Service
Stats and trends about consumption habits.

**Responsibilities:**
- Listens to events from Tracking and Review services
- Aggregates stats over time (books read per year, avg rating by genre, hours played)
- Powers a personal "year in review" style summary

---

## Communication Architecture

```
                    ┌─────────────────┐
                    │   API Gateway   │  ← single entry point
                    └────────┬────────┘
                             │
      ┌──────────┬───────────┼───────────┬──────────┐
      │          │           │           │          │
 ┌────▼───┐ ┌────▼───┐ ┌────▼───┐ ┌────▼───┐ ┌────▼────┐
 │ Media  │ │Tracking│ │ Review │ │Recomm. │ │Insights │
 │Service │ │Service │ │Service │ │Service │ │ Service │
 └────┬───┘ └────┬───┘ └────┬───┘ └────┬───┘ └────┬────┘
      │          │           │           │          │
      └──────────┴───────────┼───────────┴──────────┘
                             │
                    ┌────────▼────────┐
                    │   Message Bus   │
                    │    (Kafka)      │
                    └─────────────────┘
```

**Synchronous (gRPC):**
- API Gateway → any service for user-initiated actions (REST externally, translates to gRPC internally)
- Recommendation Service → Media Service to fetch similar titles
- Tracking Service → Media Service to resolve a media item on add

Service contracts are defined as `.proto` files, with Go code generated via `protoc`. Each service exposes a gRPC server; callers use the generated client stubs.

**Asynchronous (Kafka):**
- Tracking Service emits `media.started`, `media.finished`, `media.dropped`
- Review Service listens to `media.finished` to know a review is now possible
- Insights Service listens to all events to update stats

> **Note on the API Gateway:** The gateway is the only service that speaks REST to the outside world (so you can use a browser or curl). Internally it translates incoming HTTP requests into gRPC calls to the appropriate service. This is the standard pattern - sometimes called a gRPC gateway or BFF (Backend for Frontend).

---

## Data Model (Simplified)

Each service owns its own Postgres database — no shared schemas.

```
Media Service
└── media_items: id, external_id, type (BOOK/MOVIE/GAME), title,
                 genres[], cover_url, metadata_json, cached_at

Tracking Service
└── tracking_entries: id, user_id, media_id, status, progress,
                      started_at, finished_at, updated_at

Review Service
└── reviews: id, user_id, media_id, rating, body, created_at

Insights Service
└── consumption_stats: user_id, year, month, media_type, count,
                       avg_rating, total_hours_games
```

---

## External APIs

| API | Covers | Notes |
|---|---|---|
| [TMDB](https://www.themoviedb.org/documentation/api) | Movies & TV | Free, excellent docs, no friction |
| [OpenLibrary](https://openlibrary.org/developers/api) | Books | Free, no API key needed |
| [IGDB](https://api-docs.igdb.com/) | Video Games | Free, requires Twitch developer account |

---

## Tech Stack

| Concern | Choice | Why |
|---|---|---|
| Language | Go | Resource efficient (~25MB/service), platform-native, familiar |
| HTTP routing (gateway only) | Chi | Lightweight, standard library compatible |
| Inter-service communication | gRPC + protobuf | Strongly typed contracts, fast, widely used in platform tooling |
| Database access | sqlc + pgx | Idiomatic Go, type-safe SQL, no ORM magic |
| Messaging | franz-go (Kafka) | Best Kafka client in Go, pure Go, no C dependencies |
| Config | Viper | Standard choice in Go ecosystem |
| Metrics | prometheus/client_golang | Official Prometheus client |
| Tracing | OpenTelemetry Go SDK | First-class gRPC interceptor support |
| Logging | slog (stdlib) | Idiomatic since Go 1.21, structured JSON out of the box |
| IaC | Pulumi Go SDK | Write infra in the same language as your services |

### Resource Estimate

| Component | Approx RAM |
|---|---|
| k3s control plane | ~512MB |
| Kafka (KRaft mode, no Zookeeper) | ~512MB |
| Postgres (shared instance while learning) | ~256MB |
| kube-prometheus-stack | ~1.5GB |
| Loki + Tempo | ~512MB |
| 5 Go services | ~125MB total |
| **Total** | **~3.4GB** |

### Project Structure

```
home-media-tracker/
├── proto/                  # Shared protobuf definitions for all services
│   ├── media/v1/
│   ├── tracking/v1/
│   ├── review/v1/
│   ├── recommendation/v1/
│   └── insights/v1/
├── services/
│   ├── gateway/            # REST → gRPC translation, public entry point
│   ├── media/              # External API integration + catalog
│   ├── tracking/           # User tracking entries + Kafka producer
│   ├── review/             # Ratings, reviews + Kafka consumer
│   ├── recommendation/     # Suggestion logic
│   └── insights/           # Stats aggregation + Kafka consumer
└── infra/                  # Pulumi Go SDK
    ├── cluster/            # k3s / Hetzner VM provisioning
    ├── platform/           # Kafka, Postgres, observability stack, ingress, ArgoCD
    └── apps/               # Per-service Kubernetes manifests
```

---



1. **Media Service** — get TMDB integration working, cache in Postgres, expose a search endpoint. Immediately satisfying because you can search for real content.
2. **Tracking Service** — add items to your list, update status. This is the core loop.
3. **Wire them together** — Tracking calls Media to resolve items; you have a working end-to-end flow.
4. **Review Service** — introduce Kafka here. Tracking emits events, Review listens.
5. **Insights Service** — subscribe to events, start accumulating stats.
6. **Recommendation Service** — last, because it depends on having real data in the other services.

---

## Skills Covered by This Project

| Thing you build | Skill you learn |
|---|---|
| Multi-service Go apps | Service decomposition, Go project structure |
| protobuf + gRPC contracts | Strongly typed inter-service communication |
| gRPC gateway (REST → gRPC) | API gateway pattern, protocol translation |
| TMDB / IGDB / OpenLibrary integration | External API patterns, caching strategies |
| Kafka between services | Event-driven architecture, consumer groups |
| Postgres per service (sqlc + pgx) | Stateful workloads on Kubernetes, idiomatic Go DB access |
| Recommendation logic | Cross-service data flow |
| Insights aggregation | Event sourcing lite, time-series thinking |
| Pulumi Go SDK | IaC in Go, layered stacks: cluster / platform / apps |
| kube-prometheus-stack | Metrics with Prometheus + Grafana |
| OpenTelemetry Go SDK + Tempo | Distributed tracing across services |
| Loki + slog | Structured log aggregation |
| ArgoCD + GitHub Actions | GitOps deployment pipeline |
| Kubernetes RBAC + network policies | Cluster hardening |

---

## Kubernetes Phases

### Phase 1 — Get it running
- Deployments, Services, ConfigMaps, Secrets per service
- Ingress controller (nginx or Traefik)
- Postgres via StatefulSet

### Phase 2 — Make it production-like
- Resource limits and requests on every pod
- Horizontal pod autoscaling
- Liveness and readiness probes (expose a `/health` endpoint in each Go service)
- Namespaces: separate infra, apps, monitoring

### Phase 3 — Harden it
- RBAC: lock down service account permissions
- Network policies: services only talk to what they need
- Pod disruption budgets

---

## Observability Stack

All free, all Kubernetes-native, all used by real platform teams.

- **Metrics** — kube-prometheus-stack (Prometheus + Grafana + Alertmanager). Instrument Go services with `prometheus/client_golang`, expose a `/metrics` endpoint. Build RED dashboards (Request rate, Error rate, Duration) per service.
- **Tracing** — OpenTelemetry Go SDK (`go.opentelemetry.io`), ship to Grafana Tempo. Trace requests across all five services including across gRPC boundaries — OTel has first-class gRPC interceptor support.
- **Logging** — Structured JSON logs via `slog` (Go standard library since 1.21), aggregated with Grafana Loki. Correlate logs with traces via injected trace IDs.

---

## Pulumi Structure

```
infra/
├── cluster/        # Kubernetes cluster (e.g. k3s locally, GKE/Hetzner in cloud)
├── platform/       # Monitoring stack, Kafka, ingress, ArgoCD
└── apps/           # Per-service Kubernetes manifests / Helm values
```

Each layer is a separate Pulumi stack. Apps stack depends on platform stack; platform depends on cluster. Mirrors how real platform teams structure IaC.

---

## Next Steps (To Be Fleshed Out)

- [ ] Media Service: proto definition, gRPC server, TMDB integration, Kafka event schema, Pulumi infra, Kubernetes manifests
- [ ] Tracking Service: proto definition, gRPC server, Kafka producer setup
- [ ] Review Service: proto definition, gRPC server, Kafka consumer setup
- [ ] Insights Service: event aggregation design, Kafka consumer setup
- [ ] Recommendation Service: cross-service gRPC client design
- [ ] API Gateway: REST → gRPC translation, routing
- [ ] Observability: dashboard designs, alert rules, OTel gRPC interceptors
- [ ] CI/CD: GitHub Actions pipeline (proto generation + build + push), ArgoCD setup
