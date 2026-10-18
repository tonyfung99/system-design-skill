# Design an Online Judge (LeetCode)

Source: full breakdown · Patterns: long-running tasks · Difficulty: medium

## Scope
- **FR:** browse problems; view a problem and code in multiple languages; submit and get feedback; live leaderboard for competitions. Below the line: auth details, discussion, plagiarism.
- **NFR:** availability > consistency; isolation & security of user code; results within 5 s; competitions with 100k users.

## Crux
Running untrusted code safely and quickly at bursty contest scale, and serving a leaderboard that changes constantly.

## Skeleton
Entities: Problem (statement, code stubs per language, test cases), Submission (user, problem, competition, code, status, results), Leaderboard. API: `GET /problems?page`, `GET /problems/{id}?language`, `POST /problems/{id}/submit {code, language}`, `GET /submissions/{id}`, `GET /leaderboard/{competitionId}?page`.
API server → execution workers (per-language containers) → results to DB; client polls submission status.

## Deep dives

**1. Isolation & security**
Containers (fast start) vs VMs/microVMs (Firecracker, gVisor — stronger isolation) vs serverless functions (cold starts, limits). Hardening: read-only filesystem + temp scratch dir, CPU/memory cgroup limits, wall-clock timeout (≤ 5 s), no network (VPC/security groups), seccomp syscall filter, non-root user, pre-warmed pools, destroy after use.

**2. Leaderboard**
✗ Poll + sort in DB every few seconds. ◐ Cache refreshed every 30 s. ✓ **Redis sorted set per competition** updated on each accepted submission (score = solved count weighted by time), clients poll every ~5 s (polling is fine here — no need for WebSockets).

**3. 100k-user competitions**
Do the math: 10k concurrent submissions × ~100 test cases × ~10–100 ms = CPU-bound fast. ✗ Vertical scaling. ✓ Autoscaled per-language worker pools. ✓ **Queue (SQS) in front of workers** to buffer bursts and retry; trades a little latency for no lost submissions; client polls status.

**4. Running test cases for any language**
One language-agnostic test-case set per problem with a serialization format (e.g., trees as level-order arrays); per-language harness deserializes inputs, invokes the user's function, compares outputs. Breaks candidates out of box-drawing mode.

## Staff+ signals
- Choose sandbox tech by threat model and density (gVisor/Firecracker vs Docker), warm pools for latency.
- Fairness in contests (deterministic resource limits, noisy-neighbor isolation, consistent hardware).
- Abuse: fork bombs, output flooding, crypto mining → limits + kill.
- Result streaming per test case via SSE as an option; idempotent re-execution on worker death.
- Leaderboard tie-breaking and the "freeze" in final minutes as product decisions.

## Level bar
Mid: functional design, recognizes need for isolation (container/VM/serverless). Senior: compare isolation options with justification; explain how test cases actually run. Staff: depth across security, scaling, and harness design with clear judgment.
