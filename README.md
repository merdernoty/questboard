# Questboard

A microservice-based task marketplace built to practice **fault-tolerant distributed systems**.
The platform lets users create paid tasks, browse a category catalogue, manage profiles with
tariff-based permissions, and collect per-user analytics. Every service is independently
deployable, communicates over gRPC for synchronous calls and Kafka for asynchronous events,
and ships with health probes, database migrations, and self-healing infrastructure.

## Architecture

```
                         ┌──────────────┐
        HTTP / gRPC ───► │  api-gateway │  (public entry point, port 8081/7001)
                         └──────┬───────┘
                                │ gRPC
                ┌───────────────┼────────────────┐
                ▼               ▼                 ▼
        ┌──────────────┐ ┌──────────────┐ ┌──────────────────┐
        │ task-service │ │profile-service│ │ analytic-service │
        └──────┬───────┘ └──────┬───────┘ └────────┬─────────┘
               │ gRPC           │ gRPC             ▲
               │  (CheckPermission)                │ gRPC
               │                                   │
               │  Kafka "task_created" event ──────┘
               ▼                ▼                  ▼
         task-postgres   profile-postgres   analytic-postgres
                         + Redis OSS cluster
```

### Services

| Service | Language | Sync API (gRPC) | Responsibility |
|---|---|---|---|
| **api-gateway** | Go | — | Single public entry point. Exposes HTTP + gRPC, fans requests out to the internal services, and aggregates their responses. |
| **profile-service** | Go | `GetProfile`, `GetProfileList`, `CheckPermission`, `CreateProfile` | Owns user profiles and **tariff-based permissions**. Backed by Postgres and a 6-node Redis OSS cluster (3 masters / 3 replicas) for caching. |
| **task-service** | Go | `GetCategories`, `CreateTask`, `GetTask`, `ListTasks` | Owns the task catalogue and task lifecycle. Categories (with prices) are seeded from `categories.json`. Calls profile-service to authorize task creation and publishes a `task_created` event to Kafka. |
| **analytic-service** | Go | `GetUserTaskCount` | Consumes `task_created` events from Kafka and maintains per-user task counters for reporting. |

### Communication

- **Synchronous:** gRPC between the gateway and the domain services, and service-to-service
  (e.g. task-service → profile-service `CheckPermission`).
- **Asynchronous:** Kafka. `task-service` produces a `task_created` event; `analytic-service`
  consumes it to update analytics — decoupling the write path from analytics processing.

### Fault tolerance

- **Health probes** — every service exposes a liveness/readiness probe (`deploy/scripts/probes.sh`)
  wired into Docker `healthcheck`.
- **Autoheal** — the `willfarrell/autoheal` container watches health status and automatically
  restarts unhealthy containers.
- **Redis OSS cluster** — profile caching runs on a 6-node cluster with replicas to survive node loss.
- **Ordered startup** — `depends_on` conditions and migration jobs guarantee databases, Kafka,
  and schemas are ready before application containers accept traffic.

## Tech Stack

- **Go** for all application services
- **gRPC / Protocol Buffers** for service contracts (`services/*/api/**/v1/*.proto`)
- **PostgreSQL 15** — one database per service
- **Redis 7 (OSS cluster)** — profile cache
- **Apache Kafka** (Confluent 7.6) with **AKHQ** UI for topic inspection
- **goose** for migrations, **xo** for typed DB code generation, **mockgen** + **gotestsum** for tests
- **Docker Compose** for local orchestration, **autoheal** for self-healing

## Project Layout

```
.
├── docker-compose.yaml      # full stack: services, DBs, Redis cluster, Kafka, migrations
├── Makefile                 # up-all / down-all / build-all / rebuild SERVICE=...
├── deploy/
│   ├── redis/               # per-node Redis cluster configs
│   └── scripts/probes.sh    # shared health probe
└── services/
    ├── api-gateway/
    ├── profile-service/
    ├── task-service/
    └── analytic-service/
```

Each service follows a layered (DDD-flavoured) structure:

```
services/<name>/
├── api/        # .proto contracts + generated code
├── cmd/        # main entry point
├── config/     # configuration
├── internal/
│   ├── app/            # gRPC/transport handlers
│   ├── application/    # use cases
│   ├── domain/         # entities & value objects
│   ├── infrastructure/ # DB, cache, Kafka adapters
│   └── pkg/
├── tools/migrations/   # goose SQL migrations
└── build/              # Dockerfile + Dockerfile.migrations
```

## Getting Started

### Prerequisites

- Docker & Docker Compose

### Run the whole stack

```bash
make up-all        # docker-compose up --remove-orphans
```

This builds and starts every service, applies database migrations, bootstraps the Redis
cluster, and brings up Kafka. On first run, allow time for migrations and the Redis cluster
creator to finish (their dependents wait on `service_completed_successfully`).

### Tear down

```bash
make down-all      # docker-compose down -v  (also removes volumes)
```

### Rebuild a single service

```bash
make rebuild SERVICE=task-service
```

## Ports

| Component | Host port | Notes |
|---|---|---|
| api-gateway | `8081` (HTTP), `7001` (gRPC) | public entry point |
| profile-service | `8082` (HTTP), `7002` (gRPC) | |
| task-service | `8083` (HTTP), `7003` (gRPC) | |
| analytic-service | `8084` (HTTP), `7004` (gRPC) | |
| profile-postgres | `5432` | |
| task-postgres | `5433` | |
| analytic-postgres | `5434` | |
| Redis OSS cluster | `6000–6005` | 3 masters + 3 replicas |
| Kafka | `9092`, `9093` | |
| AKHQ (Kafka UI) | `8090` | browse topics & messages |

## Typical Flow

1. A client calls **api-gateway** to create a task.
2. The gateway forwards the request to **task-service**.
3. task-service asks **profile-service** (`CheckPermission`) whether the user's tariff allows it.
4. On success the task is persisted and a `task_created` event is published to **Kafka**.
5. **analytic-service** consumes the event and increments the user's task counter, later
   queryable via `GetUserTaskCount`.
