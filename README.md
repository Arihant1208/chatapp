# chatapp

System design study + implementation plans for building a real-time chat application at
ChatGPT scale (1M+ requests/second).

## Contents

- **[chat-system-design.html](chat-system-design.html)** — an interactive field guide: the L0→L7
  scaling ladder, six cross-cutting concerns (auth, rate limiting, observability, deployment, cost,
  resilience), and a hands-on measurement & benchmarking framework.
- **[implementation-plan/](implementation-plan/)** — one self-contained, buildable implementation
  plan per architecture level (L0 through L7), plus an index README with the reference tech stack.

## The scaling ladder

| Level | Focus |
|-------|-------|
| L0 | Monolith — one box, correct data model |
| L1 | Stateless app + real data tier |
| L2 | Real-time transport (SSE token streaming) |
| L3 | Horizontal scale & load balancing |
| L4 | Fan-out via pub/sub backbone |
| L5 | Data at scale (sharding + polyglot) |
| L6 | LLM inference layer (GPU fleet) |
| L7 | Going global (multi-region + edge) |

Start with [implementation-plan/README.md](implementation-plan/README.md).
