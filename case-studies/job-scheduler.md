# Design a Distributed Job Scheduler

Source: Hello Interview outline (premium); answers are independent · Patterns: long-running tasks, scaling writes · Difficulty: medium

## Scope
- **FR:** schedule jobs to run immediately, at a future time, or on a recurring (cron) schedule; monitor job status/history. Below the line: DAG dependencies, priorities, cancellation UI.
- **NFR:** execute within **2 s** of scheduled time; up to **10k jobs/s**; **at-least-once** execution; highly available; job durations from seconds to minutes.

## Crux
Separating *definitions* from *executions*, finding due executions cheaply without scanning everything, and dispatching precisely and reliably at 10k/s.

## Skeleton
Entities: Job (definition: schedule/cron, payload, owner), Execution (job_id, scheduled_time, status, attempt, worker). API: `POST /jobs`, `GET /jobs/{id}`, `GET /jobs/{id}/executions?status&cursor`.
Data flow: create job → compute next execution(s) → store Execution keyed by **time bucket** → scheduler loads soon-due executions → enqueue → workers run → update status → (recurring) create next execution.

## Deep dives

**1. Within 2 s of scheduled time**
Two-tier scheduling: a periodic loader (every ~5 min) queries executions due in the next window from the DB (DynamoDB: PK = time bucket e.g. minute/hour, SK = time+jobId; or Postgres index on (scheduled_time, status)) and pushes them into a **delay mechanism** that releases exactly on time — SQS delay messages (≤ 15 min), Redis sorted set scored by timestamp polled every ~100 ms, or a timing wheel. Jobs created for "now / within the window" bypass the loader and go straight to the delay queue.

**2. 10k jobs/s**
Partition time buckets with a shard suffix (`bucket#0..N`) to avoid hot partitions; multiple loader instances each own shards (lease via DynamoDB conditional writes / etcd); queue sized for bursts (top of the hour cron spikes — spread with jitter where acceptable); autoscaled worker pools; separate queues per workload class.

**3. At-least-once execution**
Workers claim via visibility timeout/lease and heartbeat for long jobs; on crash the message reappears → retry with exponential backoff, max attempts → DLQ + status `failed`. Duplicates are therefore possible → **jobs must be idempotent** (execution id as idempotency key; conditional status transitions `queued → running WHERE status='queued'`). Status updates are best-effort; reconciliation sweeps find executions stuck in `running` past their lease.

## Staff+ signals
- Clear model: definitions vs executions; recurring jobs generate their next execution on completion *or* ahead of time (avoid drift and double-generation).
- Clock/time-zone/DST semantics for cron; missed-fire policy after outages (run once vs catch up all).
- Leader election only where needed (loader shard ownership), not a single global scheduler.
- Fairness/isolation between tenants (per-tenant quotas); priority lanes.
- Observability: schedule lag (actual − scheduled) as the key SLO metric.

## Traps
Polling the whole table every second; single scheduler SPOF; promising exactly-once; hot time-bucket partitions at :00.
