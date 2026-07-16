# Load Balancers, Proxies & Nginx — Complete Reference

> A from-first-principles guide to load balancing, reverse proxies, Nginx internals, SSL/TLS, PM2, and cloud load balancers — built for interview prep and system design reference.

## Table of Contents

1. [Why Load Balancers Are Needed](#1-why-load-balancers-are-needed)
2. [Request Flow (End to End)](#2-request-flow-end-to-end)
3. [What Is a Proxy? Forward vs Reverse Proxy](#3-what-is-a-proxy-forward-proxy-vs-reverse-proxy)
4. [Reverse Proxy vs Load Balancer](#4-reverse-proxy-vs-load-balancer)
5. [Load Balancing Algorithms](#5-load-balancing-algorithms)
6. [Health Checks](#6-health-checks)
7. [Sticky Sessions](#7-sticky-sessions)
8. [Layer 4 vs Layer 7](#8-layer-4-vs-layer-7)
9. [Nginx Deep Dive](#9-nginx-deep-dive)
10. [SSL / TLS Explained Properly](#10-ssl--tls-explained-properly)
11. [PM2 Explained](#11-pm2-explained)
12. [Cloud Load Balancers Deep Dive](#12-cloud-load-balancers-deep-dive)
13. [Production Reference Architecture](#13-production-reference-architecture)
14. [Advanced Concepts](#14-advanced-concepts)
15. [Mapping to Your Own Projects](#15-mapping-to-your-own-projects)
16. [Interview Question Bank](#16-interview-question-bank)
17. [Metrics to Monitor in Production](#17-metrics-to-monitor-in-production)

---

## 1. Why Load Balancers Are Needed

**Single server, low traffic — fine:**

```
                 Users
                   │
                   ▼
           ┌───────────────┐
           │   Server A    │
           └───────────────┘
```

**Single server, high traffic (~10,000 users) — falls over:**

```
   10,000 users ──▶ ┌───────────────┐
                     │   Server A    │   CPU: 100%
                     │               │   RAM:  Full
                     └───────────────┘   Response: 8s → CRASH
```

**First instinct — Vertical Scaling** (make the one server bigger):

```
   2 CPU  ──▶  8 CPU
  16 GB   ──▶ 64 GB RAM
```
Has a hard ceiling, costs rise fast, and it's still a single point of failure (SPOF). Eventually it crashes again.

**Real fix — Horizontal Scaling** (add more servers):

```
   ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐
   │ Server A  │  │ Server B  │  │ Server C  │  │ Server D  │
   └───────────┘  └───────────┘  └───────────┘  └───────────┘
```

New problem: **which server should a given request go to?** If everyone defaults to Server A, you've recreated the original bottleneck. This routing problem is exactly what a **Load Balancer** solves.

### Definition

A load balancer is a traffic manager that intelligently distributes incoming requests across multiple backend servers. **It does not process business logic itself — it only forwards/routes traffic.**

```
                      Users
                        │
                        ▼
              ┌───────────────────┐
              │   Load Balancer   │
              └─────────┬─────────┘
              ┌──────────┼──────────┐
              ▼          ▼          ▼
        ┌─────────┐┌─────────┐┌─────────┐
        │Server A ││Server B ││Server C │
        └─────────┘└─────────┘└─────────┘
```

### Real-World Analogy

A restaurant receptionist:
- Customers (requests) don't walk straight into the kitchen (server).
- Receptionist (load balancer) assigns: *"Table 4," "Table 8," "Table 2."*
- Tables = servers.

### Without vs With Load Balancer

```
WITHOUT:   100 users ──▶ Server A ──▶ CRASH

WITH:      100 users ──▶ LB ──┬──▶ 25 → Server A
                               ├──▶ 25 → Server B
                               ├──▶ 25 → Server C
                               └──▶ 25 → Server D
```
Each server now only handles a fraction of the load.

---

## 2. Request Flow (End to End)

```
   Browser
      │
      ▼
     DNS
      │
      ▼
  Public IP
      │
      ▼
┌───────────────┐
│ Load Balancer │
└───────┬───────┘
        │
        ▼
 Backend Server (one of many)
        │
        ▼
    Database
        │
        ▼
     Response
```

**Key point:** the client never knows the IP of the actual backend server. It only ever talks to the Load Balancer's IP/domain — important for security (backend IPs stay hidden) and flexibility (servers can be added/removed without the client noticing).

---

## 3. What Is a Proxy? Forward Proxy vs Reverse Proxy

Before Nginx or load balancers make any sense, you need one core idea: a **proxy** is simply *"a middleman that sits between two parties and forwards traffic on their behalf."* Everything else (reverse proxy, load balancer, API gateway) is a specialized version of this.

There are exactly two flavors — almost all confusion comes from mixing them up.

### Forward Proxy — hides the CLIENT

```
  You (client) ──▶ ┌───────────────┐ ──▶ Internet / Server
                    │ Forward Proxy │
                    └───────────────┘
```

Example: office/college wifi. The website you visit sees the **proxy's** IP, not your laptop's. Used for:
- Blocking certain websites (firewall policy)
- Anonymizing the client (a VPN is essentially a forward proxy)
- Caching frequently-visited sites for the whole network

**Key idea:** a forward proxy **protects/represents the client**.

### Reverse Proxy — hides the SERVER

```
  Client ──▶ ┌───────────────┐ ──▶ Actual Backend Server
             │ Reverse Proxy │
             └───────────────┘
```

The client believes it's talking directly to "the server." In reality it only ever talks to the reverse proxy, which decides which real backend handles the request.

**Key idea:** a reverse proxy **protects/represents the server**.

> This single question — *"which side is being hidden, client or server?"* — is the entire distinction, and the #1 thing interviewers test with a quick "forward vs reverse proxy" question.

### Why Would You Even Want a Reverse Proxy?

**Without one:**

```
  Client ──▶ Node.js server (port 3000) directly
```

Problems:
- Server's IP/port directly exposed — bypasses future protections, easy to fingerprint your stack.
- No SSL/TLS handling built in — Node *can* do HTTPS, but that's not what it's optimized for; crypto work steals CPU from your actual app logic.
- No easy way to run multiple backend copies behind one address.
- No caching layer — every request re-executes app code, even for a static image requested 1000 times.

**With a reverse proxy (Nginx):**

```
  Client ──▶ ┌────────────┐ ──▶ ┌──────────────────────┐
             │ Nginx      │     │ Node.js app           │
             │ :80 / :443 │     │ :3000 (internal only) │
             └────────────┘     └──────────────────────┘
```

Now:
- The client only ever talks to Nginx. Node can bind to `localhost` only — literally unreachable from outside the machine.
- Nginx handles HTTPS (**SSL/TLS termination** — see [Section 10](#10-ssl--tls-explained-properly)).
- Nginx serves static files (images/CSS/JS) straight from disk, without touching Node.
- Nginx compresses responses (gzip) before sending to the client.

> This exact setup is already running in production for **CodeArena** (`Nginx + PM2` on a DigitalOcean VPS) — none of this is abstract theory.

---

## 4. Reverse Proxy vs Load Balancer

Another classic point of confusion, now easy given Section 3:

| | Reverse Proxy | Load Balancer |
|---|---|---|
| Core purpose | Hide/front a backend (SSL termination, caching, compression, auth) | Distribute traffic across **multiple** servers |
| Works with 1 server? | Yes — useful even fronting a single server | No — inherently needs multiple backends |
| Diagram | `Client → Nginx → Backend` | `Client → LB → Server A / B / C` |

**Can one tool do both?** Yes — Nginx very commonly acts as a reverse proxy *and* a load balancer simultaneously.

> Standard interview line: **"Nginx acts as a reverse proxy and a load balancer."**

---

## 5. Load Balancing Algorithms

### A) Round Robin
```
Req1 → A    Req2 → B    Req3 → C
Req4 → A    Req5 → B    Req6 → C
```
Simple, fast, most common interview answer.
**Problem:** treats all requests as equal cost. If A's requests take 2s but B's take 20s, Round Robin still sends equal *counts* — B silently overloads.

### B) Least Connections
```
Server A ── 100 active connections
Server B ──  20 active connections
Server C ──   5 active connections      ◀── next request goes here
```
Dynamic, adapts to real load. Widely used in production.

### C) Least Response Time
```
Server A = 20 ms
Server B = 300 ms
Server C = 15 ms      ◀── next request goes here
```

### D) Weighted Round Robin
Not all servers are equally powerful:
```
Server A (16 CPU) → weight 5
Server B ( 4 CPU) → weight 2
Server C ( 2 CPU) → weight 1

Distribution:  A  A  A  A  A  B  B  C
```
*(Weighted Least Connections also exists — same idea applied to the least-connections algorithm.)*

### E) IP Hash
```
Client IP 192.x.x.x ──▶ hash() ──▶ always Server B
```
Same user always lands on the same server — useful for in-memory carts/sessions.
**Caveat:** naive hashing causes a big reshuffle if a server goes down — real systems use **consistent hashing** (CDNs, Redis Cluster) so only a small fraction of clients remap.

### F) Random / Power of Two Choices
Pick 2 random servers, route to the less loaded one — cheaper than tracking global state, nearly as effective at scale.

---

## 6. Health Checks

```
                    ┌───────────┐
   LB ── GET /health ▶│ Server A │──▶ 200 OK  → stays in rotation
                    └───────────┘

                    ┌───────────┐
   LB ── GET /health ▶│ Server B │──▶ Timeout  → removed from rotation
                    └───────────┘
```

**Without health checks:** LB keeps sending traffic to a dead server → users get 500 errors.

Two flavors:
- **Passive** — health inferred from real traffic (repeated failures/timeouts).
- **Active** — LB proactively pings a dedicated endpoint on a fixed interval.

Mandatory in any real production setup.

---

## 7. Sticky Sessions

```
  Login request ──▶ Server A  (session saved on A)
  Next request  ──▶ Server B  (B has no idea who this user is) ──▶ effectively logged out
```

**Fix — Sticky Sessions** (usually cookie-based):
```
  Client ──▶ always routed to ──▶ Server A
```
**Downside:** if A crashes, the session is gone; also works against even load distribution.

**Modern fix — centralized session store:**
```
  Server A ──▶ write session ──▶ ┌───────┐
                                  │ Redis │
  Server B ◀── read session ──── └───────┘
```
Session state lives outside any single server, so it no longer matters which backend handles a request. Still needed for stateful long-lived connections like WebSockets.

---

## 8. Layer 4 vs Layer 7

### Layer 4 (Transport — TCP/UDP)
```
  Client ──▶ [ IP + Port only ] ──▶ Server
```
No content inspection. Very fast, low latency. *Example: AWS NLB.*

### Layer 7 (Application — HTTP/HTTPS)
```
  GET /api      ──▶ Backend A
  GET /images   ──▶ Image Server
  GET /admin    ──▶ Admin Service
```
Can inspect headers, cookies, JWTs, path, host, method — enabling smart, content-based routing. *Example: AWS ALB, Nginx.*

L4 can **never** make path-based decisions like this — it doesn't look inside the packet payload.

---

## 9. Nginx Deep Dive

### What Nginx Actually Is
A standalone program, originally a **web server**, that evolved into a reverse proxy, load balancer, and caching layer — a Swiss-army-knife traffic handler sitting at the edge of your infrastructure.

### How It Works Internally

Traditional servers (classic Apache) spawn a new OS thread per connection — 10,000 connections = 10,000 threads, each with memory/scheduling overhead before any real work happens.

Nginx uses an **event-driven, asynchronous architecture** instead:

```
                  Nginx Master Process
              (reads config, supervises workers)
                          │
        ┌─────────────────┼─────────────────┐
        ▼                 ▼                 ▼
   ┌──────────┐      ┌──────────┐      ┌──────────┐
   │ Worker 1 │      │ Worker 2 │      │ Worker 3 │
   │event loop│      │event loop│      │event loop│
   │  1000s of│      │  1000s of│      │  1000s of│
   │connections     │connections     │connections
   └──────────┘      └──────────┘      └──────────┘
```

A small, **fixed** number of workers (usually one per CPU core) each handle thousands of connections via an event loop — the OS notifies Nginx the instant a connection has data ready, and idle connections cost nothing while waiting. This is why one Nginx instance handles tens of thousands of concurrent connections on modest hardware.

### Config, Explained Line by Line

```nginx
upstream backend {
    server 192.168.1.1;
    server 192.168.1.2;
    server 192.168.1.3;
}

server {
    listen 80;
    server_name codearena.example.com;

    location / {
        proxy_pass http://backend;
    }
}
```

| Line | Meaning |
|---|---|
| `upstream backend { ... }` | Defines a **named group** of backend servers — a pool of interchangeable app copies. |
| `server 192.168.1.1;` (×3) | Each is one real backend address — the list Nginx load-balances across. Default: Round Robin. Add `least_conn;` / `ip_hash;` here to switch algorithms, or `weight=5` for weighted round robin. |
| `server { listen 80; ... }` | A *separate* block describing how **Nginx itself** is reached — "listen on port 80" (standard HTTP). HTTPS would add `listen 443 ssl;` + certificate paths. |
| `server_name codearena.example.com;` | Which domain this config block applies to — one Nginx install can host many unrelated domains. |
| `location / { proxy_pass http://backend; }` | The routing rule: `/` matches every path, `proxy_pass` forwards to the `backend` group above. |

In this one file, Nginx is simultaneously a **reverse proxy** (real IPs hidden), a **load balancer** (traffic split across 3 IPs), and the **public entry point** for the domain.

### Path-Based (Layer 7) Routing

```nginx
location /api/ {
    proxy_pass http://api_backend;
}

location /images/ {
    proxy_pass http://image_backend;
}
```
Nginx reads *inside* the HTTP request (the path) to decide where traffic goes — only possible because it understands HTTP (Layer 7), not just raw TCP.

---

## 10. SSL / TLS Explained Properly

**SSL** (old name) / **TLS** (modern, actually-used version) is what turns "HTTP" into "HTTPS" — it encrypts client↔server traffic so anyone intercepting it (e.g. on public wifi) sees only scrambled bytes.

TLS does two jobs:
1. **Encryption** — scrambles data in transit.
2. **Authentication** — proves the server's identity via a certificate issued by a trusted Certificate Authority (Let's Encrypt, DigiCert, etc.) — the browser padlock icon.

**Simplified TLS handshake:**
```
Client  ──"hello, here's what I support"──▶  Server
Client  ◀── certificate + agreed method ───  Server
     both sides derive a shared secret key (never sent in plain text)
Client  ◀══ all further traffic encrypted ══▶  Server
```
This handshake adds latency/CPU cost — which is exactly why **where** you terminate TLS matters.

### Termination Strategies

**Option 1 — Terminate at the LB/Reverse Proxy (most common):**
```
Client ──(HTTPS, encrypted)──▶ Nginx/LB ──(HTTP, plain)──▶ Backend
```
Nginx holds the cert, does all crypto work. Internal traffic is plain HTTP — safe within a trusted network/same machine (e.g. CodeArena's Nginx + Node on one VPS). Backend servers never pay the handshake cost.

**Option 2 — SSL Passthrough:**
```
Client ──(HTTPS, untouched)──▶ LB ──(still encrypted)──▶ Backend
```
LB forwards raw encrypted bytes; backend holds the cert and decrypts. True end-to-end encryption, but LB **loses Layer-7 routing ability** (can't inspect encrypted content).

**Option 3 — Re-encryption (SSL Bridging):**
```
Client ──(HTTPS #1)──▶ LB ──(HTTPS #2, re-encrypted)──▶ Backend
```
LB decrypts (can do Layer-7 routing), then re-encrypts before forwarding. Best of both, at the cost of two full encrypt/decrypt cycles — common in banking/healthcare.

> For nearly everything you build as a student/early developer: **Option 1** is the standard, sufficient approach.

---

## 11. PM2 Explained

### What PM2 Actually Is
A **process manager for Node.js** — installed via `npm install -g pm2` — whose job is to keep your Node app running reliably and to run multiple copies of it on one machine.

### Why You Need It

Running `node app.js` directly:
- Crashes → **stays dead**. No auto-recovery.
- Server reboots → app does **not** restart automatically.
- Node is single-threaded for your JS code → 1 process uses 1 CPU core, even on an 8-core machine (7 cores sit idle).

### What PM2 Fixes

1. **Auto-restart on crash** — instant restart the moment the process dies.
2. **Startup on boot** — `pm2 startup` + `pm2 save` re-registers your app after a full server reboot.
3. **Cluster mode** — starts multiple instances, one per CPU core, with a built-in lightweight load balancer:

```bash
pm2 start app.js -i max
```

```
              PM2 Cluster Manager
             (built-in load balancer)
                       │
        ┌─────────┬────┴────┬─────────┐
        ▼         ▼         ▼         ▼
   App copy 1 App copy 2 App copy 3 App copy 4
    (core 1)   (core 2)   (core 3)   (core 4)
```

### PM2 vs Nginx — Not Competitors

| | PM2 | Nginx |
|---|---|---|
| Balances across | Multiple **CPU cores**, same machine | Multiple **machines/servers** |
| Scope | Local to one host | Public-facing entry point |

**Realistic full picture:**

```
                       Client
                         │
                         ▼
              ┌────────────────────┐
              │  Nginx (reverse    │
              │  proxy + LB across │
              │  multiple VPS)     │
              └──────────┬─────────┘
              ┌───────────┴───────────┐
              ▼                       ▼
        ┌───────────┐           ┌───────────┐
        │   VPS 1    │           │   VPS 2    │
        │ PM2 cluster│           │ PM2 cluster│
        │ (4 cores)  │           │ (4 cores)  │
        └───────────┘           └───────────┘
```

> This is exactly what CodeArena's **"PM2 + Nginx"** deployment means: Nginx is the outward-facing reverse proxy/LB; PM2 keeps the Node processes alive and uses the VPS's CPU cores fully underneath it.

---

## 12. Cloud Load Balancers Deep Dive

### Why They Exist
Running Nginx yourself means *you* patch it, keep it from being a SPOF, and scale it manually. A **cloud load balancer** (AWS/GCP/Azure) is a fully managed service — the provider runs, patches, and scales the LB across multiple data centers; you just configure rules.

### Core AWS Building Blocks

```
                     Internet
                         │
                         ▼
              ┌─────────────────────┐
              │  ALB Listener :443  │
              └──────────┬──────────┘
        ┌─────────────────┼─────────────────┐
        ▼                 ▼                 ▼
  Rule: /api/*       Rule: /images/*     Rule: default
        │                 │                 │
        ▼                 ▼                 ▼
 Target Group:      Target Group:      Target Group:
  api-servers        image-servers       web-servers
```

- **Listener** — watches a port/protocol ("listen on 443 for HTTPS").
- **Target Group** — the pool of backends receiving traffic (same idea as Nginx's `upstream` block).
- **Rules** — conditions deciding which target group gets which traffic.

### Application Load Balancer (ALB)
- Layer 7 — same path-based routing power as Nginx, fully managed.
- Host-based routing, managed sticky-session cookies, per-target-group health checks (auto-managed version of [Section 6](#6-health-checks)).
- Best for: web apps, REST APIs, microservices behind one domain.

### Network Load Balancer (NLB)
- Layer 4 — raw TCP/UDP forwarding, no content inspection.
- Extremely low latency, very high connection volume.
- Best for: gaming, WebSocket-heavy apps, raw-speed-critical services.

### Gateway Load Balancer (GWLB)
Transparently inserts third-party security appliances (firewalls, IDS/IPS) into the traffic path without reconfiguring every service individually.

### Global Server Load Balancing (GSLB) / DNS-Based Routing
```
  User in India ──▶ DNS lookup ──▶ Mumbai/Singapore data center
  User in US    ──▶ DNS lookup ──▶ US data center
```
Routes users to the nearest region *before* any HTTP request is even sent (e.g. AWS Route 53 latency/geo routing) — how global-scale systems balance across continents, not just servers in one data center.

---

## 13. Production Reference Architecture

```
                        Users
                          │
                          ▼
              ┌────────────────────────┐
              │  Cloudflare (CDN,      │
              │  DDoS protection,      │
              │  edge caching)         │
              └───────────┬────────────┘
                          ▼
              ┌────────────────────────┐
              │     Load Balancer      │
              └───────────┬────────────┘
     ┌───────────┬─────────┴─────────┬───────────┐
     ▼           ▼                   ▼           ▼
 ┌───────┐  ┌───────┐           ┌───────┐   ┌───────┐
 │ Node1 │  │ Node2 │           │ Node3 │   │ Node4 │
 └───────┘  └───────┘           └───────┘   └───────┘
     └───────────┴─────────┬─────────┴───────────┘
                            ▼
                     ┌─────────────┐
                     │    Redis    │  (cache / sessions / pub-sub)
                     └──────┬──────┘
                            ▼
                ┌────────────────────┐
                │ PostgreSQL Primary │
                └──────────┬─────────┘
                            ▼
                   ┌────────────────┐
                   │    Replica     │  (read scaling)
                   └────────────────┘
```

Standard shape used by most startups and mid-scale SaaS products. The CDN sits *in front of* the load balancer — absorbing static assets and traffic/attacks before they reach your LB and origin servers.

---

## 14. Advanced Concepts

- **Connection Draining / Graceful Shutdown** — when a server leaves rotation, the LB stops sending *new* requests but lets in-flight ones finish, avoiding dropped requests during deploys.
- **Blue-Green / Canary Deployments** — LB shifts traffic gradually (5% → 50% → 100%) to a new server group; bad deploys roll back before affecting everyone.
- **Autoscaling + LB** — new servers spin up under rising load and self-register with the LB (and deregister on scale-down).
- **Load Balancing WebSockets** — long-lived, stateful connections can't be round-robined per-message; needs sticky sessions + a shared pub/sub layer (e.g. Redis Pub/Sub) for cross-server message delivery.
- **The LB Itself Can Be a SPOF** — real systems run LBs in active-passive/active-active HA pairs with a floating/virtual IP (VRRP/keepalived) so a backup takes over instantly.

---

## 15. Mapping to Your Own Projects

### CodeArena *(Next.js 15, BullMQ, Redis, Prisma/Postgres)*
```
   Users ──▶ Load Balancer ──┬──▶ Judge Worker 1
                              ├──▶ Judge Worker 2
                              └──▶ Judge Worker 3
```
Code execution/judging requests distribute across sandboxed workers. Current deployment (Nginx + PM2 on one DigitalOcean VPS) already load-balances locally via PM2 cluster mode; scaling to multiple VPS nodes means Nginx (or a cloud LB) in front of several instances, with BullMQ/Redis as the shared job queue so any worker can pick up any submission.

### Chat App
```
   Users ──▶ Load Balancer ──┬──▶ Socket Server 1
                              ├──▶ Socket Server 2
                              └──▶ Socket Server 3
```
Needs **sticky sessions** (a socket must stay on one server) + **Redis Pub/Sub** so a message from a user on Server 1 reaches a user on Server 3.

### Postman Clone
```
   Users ──▶ Load Balancer ──┬──▶ API Runner 1
                              ├──▶ API Runner 2
                              └──▶ API Runner 3
```
Stateless worker pool — plain Round Robin or Least Connections works fine, no stickiness needed.

---

## 16. Interview Question Bank

**Q: Why do we need a load balancer?**
A: To distribute traffic across multiple servers so no single one becomes a bottleneck/SPOF, enabling horizontal scaling and higher availability.

**Q: Difference between vertical and horizontal scaling?**
A: Vertical = making one server bigger — hard ceiling, stays a SPOF. Horizontal = adding more servers — scales further, needs a load balancer to coordinate.

**Q: Forward proxy vs reverse proxy?**
A: Forward proxy hides the *client* from the server (office network, VPN). Reverse proxy hides the *server* from the client — the client only ever talks to the proxy.

**Q: Round Robin vs Least Connections?**
A: Round Robin rotates regardless of load; Least Connections routes to whichever server has the fewest active connections — better for uneven request costs.

**Q: L4 vs L7 Load Balancer?**
A: L4 = TCP/UDP, IP+port only, no content inspection, very fast. L7 = HTTP-aware, can route on headers/cookies/path, more overhead.

**Q: What happens if a backend server dies?**
A: Health checks detect it (timeout/non-200), LB marks it unhealthy, removes it from rotation, redistributes its share to healthy servers.

**Q: What are health checks?**
A: Periodic probes (e.g. `GET /health`) verifying a backend is alive — active (LB-initiated pings) or passive (inferred from live traffic failures).

**Q: What are sticky sessions, and why are they less preferred today?**
A: Force a client to always hit the same server (cookie-based) — needed for server-local session state. Less preferred now since Redis-backed shared sessions remove the need in most cases; still relevant for WebSockets.

**Q: Can Nginx act as a load balancer?**
A: Yes — via `upstream` + `proxy_pass`, supporting round robin (default), weighted round robin, `least_conn`, and `ip_hash`.

**Q: Reverse Proxy vs Load Balancer?**
A: Reverse proxy fronts/hides a backend (can be just one); load balancer distributes across multiple backends. Nginx commonly does both.

**Q: What is SSL/TLS termination, and where should it happen?**
A: The point where HTTPS is decrypted back to plain data. Usually at the LB/reverse proxy (saves backend CPU, centralizes certs); passthrough or re-encryption used when true end-to-end encryption is required.

**Q: What does PM2 do, and how does it differ from a load balancer?**
A: A Node.js process manager — auto-restarts crashed apps, starts on boot, and cluster-mode load-balances across CPU cores on *one* machine. A load balancer distributes across *multiple* machines. They complement each other.

**Q: Why put a CDN before a Load Balancer?**
A: Serves static assets from the edge, absorbs traffic spikes/DDoS before they hit your infrastructure, reduces load on the LB and origin servers.

**Q: How would you load balance WebSocket connections?**
A: Sticky sessions (or L4/L7 connection affinity) to keep a connection on one server, plus a shared pub/sub layer (Redis Pub/Sub) for cross-server message delivery.

**Q: How do load balancers affect session management/authentication?**
A: Since requests can land on different servers, session/auth state can't live in one server's local memory — needs a shared store (Redis/DB) or a stateless token (JWT) any server can validate.

**Q: Core building blocks of an AWS ALB?**
A: Listener (watches port/protocol), Target Groups (backend pools), Rules (conditions deciding which target group handles which traffic).

**Q: What metrics would you monitor on a production load balancer?**
A: See [Section 17](#17-metrics-to-monitor-in-production).

---

## 17. Metrics to Monitor in Production

- **Latency** (p50 / p95 / p99) — averages hide tail-latency problems; p99 matters most for UX.
- **Error rate** (4xx/5xx %) — spikes indicate backend issues.
- **Active connections per backend** — spots uneven load distribution.
- **Backend health status** — how many servers are healthy vs removed from rotation.
- **Throughput** (requests/sec) — overall and per-backend.
- **Connection/queue depth** — early warning sign of saturation before latency/errors spike.
- **SSL handshake time** — if TLS termination happens at the LB.