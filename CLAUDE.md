# Upswing: project brief

A live "trending front page" backend, inspired by Reddit and Hacker News. Users post links and vote, and every click is tracked. A ranking worker turns the vote and click stream into a front page that updates in real time.

## Why I'm building this
- I'm a backend engineer with 5+ years of Java/Spring Boot, REST and GraphQL experience, preparing for senior backend interviews.
- Goal: build **real depth** in Kafka, Redis and Postgres, and come out with system-design stories and **measured numbers** I can defend in interviews.
- This is a learning project. Prefer depth over breadth. Don't add tools or services unless I ask.

## Constraints
- **Solo developer.** Keep it to one main service plus one worker.
- **Open source and free only.** Everything runs locally with Docker Compose. GCP is used only for demos (one VM, stopped when not in use).
- **Java 21 + Spring Boot 3.**

## Stack
| Piece | Choice | Role |
|---|---|---|
| API service | Spring Boot 3, Java 21 | REST API, writes to Postgres, outbox relay, publishes click events |
| Ranking worker | Spring Boot 3 + Spring Kafka | Consumes events, updates Redis rankings and counters |
| Event streaming | Apache Kafka (KRaft mode, `apache/kafka` image) | Events between services + click stream |
| Database | PostgreSQL 17 + Flyway | Source of truth: users, posts, votes, outbox |
| Cache / rankings | Valkey 8 (Redis-compatible), Lettuce client | Sorted-set rankings, counters, dedupe, rate limits, cache |
| Observability | Micrometer → Prometheus → Grafana | Latency, throughput, consumer lag, correctness metrics |
| Testing | JUnit 5, Testcontainers, k6 | Integration tests against real infrastructure; load tests |
| Build | Maven multi-module (`events`, `api`, `worker`) + Maven Wrapper | One parent POM; shared event classes in `events` |
| Local dev / CI | Docker Compose, GitHub Actions | One-command startup; CI runs the tests |

Out of scope unless I ask: gRPC, Kubernetes, Terraform (phase 6 only), Elasticsearch, Flink, Debezium, auth providers (a stub user header is fine), frontend beyond a minimal page.

## Architecture
```
Client ──► API service ──► Postgres (posts, votes, outbox: same transaction)
   │            │                         │
   │            │                   outbox relay (polling, FOR UPDATE SKIP LOCKED)
   │            │                         ▼
   │            └── click events ──► Kafka: post.events / vote.events / click.events
   │                                        │
   │                                  Ranking worker (consumer group: ranking)
   │                                        │
   └──── GET /frontpage ◄── Redis (ZSETs, counters, cache) ◄──┘
```
- Domain events (`post.events`, `vote.events`) go through the **transactional outbox**. Never write to Postgres and Kafka separately (the dual-write problem).
- Click and impression events are high-volume and lower-stakes. The API publishes them **directly** to Kafka, at-least-once, and the worker dedupes them.

## Data model (Postgres)
- `users(id bigserial pk, handle text unique, created_at timestamptz)`
- `posts(id bigserial pk, hn_id bigint unique null, author_id fk, title text, url text, created_at timestamptz)`
- `votes(user_id fk, post_id fk, value smallint check (value in (-1, 1)), created_at, updated_at, primary key (user_id, post_id))`. The primary key enforces **one vote per user per post**. Changing a vote is an upsert that emits the delta.
- `outbox(id bigserial pk, aggregate_type text, aggregate_id bigint, event_type text, event_id uuid unique, payload jsonb, created_at, published_at null)`. Add a partial index on unpublished rows.
- `processed_events(consumer text, event_id uuid, processed_at, primary key (consumer, event_id))`. Use only where idempotency must be exact; Redis `SET NX` dedupe covers the rest.

## Kafka topics
| Topic | Key | Producer | Events | Notes |
|---|---|---|---|---|
| `post.events` | postId | outbox relay | PostCreated | |
| `vote.events` | postId | outbox relay | VoteCast (with delta: +1, -1, +2, -2) | Keying by post keeps per-post order |
| `click.events` | postId | API (direct) | Impression, Click, Dwell | Each event has a UUID `eventId` for dedupe |
| `*.dlt` | same | worker error handler | failed records | Retry with backoff, then send to the dead-letter topic |

- Start with 12 partitions per topic. Use idempotent producers with `acks=all` for domain events.
- Consumers commit offsets only after the Redis/Postgres side effects succeed (at-least-once delivery + idempotent handling).

