# Migration Plan: Node.js Backend → .NET

## What the Node.js backend does

### Stack & Framework
- **NestJS** (Node.js framework) running on port `3001`
- **MongoDB** via Mongoose (ODM)
- **RabbitMQ** as a message queue consumer (via `@nestjs/microservices`)
- **Swagger/OpenAPI** docs exposed at `/api`

---

### Architecture

The app runs **two modes simultaneously** in one process:
1. **HTTP server** — serves REST API requests
2. **RabbitMQ consumer** — consumes messages from a queue (wired via `app.connectMicroservice()`)

```
┌─────────────────────────────────────┐
│            NestJS App               │
│                                     │
│  HTTP  ──►  AppController           │
│                  GET /          → "Hello World"
│                  GET /jobs      → list all jobs from MongoDB
│                                     │
│  RabbitMQ ──► JobsMessageHandler    │
│               queue: "jobs"         │
│               → parse message       │
│               → save to MongoDB     │
│               → ack/nack            │
└─────────────────────────────────────┘
         │                  │
         ▼                  ▼
      MongoDB          RabbitMQ
   collection: jobs   queue: jobs
```

---

### Data model — `Job`
Stored in MongoDB collection `jobs`:

| Field | Type | Required |
|-------|------|----------|
| `title` | string | yes |
| `company` | string | yes |
| `location` | string | no |

---

### Key components

| File | Role |
|------|------|
| `main.ts` | App bootstrap — HTTP + microservice startup, Swagger setup |
| `app.controller.ts` | `GET /` and `GET /jobs` HTTP endpoints |
| `app.service.ts` | Trivial "Hello World" service |
| `jobs.service.ts` | MongoDB CRUD + queue message parsing/validation |
| `job.schema.ts` | Mongoose schema for `Job` document |
| `rabbitmq.config.ts` | RabbitMQ connection config (durable queue, manual ack) |
| `jobs-message.handler.ts` | Consumes `jobs` queue — acks on success, nacks (dead-letter) on failure |
| `jobs-message.deserializer.ts` | Custom deserializer — handles raw JSON or NestJS envelope format |
| `job.dto.ts` | API response shape with Swagger annotations |
| `seed-jobs.ts` | Dev script — wipes `jobs` collection and inserts 5 sample records |

---

### Message flow (RabbitMQ → MongoDB)
1. A message arrives on queue `jobs` (published by the Python scraper)
2. The deserializer parses it — accepts raw JSON `{title, company, location}` or NestJS envelope `{pattern, data}`
3. `JobsMessageHandler` calls `JobsService.createFromQueueMessage(payload)`
4. `JobsService` validates fields (`title` and `company` required, `location` optional) and inserts into MongoDB
5. On success → `channel.ack()` (message removed from queue)
6. On failure → `channel.nack(..., false, false)` (dead-lettered, not requeued)

---

## Architecture decision: monolith-first, Worker as infra

The .NET version mirrors the NestJS approach: **one deployable, one process**, one Docker container.
The RabbitMQ consumer is not a separate service — it is infrastructure inside the backend, running as a
hosted background service alongside the HTTP API.

This is the correct starting point. Extracting the consumer to a standalone `worker-dotnet/` at the
repo root is a clean, well-defined step if independent scaling is needed later.

**Monorepo layout (unchanged):**
```
/
├── frontend/
├── backend/           ← NestJS (kept as reference)
├── backend-dotnet/    ← .NET monolith (API + RabbitMQ consumer)
└── scraper/           ← Python, publishes to RabbitMQ
```

---

## Proposed solution structure

```
backend-dotnet/
├── JobsAggregator.sln
│
└── src/
    ├── JobsAggregator.Domain/
    │   ├── Jobs/
    │   │   ├── Job.cs                        ← domain entity (replaces job.schema.ts)
    │   │   └── IJobsRepository.cs            ← repository interface
    │   └── JobsAggregator.Domain.csproj
    │
    ├── JobsAggregator.Application/
    │   ├── Jobs/
    │   │   ├── JobDto.cs                     ← response DTO (replaces job.dto.ts)
    │   │   ├── IJobsService.cs               ← application service interface
    │   │   └── JobsService.cs                ← orchestrates IJobsRepository
    │   └── JobsAggregator.Application.csproj
    │
    ├── JobsAggregator.Infrastructure/
    │   ├── MongoDB/
    │   │   └── JobsRepository.cs             ← MongoDB.Driver implementation of IJobsRepository
    │   ├── RabbitMQ/
    │   │   ├── JobsMessageConsumer.cs        ← replaces jobs-message.handler.ts
    │   │   └── JobMessage.cs                 ← inbound message contract (replaces deserializer)
    │   └── JobsAggregator.Infrastructure.csproj
    │
    └── JobsAggregator.Api/
        ├── Program.cs                        ← replaces main.ts + app.module.ts
        ├── appsettings.json                  ← replaces .env
        ├── Controllers/
        │   └── JobsController.cs             ← replaces app.controller.ts
        └── JobsAggregator.Api.csproj
```

