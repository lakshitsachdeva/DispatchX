# DispatchX: real-time delivery & dispatch platform

Customers book parcel deliveries, the system matches the best nearby rider in
seconds, and everyone tracks the delivery live on a map. Dispatchers watch the
whole fleet from a control room.

**Stack:** Next.js · TypeScript · Tailwind · FastAPI · Python · MySQL · SQLAlchemy ·
Alembic · JWT · Socket.IO · Celery · Redis · SNS · SQS · Docker · EC2 · RDS · S3 ·
CloudWatch · Prometheus · Grafana

## Present the project

Start at **`/demo`** for the eight-minute presentation, **Explain / Busy fleet / Failure/retry**
presets, editable bounded settings, and copyable PowerShell/POSIX commands. The page only
generates commands; it never starts a simulator or cloud resources. Complete the three-role
delivery before running optional load.

- [Exact click/say/observe runbook](docs/DEMO-RUNBOOK.md)
- [Beginner cloud guide: how the app and AWS architecture work](docs/BEGINNER-CLOUD-GUIDE.md)
- [Implementation progress, test evidence and measured limits](docs/SCALABILITY-PROGRESS.md)
- **`/dispatch/system`**: authenticated application signals with freshness, unknown/error
  states, pause/retry, and bounded browser history. Its Celery ready-queue count is not SQS
  backlog or total work in flight.

Demo navigation remains available across desktop and mobile consoles. Infrastructure stop
actions are disabled by default; they require both an administrator and deployment opt-in.
Do not enable chaos testing in a deployment exposing the seeded public admin account.

## Explore the cloud architecture

Open **`/architecture`** in the web app for the public, backend-independent cloud explorer.
Inspect services, Docker images/containers, Kubernetes orchestration and the AWS ECS deployment;
play guided delivery, GPS, scaling and recovery flows, or copy a link to an individual step.
The diagrams explain repository configuration, **not live health**. Authenticated dispatchers
can inspect actual platform snapshots under `/dispatch/cluster` when a stack is running.

See [the explorer handoff](docs/cloud-explorer-handoff.md) for the presentation sequence,
frontend-only preview, verification commands and known limitations.

The [local delivery demo handoff](docs/local-demo-handoff.md) covers the verified three-role
browser journey, console reliability fixes, safe test setup and finite fleet simulation.

## Repository layout

```
backend/     FastAPI API + Socket.IO + Celery workers + SQS consumer (one image, several roles)
frontend/    Next.js app: customer, rider and dispatcher experiences
simulator/   Load / demo generator: N virtual riders + customers
monitoring/  Prometheus + Grafana config
infra/       AWS CloudFormation (nested stacks) + CI/CD → see infra/README.md
k8s/         Same app on Kubernetes (kind) with HPA, Ingress, StatefulSet → see k8s/README.md
```

## Run locally

```bash
docker compose up --build -d
docker compose exec api python -m app.seed --riders 100 --customers 30
```

| What | URL |
|---|---|
| Web app | http://localhost:3000 |
| API docs (Swagger) | http://localhost:8000/api/docs |
| Prometheus | http://localhost:9090 |
| Grafana | http://localhost:3001 (admin / admin) |

Demo logins (password `dispatchx123`): `customer@dispatchx.dev`, `rider@dispatchx.dev`,
`dispatch@dispatchx.dev`, `admin@dispatchx.dev`. The login page has one-click demo buttons.

### Bring the city to life

```bash
cd simulator
pip install -r requirements.txt
python simulator.py --riders 20 --customers 5 --orders-per-min 6 --duration 300
```

Open the dispatcher view to watch the fleet move, orders get matched and deliveries complete.
The run ends after five minutes; Ctrl+C stops it early. Saved orders/queued work can remain.

### End-to-end smoke test

```bash
pip install httpx "python-socketio[client]"
python backend/tests/smoke_e2e.py                          # BASE_URL=... for a deployed stack
```

### Frontend dev mode

```bash
cd frontend && cp .env.example .env.local && npm install && npm run dev
```

## How it works

### Order lifecycle
`pending → assigned → picked_up → delivered` (plus `cancelled`, and `unassigned` when no rider
can be found). Every transition goes through one state machine (`backend/app/services/orders.py`),
which writes the audit log, pushes the update over Socket.IO and publishes a domain event.

