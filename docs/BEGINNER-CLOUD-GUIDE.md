# DispatchX explained like you are seeing it for the first time

This is the beginner's guide to DispatchX.

If you remember only one sentence, remember this:

> **DispatchX is a parcel-delivery application whose code runs in different processes,
> stores important data in MySQL, keeps fast live state in Redis-compatible storage,
> and can be packaged locally with Docker or deployed to AWS using ECS on EC2.**

The project has two sides:

1. **The product side:** a customer books a delivery, a rider accepts it, and a dispatcher
   watches it.
2. **The cloud side:** different AWS services provide the front door, containers, database,
   cache, files, queues, security and monitoring.

This guide explains both sides without pretending that a diagram is the same thing as a
currently running cloud deployment.

### If you only have 10 minutes

Read these sections in order:

1. **Section 2:** the whole system in one picture.
2. **Section 4:** what happens during one booking.
3. **Section 5:** why MySQL and Redis are both needed.
4. **Section 8:** the AWS architecture.
5. **Section 9:** the AWS services table.
6. **Section 17:** what we are not using.

---

## 1. What problem are we solving?

Imagine a small-city delivery company.

- A **customer** wants to send a parcel.
- A **rider** needs to receive and complete the job.
- A **dispatcher** needs to see what is happening across the fleet.

DispatchX gives each person a different screen:

| Person | What they do |
|---|---|
| Customer | Selects pickup and drop-off, gets a quote, books, tracks the rider, sees proof of delivery |
| Rider | Goes online, shares location, accepts an offer, confirms pickup, completes delivery |
| Dispatcher | Watches orders, riders, metrics, logs and infrastructure snapshots |

The normal order journey is:

```text
pending → assigned → picked_up → delivered
```

Other possible endings are `cancelled` or `unassigned` when the system cannot find a rider
after its retry attempts.

---

## 2. The whole system in one picture

Here is the simplest useful picture:

```text
 Customer browser ───────┐
 Rider browser ──────────┼──> DispatchX API ───> MySQL
 Dispatcher browser ────┘          │              │
                                   │              └─ orders, users, history
                                   │
                                   ├──> Redis/Valkey
                                   │      ├─ rider locations
                                   │      ├─ nearby-rider search
                                   │      ├─ temporary locks and offers
                                   │      ├─ Socket.IO pub/sub
                                   │      └─ Celery broker
                                   │
                                   ├──> Celery worker
                                   │      └─ dispatch, timeouts, notifications
                                   │
                                   └──> object storage for delivery photos
                                          └─ S3 in AWS; local disk locally
```

The browser does not directly decide who gets an order. The browser asks the API. The API
checks the database and fast live state, then background processes do work that should not
block the user's request.

---

## 3. The first important idea: one codebase, several process roles

People often hear “cloud” and immediately imagine many independent microservices. That is
not exactly what we built.

DispatchX has **one backend codebase and one backend Docker image**, but that image can be
started with different commands:

| Process | Simple explanation |
|---|---|
| `api` | Answers HTTP requests and handles Socket.IO live connections |
| `worker` | Performs background jobs, such as dispatching and offer timeouts |
| `beat` | A scheduler that periodically starts maintenance jobs; it should have one active copy |
| `consumer` | AWS-only event consumer that reads SQS messages |
| `web` | Serves the Next.js frontend |
| `monitoring` | Runs Prometheus and Grafana containers in the AWS design |

So this is a **separation of responsibilities**, not six totally separate products.

Why separate them?

- API work should stay responsive even when many background jobs exist.
- Workers can be increased without creating more schedulers.
- The web frontend can be deployed independently from the API process.
- The AWS deployment can scale the API, worker and event-consumer roles differently.

The important exception is `beat`: having two schedulers can duplicate periodic work, so the
design keeps it as a singleton role.

---

## 4. What happens when a customer books?

Let us follow one order slowly.

### Step 1: the customer chooses two places

The customer page lets the user choose a pickup and drop-off. The browser gets coordinates
from the map or address search.