**Notes on structure:**
- `Domain` — pure C#, no framework dependencies. Defines the entity and repository interface
- `Application` — orchestration layer. Houses application service interfaces, their implementations, and DTOs. References Domain only
- `Infrastructure` — implements `Domain` interfaces. Contains both MongoDB and RabbitMQ code, mirroring the `src/infra/` layout from NestJS
- `Api` — startup project. Hosts the HTTP server and registers MassTransit as a hosted background service (the consumer runs inside this process)
- No separate Worker project — `JobsMessageConsumer` runs inside `Api` via `IHostedService`, exactly as NestJS does via `connectMicroservice()`

---

## Equivalent .NET stack

| NestJS concept | .NET equivalent |
|---|---|
| NestJS framework | ASP.NET Core (controller-based) |
| `@nestjs/mongoose` + Mongoose | `MongoDB.Driver` |
| `@nestjs/microservices` RMQ + `connectMicroservice()` | **MassTransit** (RabbitMQ transport, hosted inside Api) |
| Swagger / `@nestjs/swagger` | `Swashbuckle.AspNetCore` |
| DI container | ASP.NET Core built-in DI |
| `.env` / `ConfigModule` | `appsettings.json` + `IConfiguration` |
| Seed script | `--seed` arg or separate console app |

---

## Step-by-step implementation plan

### Phase 1 — Solution scaffold
- Create `JobsAggregator.sln` inside `backend-dotnet/`
- Create three projects: `Domain`, `Infrastructure`, `Api`
- Add NuGet packages:
  - `MongoDB.Driver` → Infrastructure
  - `MassTransit.RabbitMQ` → Infrastructure
  - `Swashbuckle.AspNetCore` → Api
- Configure `appsettings.json` in Api:
  - `MongoDb:Uri`, `MongoDb:Database`
  - `RabbitMq:Host`, `RabbitMq:Queue`

### Phase 2 — Domain layer
- Define `Job.cs` with `Title`, `Company`, `Location?`
- Define `IJobsRepository` with `FindAll()` and `Create()`

### Phase 3 — Application layer
- Define `JobDto.cs` (response shape matching current API)
- Define `IJobsService` with `GetAllJobs()` and `CreateJob()`
- Implement `JobsService` — calls `IJobsRepository`, maps `Job` → `JobDto`
- Register `IJobsService` as scoped in a DI extension method

### Phase 4 — Infrastructure: MongoDB
- Implement `JobsRepository` using `IMongoCollection<Job>`
- Register `IJobsRepository` as scoped in a DI extension method

### Phase 5 — Infrastructure: RabbitMQ consumer
- Define `JobMessage.cs` with `Title`, `Company`, `Location?` (inbound contract from scraper)
- Implement `JobsMessageConsumer : IConsumer<JobMessage>`
  - Validate required fields (`Title`, `Company`)
  - Call `IJobsService.CreateJob()`
  - MassTransit handles ack on success; throw to nack/dead-letter on failure
- Configure MassTransit with `RawJsonDeserializer` to match Python scraper's plain JSON output

### Phase 6 — HTTP API
- `JobsController`: `GET /jobs` returns `List<JobDto>` via `IJobsService`
- `GET /` returns health string (parity with NestJS)
- Wire Swagger with same title/version/server as NestJS version
- Register MassTransit as a hosted service in `Program.cs` — consumer starts with the app

### Phase 7 — Seed script
- `--seed` flag handled in `Program.cs` before the host starts
- Drops and recreates the `jobs` collection with the same 5 sample records as `seed-jobs.ts`

### Phase 8 — Parity validation
- `GET /jobs` returns same JSON shape as NestJS version
- Publish a test message via the Python scraper (or `rabbitmqadmin`) → confirm it lands in MongoDB
- Compare Swagger output at `/api`
- Verify nack behavior: send an invalid message (missing `title`) → confirm it is dead-lettered and not requeued
