# System Design

Personal system design notes — built from first principles for interview prep and as a reference I can point back to while actually building things. Each topic goes deep enough to explain *why* something exists, not just define it, and ends by mapping the concept onto my own projects instead of a generic textbook example.

## Format

Every topic doc follows the same shape:
- **From-first-principles walkthrough** — starts from "why would this even be needed" before naming the concept
- **ASCII diagrams** for request flow / architecture, no external image dependencies
- **Mapping to My Own Projects** — how the concept actually applies to things I've built (CodeArena, the chat app, Postman clone, etc.), not a hypothetical
- **Interview Question Bank** — Q&A pairs at the end for quick pre-interview review
- **Production metrics** to monitor, where relevant

## Topics

| Topic | Status | Notes |
|---|---|---|
| [Load Balancers, Proxies & Nginx](./LoadBalancer/Readme.md) | ✅ Done | Forward/reverse proxy, LB algorithms, L4 vs L7, Nginx, SSL/TLS, PM2, AWS ALB/NLB/GWLB, production architecture |
| Caching | ⬜ Planned | |
| CDN | ⬜ Planned | |
| Sharding | ⬜ Planned | |
| Consistent Hashing | ⬜ Planned | |
| Replication | ⬜ Planned | High-level |
| Message Queue (RabbitMQ basics) | ⬜ Planned | |
| CAP Theorem | ⬜ Planned | High-level |
| Stateless vs Stateful | ⬜ Planned | |
| Idempotency | ⬜ Planned | |
| Rate Limiting | ⬜ Planned | |

## Case Studies (Planned)

- URL Shortener
- WhatsApp
- Instagram Feed
- Notification System
- File Upload Service
- Additional case studies TBD

## Out of Scope Here

Horizontal vs vertical scaling, indexing, normalization, database/schema design, and ER diagrams are already covered in the Advance Dev repo — not duplicated here.

## Structure

```
System-Design/
├── LoadBalancer/
│   └── Readme.md
├── topics.txt        ← running list this README is generated from
└── README.md
```

Each future topic gets its own folder with a `Readme.md`, same pattern as `LoadBalancer/`.