The browser sends those coordinates to the API. The browser is not trusted to decide the
final price or assignment.

### Step 2: the API calculates a quote

The backend calculates:

- distance,
- a fare,
- an estimated delivery time.

The current model is intentionally simple: straight-line distance, a road-distance factor,
and an average speed. It is not live traffic routing.

### Step 3: the customer creates the order

The API writes the order into **MySQL**. It also writes an initial audit event such as
`status.pending`.

The optional `Idempotency-Key` protects against a common mobile/web problem: the customer
clicks Book, the request succeeds, but the response gets lost. If the browser retries the
same customer/key/payload, the API returns the original order instead of creating a duplicate.

This is durable database protection, not a claim of exactly-once event delivery. The order
commit and later event publication are still separate operations.

### Step 4: background dispatch begins

The API publishes an `order.created` event. Locally, the event is handled through Celery.
In the AWS design, events can go through SNS and SQS.

The worker searches Redis/Valkey for nearby riders:

1. Look in the rider GEO index.
2. Keep riders who are online, available and have fresh location data.
3. Ignore riders already tried for this order.
4. Score candidates using distance and rating.
5. Offer the job to the best candidate.

The search radius can grow from 3 to 6 to 12 to 25 km on later attempts.

### Step 5: the rider sees an offer

The rider receives a live Socket.IO event with an offer and countdown.

Redis holds short-lived offer/reservation keys so two dispatch workers do not casually give
the same rider two offers at the same time.

### Step 6: acceptance is checked again on the server

When the rider clicks Accept, the API does not blindly trust the button click. It checks:

- whether the offer still belongs to that rider,
- whether the rider profile can be locked,
- whether the rider already has active work,
- whether the order is still pending.

The database then changes the order to `assigned` using a conditional update. The rider's
profile row is locked during the important MySQL decision, which protects competing accepts
under the configured MySQL isolation behavior.

### Step 7: live tracking happens

The rider sends location updates. Redis stores the latest live location. Socket.IO broadcasts
updates to the customer and dispatcher screens.

MySQL stores business history. Redis holds the fast-changing “where is the rider right now?”
state. These are different jobs.

### Step 8: pickup and delivery are recorded

The rider confirms pickup, then completes delivery. Every status transition writes an audit
event and broadcasts an update.

For proof of delivery:

- **AWS:** the browser uploads the photo directly to S3 using a presigned URL.
- **Local development:** a local disk endpoint stands in for S3.

The photo upload is not proof that an AWS S3 upload happened when the app is running locally.

---

## 5. Why do we use both MySQL and Redis?

This is one of the most important explanations in the project.

### MySQL is the durable record

MySQL stores things we cannot casually lose:

- users and roles,
- rider profiles,
- orders,
- order status,
- prices and locations used by the order,
- audit events,
- durable booking retry identities.

If the API restarts, this is the place we read the order history back from.

### Redis/Valkey is the fast coordination layer

Redis-compatible storage is used for things that change quickly or need fast coordination:

- rider GEO locations,
- online and available rider sets,
- current rider state,
- temporary offer keys,
- dispatch locks,
- Socket.IO pub/sub between API instances,
- Celery's local broker/ready queue,
- short-lived counters and timing samples.

The AWS deployment uses **Amazon ElastiCache with Valkey**, which speaks the Redis protocol.
The local stack uses Redis.

Simple rule:

> **MySQL answers “what is the durable history?” Redis answers “what is happening quickly
> right now?”**

---

## 6. What runs on our laptop?

The normal local environment uses Docker Compose.

```text
Docker host / Colima
├─ MySQL container
├─ Redis container
├─ API container
├─ Celery worker container
├─ Celery Beat container
└─ Next.js web container or local frontend preview
```

The local browser normally uses:

| Local address | Meaning |
|---|---|
| `http://localhost:3000` | Full application gateway/frontend arrangement |
| `http://localhost:8000` | API and Socket.IO |
| `http://localhost:8000/api/docs` | Swagger API documentation |
| `http://localhost:3200` | Standalone frontend production preview when used |

