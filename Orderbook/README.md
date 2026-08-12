# AB Crypto (Orderbook) — Event-Driven Trading Platform

> Portfolio notes for the `Orderbook` project. Source of truth is `Documents/Orderbook`
> (backend microservices, frontend, Terraform, Docker/Jenkins CI-CD). Note: the real repo's
> remote is `github.com/kpashindla25/order-book-youssefamr74` — a training-org account, not a
> personal GitHub — see the earlier portfolio-audit summary for that flag.

## What it does

A crypto trading platform that accepts buy/sell orders, matches them against a live order
book, and records the resulting trades — the same basic shape as a real exchange's core
engine, built as a set of independently deployable Spring Boot microservices communicating
over Kafka rather than direct HTTP calls to each other.

From the actual code:

- **Gateway** (`backend/gateway`) — Spring Cloud Gateway, the single entry point the frontend
  talks to.
- **Auth service** (`backend/auth-service`) — registration, login, JWT issuing.
- **Order service** (`backend/order-service`) — owns the lifecycle of buy/sell orders,
  publishes them to Kafka.
- **Matching engine** (`backend/matching-engine-service`) — the core algorithm. Consumes
  order events from Kafka (`kafka/OrderConsumer.java`), maintains an in-memory order book per
  instrument (`model/OrderBook.java`), matches incoming orders against the opposite side, and
  publishes resulting trades (`kafka/TradeProducer.java`). Also pushes live book updates to
  the frontend over WebSocket/STOMP (`config/WebSocketConfig.java`).
- **Trade service** (`backend/trade-service`) — persists the history of executed trades.
- **Transaction service** (`backend/transaction-service`) — a Gradle-based service tracking
  account balances/transactions (separate build tooling from the Maven services — visibly a
  later addition).
- **Frontend** (`frontend/`) — Vite + React trading UI (`TradingChart.jsx` and friends),
  served via Nginx.

Each service has its own isolated Postgres database (`user_db`, `order_db`, `trade_db`) — no
service reaches into another's schema directly; everything crosses service boundaries via
Kafka events or the gateway.

## Tech stack

- **Backend:** Java 21, Spring Boot, Spring Cloud Gateway, Maven (+ one Gradle service),
  Lombok
- **Messaging:** Apache Kafka 4.1.1 (KRaft mode, no Zookeeper), Kafdrop for topic inspection
- **Database:** PostgreSQL 15, one isolated database per service
- **Frontend:** Vite, React, Nginx
- **Infra/CI-CD:** Docker + multi-stage Dockerfiles, Terraform (AWS EC2/ECR/IAM/security
  group), Jenkins pipeline (build → push to ECR → SSH-deploy to EC2 via
  `docker-compose.prod.yml`)

## Architecture

```
                         ┌─────────────────────┐
                         │      Frontend         │
                         │   (Vite + React,      │
                         │   served via Nginx)    │
                         └──────────┬────────────┘
                                    │ HTTP
                                    ▼
                         ┌─────────────────────┐
                         │       Gateway          │
                         │ (Spring Cloud Gateway) │
                         └──┬───────┬───────┬────┘
                            │       │       │
                 ┌──────────┘       │       └──────────┐
                 ▼                  ▼                  ▼
         ┌───────────────┐ ┌────────────────┐ ┌─────────────────┐
         │  Auth Service   │ │  Order Service  │ │  Trade Service    │
         │  (user_db)      │ │  (order_db)     │ │  (trade_db)       │
         └───────────────┘ └────────┬────────┘ └────────▲──────────┘
                                    │ orders-created/            │ trades-executed
                                    │ orders-cancelled            │
                                    ▼                            │
                         ┌────────────────────────┐              │
                         │   Apache Kafka (KRaft)   │──────────────┘
                         └───────────┬─────────────┘
                                     │ orders-created/cancelled
                                     ▼
                         ┌────────────────────────┐
                         │   Matching Engine         │
                         │  in-memory OrderBook per   │
                         │  instrument (price/time    │
                         │  priority) → trades-executed│
                         │  → also pushes live book    │
                         │  updates via WebSocket/STOMP│
                         └────────────────────────┘
```

## Interesting technical decisions worth calling out

- **Price/time priority order book with lock-free-ish concurrent structures.**
  `OrderBook.java` uses a `ConcurrentSkipListMap<BigDecimal, ConcurrentLinkedQueue<...>>` for
  each side — bids sorted in reverse (highest price first), asks sorted ascending (lowest
  price first), with a FIFO queue at each price level for time priority. That's the standard
  data-structure shape a real matching engine uses, not a naive list-scan.
- **Per-book locking, not a single global lock.** `MatchingService.process()` synchronizes on
  the specific `OrderBook` instance for the instrument being traded (`synchronized (book)`),
  so trading BTC/USD doesn't block matching on ETH/USD.
- **Kafka as the service boundary, not REST-to-REST calls.** Order service never calls the
  matching engine directly — it publishes an event, and the engine (and anything else that
  cares) subscribes. That decouples the high-throughput matching hot path from the
  request/response latency of whatever else is happening in order-service.
- **Isolated database per service.** Each service owns its schema; nothing shares a database
  connection across service boundaries, which is what actually makes the "microservices" label
  honest here rather than just a namespacing convention.

## Running it locally

```bash
git clone <this repo>
cd Orderbook
docker-compose -f docker-compose.dev.yml up -d
```

- Frontend: http://localhost
- Gateway: http://localhost:8080
- Kafdrop (Kafka topic inspector): http://localhost:9000
- Postgres: localhost:5432 (dev credentials only — see `frontend.env.example` /
  `deploy.env.example` in this folder, do not reuse dev defaults in production)

To seed sample Kafka messages: `docker-compose -f docker-compose.dev.yml --profile seed up`.

## Deployment (for context — not required to run locally)

Production deploys via a Jenkins pipeline: build multi-stage Docker images → push to AWS ECR
→ SSH into an EC2 instance → `docker-compose -f docker-compose.prod.yml up -d --build`. The
EC2 instance, ECR repositories, IAM role, and security group are provisioned by Terraform
(`terraform/main.tf`).

## Configuration

See `frontend.env.example`, `deploy.env.example`, and `terraform.tfstate.example` in this
folder. The real project's `.env.production` (frontend), `orderbook-keypair.pem` (AWS SSH key
pair), and `terraform.tfstate`/`terraform.tfstate.backup` (Terraform state) were **not**
copied here:

- The frontend env only holds a relative API path (`VITE_API_URL=/api`), which is why
  `frontend.env.example` looks nearly identical to the real thing — there was nothing
  sensitive in it to begin with.
- There is no meaningful "example" version of a private SSH key — that's just documented
  below as a setup step instead of templated.
- `terraform.tfstate.example` is a **hand-built, fictitious** state file showing the shape of
  what Terraform tracks once the infrastructure is applied (EC2 instance, ECR repos, IAM
  role, security group) — all IDs/IPs/ARNs in it are made up. The real `terraform.tfstate`
  currently has an empty `resources` list (infrastructure isn't provisioned right now), so
  there was no real deployed-state data to redact in the first place.

To provision your own infrastructure: create an AWS EC2 key pair named `orderbook-key` (or
change `key_name` in `terraform/main.tf`), configure AWS credentials locally (`aws configure`
or environment variables — never commit them), then `terraform init && terraform apply` from
the `terraform/` directory.
