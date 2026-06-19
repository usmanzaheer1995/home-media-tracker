# Media Service — Hexagonal Architecture (First Slice)

> Design doc to implement against. Build it yourself; pull in help per step as needed.

## Context

The media service is the first service to build (per CLAUDE.md build order): external API
integration (TMDB to start), a Postgres-backed cache of canonical media records, and a search
endpoint exposed over gRPC. It's structured with **hexagonal architecture (ports & adapters)**
so external concerns (TMDB, persistence, gRPC) sit at the edges and the domain core has zero
knowledge of them — making providers and storage swappable.

Decisions locked for this first slice:
- **Provider:** TMDB only, behind a `MediaProvider` port. OpenLibrary/IGDB become new adapters later.
- **Persistence:** in-memory `Repository` adapter first; swap to sqlc+pgx later with no core changes.
- **Inbound:** gRPC only (gateway owns REST). Verify with `grpcurl`.

Deferred to follow-up passes (explicitly *not* in this slice): Postgres/sqlc adapter,
Prometheus metrics, OpenTelemetry tracing, Kafka. The hexagon is designed so these slot in
without touching the core.

## The hexagon

```
            inbound (driving)                          outbound (driven)
                 │                                            │
   grpc.Server ──┤                          ┌── tmdb.Client (MediaProvider)
   (proto<->dom) │   ┌──────────────────┐   │   (TMDB HTTP + DTO->domain)
                 └──>│  core.MediaSvc   │<──┤
   MediaService     │  (domain logic)  │   └── memory.Repository (Repository)
   port  ───────────┤                  │       (later: postgres adapter)
                     └──────────────────┘
```

- **Inbound port** `MediaService` — what the core offers: `Search`, `GetByID`.
- **Outbound ports** the core needs: `MediaProvider` (live external search), `Repository` (cache).
- Dependencies point **inward**: adapters import the core; the core imports nothing from adapters.

### Behavior
- `Search(ctx, query, type)` → call `MediaProvider` (TMDB) live → `Repository.Upsert` the
  results (write-through cache) → return them. This is the path that hits the external API.
- `GetByID(ctx, id)` → `Repository` only (the cheap cached canonical-record path; never hits TMDB).
- Dedup/identity: cache unique key is `(type, external_id)`; internal canonical `id` is a UUID
  minted on first upsert. `metadata_json` stores raw provider payload as `json.RawMessage`.

## Layout

Module: `github.com/usmanzaheer1995/home-media-tracker/services/media`

```
proto/                                   # shared, its own Go module
└── media/v1/media.proto
   gen/media/v1/*.pb.go                  # generated (buf)
buf.yaml, buf.gen.yaml                   # codegen config at repo root
go.work                                  # ties services/media + proto for local dev

services/media/
├── cmd/media/main.go                    # composition root (wires everything)
└── internal/
    ├── core/
    │   ├── domain/media.go              # MediaItem, MediaType, sentinel errors
    │   ├── port/inbound.go              # MediaService interface
    │   ├── port/outbound.go             # MediaProvider, Repository interfaces
    │   └── service/media_service.go     # implements MediaService; depends only on ports
    ├── adapter/
    │   ├── inbound/grpc/server.go       # implements generated MediaServiceServer
    │   ├── inbound/grpc/mapper.go       # proto <-> domain
    │   └── outbound/
    │       ├── tmdb/client.go           # implements MediaProvider (net/http)
    │       ├── tmdb/mapper.go           # TMDB DTO -> domain
    │       └── memory/repository.go     # implements Repository (map + mutex)
    └── config/config.go                 # Viper: TMDB key, grpc port, log level
```

## Proto contract (`proto/media/v1/media.proto`)

```proto
syntax = "proto3";
package media.v1;
option go_package = ".../proto/gen/media/v1;mediav1";

service MediaService {
  rpc Search(SearchRequest)   returns (SearchResponse);
  rpc GetMedia(GetMediaRequest) returns (GetMediaResponse);
}
enum MediaType { MEDIA_TYPE_UNSPECIFIED=0; MEDIA_TYPE_BOOK=1; MEDIA_TYPE_MOVIE=2; MEDIA_TYPE_GAME=3; }
message MediaItem {
  string id=1; string external_id=2; MediaType type=3; string title=4;
  repeated string genres=5; string cover_url=6; string metadata_json=7;
  google.protobuf.Timestamp cached_at=8;
}
message SearchRequest  { string query=1; MediaType type=2; }
message SearchResponse { repeated MediaItem items=1; }
message GetMediaRequest  { string id=1; }
message GetMediaResponse { MediaItem item=1; }
```

Codegen via **buf** (`buf generate`) — simpler than raw `protoc`. (If you keep this, update the
"generated to Go via protoc" line in CLAUDE.md accordingly.)

## Implementation steps

1. **proto module + codegen.** Write `media.proto`; add `buf.yaml`/`buf.gen.yaml`; init the
   `proto/` Go module; `buf generate`; create root `go.work` with `services/media` + `proto`. ✅
2. **Core domain & ports.** `domain/media.go` (MediaItem, MediaType, `ErrNotFound`);
   `port/inbound.go`, `port/outbound.go`.
3. **Core service + unit tests.** `service/media_service.go`; table tests using a fake
   `MediaProvider` and fake `Repository` — this is the payoff of the hexagon (core tested with
   no TMDB, no DB).
4. **Driven adapters.** `memory.Repository` (map keyed by `(type,external_id)` + UUID id);
   `tmdb.Client` (hand-rolled `net/http`, API key from config) + DTO→domain mapper.
5. **Inbound gRPC adapter.** `grpc.Server` implementing the generated server, proto↔domain
   mapping, delegating to the inbound port.
6. **Composition root** `cmd/media/main.go`: load config (Viper) → `slog` JSON logger → build
   tmdb client + memory repo → core service → gRPC server; register service + grpc **health**
   server + **reflection**; listen; graceful shutdown on SIGINT/SIGTERM.

### Dependencies (first slice, kept lean)
`google.golang.org/grpc`, `google.golang.org/protobuf`, `github.com/google/uuid`,
`github.com/spf13/viper`, gRPC health + reflection. (No Postgres/Prometheus/OTel yet.)

### Config (env via Viper)
`MEDIA_TMDB_API_KEY`, `MEDIA_GRPC_PORT` (default 50051), `MEDIA_LOG_LEVEL`.

## Verification

- **Unit:** `go test ./...` in `services/media` — core service tests pass against fakes (no
  network/DB).
- **End-to-end (manual):** export `MEDIA_TMDB_API_KEY`, `go run ./cmd/media`, then:
  - `grpcurl -plaintext localhost:50051 list` (reflection shows `media.v1.MediaService`)
  - `grpcurl -plaintext -d '{"query":"Project Hail Mary","type":"MEDIA_TYPE_MOVIE"}' localhost:50051 media.v1.MediaService/Search`
    → returns TMDB results; a follow-up `GetMedia` with a returned `id` resolves from cache.
  - `grpcurl -plaintext -d '{"service":""}' localhost:50051 grpc.health.v1.Health/Check` → `SERVING`.

## Follow-ups (next slices, out of scope here)
Swap `memory.Repository` → sqlc+pgx Postgres adapter (+ migrations); add Prometheus `/metrics`
and OTel gRPC interceptors; then move on to the tracking service per build order.
