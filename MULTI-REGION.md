# Multi-region scaling plan for Sentinel API

## Current architecture (single region — us-west2)
```
client → Vercel edge (anycast, ~30 PoPs) → Railway us-west2 → Postgres us-west2
                                                            → Redis us-west2
```

Vercel handles TLS at the edge close to the user (this proxy). After TLS termination,
the proxy makes an HTTP/2 keep-alive call to Railway. Latency floor for a non-US
user dropped from `RTT(user→Railway)` to `RTT(user→nearest_edge) + 70ms backbone`.

## When you outgrow this

Triggers for multi-region:
- p95 latency from a target market exceeds your SLO (e.g., EU customers >250 ms)
- Single-region outage means full product outage
- Customer demands data residency (EU/UK GDPR, AU sovereignty)

## Topology A — Read replicas, single write region (simplest)
```
                     ┌── Railway us-west2 (primary)
                     │   Postgres primary + LISTEN/NOTIFY hub
client → Vercel edge ┼── Railway us-east2  (replica) → Postgres read replica
                     └── Railway eu-west1  (replica) → Postgres read replica
```

- Each region runs an identical FastAPI app
- All writes (`POST /v1/approvals`, `POST /decision`, etc.) routed to us-west2
- Reads (`GET /v1/approvals*`, `GET /audit-events`, `/wait`) served locally
- Vercel rewrites split by HTTP method or path

Effort: ~1 day. Postgres logical replication on Railway is Pro feature; cheaper
on a separate provider (Neon, Supabase) that natively supports read replicas.

## Topology B — Active-active with conflict-free state
```
client → Vercel edge → Railway nearest → Postgres (CockroachDB or Neon branching)
                                       → shared Redis or NATS for NOTIFY
```

- Replace Postgres with a multi-region SQL store (CockroachDB Serverless, Neon, Aurora DSQL)
- LISTEN/NOTIFY → swap to NATS or Redis Streams (works across regions)
- Every region writes locally; CRDB handles consensus

Effort: ~1 week. Cost goes up. Justified when EU/AU customers can't tolerate
the 100 ms NA-roundtrip on writes.

## Topology C — Edge-native (no origin server)
```
client → Vercel Function (Edge runtime) → Neon Postgres (HTTP driver) globally
```

- Rewrite the hot endpoints (`POST /approvals`, `GET /wait`) as Vercel Edge
  Functions using Neon's HTTP driver (1 ms cold start)
- Background dispatch via Upstash Redis / Vercel Queues
- Audit log appends via the Neon HTTP driver

Effort: ~2 weeks. Highest performance ceiling. Forces rewriting in TypeScript
or Python-on-Lambda. Tradeoff: lose some of Python's ecosystem (no asyncpg
LISTEN/NOTIFY — replace with Postgres logical decoding via NATS/Pulsar).

## Recommendation

Start with **Topology A** when you have your first paying EU customer. The
Vercel proxy you just deployed already gives 30–50% of the win for free.
Don't pre-build B or C; the API surface is small enough that rewrites are
cheap when you actually have the revenue to justify them.

## Right now: the proxy is the multi-region groundwork

The fact that this proxy exists means:
- Customers point at `api.pauseapi.app` (stable forever)
- You can swap the Railway origin to a load balancer later without breaking clients
- You can A/B route traffic via Vercel rewrites (`/v1/approvals → eu-write.railway.app`)
- TLS terminates globally already
