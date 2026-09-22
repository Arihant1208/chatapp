# Building a ChatGPT-scale Chat System — Implementation Plans

This folder contains **one self-contained implementation plan per architecture level** (L0 → L7).
Each file can be picked up and built independently: it lists its prerequisites, the exact
components to build, the config, and the acceptance + load tests that prove the level is done
before you move up.

> Companion reference: the [system-design field guide](../chat-system-design.html) explains *why*
> each level exists. These files are the *how*.

---

## How to use these files

1. **Build in order.** Each level assumes the previous one exists. Don't skip — every level exists
   to solve a problem the previous one literally cannot survive.
2. **Each level has an exit gate.** You do not advance until the level's *Acceptance criteria* pass
   and the *Exit trigger* condition is actually true for you. Premature jumps are the #1 way these
   projects die of complexity.
3. **Measurement is part of "done."** Every plan ends with a load/measurement step tied to the
   framework in the field guide (Chapter 08). If you can't measure it, it isn't done.
4. **Cross-cutting concerns are woven in.** Auth, observability, deployment, security, rate limiting
   and cost appear inside the level where they first become necessary, not bolted on at the end.

---

## The levels at a glance

| Level | File | Goal | Unlocks | Exit trigger |
|-------|------|------|---------|--------------|
| L0 | [L0-monolith.md](L0-monolith.md) | One working chat app on one box | A shippable product | Can't deploy w/o downtime, or box saturates |
| L1 | [L1-stateless-app.md](L1-stateless-app.md) | Stateless app + real data tier | HA + zero-downtime deploys | Server must *push*; write ceiling |
| L2 | [L2-realtime-transport.md](L2-realtime-transport.md) | SSE token streaming | Real-time answers | One box can't hold the connections |
| L3 | [L3-horizontal-scale.md](L3-horizontal-scale.md) | Fleet behind a load balancer | Horizontal scale | Cross-server delivery needed |
| L4 | [L4-fanout-pubsub.md](L4-fanout-pubsub.md) | Fan-out via pub/sub backbone | Real multi-user chat | Single DB can't hold writes/storage |
| L5 | [L5-data-at-scale.md](L5-data-at-scale.md) | Sharded, polyglot data tier | Petabyte-scale history | You add AI generation |
| L6 | [L6-inference-layer.md](L6-inference-layer.md) | GPU inference + streaming | It becomes ChatGPT | Users are global |
| L7 | [L7-global.md](L7-global.md) | Multi-region + edge | Local-feel worldwide | Endless refinement |

---

## Reference tech stack (all swappable)

These plans pick a concrete, mainstream stack so tasks and config are copy-pasteable. Swap any box
for its equivalent; the *architecture* is what matters, not the vendor.

| Concern | Choice | Common alternatives |
|---|---|---|
| Frontend | React + TypeScript (Next.js) | SvelteKit, Vue/Nuxt |
| App / API tier | Node.js + TypeScript (Fastify) | Go, Python/FastAPI, Java/Spring |
| Connection tier (sockets) | Go | Elixir/Phoenix, Node.js |
| Relational DB | PostgreSQL | MySQL, CockroachDB |
| Message store (at scale) | Cassandra / ScyllaDB | DynamoDB, Bigtable |
| Cache / presence / pub-sub | Redis | Valkey, KeyDB |
| Durable event log | Kafka | Redpanda, Pulsar, NATS JetStream |
| Object storage | S3 (MinIO for local) | GCS, R2 |
| Search / vector | OpenSearch + pgvector → dedicated vector DB | Elasticsearch, Pinecone, Milvus |
| Inference server | Python + vLLM on NVIDIA GPUs | TGI, TensorRT-LLM, SGLang |
| Containers / orchestration | Docker + Kubernetes | ECS, Nomad |
| IaC | Terraform | Pulumi, CDK |
| CI/CD | GitHub Actions + Argo CD | GitLab CI, Jenkins |
| Metrics / dashboards | Prometheus + Grafana | VictoriaMetrics, Datadog |
| Tracing | OpenTelemetry + Tempo/Jaeger | Honeycomb |
| Logs | Loki | OpenSearch/ELK |
| GPU telemetry | NVIDIA DCGM exporter | — |
| Load testing | k6 (+ xk6-sse) | Gatling, Locust, GenAI-Perf |
| Chaos | Chaos Mesh | Litmus, Gremlin |

---

## Conventions used in every file

- **Task lists are ordered and checkable** — treat them as a build backlog.
- **Acceptance criteria are measurable** — a number and a percentile, never "fast".
- **`▶ Exit trigger`** marks the single condition that means "now build the next level."
- Code/config blocks are minimal starters, not production-complete — adapt names and secrets.

## Suggested repo layout (target end state)

```
/apps
  /web              # Next.js frontend
  /api              # Fastify app/API tier (stateless)
  /gateway          # Go connection tier (WebSocket/SSE)
  /inference        # Python vLLM workers + orchestrator
/services
  /worker           # async consumers (fan-out, indexing, notifications)
/packages
  /shared           # shared types, auth, telemetry helpers
/infra
  /terraform        # cloud resources
  /k8s              # manifests / helm charts
  /observability    # prometheus rules, grafana dashboards, otel config
/loadtests          # k6 scripts, one per level
```