The local Compose stack proves that the application can work with MySQL, Redis, Celery and
real browser sessions on one Docker host. It does **not** prove multi-host failure recovery,
AWS autoscaling or production availability.

The optional simulator creates virtual riders/customers and orders. The `/demo` page only
generates commands; it does not start the simulator by itself.

---

## 7. Where does Kubernetes fit?

The `k8s/` folder runs the same Docker images on Kubernetes, usually with **kind**.

Kind means Kubernetes running inside Docker on a local machine or CI runner.

```text
kind cluster
├─ control-plane node
├─ worker node: zone-a
└─ worker node: zone-b
```

Kubernetes provides:

- Deployments to keep the desired number of pods running,
- Services for stable internal routing,
- Ingress/NGINX for HTTP and WebSocket entry,
- readiness and liveness probes,
- a MySQL StatefulSet and volume,
- Redis deployment,
- Horizontal Pod Autoscalers for API and workers.

The HPA can increase **pod count**. It does not automatically add more kind nodes in this
configuration.

Kubernetes is our local/CI alternative deployment. It is not the AWS deployment in this
repository.

---

## 8. The AWS architecture in plain English

The AWS design is deployed with **CloudFormation**. CloudFormation is AWS's infrastructure-as-
code system: instead of manually clicking every resource, YAML templates describe what should
exist.

The target AWS region is **`ap-southeast-2` (Sydney)**.

The high-level AWS path is:

```text
User browser
   │ HTTPS / WebSocket
   ▼
CloudFront
   │ secret origin header
   ▼
Application Load Balancer + regional WAF
   │
   ▼
ECS cluster running on EC2 instances
   ├─ web
   ├─ api
   ├─ worker
   ├─ beat
   ├─ consumer
   └─ monitoring
        │
        ├─ RDS MySQL
        ├─ ElastiCache Valkey
        ├─ S3
        └─ SNS/SQS
```

The key sentence is:

> **We use Amazon ECS running on EC2, not Amazon EKS.**

ECS is the container orchestrator. EC2 instances are the machines underneath it. ECS places
tasks on those machines and keeps the desired services running.

For a browser, the outside connection is HTTPS or WebSocket Secure. In the configured AWS
template, CloudFront forwards to the ALB origin on HTTP port 80 inside the AWS network; this
does not mean the user's browser connection is HTTP.

---

## 9. AWS services we use or configure

This table is the “what AWS stuff are we using?” answer.

| AWS service | What it does in this project | Easy mental model |
|---|---|---|
| **Amazon CloudFront** | Public HTTPS edge, static asset caching, API/WebSocket forwarding | The front door and fast global-ish delivery layer |
| **Application Load Balancer** | Routes traffic from CloudFront to healthy ECS targets | The traffic distributor inside the VPC |
| **AWS WAF** | Filters suspicious/rate-limited web requests before the ALB | A security checkpoint |
| **Amazon ECS** | Runs the web/API/worker/Beat/consumer/monitoring containers | The container manager |
| **Amazon EC2 Auto Scaling Group** | Supplies the virtual machines where ECS tasks run | The pool of machines under ECS |
| **Amazon ECR** | Stores backend, web, Prometheus and Grafana container images | The private container-image warehouse |
| **Amazon RDS for MySQL** | Durable relational database for users, orders and audit history | The managed MySQL server |
| **Amazon ElastiCache for Valkey** | Redis-compatible live state, GEO search, pub/sub and broker use | The fast shared coordination store |
| **Amazon S3** | Proof-of-delivery photos and analytics data lake | Durable object/file storage |
| **Amazon SNS** | Fans order events or notifications out to subscribers | A broadcast announcement service |
| **Amazon SQS** | Buffers work for dispatch, notification and analytics consumers | A queue of tasks waiting to be processed |
| **SQS dead-letter queues** | Hold messages that repeatedly fail | The “failed messages go here” box |
| **AWS Lambda** | Validates uploaded POD metadata and ingests analytics messages | Small event-triggered functions |
| **Amazon EventBridge** | Triggers the POD validator from S3 events | An event wiring/routing service |
| **AWS Glue** | Describes the analytics data for querying | A data catalog |
| **Amazon Athena** | Runs SQL queries over analytics files in S3 | Querying the data lake without a database server |
| **Amazon CloudWatch** | Dashboards, alarms, container insights and log/flow visibility | AWS monitoring and alarms |
| **AWS Secrets Manager** | Stores database/JWT/Grafana secrets | A locked cabinet for credentials |
| **AWS Cloud Map** | Private service discovery for internal ECS services | Internal DNS/service directory |
| **Amazon VPC** | Private network, subnets, routes and security groups | The private AWS neighborhood |
| **NAT Gateway** | Lets private application resources reach required outside services | A controlled exit from private subnets |