### Dispatch engine (`backend/app/services/dispatch.py`)
1. Online riders' positions live in a **Redis GEO set**; `GEOSEARCH` finds riders near the pickup,
   widening the radius 3 → 6 → 12 → 25 km on each failed round.
2. Candidates are scored on distance and rating. Busy riders, riders with stale GPS and riders who
   already declined are skipped.
3. The best rider gets an **offer with a 20 s countdown**. A per-rider `SET NX` reservation
   suppresses competing offers while the key is live; conditional cleanup preserves newer reservations.
4. A **Celery countdown task** expires ignored offers and moves on to the next rider.
5. Acceptance locks the rider profile, checks active work using a current read, then conditionally
   updates the pending order. This coordinates competing accept calls in MySQL.
6. Safety nets: an ownership-checked per-order lock suppresses duplicate dispatch within its lease,
   and a Celery Beat job re-dispatches pending orders whose offer was lost. SQL, Redis and event
   publication are not one transaction; this is not an exactly-once guarantee.

### Safe booking retries
`POST /api/orders` accepts an optional `Idempotency-Key` (8–128 characters from
`A-Z`, `a-z`, `0-9`, `.`, `_`, `:`, `-`). The same customer/key/validated payload returns the
original order ID with HTTP 201; conflicting payload reuse returns 409. Invalid keys return 422.
Without the header each request is a new booking. Keys are case-sensitive, stored as hashes,
retained without expiry, and scoped to the customer. Migration `0002` adds the durable identity.

The customer form reuses its key for unchanged retries while mounted; changing the request or
reloading starts a new attempt. Order, identity and initial audit commit together. Event
publication remains after commit; a transactional outbox is still needed to close that gap.

### Real time across many servers
Each API instance runs a Socket.IO server attached to a shared **Redis pub/sub** channel. Workers
emit through a write-only Redis manager. A rider connected to instance A and a customer connected to
instance B still see each other's updates, which is what lets the API scale horizontally behind a
load balancer.

### Event-driven fan-out
`order.*` events are published to the **SNS** topic `order-events`, which fans out to **SQS** queues
(dispatch, notify, analytics) with subscription filter policies and dead-letter queues. Locally,
without AWS, the same handlers run through Celery, so the code path is identical.

### Background work
| Process | Role |
|---|---|
| `api` | REST + Socket.IO (runs Alembic migrations under a MySQL `GET_LOCK` on boot) |
| `worker` | Celery: dispatch rounds, offer timeouts |
| `beat` | Celery Beat: SLA monitor (30 s), stale-rider pruning (60 s), stuck-order rescue (45 s) |
| `consumer` | AWS only: long-polls the SQS queues and runs the event handlers |

### Proof of delivery
The rider app requests a **presigned S3 PUT URL** and uploads the photo straight from the browser to
S3, so image bytes never pass through the API servers. Locally a disk-backed endpoint stands in for S3.

### Observability
`/metrics` exposes HTTP metrics (latency, throughput, status codes) plus business metrics: orders,
offers sent/accepted/declined/expired, riders online/available, average assign time, active SLA
breaches. `/api/health` serves ALB health checks; `/api/ready` checks the database and Redis.

## Configuration (environment variables)

| Variable | Purpose |
|---|---|
| `DB_HOST` `DB_USER` `DB_PASSWORD` `DB_NAME` (or `DATABASE_URL`) | MySQL / RDS |
| `REDIS_URL` | Redis / ElastiCache |
| `JWT_SECRET` | Token signing key |
| `SNS_TOPIC_ARN` | Order-events topic. When unset, events go through Celery |
| `NOTIFY_TOPIC_ARN` | Customer email/SMS notifications |
| `SQS_DISPATCH_QUEUE_URL` `SQS_NOTIFY_QUEUE_URL` | Queues read by the `consumer` role |
| `S3_BUCKET` | Proof-of-delivery photos. When unset, photos are stored on local disk |
| `PUBLIC_API_URL` | Base URL used for local upload links |
| `ENABLE_CHAOS_TESTS` | Default `false`; task-stop additionally requires admin. Never enable with public demo admin access |
