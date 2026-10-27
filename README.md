# System Design

Personal system design notes, built from first principles for interview prep and as a reference I can point back to while actually building things. Each topic goes deep enough to explain *why* something exists, not just define it, and ties the concept back to systems I have built or am building.

**Progress: all 9 core topics for this repo are done.** Caching and Message Queues are covered in my [Advance-Dev](https://github.com/TridibeshSam31/Advance-Dev) repo (see [Covered in Advance-Dev](#covered-in-advance-dev)), so they are not duplicated here. Next up: the case studies.

## Format

Every topic follows a similar shape:

- **Terminology first**: basic terms are defined up front so the rest of the doc never assumes prior knowledge
- **From-first-principles walkthrough**: starts from "what problem breaks without this" before naming the concept
- **ASCII diagrams** for request flow and architecture, with no external image dependencies
- **Mapping to my own projects**: how the concept applies to things I have built (chat app, Exchange, Movie Search API and others), not a generic textbook example
- **Interview question bank**: Q&A with deeper answers at the end for quick pre-interview review
- **One-line summary / mental model** to close each topic

> **Language note:** the Load Balancer doc is written in English. The rest of the notes are in Hinglish (Hindi written in Roman script, with technical terms in English), since that is how I think through these concepts.

## Topics

| # | Topic | Status | What it covers |
|---|---|---|---|
| 1 | [Load Balancers, Proxies & Nginx](./LoadBalancer/Readme.md) | ✅ Done | Forward/reverse proxy, LB algorithms, health checks, sticky sessions, L4 vs L7, Nginx, SSL/TLS, PM2, AWS ALB/NLB/GWLB, production architecture, metrics |
| 2 | [Stateful vs Stateless, Singleton & Pub/Sub](./stateless/notes.md) | ✅ Done | Stateless vs stateful backends, stickiness, managing in-process state in Node.js, Singleton pattern, Pub/Sub with a PubSubManager |
| 3 | [Sharding](./Sharding/notes.md) | ✅ Done | Scaling options, shard keys, range vs hash vs directory-based sharding, rebalancing, hot shards, cross-shard queries and transactions |
| 4 | [Consistent Hashing](./ConsistentHashing/notes.md) | ✅ Done | Why modulo hashing breaks, the hash ring, adding/removing nodes, virtual nodes, consistent hashing vs replication vs load balancing |
| 5 | [Replication](./Replication/notes.md) | ✅ Done | Primary/replica, sync vs async replication, replication lag, failover, read-after-write problem, replication vs sharding |
| 6 | [CAP Theorem](./CAP/notes.md) | ✅ Done | Partitions, consistency vs availability, CP and AP examples, CAP consistency vs ACID consistency, applying CAP to real systems |
| 7 | [CDN](./CDN/notes.md) | ✅ Done | What a CDN caches, cache hit/miss, CDN vs Redis, dynamic API requests, cache invalidation |
| 8 | [Idempotency](./Idempotency/notes.md) | ✅ Done | Idempotency keys, retries, Redis-based design, race conditions, DB unique constraints, key lifecycle, idempotency vs transactions |
| 9 | [Rate Limiting](./RateLimitter/notes.md) | ✅ Done | Leaky bucket, token bucket, fixed and sliding window, algorithm comparison, distributed design with Redis, atomicity, limiting keys |
| 10 | Caching | 🔗 In Advance-Dev | [Redis & caching](https://github.com/TridibeshSam31/Advance-Dev/blob/main/Backend/phase7.txt) |
| 11 | Message Queues | 🔗 In Advance-Dev | [Background jobs & queues](https://github.com/TridibeshSam31/Advance-Dev/blob/main/Backend/phase8.txt) |

## Suggested Reading Order

Some topics build directly on earlier ones:

1. **Load Balancer** → **Stateless vs Stateful**: why stateless servers are easy to put behind a load balancer
2. **Sharding** → **Consistent Hashing**: how to decide which shard owns a key without a full reshuffle on every change
3. **Replication** → **CAP Theorem**: what consistency you give up when data lives on multiple nodes
4. **CDN** → **Caching** (Advance-Dev): edge caching first, then Redis caching in the application layer
5. **Idempotency** → **Rate Limiting**: both are about keeping APIs safe under retries and heavy traffic

## Covered in Advance-Dev

These topics are part of the system design roadmap but live in the [Advance-Dev](https://github.com/TridibeshSam31/Advance-Dev) repo:

| Topic | Where | What it covers |
|---|---|---|
| Caching | [`Backend/phase7.txt`](https://github.com/TridibeshSam31/Advance-Dev/blob/main/Backend/phase7.txt), [`Backend/caching.txt`](https://github.com/TridibeshSam31/Advance-Dev/blob/main/Backend/caching.txt) | Redis as an in-memory store, why it does not replace a relational DB, cache hit/miss flow, key naming, TTLs, use cases (sessions, OTPs, rate limiting, leaderboards) |
| Message Queues | [`Backend/phase8.txt`](https://github.com/TridibeshSam31/Advance-Dev/blob/main/Backend/phase8.txt) | Producers/consumers, BullMQ and RabbitMQ as backends, retries, dead letter queues, background-job patterns |
| Redis queues & Pub/Sub | [`Backend/phase7.txt`](https://github.com/TridibeshSam31/Advance-Dev/blob/main/Backend/phase7.txt) | Redis lists as a queue, blocking pop, publish/subscribe |
| SQS & SNS | [`AWS/15-SQS-SNS.md`](https://github.com/TridibeshSam31/Advance-Dev/blob/main/AWS/15-SQS-SNS.md) | Managed queues and fan-out on AWS, visibility timeout, DLQ, at-least-once delivery |

## Case Studies (Planned)

- URL Shortener
- WhatsApp
- Instagram Feed
- Notification System
- File Upload Service
- Additional case studies TBD

## Out of Scope Here

Horizontal vs vertical scaling, indexing, normalization, database/schema design, ER diagrams, caching and message queues are already covered in the [Advance-Dev](https://github.com/TridibeshSam31/Advance-Dev) repo and are not duplicated here.

## Structure

```
System-Design/
├── LoadBalancer/
│   └── Readme.md
├── stateless/
│   └── notes.md
├── Sharding/
│   └── notes.md
├── ConsistentHashing/
│   └── notes.md
├── Replication/
│   └── notes.md
├── CAP/
│   └── notes.md
├── CDN/
│   └── notes.md
├── Idempotency/
│   └── notes.md
├── RateLimitter/
│   ├── notes.md
│   └── code.txt
├── topics.txt        ← running list of planned topics
└── README.md
```

Each new topic gets its own folder with a `notes.md`, following the same pattern.