## Redis keys
- `rank:hot`: ZSET postId → hot score
- `rank:clicks:min:{epochMinute}`: per-minute ZSET buckets; `ZUNIONSTORE` combines the last 60 into `rank:trending:1h`
- `post:{id}:counts`: HASH with `up`, `down`, `clicks`, `impressions`
- `dedupe:{eventId}`: `SET NX EX 86400`
- `ratelimit:vote:{userId}`: sliding-window rate limit implemented as a Lua script
- `cache:frontpage:{type}:{page}`: short TTL (around 5 s). Add an in-process Caffeine cache and request coalescing so a viral post doesn't overload Redis with one hot key.

## Ranking
- **Hot score:** Reddit-style, so scores never need recomputing as posts age:
  `score = sign(net) * log10(max(|net|, 1)) + (createdEpochSeconds - EPOCH) / 45000`
- **Trending (last hour):** windowed click counts from the per-minute buckets.
- **Stretch goal:** compare exact counts with a Count-Min Sketch + heap, and measure the error vs memory used.

## Correctness rules (must always hold)
1. A user has at most one vote per post, enforced by the database, not the application.
2. Every vote change produces exactly one outbox event in the same transaction.
3. Consumers are idempotent: replaying any topic from offset 0 produces the same Redis state.
4. A nightly reconciliation job compares Redis `post:{id}:counts` against Postgres aggregates, records the drift as a metric, and repairs it.
5. Redis can be flushed and rebuilt from Postgres or a Kafka replay.

## Hard problems to build deliberately (these are the interview stories)
- **Hot key:** a single viral post gets 50% of the traffic.
- **Duplicate events:** retries and producer re-sends; dedupe them and show the counts stay exact.
- **Vote manipulation:** detect bursts from new accounts within a sliding window (stretch goal).
- **Failure injection:** kill the worker mid-stream, stop a Kafka broker, flush Redis. Show that recovery works and nothing drifts.
- **Backpressure:** watch consumer lag grow under load and recover once load drops.

## Observability (Grafana dashboard)
- API p50/p95/p99 latency per endpoint; requests/sec
- Votes/sec and clicks/sec produced vs consumed
- Consumer lag per partition; age of the oldest unpublished outbox row
- Dedupe hit rate; dead-letter topic count
- Reconciliation drift (target: 0)

## Data and load
- **Seeder:** import real posts from the Hacker News API (`https://hacker-news.firebaseio.com/v0/topstories.json`, then `/v0/item/{id}.json`).
- **k6 scenarios:**
  1. steady browsing
  2. a viral post spike
  3. a vote storm from 10K users
  4. duplicate and retried events

## Build phases (time box: about 5–6 weeks)
1. **Foundation:** Docker Compose (Postgres, Kafka, Valkey, Prometheus, Grafana); Flyway schema; posts and votes API with the unique vote rule; HN seeder; Testcontainers tests
2. **Events:** outbox table + relay; topics; ranking worker with idempotent consumers; `rank:hot` and `GET /frontpage`
3. **Click tracking:** click/impression pipeline; dedupe; per-minute buckets → trending over the last hour; vote rate limiting
4. **Performance:** metrics and dashboard; k6 scenarios; hot-key handling (Caffeine + request coalescing); record baseline numbers
5. **Resilience:** retries + dead-letter topics; failure-injection runs; reconciliation job; rebuild Redis from a replay; README with architecture, tradeoffs and measured results
6. **Optional:** deploy to k3s on one GCP VM provisioned with Terraform; Count-Min Sketch comparison; bot-burst detection

## How Claude Code should work with me
- Explain the tradeoff behind each design decision in a sentence or two. I want to be able to defend every choice in an interview.
- Prefer plain, readable code over clever abstractions. Use JdbcClient or explicit SQL where it makes the database behavior clearer.
- Write or update tests with every feature (Testcontainers for anything that touches Postgres, Kafka or Redis).
- Ask before adding a new dependency, tool or service.
- Keep the README's "Tradeoffs" and "Results" sections up to date as we go.

## Resume line (fill in measured numbers at the end)
> Built Upswing, an event-driven trending engine (Kafka, Redis, Postgres, transactional outbox) sustaining **X** events/s with **0** count drift under failure injection; p99 front-page latency **Y** ms during a viral-post spike.