Not every service is part of every local run. Many of the entries above describe the AWS
target defined by the infrastructure templates.

---

## 10. AWS network layout, slowly

The AWS VPC is divided into subnet types across two availability zones.

### Public subnets

The public-facing load balancer lives here. “Public” does not mean every application container
is directly exposed to the internet.

### Private application subnets

The ECS EC2 instances and tasks live here. Users reach them through CloudFront and the ALB,
not by directly opening random container ports to the world.

### Isolated data subnets

RDS MySQL and ElastiCache live here. They are intended to be reachable by the application
network, not directly by internet users.

### Traffic protection

- Security groups restrict which layer can talk to which other layer.
- CloudFront sends a secret origin header that the ALB rules expect.
- Direct ALB access is rejected by design.
- WAF applies web request rules at the load-balancer edge.
- VPC flow logs record rejected network flows for investigation.

The exact availability, backup and redundancy settings still depend on the deployment
parameters. Multi-AZ RDS and cache replicas are optional settings, not automatic proof of
high availability.

---

## 11. What exactly runs inside ECS?

ECS runs several services from the project images:

| ECS service | Why it exists |
|---|---|
| `web` | Serves the Next.js frontend |
| `api` | Handles REST, Socket.IO and application requests |
| `worker` | Runs Celery background jobs |
| `beat` | Starts periodic maintenance jobs; one copy |
| `consumer` | Reads AWS SQS event queues and calls handlers |
| `monitoring` | Runs Prometheus and Grafana containers |

The API and web are reached through the ALB. Workers and Beat usually do not need public
HTTP access; they consume internal work.

ECS has two different scaling ideas here:

1. **Service scaling:** create more copies of an API, worker or consumer task.
2. **Capacity scaling:** add more EC2 container instances when the current machines are full.

The design configures CPU-based service scaling for API/workers and queue-depth-based scaling
for the SQS consumer. The EC2 capacity provider can add hosts within its limits.

Those are configured behaviors. A local laptop test does not prove that AWS actually scaled
or recovered a task.

---

## 12. Why SNS and SQS are both present

Suppose an order is created. Multiple things may need to react:

- dispatch should find a rider,
- notification should tell a customer something,
- analytics should record the event.

Instead of making the API call all three consumers directly:

```text
API publishes order.created
          │
          ▼
       SNS topic
       /   |   \
      /    |    \
 dispatch notify analytics
   SQS     SQS      SQS
    │       │        │
 consumer consumer  Lambda
```

SNS is the broadcast/fan-out part. SQS is the durable queue for each subscriber.

This gives consumers their own pace. If analytics is slow, it should not directly block
dispatch. If a message repeatedly fails, the dead-letter queue keeps it for investigation.

Locally, we do not need AWS SNS/SQS to demonstrate the business flow. The local event path
uses Celery and the same application handlers.

---

## 13. What happens to a delivery photo in AWS?

The intended AWS path is:

```text
Rider browser ──asks API for signed upload URL──> API
Rider browser ──uploads photo directly─────────> S3
S3 event ──> EventBridge ──> validation Lambda
```

The API does not need to carry the whole image through its own container. That reduces work
for the API and lets S3 handle object storage.

