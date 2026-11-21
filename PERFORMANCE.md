# Performance & Reliability Engineering — Measurement Log

This fork of [Spring PetClinic Microservices](https://github.com/spring-petclinic/spring-petclinic-microservices) is used to practise backend performance and reliability engineering: load testing, query tuning, resilience testing and distributed tracing.

**Rules for this log**

- Every number here comes from a run that actually happened. No estimated or invented figures.
- Every result records the command, environment, data volume and commit hash so it can be reproduced.
- Each experiment states a hypothesis first, then the baseline, the change and the result. A change that made things worse is recorded too.

---

## What the project already gives us

| Capability | Where |
|---|---|
| JMeter load test plan | `spring-petclinic-api-gateway/src/test/jmeter/petclinic_test_plan.jmx` (see upstream README) |
| Metrics | Micrometer + Prometheus (`http://localhost:9091`), Grafana dashboard (`http://localhost:3030`) |
| Custom timers | `@Timed`: `petclinic.owner`, `petclinic.pet` (customers-service), `petclinic.visit` (visits-service) |
| Tracing | Micrometer Tracing + Zipkin (`http://localhost:9411/zipkin/`) |
| Resilience | Spring Cloud Circuit Breaker with Resilience4j in the API gateway |
| Fault injection | Chaos Monkey for Spring Boot (`./scripts/run_all.sh --chaos-monkey`, `./scripts/chaos/call_chaos.sh`) |
| Databases | HSQLDB in memory by default; MySQL via the `mysql` Spring profile |

## Test environment (fill in once, update if it changes)

| Item | Value |
|---|---|
| Machine | _CPU, cores, RAM_ |
| OS / Docker | _versions, Docker VM memory_ |
| Java | _version (images use Eclipse Temurin 17)_ |
| Commit | _hash of this fork under test_ |
| Load tool | _JMeter / k6 version_ |
| Database | _HSQLDB or MySQL 8.4, row counts_ |

---

## Experiments

### 1. Baseline load test through the API gateway

- **Goal:** establish p50/p95/p99 latency, throughput and error rate for the main read and write paths before changing anything.
- **How:** start the stack with `docker compose up`, wait for all services to register in Eureka (`http://localhost:8761`), warm up for 2 minutes, then run a fixed load for 10 minutes. Use the existing JMeter plan, or add a k6 script under `tests/performance/`.
- **Endpoints:** owner list, owner detail (gateway aggregates customers + visits), vets list, create visit. Confirm the exact routes in the gateway configuration before scripting.

| Run | Users / RPS | Duration | p50 | p95 | p99 | Throughput | Errors | Commit |
|---|---|---|---|---|---|---|---|---|
| baseline | | | | | | | | |

### 2. Database: query plans, indexes and N+1

- **Goal:** find slow queries on the MySQL profile and fix them with evidence.
- **How:** run with `--spring.profiles.active=mysql`, seed a realistic data volume (e.g. 10k owners, 30k pets, 100k visits), enable SQL logging, and capture `EXPLAIN ANALYZE` for:
  - owners by last name
  - visits by pet ID (visits-service)
  - owner detail with pets (check for N+1 fetches)
- **Change:** add or adjust indexes and fetch strategies one at a time; re-run experiment 1.

| Query | Rows | Before (plan, time) | Change | After (plan, time) |
|---|---|---|---|---|
| | | | | |

### 3. Resilience: circuit breaker under injected faults

- **Goal:** show what users see when a downstream service is slow or down, and how quickly the system recovers.
- **How:** run with `--chaos-monkey`, inject latency and then exceptions into visits-service using `call_chaos.sh`, while running a steady load through the gateway. Watch the Resilience4j circuit-breaker state and gateway error rate in Grafana.
- **Record:** time until the breaker opens, fallback response behaviour, error rate during the fault, time to recover after the fault is removed.

| Fault | Load | Breaker opened after | Error rate during | Recovery time |
|---|---|---|---|---|
| | | | | |

### 4. Tracing: where the time goes

- **Goal:** use Zipkin traces to find the slowest span on the owner-detail path (gateway → customers-service → visits-service).
- **Record:** trace screenshots, span breakdown before and after any fix from experiments 2 or 5.

### 5. Runtime tuning

- **Goal:** measure, rather than assume, the effect of runtime settings.
- **Candidates (one at a time):** HikariCP pool size; Java 21 with virtual threads (`spring.threads.virtual.enabled=true`) versus the Java 17 baseline; container memory limits.
- **Record:** the experiment 1 table for each setting.

### 6. Horizontal scaling

- **Goal:** check how throughput changes when a service is scaled out behind Eureka and the gateway's load balancer.
- **How:** `docker compose up --scale customers-service=2` (then 3), re-run experiment 1.

| Replicas | Throughput | p95 | Errors |
|---|---|---|---|
| 1 | | | |
| 2 | | | |
| 3 | | | |

---

## Findings

_Summaries go here only after the results above are recorded, each linked to its run and commit._

## Upstream

Original project: [spring-petclinic/spring-petclinic-microservices](https://github.com/spring-petclinic/spring-petclinic-microservices) (Apache 2.0). Improvements that are generally useful may be offered back upstream as pull requests.