The validator checks metadata/tags according to the configured Lambda flow. It does not
replace the order state machine, approve the delivery by itself, or prove that a photo is
available in every environment.

Locally, uploads use a disk-backed fallback because there is no S3 bucket in the Compose demo.

---

## 14. What is the analytics path?

Analytics is deliberately separate from the transactional MySQL path:

```text
Order events
   ▼
SQS analytics queue
   ▼
Analytics Lambda
   ▼
S3 data lake, partitioned by date
   ▼
Glue catalog
   ▼
Athena SQL queries
```

This is useful for reporting and historical analysis. It is not the live source of truth for
whether a rider accepted an order. MySQL and the application event handlers handle the live
product flow.

---

## 15. AWS deployment process

The intended deployment path is:

```text
Developer pushes code
        ▼
GitHub Actions
  1. test
  2. build container images
  3. push images to ECR
  4. package CloudFormation
  5. deploy/update the stack
  6. run seed task
  7. run smoke test
```

There are two infrastructure layers:

### Bootstrap stack

Created once. It creates ECR repositories and an artifacts bucket.

### Main application stack

The parent CloudFormation stack wires together nested stacks for:

- network,
- data,
- messaging,
- application/ECS,
- edge/CloudFront,
- analytics,
- observability.

The repository keeps most of this in `infra/`. The ECS application template is generated
from `infra/src/app.template.yml` into `infra/stacks/app.yml`.

Important operational rule:

> **Pushing to `main` can deploy AWS. Work on the feature branch unless deployment is
> intentionally wanted.**

The current documentation and local verification pass did not create or modify AWS resources.
Always check the current GitHub Actions result and stack outputs before claiming a live AWS
deployment is healthy.

---

## 16. AWS versus local: the translation table

| Responsibility | Local Compose | Local/CI Kubernetes | AWS target |
|---|---|---|---|
| Run containers | Docker Compose | Kubernetes pods | ECS tasks |
| Machines | One Docker host | kind nodes | EC2 instances in an Auto Scaling group |
| Front door | Local port/gateway | NGINX Ingress | CloudFront → ALB |
| Database | MySQL container | MySQL StatefulSet + volume | RDS MySQL |
| Fast state | Redis container | Redis Deployment | ElastiCache Valkey |
| Background jobs | Celery | Celery | Celery plus ECS roles; AWS consumer reads SQS |
| Event fan-out | Local Celery event path | Local Celery event path | SNS → SQS |
| Photos | Local disk fallback | Pod disk fallback unless configured otherwise | S3 |
| Scaling | Manual container changes | HPA changes pod replicas | ECS service scaling plus EC2 capacity scaling |
| Service discovery | Compose service names | Kubernetes DNS | Cloud Map |
| Monitoring | Optional local containers | Kubernetes tools / app view | CloudWatch plus Prometheus/Grafana |

The application code tries to preserve the same business responsibilities across environments,
but the infrastructure around it is different.

---

## 17. What we are **not** using

This section is important because it prevents accidental overclaiming.

### Not EKS

The AWS target is **ECS on EC2**. The Kubernetes manifests are for local/CI Kubernetes and
could support a future Kubernetes deployment, but this repository is not currently describing
an EKS deployment.

### Not a collection of independent microservices

The API, worker, Beat and consumer share the backend codebase and image. They are separate
process roles, not independently designed databases and repositories.

### Not DynamoDB

Orders and audit history use MySQL/RDS, not DynamoDB.

### Not API Gateway as the main front door

The configured front door is CloudFront and an Application Load Balancer.

### Not Lambda for the main API

FastAPI runs in ECS containers. Lambda is used for event-driven helper paths such as analytics
ingest and POD validation.

### Not a live-routing or traffic-optimization platform

The demo uses straight-line distance and simulated movement. It does not include a paid road
routing/traffic provider by default.

### Not automatically high-availability everywhere

The templates contain multi-AZ and replica options, but the safer reading is:

- RDS Multi-AZ can be enabled,
- cache replicas can be enabled,
- local Compose is still one host,
- local/kind Redis is not configured as a production durable cache,
- no failover claim should be made without observing it.

### Not exactly-once processing

The booking retry key prevents a class of duplicate order creations. It does not make every
database write, Redis update and event publication one atomic operation. A transactional
outbox is not implemented.

### Not proven live right now by a diagram

The public cloud explorer is a source-backed explanation. It does not query AWS, scale a
cluster, kill a task or generate load. The authenticated cluster view only becomes live when
it can reach a configured ECS or Kubernetes platform.

---

## 18. What to say in a viva or presentation

Here is a simple script.

> “DispatchX is a real-time parcel delivery application. The customer, rider and dispatcher
> use a Next.js frontend and FastAPI backend. MySQL stores durable orders and audit history.
> Redis-compatible storage keeps live rider locations, nearby search and coordination locks.
> Celery handles local background work. For AWS, we package the same roles as containers and
> run them on ECS on EC2. CloudFront and the ALB provide the entry path, RDS provides MySQL,
> ElastiCache provides Valkey, S3 stores delivery photos, and SNS/SQS separate event consumers.
> We also configure CloudWatch, WAF, Secrets Manager and Cloud Map. Kubernetes is the local/CI
> alternative, not EKS. Our local tests prove the delivery flow and selected concurrency behavior;
> they do not prove AWS failover or unlimited capacity.”

If someone asks “what scales?”, answer:

- API task copies can scale.
- Worker task copies can scale.
- SQS consumer task copies can scale from queue backlog.
- EC2 capacity can scale when ECS needs more room.
- Kubernetes HPA can change pod copies in the Kubernetes environment.

If someone asks “what did you actually prove locally?”, answer:

- a real customer/rider/dispatcher delivery flow,
- MySQL persistence and audit read-back,
- Redis/Celery dispatch behavior,
- concurrent keyed booking protection,
- competing rider-assignment protection,
- responsive demo and system screens.

Do not turn those results into a claim that AWS autoscaling, failover or production throughput
was measured.

---

## 19. Glossary

| Word | Meaning in simple language |
|---|---|
| API | The backend endpoint that receives and answers requests |
| Container | A packaged process with its code and dependencies |
| Docker image | The package used to create a container |
| ECS task | A running ECS copy of a container task definition |
| EC2 | A virtual machine in AWS |
| EKS | AWS managed Kubernetes; not our configured AWS orchestrator |
| ALB | A load balancer that sends traffic to healthy targets |
| CloudFront | AWS edge/front-door service for HTTP and WebSockets |
| VPC | A private network in AWS |
| RDS | AWS-managed database service; ours is MySQL |
| Valkey | Redis-compatible in-memory data store used through ElastiCache |
| GEO search | Finding nearby riders by coordinates |
| Pub/sub | One process publishes a message and other processes receive it |
| SNS | Broadcast/fan-out messaging |
| SQS | Durable queue messaging |
| DLQ | Dead-letter queue for messages that keep failing |
| Presigned URL | Temporary permission to upload directly to S3 |
| HPA | Kubernetes Horizontal Pod Autoscaler |
| Capacity provider | ECS mechanism that helps manage EC2 capacity |
| CloudFormation | AWS infrastructure-as-code templates |
| Cloud Map | Internal service discovery/private DNS |
| CloudWatch | AWS monitoring, dashboards, logs and alarms |
| Idempotency | Retrying the same request does not create a duplicate result |

---

## 20. Where to read next

- Start the presentation at **`/demo`**.
- Read [`DEMO-RUNBOOK.md`](DEMO-RUNBOOK.md) for the exact eight-minute flow.
- Read [`SCALABILITY-PROGRESS.md`](SCALABILITY-PROGRESS.md) for measured results and limits.
- Read [`../infra/README.md`](../infra/README.md) for AWS deployment operations.
- Read [`../k8s/README.md`](../k8s/README.md) for the Kubernetes alternative.
- Open **`/architecture`** for the interactive, source-backed visual explorer.
