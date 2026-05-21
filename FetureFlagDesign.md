# Feature Management Service — System Design

## Table of Contents

1. [Problem Statement](#1-problem-statement)
2. [Architecture Overview](#2-architecture-overview)
3. [Core Components](#3-core-components)
4. [Caching Strategy](#4-caching-strategy)
5. [Client SDK Design](#5-client-sdk-design)
6. [API Design](#6-api-design)
7. [Observability](#7-observability)
8. [Explainability](#8-explainability)
9. [Security](#9-security)
10. [High Availability & Fault Tolerance](#10-high-availability--fault-tolerance)
11. [MVP Phasing](#11-mvp-phasing)

---

## 1. Problem Statement

Design a Feature Management Service for an e-commerce platform that:

- Manages **thousands of feature flags** across **100+ applications and services** (web portals, backend APIs, mobile clients).
- Evaluates flags at **high throughput and low latency**.
- Maintains **reasonable resource cost** as the number of flags grows.

The design must cover:

| # | Requirement | Section |
|---|-------------|---------|
| 1 | A caching strategy that is efficient and cost-effective at scale | [§4](#4-caching-strategy) |
| 2 | A client SDK that is easy to integrate and consistent across client types | [§5](#5-client-sdk-design) |
| 3 | Complete API design — Management, Evaluation, and auxiliary APIs | [§6](#6-api-design) |
| 4 | An observability solution for health monitoring, debugging, and behaviour analysis | [§7](#7-observability) |
| 5 | An explainability model (who, where, which release, etc.) | [§8](#8-explainability) |

---

## 2. Architecture Overview

### 2.1 High-Level Architecture

```
┌──────────────────────────┐
│        Admin UI           │
│  Feature / Rule / Debug   │
└─────────────┬────────────┘
              │
              ▼
┌──────────────────────────┐
│ Spring Boot Management API│
│ Auth / RBAC / Audit       │
│ Publish Orchestration     │
└─────────────┬────────────┘
              │
              ▼
┌──────────────────────────┐
│        Feature DB         │
│ Source of Truth            │
│ Feature / Rule / Audit    │
│ Snapshot / Release        │
└─────────────┬────────────┘
              │
   ┌──────────┴──────────┐
   ▼                     ▼
┌────────────────┐  ┌────────────────────┐
│ Rule Compiler   │  │ Snapshot Builder    │
│ (Rust)          │  │ (Rust)             │
│ Validate / IR   │  │ Full / Delta / Diff│
└───────┬────────┘  └───────┬────────────┘
        └────────┬──────────┘
                 ▼
┌──────────────────────────┐
│ Object Storage / CDN      │
│ server-full.json.gz       │
│ client-full.json.gz       │
│ delta.json.gz / latest    │
└─────────────┬────────────┘
              │
     ┌────────┴────────┐
     ▼                 ▼
┌──────────┐   ┌──────────────────────────┐
│SSE GW    │   │ CDN Edge (public)         │
│(internal)│   │ Client snapshots only     │
└────┬─────┘   └────────┬─────────────────┘
     │                  │
     ▼                  │
┌──────────────────┐    │
│   Relay Proxy     │    │
│ Cache / SSE / API │    │
└────────┬─────────┘    │
         │              │
  ┌──────┼──────┐       │
  ▼      ▼      ▼       ▼
┌─────┐┌─────┐┌──────┐┌──────────┐
│Java ││Go   ││Node  ││Web SDK   │
│SDK  ││SDK  ││SDK   ││(Browser) │
│     ││     ││      │├──────────┤
│Back-││Back-││Back- ││Mobile SDK│
│end  ││end  ││end / ││iOS/Andrd │
│API  ││API  ││SSR   ││          │
└──┬──┘└──┬──┘└──┬───┘└────┬─────┘
   │      │      │         │
   └──────┼──────┘         │
          ▼                ▼
┌──────────────┐  ┌────────────────┐
│Server-Side   │  │Client-Side     │
│Evaluation    │  │Evaluation      │
│Full rules    │  │Stripped rules  │
│HashMap+Hash  │  │WASM / Native   │
└──────────────┘  └────────────────┘
```
## SSE 

> SSE is the server proactively "streaming messages" to the client over a plain HTTP connection that stays open — data flows one message at a time.

---

### How it differs from polling

**Polling:** The client asks "any new messages?" every few seconds — most of the time, nothing comes back.

**SSE:** Once the connection is established, the server pushes when there's something; otherwise it waits — saves bandwidth, lower latency.

---

### How it differs from WebSocket

| | SSE | WebSocket |
|--|-----|-----------|
| Direction | Server → Client (unidirectional) | Bidirectional |
| Protocol | Plain HTTP | Requires protocol upgrade (ws://) |
| Reconnection | Browser auto-reconnects | Must be handled manually |
| Best for | Push notifications, status updates | Chat, gaming, bidirectional interaction |

---

The system serves **three distinct client types** with different SDK strategies:

| Client Type | Examples | Snapshot Type | Connectivity | Evaluation |
|-------------|----------|--------------|--------------|------------|
| **Backend API** | Java microservices, Go services, Node.js servers | Server snapshot (full rules) | Relay Proxy (internal network) | In-process, full local evaluation |
| **Web Portal** | React SPA, Next.js SSR, Angular apps | Client snapshot (stripped) for CSR; Server snapshot for SSR | CDN-direct (CSR) or Relay Proxy (SSR) | WASM or JS evaluation (CSR); server-side (SSR) |
| **Mobile Client** | iOS app, Android app, React Native | Client snapshot (stripped) | CDN-direct (public internet) | Native or WASM evaluation, offline-capable |

### 2.2 Control Plane vs Data Plane Separation

This is the fundamental architectural principle. The **control plane** handles configuration, validation, and publishing. The **data plane** handles distribution and evaluation.

| Concern | Control Plane | Data Plane |
|---------|--------------|------------|
| Components | Management API, Feature DB, Rule Compiler, Snapshot Builder | CDN, SSE Gateway, Relay Proxy, SDKs |
| Consistency | Strong (ACID transactions) | Eventual (snapshot propagation) |
| Availability | Standard (brief downtime tolerable) | Extreme (must never block business requests) |
| Latency | Seconds acceptable | Sub-millisecond evaluation |

**Key invariant:** The Feature DB never participates in online flag evaluation. All runtime evaluation is performed locally in SDK memory.

```
✗  User Request → Business Service → Feature DB query
✓  User Request → Business Service → SDK in-memory evaluation
```

### 2.3 Data Flow

**Control plane (write path):**
```
Admin UI → Management API → Feature DB → Rule Compiler → Snapshot Builder → Object Storage/CDN → SSE Gateway publishes snapshot_published event
```

**Data plane (read path):**
```
Relay Proxy subscribes to SSE Gateway → pulls full/delta snapshot from CDN → caches locally → distributes to internal SDKs via API/SSE
```

**Business request (evaluation path):**
```
Business Service → SDK.isEnabled(flagKey, context) → local in-memory HashMap lookup → return value in <1ms
```

### 2.4 Technology Choices

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| Management API | Spring Boot | Mature ecosystem for auth, transactions, audit, DB access |
| Feature DB | PostgreSQL | ACID, JSONB, rich query support |
| Rule Compiler | Rust | Type safety, deterministic output, high-performance parsing |
| Snapshot Builder | Rust | Efficient serialization, compression, checksumming |
| SSE Gateway | Rust (tokio + axum) | High-concurrency long-connections, low resource usage |
| Relay Proxy | Rust (tokio + hyper + moka) | High-concurrency proxy, low memory, consistent with Rust data plane |
| Admin UI | React + Ant Design | Rich component library, RBAC integration |
| Object Storage | S3 + CloudFront (or equivalent) | Global CDN distribution |

---
### 2.5 Deployment Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                              GitHub Repository (Monorepo)                                 │
│  /services/management-api  /services/rule-compiler  /services/sse-gateway  /admin-ui     │
└──────────────────────┬──────────────────────────────────────────────────────────────────┘
                       │  push / PR
                       ▼
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                           GitHub Actions (CI)                                             │
│                                                                                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                │
│  │ management-  │  │ rule-compiler│  │ sse-gateway  │  │  admin-ui    │                │
│  │ api.yml      │  │ .yml         │  │ .yml         │  │  .yml        │                │
│  │              │  │              │  │              │  │              │                │
│  │ ┌──────────┐│  │ ┌──────────┐│  │ ┌──────────┐│  │ ┌──────────┐│                │
│  │ │Unit Test ││  │ │Cargo Test││  │ │Cargo Test││  │ │npm test   ││                │
│  │ │SAST Scan ││  │ │Clippy    ││  │ │Clippy    ││  │ │ESLint     ││                │
│  │ │OWASP Dep ││  │ │Audit     ││  │ │Audit     ││  │ │           ││                │
│  │ └────┬─────┘│  │ └────┬─────┘│  │ └────┬─────┘│  │ └────┬─────┘│                │
│  │      │      │  │      │      │  │      │      │  │      │      │                │
│  │ ┌────▼─────┐│  │ ┌────▼─────┐│  │ ┌────▼─────┐│  │ ┌────▼─────┐│                │
│  │ │Docker    ││  │ │Docker    ││  │ │Docker    ││  │ │npm build  ││                │
│  │ │Build+Push││  │ │Build+Push││  │ │Build+Push││  │ │→ artifact ││                │
│  │ └────┬─────┘│  │ └────┬─────┘│  │ └────┬─────┘│  │ └────┬─────┘│                │
│  └──────┼──────┘  └──────┼──────┘  └──────┼──────┘  └──────┼──────┘                │
└─────────┼─────────────────┼─────────────────┼─────────────────┼─────────────────────────┘
          │                 │                 │                 │
          ▼                 ▼                 ▼                 ▼
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                              AWS ECR                              │   S3 (staging)       │
│  ┌────────────────┐ ┌────────────────┐ ┌────────────────┐       │   ┌──────────────┐   │
│  │management-api  │ │rule-compiler   │ │sse-gateway     │  ...  │   │admin-ui dist │   │
│  │:abc123f        │ │:abc123f        │ │:abc123f        │       │   │              │   │
│  └────────────────┘ └────────────────┘ └────────────────┘       │   └──────────────┘   │
└──────────────────────────────────────────┬──────────────────────────────────┬───────────┘
                                           │                                  │
                                           ▼                                  ▼
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                           Deploy to Staging (auto)                                        │
│                                                                                          │
│  ┌────────────────────────────────────────────┐    ┌────────────────────────────┐       │
│  │ ECS Cluster: feature-mgmt-staging          │    │ CloudFront (Admin UI)      │       │
│  │                                            │    │ → S3 staging bucket        │       │
│  │  ┌─────────────┐ ┌────────────────────┐   │    │ → Invalidate /index.html   │       │
│  │  │mgmt-api (2) │ │rule-compiler (1)   │   │    └────────────────────────────┘       │
│  │  ├─────────────┤ ├────────────────────┤   │                                          │
│  │  │snapshot-    │ │sse-gateway (2)     │   │                                         │
│  │  │builder (1)  │ ├────────────────────┤   │                                          │
│  │  ├─────────────┤ │relay-proxy (2)     │   │                                         │
│  │  └─────────────┘ └────────────────────┘   │                                          │
│  └────────────────────────────────────────────┘                                          │
└─────────────────────────────────────────────────────────────────┬───────────────────────┘
                                                                  │
                                                                  │ ✅ Staging verified
                                                                  │ 🔒 Manual Approval
                                                                  ▼
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                           Deploy to Production (manual approve)                           │
│                                                                                          │
│  ┌────────────────────────────────────────────┐    ┌────────────────────────────┐       │
│  │ ECS Cluster: feature-mgmt-prod             │    │ CloudFront (Admin UI prod) │       │
│  │                                            │    │ → S3 prod bucket           │       │
│  │  ┌─────────────┐  ┌────────────────────┐  │    └────────────────────────────┘       │
│  │  │mgmt-api (3) │  │rule-compiler (2)   │  │                                          │
│  │  │Rolling      │  │Rolling             │  │                                          │
│  │  ├─────────────┤  ├────────────────────┤  │                                        │
│  │  │snapshot-    │  │sse-gateway (3)     │  │                                         │
│  │  │builder (2)  │  │Blue/Green+CodeDeploy│  │                                        │
│  │  ├─────────────┤  ├────────────────────┤  │                                          │
│  │  │relay-proxy  │  │                    │  │                                          │
│  │  │(2-3/cluster)│  │                    │  │                                          │
│  │  └─────────────┘  └────────────────────┘  │                                          │
│  └────────────────────────────────────────────┘                                          │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

###  Deployment Strategy Summary

| Component        | AWS Service            |
| ---------------- | ---------------------- |
| Admin UI         | S3 + CloudFront        |
| Management API   | ECS Fargate + ALB      |
| Rule Compiler    | ECS Fargate            |
| Snapshot Builder | ECS Fargate            |
| SSE Gateway      | ECS + NLB + CodeDeploy |
| Relay Proxy      | ECS Fargate            |
| DB Schema        | RDS                    |

**ALB VS NLB:**
- ALB → APIs (HTTP traffic)
- NLB → Real-time services (SSE/WebSocket/TCP)

---
## 3. Core Components

### 3.1 Management API (Spring Boot)

The Management API is the single entry point for all configuration mutations. It enforces consistency, authorization, validation, and auditability.

**Responsibilities:**

- Feature CRUD and Rule CRUD with transactional integrity
- RBAC authorization (OAuth2/OIDC/SSO)
- Input validation (Bean Validation)
- Audit log recording (actor, before/after state, reason, ticket ID)
- Publish job orchestration (trigger compile → build → upload → notify)
- Debug Explain API for troubleshooting
### 3.1.1 Kafka-Based Publish job orchestration

The publish pipeline uses Kafka topics as a **choreography-based event chain**, replacing synchronous RPC calls between stages:

```
┌────────────┐   publish.requested   ┌───────────────┐   publish.compiled   ┌──────────────────┐   publish.completed   ┌─────────────┐
│Management  │ ───────────────────▶  │Rule Compiler  │ ───────────────────▶ │Snapshot Builder  │ ───────────────────▶  │SSE Gateway  │
│API         │                       │(consumer)     │                      │(consumer)        │                       │(consumer)   │
└────────────┘                       └───────────────┘                      └──────────────────┘                       └─────────────┘
                                            │                                      │                                        │
                                            ▼                                      ▼                                        ▼
                                     publish.dlq (on failure)              publish.dlq (on failure)                  Notify Relay/SDKs
```

**Kafka Topics:**

| Topic | Producer | Consumer | Payload (key fields) |
|-------|----------|----------|---------------------|
| `publish.requested` | Management API | Rule Compiler | `jobId`, `appId`, `env`, `requestedBy`, `flagKeys` |
| `publish.compiled` | Rule Compiler | Snapshot Builder | `jobId`, `appId`, `env`, `compiledIrUrl`, `checksum` |
| `publish.completed` | Snapshot Builder | SSE Gateway | `jobId`, `appId`, `env`, `snapshotVersion`, `fullUrl`, `deltaUrl`, `checksum` |
| `publish.failed` | Any stage | Alerting / Management API | `jobId`, `stage`, `error`, `retryable` |
| `publish.dlq` | Kafka (on max retries) | Ops team | Original message + error metadata |


**Kafka consumer config (per stage):**
```yaml
spring.kafka:
  consumer:
    group-id: rule-compiler-group     # or snapshot-builder-group
    auto-offset-reset: earliest
    enable-auto-commit: false          # Manual commit after processing
    max-poll-records: 1                # Process one job at a time
  producer:
    acks: all
    retries: 3
    idempotence: true                  # Exactly-once semantics
```

**Job status tracking:** The Management API also consumes `publish.compiled`, `publish.completed`, and `publish.failed` to update the `publish_jobs` table status (PENDING → COMPILING → BUILDING → SUCCEEDED / FAILED). This gives the Admin UI real-time pipeline visibility.


### 3.2 Feature DB (PostgreSQL)

Source of truth for all configuration. Serves only the control plane.

**Core Schema:**

```sql
CREATE TABLE features (
    id            BIGSERIAL PRIMARY KEY,
    flag_key      VARCHAR(255) UNIQUE NOT NULL,
    name          VARCHAR(255) NOT NULL,
    description   TEXT,
    type          VARCHAR(50) NOT NULL,          -- boolean | string | number | json
    default_value JSONB NOT NULL,
    enabled       BOOLEAN NOT NULL DEFAULT true,
    owner         VARCHAR(255),
    tags          JSONB DEFAULT '[]',
    release_id    VARCHAR(255),
    created_at    TIMESTAMP NOT NULL DEFAULT now(),
    updated_at    TIMESTAMP NOT NULL DEFAULT now()
);

CREATE TABLE feature_rules (
    id                  BIGSERIAL PRIMARY KEY,
    flag_key            VARCHAR(255) NOT NULL REFERENCES features(flag_key),
    rule_id             VARCHAR(100) NOT NULL,
    priority            INT NOT NULL,
    description         TEXT,
    serve_value         JSONB NOT NULL,
    rollout_percentage  INT CHECK (rollout_percentage BETWEEN 0 AND 100),
    salt                VARCHAR(255),
    enabled             BOOLEAN NOT NULL DEFAULT true,
    created_at          TIMESTAMP NOT NULL DEFAULT now(),
    updated_at          TIMESTAMP NOT NULL DEFAULT now(),
    UNIQUE (flag_key, rule_id)
);

CREATE TABLE rule_conditions (
    id        BIGSERIAL PRIMARY KEY,
    rule_id   BIGINT NOT NULL REFERENCES feature_rules(id) ON DELETE CASCADE,
    attribute VARCHAR(255) NOT NULL,
    operator  VARCHAR(50) NOT NULL,              -- eq, neq, in, not_in, gt, gte, lt, lte, contains, regex, semver_gte, semver_lte
    value     JSONB NOT NULL
);

CREATE TABLE audit_logs (
    id           BIGSERIAL PRIMARY KEY,
    actor        VARCHAR(255) NOT NULL,
    action       VARCHAR(100) NOT NULL,          -- FEATURE_CREATED, RULE_UPDATED, SNAPSHOT_PUBLISHED, ...
    target_type  VARCHAR(100),
    target_id    VARCHAR(255),
    before_value JSONB,
    after_value  JSONB,
    reason       TEXT,
    ticket_id    VARCHAR(255),
    release_id   VARCHAR(255),
    created_at   TIMESTAMP NOT NULL DEFAULT now()
);

CREATE TABLE publish_jobs (
    id               BIGSERIAL PRIMARY KEY,
    app_id           VARCHAR(255) NOT NULL,
    env              VARCHAR(50) NOT NULL,
    status           VARCHAR(50) NOT NULL DEFAULT 'PENDING',  -- PENDING, RUNNING, SUCCEEDED, FAILED
    requested_by     VARCHAR(255) NOT NULL,
    snapshot_version VARCHAR(100),
    error_message    TEXT,
    created_at       TIMESTAMP NOT NULL DEFAULT now(),
    updated_at       TIMESTAMP NOT NULL DEFAULT now()
);

CREATE TABLE snapshot_artifacts (
    id               BIGSERIAL PRIMARY KEY,
    app_id           VARCHAR(255) NOT NULL,
    env              VARCHAR(50) NOT NULL,
    snapshot_version VARCHAR(100) NOT NULL,
    artifact_type    VARCHAR(50) NOT NULL,        -- FULL, DELTA
    from_version     VARCHAR(100),
    url              TEXT NOT NULL,
    checksum         VARCHAR(255) NOT NULL,
    size_bytes       BIGINT,
    flag_count       INT,
    rule_count       INT,
    created_at       TIMESTAMP NOT NULL DEFAULT now()
);

CREATE INDEX idx_features_owner ON features(owner);
CREATE INDEX idx_features_release ON features(release_id);
CREATE INDEX idx_feature_rules_flag ON feature_rules(flag_key);
CREATE INDEX idx_audit_logs_target ON audit_logs(target_type, target_id);
CREATE INDEX idx_audit_logs_actor ON audit_logs(actor);
CREATE INDEX idx_audit_logs_time ON audit_logs(created_at);
CREATE INDEX idx_publish_jobs_app_env ON publish_jobs(app_id, env);
CREATE INDEX idx_snapshot_artifacts_version ON snapshot_artifacts(app_id, env, snapshot_version);
```

### 3.3 Rule Compiler (Rust)

Transforms human-readable rules into an optimized intermediate representation (IR) that SDKs can evaluate efficiently.

**Why a separate compilation step?**

- Pre-parses semver strings, pre-compiles regexes, pre-normalizes operators
- Reduces per-evaluation CPU cost in SDKs to simple comparisons
- Guarantees deterministic, cross-language-consistent output
- Detects rule conflicts and validation errors before publish

**Compilation example:**

*Input (human-readable):*
```json
{
  "attribute": "appVersion",
  "operator": ">=",
  "value": "5.3.0"
}
```

*Output (compiled IR):*
```json
{
  "attrId": 2,
  "opCode": "SEMVER_GTE",
  "value": [5, 3, 0]
}
```

**Full compilation example:**

*Before:*
```json
{
  "flagKey": "checkout_new_ui",
  "rules": [{
    "conditions": [
      { "attribute": "region", "operator": "in", "values": ["CN", "SG"] },
      { "attribute": "appVersion", "operator": ">=", "value": "5.3.0" }
    ],
    "serve": true,
    "rollout": 30
  }]
}
```

*After:*
```json
{
  "flagKey": "checkout_new_ui",
  "defaultValue": false,
  "compiledRules": [{
    "ruleId": "rule_001",
    "priority": 1,
    "conditions": [
      { "attrId": 1, "opCode": "IN", "values": ["CN", "SG"] },
      { "attrId": 2, "opCode": "SEMVER_GTE", "value": [5, 3, 0] }
    ],
    "rollout": {
      "threshold": 30000,
      "bucketSize": 100000,
      "salt": "checkout_new_ui"
    },
    "serveValue": true
  }]
}
```

The compiler also emits **explain metadata** per rule (human-readable descriptions, condition summaries) for the explainability system.

### 3.4 Snapshot Builder (Rust)

Converts compiled flags into immutable, distributable snapshot files. This is the bridge from control plane to data plane.

**Responsibilities:**
1. Read compiled flags from Feature DB
2. Generate **server snapshot** (full rules — for backend SDKs via Relay Proxy)
3. Generate **client snapshot** (stripped rules — for web/mobile SDKs via public CDN)
4. Generate **delta snapshot** (diff from previous version, for both server and client)
5. Compute SHA-256 checksums (delta, result snapshot)
6. Compress with gzip/brotli
7. Upload to Object Storage
8. Update `latest.json`
9. Record snapshot metadata

#### Server Snapshot vs Client Snapshot

The Snapshot Builder produces **two variants** of each snapshot to address the different security and distribution requirements of backend vs. client-side SDKs:

| Property | Server Snapshot | Client Snapshot |
|----------|----------------|----------------|
| Contains full rule conditions | Yes | No — conditions are pre-evaluated into **segment buckets** |
| Contains rollout salt/threshold | Yes | No — rollout decisions are baked in per segment |
| Contains explain metadata | Yes | Optional (debug builds only) |
| Distribution channel | Relay Proxy (internal network) | Public CDN |
| Who consumes it | Backend API SDKs (Java, Go, Node.js server) | Web browser SDKs, Mobile SDKs |
| Security exposure | Low (internal only) | High (shipped to end-user devices) |

**Client snapshot structure** (simplified, pre-evaluated):
```json
{
  "appId": "checkout-mobile",
  "env": "prod",
  "snapshotVersion": "2026.05.12-001",
  "schemaVersion": 3,
  "snapshotType": "client",
  "flags": {
    "checkout_new_ui": {
      "type": "boolean",
      "defaultValue": false,
      "enabled": true,
      "compiledRules": [
        {
          "ruleId": "rule_001",
          "priority": 1,
          "conditions": [
            { "attrId": 1, "opCode": "IN", "values": ["CN", "SG"] }
          ],
          "rollout": { "threshold": 30000, "bucketSize": 100000, "salt": "checkout_new_ui" },
          "serveValue": true
        }
      ],
      "releaseId": "checkout-v2"
    }
  }
}
```

Note: Sensitive conditions (e.g., internal employee email regex, internal IP ranges) are excluded from the client snapshot. Flags that should never be visible to end-user devices can be marked `serverOnly: true` and omitted entirely.

**Full (server) snapshot structure:**
```json
{
  "appId": "checkout-web",
  "env": "prod",
  "snapshotVersion": "2026.05.12-001",
  "createdAt": "2026-05-12T10:00:00Z",
  "schemaVersion": 3,
  "flags": {
    "checkout_new_ui": {
      "type": "boolean",
      "defaultValue": false,
      "enabled": true,
      "compiledRules": [ /* ... */ ],
      "releaseId": "checkout-v2"
    }
  },
  "metadata": {
    "flagCount": 342,
    "ruleCount": 1205,
    "attributeDictionary": { "1": "region", "2": "appVersion", "3": "userSegment" }
  }
}
```

**Delta snapshot structure:**
```json
{
  "appId": "checkout-web",
  "env": "prod",
  "fromVersion": "2026.05.12-001",
  "toVersion": "2026.05.12-002",
  "schemaVersion": 3,
  "operations": [
    { "op": "upsert", "flagKey": "checkout_new_ui", "value": { /* compiled flag */ } },
    { "op": "delete", "flagKey": "legacy_checkout_ui" }
  ],
  "checksum": {
    "delta": "sha256:abc123...",
    "resultSnapshot": "sha256:def456..."
  }
}
```

### 3.5 Object Storage / CDN

Hosts immutable snapshot artifacts for global distribution.

**File layout:**
```
s3://feature-config/
  {env}/
    {appId}/
      latest.json                              ← shared pointer
      {snapshotVersion}/
        server-full.json.gz                    ← full rules (Relay Proxy only, not public CDN)
        server-delta-from-{prev}.json.gz
        client-full.json.gz                    ← stripped rules (public CDN)
        client-delta-from-{prev}.json.gz
        metadata.json
        diff-report.json
```

CDN distribution rules:
- `server-*.json.gz` files are **not exposed** on the public CDN; they are fetched by Relay Proxy directly from Object Storage or via an authenticated CDN path.
- `client-*.json.gz` files are served on the public CDN for web and mobile SDKs.

**Cache headers:**

| File | Cache-Control | Rationale |
|------|--------------|-----------|
| `full.json.gz` | `public, max-age=31536000, immutable` | Versioned, never changes |
| `delta-*.json.gz` | `public, max-age=31536000, immutable` | Versioned, never changes |
| `latest.json` | `no-cache` with `ETag` | Must always reflect current version |

### 3.6 SSE Gateway

Pushes lightweight change notifications to downstream consumers. Does **not** push full configuration — only event metadata.

**SSE event format:**
```
id: 2026.05.12-002
event: snapshot_published
data: {"appId":"checkout-web","env":"prod","fromVersion":"2026.05.12-001","toVersion":"2026.05.12-002","deltaUrl":"...","fullUrl":"...","checksum":"sha256:xxx"}
```

**Features:** heartbeat (every 30s), `Last-Event-ID` for reconnection, connection authentication, rate limiting.

**Design principle:**
- SSE = fast notification
- Polling = reliable fallback
- CDN = payload delivery

### 3.7 Relay Proxy

A configuration relay layer deployed close to business clusters (e.g., per Kubernetes cluster, 2–3 replicas).

**Value proposition:**
- Reduces connections to central services from thousands of SDK instances to a handful of Relay Proxies
- Enables air-gapped / private network deployments
- Provides local cache for resilience during upstream outages
- Unifies authentication for internal SDKs

**Key invariant:** Relay Proxy does **not** perform remote per-request evaluation. It only syncs, caches, and distributes snapshots.

---

## 4. Caching Strategy

### 4.1 Multi-Tier Cache Architecture

The system employs **six cache tiers** to minimise cost, latency, and blast radius:

```
Tier 1: CDN Edge Cache           (global, ~ms access, cost: pennies per GB)
    ↓
Tier 2: Relay Proxy Memory       (cluster-local, ~μs access, moka / DashMap cache)
    ↓
Tier 3: Relay Proxy Disk         (cluster-local, ~ms access, survives restart)
    ↓
Tier 4: SDK In-Memory Snapshot   (process-local, ~ns access, ArcSwap/AtomicReference)
    ↓
Tier 5: SDK Disk Cache           (process-local, ~ms access, cold-start fallback)
    ↓
Tier 6: Code Default Value       (hardcoded, zero-cost, ultimate fallback)
```

### 4.2 Cache Update Flow

```
Snapshot Builder uploads to Object Storage
  → CDN invalidation (or short TTL on latest.json)
  → SSE Gateway pushes snapshot_published event
  → Relay Proxy receives event, fetches delta/full from CDN
  → Relay Proxy atomically swaps in-memory snapshot
  → Relay Proxy pushes downstream SSE event to SDKs
  → SDK receives event, fetches delta/full from Relay Proxy
  → SDK atomically swaps in-memory snapshot (ArcSwap / AtomicReference)
```

### 4.3 Cache Invalidation Strategy

| Cache Tier | Invalidation Trigger | Mechanism |
|-----------|---------------------|-----------|
| CDN | Snapshot publish | `latest.json` with `no-cache` + `ETag`; versioned files are immutable |
| Relay Memory | SSE event or polling | SSE `snapshot_published` → pull new version; polling fallback every 30s–5min |
| Relay Disk | Same as memory | Write-through on snapshot update |
| SDK Memory | Downstream SSE or polling | Atomic swap (lock-free); no evaluation downtime during update |
| SDK Disk | Same as memory | Write-through; read on cold start |

### 4.4 Delta Snapshots for Cost Efficiency

When a platform manages thousands of flags but only a few change per publish:

- **Full snapshot** may be 500KB–2MB compressed
- **Delta snapshot** for a single-flag change may be 1–5KB

Delta rules:
1. Delta contains `fromVersion` and `toVersion`
2. SDK applies delta only if `localVersion == fromVersion`
3. After applying delta, SDK verifies the resulting snapshot checksum against `checksum.resultSnapshot`
4. If verification fails or `fromVersion` mismatches → fallback to full snapshot download

This reduces CDN egress costs and SDK parse time proportionally to the change size rather than the total flag count.

### 4.5 Cost Analysis at Scale

| Scenario | Without Caching | With Multi-Tier Cache |
|----------|----------------|----------------------|
| 10,000 SDK instances polling every 30s | 20,000 req/min to origin | 3 Relay Proxies poll origin; SDKs read locally |
| Snapshot size 1MB × 10,000 instances | 10 GB/update | Delta ~5KB × 3 relays = 15KB to origin; SDKs pull from relay |
| Cold start (1,000 pods restarting) | 1,000 concurrent CDN fetches | SDK reads disk cache; no external requests |

### 4.6 Snapshot Atomicity

SDK memory updates use lock-free atomic swap:
- **Java:** `AtomicReference<Snapshot>` swap
- **Rust:** `arc_swap::ArcSwap<Snapshot>`
- **Go:** `atomic.Value`
- **Node.js:** single-threaded reference replacement
- **iOS (Swift):** `os_unfair_lock` or actor-based swap
- **Android (Kotlin):** `AtomicReference<Snapshot>` or `StateFlow`
- **Browser (JS):** single-threaded reference replacement; `postMessage` for Web Worker sync

This ensures evaluations never block on snapshot updates and never see a partially-applied state.

### 4.7 Caching by Client Type

Different client types use different subsets of the cache tiers:

| Cache Tier | Backend API | Web Portal | Mobile Client |
|-----------|------------|------------|---------------|
| CDN Edge | — (via Relay) | Yes (primary source) | Yes (primary source) |
| Relay Proxy Memory | Yes | — | — |
| Relay Proxy Disk | Yes | — | — |
| SDK In-Memory | `AtomicReference` | JS variable / WASM heap | Native memory |
| SDK Persistent Cache | File system | `localStorage` / IndexedDB | App sandbox (SQLite / file) |
| Bundled Snapshot | — | SSR bootstrap blob | Binary-embedded JSON |
| Code Default | Yes | Yes | Yes |

**Web-specific caching notes:**
- `localStorage` is limited to ~5MB; sufficient for typical snapshots (<500KB)
- IndexedDB is preferred for large snapshots or when structured query is needed
- SSR bootstrap injects the snapshot into the HTML payload, eliminating the first-render fetch entirely
- Tab-level deduplication: if multiple tabs are open, only one polls; others receive updates via `BroadcastChannel`

**Mobile-specific caching notes:**
- Bundled snapshot baked into app binary guarantees evaluation on first-ever launch with zero network
- Disk cache in app sandbox survives app restarts but not uninstall
- Background fetch (iOS Background App Refresh / Android WorkManager) keeps cache warm even when app is not in foreground
- Cache is shared across app extensions (iOS App Groups / Android ContentProvider) if needed

---

## 5. Client SDK Design

### 5.1 Design Principles

1. **Local evaluation only** — zero network calls during `isEnabled()` / `evaluate()`
2. **Consistent behaviour** — all SDKs use identical evaluation semantics (same hash, same condition matching, same precedence)
3. **Minimal integration surface** — 3 lines to integrate, 1 function to call
4. **Graceful degradation** — always returns a value, never throws on missing flags

### 5.2 SDK Integration Example

**Java / Spring Boot:**
```java
@Autowired
private FeatureClient featureClient;

// Simple boolean check
boolean enabled = featureClient.isEnabled("checkout_new_ui", context);

// Typed value
String variant = featureClient.getString("checkout_theme", "default", context);

// With explain (debug mode)
EvaluationResult result = featureClient.evaluate("checkout_new_ui", context, Explain.ENABLED);
```

**Node.js:**
```javascript
const client = new FeatureClient({ relayUrl: 'http://relay:8080', appId: 'checkout-web', env: 'prod' });
await client.initialize();

const enabled = client.isEnabled('checkout_new_ui', { userId: 'u123', region: 'CN' });
```

**Python:**
```python
client = FeatureClient(relay_url="http://relay:8080", app_id="checkout-web", env="prod")
client.initialize()

enabled = client.is_enabled("checkout_new_ui", {"user_id": "u123", "region": "CN"})
```

### 5.3 SDK Lifecycle

```
1. Initialize
   → Read disk cache (if exists) → load into memory immediately
   → Fetch latest.json from Relay Proxy
   → Download full snapshot → atomic swap into memory
   → Write snapshot to disk cache

2. Steady State
   → Subscribe to downstream SSE from Relay Proxy
   → On snapshot_published event → fetch delta (or full if delta unavailable)
   → Atomic swap in-memory snapshot
   → Low-frequency polling as fallback (configurable: 30s–5min)

3. Evaluation
   → HashMap lookup by flagKey → O(1)
   → Condition matching → iterate compiled rules by priority
   → Rollout hash → deterministic bucket assignment
   → Return value + reason

4. Shutdown
   → Flush current snapshot to disk cache
   → Close SSE connection
```

### 5.4 Evaluation Algorithm

```
evaluate(flagKey, context, explain=false):
  flag = snapshot.flags[flagKey]

  if flag is null         → return (defaultValue, FLAG_NOT_FOUND)
  if flag.enabled is false → return (flag.defaultValue, FLAG_DISABLED)

  for rule in flag.compiledRules (sorted by priority):
    if not matchConditions(rule.conditions, context):
      continue  → reason: CONDITION_NOT_MATCH
    if not matchRollout(rule.rollout, context.userId):
      continue  → reason: ROLLOUT_NOT_MATCH
    return (rule.serveValue, RULE_MATCH, ruleId, matchDetails)

  return (flag.defaultValue, DEFAULT)
```

**Condition matching operators:**

| Operator | OpCode | Example |
|----------|--------|---------|
| equals | `EQ` | `region == "CN"` |
| not equals | `NEQ` | `region != "US"` |
| in | `IN` | `region in ["CN", "SG"]` |
| not in | `NOT_IN` | `region not_in ["US"]` |
| greater than | `GT` | `age > 18` |
| greater or equal | `GTE` | `age >= 18` |
| less than | `LT` | `age < 65` |
| less or equal | `LTE` | `age <= 65` |
| contains | `CONTAINS` | `tags contains "beta"` |
| regex | `REGEX` | `email matches ".*@corp.com"` |
| semver >= | `SEMVER_GTE` | `appVersion >= 5.3.0` |
| semver <= | `SEMVER_LTE` | `appVersion <= 6.0.0` |

**Rollout hashing:**
```
bucket = murmur3_hash(salt + ":" + userId) % 100000
matched = bucket < threshold  // threshold = percentage × 1000
```

Using murmur3 ensures uniform distribution. The salt (typically the flagKey) ensures independent rollout distributions across flags.

### 5.5 Cross-Language Consistency

**Short-term (MVP):** Each SDK implements the evaluation algorithm natively. A shared **Golden Test Suite** (JSON fixture file with inputs and expected outputs) validates consistency across all SDKs in CI.

**Long-term (Phase 9):** Extract a **Rust Evaluator Core** shared via FFI:

```
                ┌─────────────────────┐
                │ Rust Evaluator Core  │
                │ (feature-eval-core)  │
                └─────────────────────┘
                   /     |      |     \
              JNI     N-API   PyO3   Native
               ↓        ↓      ↓       ↓
            Java SDK  Node SDK  Python  Rust SDK
```

Binding options per language:

| Language | Binding | Notes |
|----------|---------|-------|
| Java | JNI / JNA | Mature, well-understood |
| Node.js | N-API or WASM | WASM for browser SDK reuse |
| Python | PyO3 | Zero-copy, Pythonic API |
| Go | CGO or subprocess | CGO has overhead; consider native Go with golden tests |
| .NET | P/Invoke | NativeAOT-compatible |
| Mobile | Static library or WASM | Consider package-size tradeoffs |

### 5.6 SDK Strategies by Client Type

Different client types have fundamentally different constraints. The SDK is designed with three distinct modes:

#### 5.6.1 Backend API SDK (Java / Go / Node.js / Python / .NET)

**Deployment:** Runs inside backend microservices, behind Relay Proxy on internal network.

**Characteristics:**
- Uses **server snapshot** with full targeting rules
- Connects to Relay Proxy (never directly to public CDN)
- Supports SSE streaming for near-real-time updates
- Full local evaluation with all operators (semver, regex, etc.)
- Long-lived process — snapshot stays in memory for the process lifetime
- Disk cache for cold-start resilience

**Integration (Java/Spring Boot):**
```java
@Autowired
private FeatureClient featureClient;

boolean enabled = featureClient.isEnabled("checkout_new_ui", EvaluationContext.builder()
    .userId("user_123")
    .attribute("region", "CN")
    .attribute("appVersion", "5.4.1")
    .build());
```

**Configuration:**
```yaml
feature-management:
  relay-url: http://relay-proxy:8080
  app-id: checkout-api
  env: prod
  sdk-key: server-sdk-xxxx
  snapshot-type: server                    # Full rules
  polling-interval: 60s
  sse-enabled: true
  disk-cache-path: /tmp/feature-cache
  connect-timeout: 5s
  default-on-error: false
```

#### 5.6.2 Web Portal SDK (Browser JavaScript / WASM)

**Deployment:** Runs in end-user browsers. Downloaded as part of the web application bundle.

**Characteristics:**
- Uses **client snapshot** (stripped of sensitive rules)
- Fetches snapshot directly from **public CDN** (no Relay Proxy needed)
- Evaluation via **WASM** (Rust Evaluator Core compiled to WebAssembly) or pure JavaScript
- SSE supported in modern browsers via `EventSource` API
- Bundle size is critical — WASM evaluator ~50KB gzipped; pure JS evaluator ~15KB gzipped
- No disk cache — uses `localStorage` or `IndexedDB` for persistence across page loads
- Must handle initial page render before snapshot is available (bootstrap problem)

**Bootstrap strategy:**
1. **SSR pre-injection (recommended):** Server-side rendering injects flag values into the initial HTML as a JSON blob. The browser SDK hydrates from this immediately — zero flicker, zero network wait.
2. **Async fetch:** SDK fetches client snapshot from CDN on page load. Use code defaults during the ~100ms fetch window.
3. **Cached replay:** SDK reads `localStorage` first, then fetches latest in background. Flags are available instantly from prior session.

**Integration (React):**
```tsx
import { FeatureProvider, useFeature } from '@mycompany/feature-sdk-react';

// Provider with bootstrap (from SSR-injected data)
function App() {
  return (
    <FeatureProvider
      cdnUrl="https://cdn.example.com/flags"
      appId="checkout-web"
      env="prod"
      sdkKey="client-sdk-xxxx"
      bootstrap={window.__FEATURE_FLAGS__}   // SSR-injected
    >
      <CheckoutPage />
    </FeatureProvider>
  );
}

// Usage in component
function CheckoutPage() {
  const newUiEnabled = useFeature('checkout_new_ui', { userId: user.id, region: user.region });
  return newUiEnabled ? <NewCheckout /> : <LegacyCheckout />;
}
```

**Configuration:**
```javascript
const client = new FeatureClient({
  cdnUrl: 'https://cdn.example.com/flags',   // Public CDN, not Relay Proxy
  appId: 'checkout-web',
  env: 'prod',
  sdkKey: 'client-sdk-xxxx',                  // Client SDK key (limited scope)
  snapshotType: 'client',                      // Stripped snapshot
  pollingInterval: 300_000,                    // 5min polling (lower frequency for browsers)
  sseEnabled: true,                            // EventSource API
  storage: 'localStorage',                     // Persistence between page loads
  bootstrap: window.__FEATURE_FLAGS__,         // Optional: SSR-injected values
});
```

**Web-specific considerations:**

| Concern | Approach |
|---------|----------|
| Bundle size | WASM evaluator ~50KB gz; JS evaluator ~15KB gz; tree-shakeable |
| First render flicker | SSR bootstrap or localStorage cache |
| CDN latency | Snapshot served from edge; `Cache-Control: no-cache` with `ETag` on `latest.json` |
| No SSE in some environments | `EventSource` polyfill or polling-only mode |
| Tab visibility | Pause polling when tab is hidden (`document.visibilityState`) |
| Memory | Single snapshot in memory; typically <100KB parsed |

#### 5.6.3 Mobile Client SDK (iOS / Android / React Native / Flutter)

**Deployment:** Runs on end-user devices. Distributed via App Store / Google Play.

**Characteristics:**
- Uses **client snapshot** (stripped of sensitive rules)
- Fetches snapshot from **public CDN** directly
- Evaluation via **native code** (Swift/Kotlin) or **Rust FFI** (static library linked into app)
- Must be **offline-capable** — devices may have no network for extended periods
- Battery and data usage aware — avoids aggressive polling
- App lifecycle aware — foreground vs background state
- App binary includes a **bundled snapshot** baked in at build time for guaranteed cold-start

**Mobile SDK lifecycle:**
```
1. App Launch (cold start)
   → Load bundled snapshot (compiled into app binary)
   → Read disk cache (if newer than bundled)
   → Evaluation available immediately (zero network dependency)
   → Background: fetch latest.json from CDN
   → If newer version → download client snapshot → atomic swap → persist to disk

2. App Foreground
   → Check snapshot age
   → If stale (> configured threshold) → fetch latest.json
   → Optional: reconnect SSE for real-time updates

3. App Background
   → Disconnect SSE to save battery
   → No polling
   → Optional: iOS Background App Refresh / Android WorkManager for periodic sync

4. Offline Mode
   → Continue evaluation from disk-cached or bundled snapshot
   → All evaluations succeed (never fails due to network)
   → sdk_snapshot_age_seconds metric tracks staleness

5. App Update
   → New app binary includes fresh bundled snapshot
   → Ensures all users get a recent baseline even if they haven't opened the app in months
```

**Integration (iOS / Swift):**
```swift
import FeatureSDK

let client = FeatureClient(
    cdnUrl: "https://cdn.example.com/flags",
    appId: "checkout-ios",
    env: "prod",
    sdkKey: "client-sdk-xxxx"
)
client.initialize()  // Loads bundled → disk cache → CDN (async)

let context = EvaluationContext(
    userId: user.id,
    attributes: ["region": "CN", "appVersion": "5.4.1", "platform": "ios"]
)
let enabled = client.isEnabled("checkout_new_ui", context: context)
```

**Integration (Android / Kotlin):**
```kotlin
import com.mycompany.featuresdk.FeatureClient

val client = FeatureClient.Builder()
    .cdnUrl("https://cdn.example.com/flags")
    .appId("checkout-android")
    .env("prod")
    .sdkKey("client-sdk-xxxx")
    .build()
client.initialize(applicationContext)  // Uses app storage for disk cache

val context = EvaluationContext.builder()
    .userId(user.id)
    .attribute("region", "CN")
    .attribute("appVersion", "5.4.1")
    .attribute("platform", "android")
    .build()
val enabled = client.isEnabled("checkout_new_ui", context)
```

**Mobile-specific considerations:**

| Concern | Approach |
|---------|----------|
| Offline first | Bundled snapshot in app binary; disk cache; never requires network for evaluation |
| Battery | SSE only in foreground; no background polling (use OS-level background fetch) |
| Data usage | Delta snapshots reduce transfer; polling interval ≥ 5min; respect low-data mode |
| App Store lag | Bundled snapshot ensures a baseline; CDN fetch gets latest on first launch |
| SDK binary size | Native evaluator ~200KB; Rust FFI static lib ~500KB; WASM ~100KB |
| Thread safety | Atomic snapshot swap; evaluation is lock-free and safe from any thread |
| Startup latency | Bundled snapshot → evaluation available in <5ms; no network wait |

### 5.7 SDK Feature Matrix

| Capability | Backend SDK | Web SDK | Mobile SDK |
|-----------|------------|---------|------------|
| Snapshot type | Server (full) | Client (stripped) | Client (stripped) |
| Distribution channel | Relay Proxy | Public CDN | Public CDN |
| SSE streaming | Yes | Yes (EventSource) | Foreground only |
| Polling fallback | 30s–5min | 5min | 5–30min |
| Disk cache | File system | localStorage / IndexedDB | App sandbox storage |
| Bundled snapshot | No (disk cache is sufficient) | Optional (SSR bootstrap) | Yes (in app binary) |
| Offline evaluation | Yes (in-memory) | Yes (localStorage) | Yes (bundled + disk cache) |
| Evaluation engine | Native (Java/Go/etc.) | WASM or JS | Native or Rust FFI |
| Explain support | Full | Limited (debug builds) | Limited (debug builds) |
| Heartbeat reporting | Yes | Optional | Optional |
| Binary size impact | ~1MB (JAR) | 15–50KB (gz) | 200–500KB |

### 5.8 SDK Configuration

**Backend SDK (YAML):**
```yaml
feature-management:
  relay-url: http://relay-proxy:8080
  app-id: checkout-api
  env: prod
  sdk-key: server-sdk-xxxx
  snapshot-type: server
  polling-interval: 60s
  sse-enabled: true
  disk-cache-path: /tmp/feature-cache
  connect-timeout: 5s
  default-on-error: false
```

**Web SDK (JavaScript):**
```javascript
{
  cdnUrl: 'https://cdn.example.com/flags',
  appId: 'checkout-web',
  env: 'prod',
  sdkKey: 'client-sdk-xxxx',
  snapshotType: 'client',
  pollingInterval: 300000,
  sseEnabled: true,
  storage: 'localStorage',
  bootstrap: window.__FEATURE_FLAGS__
}
```

**Mobile SDK (programmatic):**
```
cdnUrl: https://cdn.example.com/flags
appId: checkout-ios
env: prod
sdkKey: client-sdk-xxxx
snapshotType: client
pollingInterval: 600000   (10 min)
sseEnabled: true          (foreground only)
offlineEnabled: true
bundledSnapshotPath: bundled_flags.json
```

---

## 6. API Design

### 6.1 Management API

All management operations are authenticated (OAuth2/OIDC) and authorized (RBAC).

#### 6.1.1 Feature CRUD

**Create Feature**
```http
POST /api/v1/features
Content-Type: application/json
Authorization: Bearer {token}

{
  "key": "checkout_new_ui",
  "name": "Checkout New UI",
  "description": "New checkout page experience",
  "type": "boolean",
  "defaultValue": false,
  "owner": "checkout-team",
  "tags": ["checkout", "ui"],
  "releaseId": "checkout-v2"
}
```

Response `201 Created`:
```json
{
  "id": 1042,
  "key": "checkout_new_ui",
  "name": "Checkout New UI",
  "type": "boolean",
  "defaultValue": false,
  "enabled": true,
  "owner": "checkout-team",
  "tags": ["checkout", "ui"],
  "releaseId": "checkout-v2",
  "createdAt": "2026-05-12T10:00:00Z",
  "updatedAt": "2026-05-12T10:00:00Z"
}
```

**List Features**
```http
GET /api/v1/features?page=0&size=20&owner=checkout-team&tag=ui&env=prod&enabled=true
```

**Get Feature**
```http
GET /api/v1/features/{flagKey}
```

**Update Feature**
```http
PATCH /api/v1/features/{flagKey}
{
  "enabled": false,
  "reason": "Rollback due to checkout error spike",
  "ticketId": "INC-4523"
}
```

**Delete Feature**
```http
DELETE /api/v1/features/{flagKey}
```

#### 6.1.2 Rule Management

**Set Rules**
```http
PUT /api/v1/features/{flagKey}/rules
Content-Type: application/json

{
  "rules": [
    {
      "ruleId": "rule_001",
      "priority": 1,
      "description": "CN/SG beta users with app >= 5.3.0",
      "conditions": [
        { "attribute": "region", "operator": "in", "values": ["CN", "SG"] },
        { "attribute": "appVersion", "operator": ">=", "value": "5.3.0" },
        { "attribute": "userSegment", "operator": "eq", "value": "beta" }
      ],
      "serve": true,
      "rollout": { "percentage": 30, "salt": "checkout_new_ui" }
    }
  ]
}
```

**Get Rules**
```http
GET /api/v1/features/{flagKey}/rules
```

#### 6.1.3 Publish / Snapshot

**Trigger Snapshot Build**
```http
POST /api/v1/snapshots/build
{
  "appId": "checkout-web",
  "env": "prod",
  "reason": "Release checkout-v2 phase 1",
  "ticketId": "REL-789"
}
```

Response `202 Accepted`:
```json
{
  "jobId": 5012,
  "status": "PENDING",
  "appId": "checkout-web",
  "env": "prod",
  "requestedBy": "alice@example.com",
  "createdAt": "2026-05-12T10:05:00Z"
}
```

**Get Publish Job Status**
```http
GET /api/v1/publish-jobs/{jobId}
```

Response:
```json
{
  "jobId": 5012,
  "status": "SUCCEEDED",
  "snapshotVersion": "2026.05.12-001",
  "artifacts": {
    "fullUrl": "https://cdn.example.com/flags/prod/checkout-web/2026.05.12-001/full.json.gz",
    "deltaUrl": "https://cdn.example.com/flags/prod/checkout-web/2026.05.12-001/delta-from-2026.05.11-002.json.gz",
    "fullChecksum": "sha256:abc...",
    "deltaChecksum": "sha256:def..."
  },
  "metadata": {
    "flagCount": 342,
    "ruleCount": 1205,
    "sizeBytes": 524288
  }
}
```

**List Snapshot History**
```http
GET /api/v1/snapshots?appId=checkout-web&env=prod&limit=20
```

**Rollback to Previous Snapshot**
```http
POST /api/v1/snapshots/rollback
{
  "appId": "checkout-web",
  "env": "prod",
  "targetVersion": "2026.05.11-002",
  "reason": "Checkout error rate spike after publish",
  "ticketId": "INC-4523"
}
```

#### 6.1.4 Audit Log

**Query Audit Logs**
```http
GET /api/v1/audit-logs?targetType=FEATURE&targetId=checkout_new_ui&from=2026-05-01&to=2026-05-12&actor=alice
```

### 6.2 Evaluation API — Server SDKs (SDK ↔ Relay Proxy)

These APIs are consumed by **backend API SDKs** via Relay Proxy on the internal network. They serve **server snapshots** with full targeting rules.

#### 6.2.1 Latest Version

```http
GET /v1/apps/{appId}/envs/{env}/latest
Authorization: Bearer {serverSdkKey}
If-None-Match: "2026.05.12-001"
```

Response `200 OK` (or `304 Not Modified`):
```json
{
  "appId": "checkout-api",
  "env": "prod",
  "latestVersion": "2026.05.12-002",
  "schemaVersion": 3,
  "snapshotType": "server",
  "checksum": "sha256:abc...",
  "hasDelta": true,
  "deltaFromVersion": "2026.05.12-001"
}
```

#### 6.2.2 Full Snapshot Download

```http
GET /v1/apps/{appId}/envs/{env}/snapshots/{version}/full
Authorization: Bearer {serverSdkKey}
Accept-Encoding: gzip
```

#### 6.2.3 Delta Snapshot Download

```http
GET /v1/apps/{appId}/envs/{env}/snapshots/{toVersion}/delta?fromVersion={fromVersion}
Authorization: Bearer {serverSdkKey}
Accept-Encoding: gzip
```

#### 6.2.4 Downstream SSE Stream

```http
GET /v1/stream?appId=checkout-api&env=prod
Authorization: Bearer {serverSdkKey}
Last-Event-ID: 2026.05.12-001
Accept: text/event-stream
```

Events:
```
id: 2026.05.12-002
event: snapshot_published
data: {"appId":"checkout-api","env":"prod","toVersion":"2026.05.12-002","fromVersion":"2026.05.12-001","deltaUrl":"...","fullUrl":"...","checksum":"sha256:..."}

: heartbeat
```

### 6.3 Evaluation API — Client SDKs (Web & Mobile ↔ CDN)

These APIs are consumed by **web portal and mobile SDKs** directly from the public CDN. They serve **client snapshots** with stripped/simplified rules.

#### 6.3.1 Latest Version (CDN)

```http
GET https://cdn.example.com/flags/{env}/{appId}/latest.json
If-None-Match: "2026.05.12-001"
```

Response `200 OK` (or `304 Not Modified`):
```json
{
  "appId": "checkout-web",
  "env": "prod",
  "latestVersion": "2026.05.12-002",
  "schemaVersion": 3,
  "snapshotType": "client",
  "checksum": "sha256:xyz...",
  "clientFullUrl": "/flags/prod/checkout-web/2026.05.12-002/client-full.json.gz",
  "clientDeltaUrl": "/flags/prod/checkout-web/2026.05.12-002/client-delta-from-2026.05.12-001.json.gz",
  "hasDelta": true,
  "deltaFromVersion": "2026.05.12-001"
}
```

#### 6.3.2 Client Full Snapshot (CDN)

```http
GET https://cdn.example.com/flags/{env}/{appId}/{version}/client-full.json.gz
Accept-Encoding: gzip
```

#### 6.3.3 Client Delta Snapshot (CDN)

```http
GET https://cdn.example.com/flags/{env}/{appId}/{version}/client-delta-from-{fromVersion}.json.gz
Accept-Encoding: gzip
```

#### 6.3.4 Client SSE Stream

Web and mobile SDKs can optionally subscribe to a **public SSE endpoint** for real-time updates:

```http
GET https://sse.example.com/v1/client-stream?appId=checkout-web&env=prod
Authorization: Bearer {clientSdkKey}
Last-Event-ID: 2026.05.12-001
Accept: text/event-stream
```

The client SSE stream uses a separate, publicly-accessible SSE Gateway that only emits version metadata (no rule content). The SDK then fetches the client snapshot from CDN.

#### 6.3.5 SSR Bootstrap API

For server-side rendered web portals, the SSR server (which has access to Relay Proxy) can fetch a pre-evaluated flag set to embed in the HTML:

```http
GET /v1/apps/{appId}/envs/{env}/bootstrap?context.region=CN&context.platform=web
Authorization: Bearer {serverSdkKey}
```

Response:
```json
{
  "flags": {
    "checkout_new_ui": { "value": true, "reason": "RULE_MATCH" },
    "dark_mode": { "value": false, "reason": "DEFAULT" }
  },
  "snapshotVersion": "2026.05.12-002",
  "evaluatedAt": "2026-05-12T10:15:00Z"
}
```

This is embedded into the HTML as `<script>window.__FEATURE_FLAGS__ = {...}</script>` so the browser SDK hydrates instantly.

### 6.4 Auxiliary APIs

#### 6.4.1 Debug Evaluate

Server-side evaluation for debugging (admin-only).

```http
POST /api/v1/debug/evaluate
Authorization: Bearer {token}

{
  "flagKey": "checkout_new_ui",
  "appId": "checkout-web",
  "env": "prod",
  "context": {
    "userId": "user_123",
    "region": "CN",
    "appVersion": "5.4.1",
    "userSegment": "beta"
  }
}
```

Response (see [§8 Explainability](#8-explainability) for full structure).

#### 6.4.2 Diff Report

```http
GET /api/v1/snapshots/{version}/diff?appId=checkout-web&env=prod
```

Response:
```json
{
  "fromVersion": "2026.05.12-001",
  "toVersion": "2026.05.12-002",
  "changes": [
    { "flagKey": "checkout_new_ui", "changeType": "MODIFIED", "fields": ["rules[0].rollout.percentage"] },
    { "flagKey": "legacy_checkout_ui", "changeType": "DELETED" }
  ]
}
```

#### 6.4.3 SDK Heartbeat

SDKs periodically report their state for fleet-wide visibility.

```http
POST /v1/sdk/heartbeat
Authorization: Bearer {sdkKey}

{
  "sdkId": "checkout-web-pod-abc123",
  "sdkVersion": "2.5.1",
  "language": "java",
  "snapshotVersion": "2026.05.12-002",
  "snapshotAge": 45,
  "uptime": 86400,
  "evaluationCount": 1523400,
  "defaultFallbackCount": 12
}
```

### 6.5 API Summary

| Category | Method | Path | Auth | Client Type | Purpose |
|----------|--------|------|------|-------------|--------|
| **Management** | POST | `/api/v1/features` | OAuth2 | Admin | Create feature |
| | GET | `/api/v1/features` | OAuth2 | Admin | List features |
| | GET | `/api/v1/features/{key}` | OAuth2 | Admin | Get feature |
| | PATCH | `/api/v1/features/{key}` | OAuth2 | Admin | Update feature |
| | DELETE | `/api/v1/features/{key}` | OAuth2 | Admin | Delete feature |
| | PUT | `/api/v1/features/{key}/rules` | OAuth2 | Admin | Set rules |
| | GET | `/api/v1/features/{key}/rules` | OAuth2 | Admin | Get rules |
| **Publishing** | POST | `/api/v1/snapshots/build` | OAuth2 | Admin | Trigger build |
| | GET | `/api/v1/publish-jobs/{id}` | OAuth2 | Admin | Job status |
| | GET | `/api/v1/snapshots` | OAuth2 | Admin | Snapshot history |
| | POST | `/api/v1/snapshots/rollback` | OAuth2 | Admin | Rollback |
| **Audit** | GET | `/api/v1/audit-logs` | OAuth2 | Admin | Query logs |
| **Debug** | POST | `/api/v1/debug/evaluate` | OAuth2 | Admin | Debug evaluate |
| | GET | `/api/v1/snapshots/{v}/diff` | OAuth2 | Admin | Diff report |
| **Server Eval** | GET | `/v1/apps/{a}/envs/{e}/latest` | Server SDK Key | Backend API | Latest version |
| | GET | `/v1/apps/{a}/envs/{e}/snapshots/{v}/full` | Server SDK Key | Backend API | Server snapshot |
| | GET | `/v1/apps/{a}/envs/{e}/snapshots/{v}/delta` | Server SDK Key | Backend API | Server delta |
| | GET | `/v1/stream` | Server SDK Key | Backend API | SSE stream |
| | GET | `/v1/apps/{a}/envs/{e}/bootstrap` | Server SDK Key | Web (SSR) | SSR bootstrap |
| **Client Eval** | GET | `CDN /{e}/{a}/latest.json` | — (public) | Web / Mobile | Latest version |
| | GET | `CDN /{e}/{a}/{v}/client-full.json.gz` | — (public) | Web / Mobile | Client snapshot |
| | GET | `CDN /{e}/{a}/{v}/client-delta-*.json.gz` | — (public) | Web / Mobile | Client delta |
| | GET | `/v1/client-stream` | Client SDK Key | Web / Mobile | Client SSE |
| **Auxiliary** | POST | `/v1/sdk/heartbeat` | Any SDK Key | All | SDK heartbeat |

---

## 7. Observability

### 7.1 Three Pillars

| Pillar | Tools | Purpose |
|--------|-------|---------|
| **Metrics** | Micrometer → Prometheus → Grafana | Quantitative health indicators, SLOs |
| **Logs** | Structured JSON → ELK / OpenSearch | Event-level debugging, audit trail |
| **Traces** | OpenTelemetry → Jaeger / Tempo | End-to-end publish-to-evaluate latency |

### 7.2 Key Metrics by Component

#### Control Plane (Management API)

| Metric | Type | Labels | Alert Threshold |
|--------|------|--------|----------------|
| `mgmt_api_request_duration_ms` | Histogram | method, path, status | p99 > 500ms |
| `mgmt_api_error_total` | Counter | method, path, status | > 1% error rate |
| `mgmt_db_query_duration_ms` | Histogram | query_type | p99 > 200ms |
| `mgmt_permission_denied_total` | Counter | actor, action | Spike detection |
| `mgmt_rule_validation_failed_total` | Counter | flag_key | — |
| `mgmt_audit_log_write_failed_total` | Counter | — | > 0 (critical) |

#### Publish Pipeline

| Metric | Type | Labels | Alert Threshold |
|--------|------|--------|----------------|
| `publish_compile_duration_ms` | Histogram | app_id, env | p99 > 5s |
| `publish_compile_failed_total` | Counter | app_id, env | > 0 |
| `publish_snapshot_build_duration_ms` | Histogram | app_id, env | p99 > 10s |
| `publish_snapshot_build_failed_total` | Counter | app_id, env | > 0 (critical) |
| `publish_snapshot_size_bytes` | Gauge | app_id, env, type | > 10MB (warning) |
| `publish_latest_json_update_failed_total` | Counter | app_id, env | > 0 |
| `publish_end_to_end_duration_ms` | Histogram | app_id, env | p99 > 30s |

#### SSE Gateway

| Metric | Type | Labels | Alert Threshold |
|--------|------|--------|----------------|
| `sse_connected_clients` | Gauge | app_id, env | — |
| `sse_event_publish_lag_ms` | Histogram | — | p99 > 1s |
| `sse_reconnect_total` | Counter | — | Spike detection |
| `sse_heartbeat_timeout_total` | Counter | — | > 0 |

#### Relay Proxy

| Metric | Type | Labels | Alert Threshold |
|--------|------|--------|----------------|
| `relay_upstream_sse_connected` | Gauge | — | < 1 (critical) |
| `relay_snapshot_version` | Info | app_id, env, version | — |
| `relay_snapshot_age_seconds` | Gauge | app_id, env | > 600 (warning) |
| `relay_cache_hit_ratio` | Gauge | — | < 90% (warning) |
| `relay_delta_apply_failed_total` | Counter | — | > 0 |
| `relay_downstream_sse_clients` | Gauge | — | — |

#### SDK (reported via heartbeat + metrics export)

| Metric | Type | Labels | Alert Threshold |
|--------|------|--------|----------------|
| `sdk_snapshot_version` | Info | app_id, env, sdk_id | — |
| `sdk_snapshot_age_seconds` | Gauge | app_id, env | > 600 (warning) |
| `sdk_snapshot_update_failed_total` | Counter | — | > 0 |
| `sdk_full_fallback_total` | Counter | — | Spike detection |
| `sdk_evaluation_duration_ns` | Histogram | — | p99 > 1ms |
| `sdk_flag_not_found_total` | Counter | flag_key | > 0 |
| `sdk_default_fallback_total` | Counter | flag_key | Spike detection |

### 7.3 Dashboards

| Dashboard | Key Panels |
|-----------|-----------|
| **Publish Pipeline** | End-to-end publish latency, success/failure rate, snapshot size trend, publish frequency |
| **Distribution Health** | Snapshot version distribution across Relay Proxies and SDKs, staleness, delta vs full ratio |
| **SDK Fleet** | SDK count by version, snapshot age heatmap, evaluation rate, error rate, language distribution |
| **Flag Activity** | Most-evaluated flags, highest default-fallback-rate flags, recently modified flags |
| **Alerting** | Publish failures, snapshot staleness > threshold, SSE disconnect rate, SDK error spike |

### 7.4 End-to-End Trace Correlation

Each publish operation generates a trace ID that flows through:

```
Management API (span: publish_requested)
  → Rule Compiler (span: compile_rules)
  → Snapshot Builder (span: build_snapshot)
  → Object Storage Upload (span: upload_artifacts)
  → SSE Event Publish (span: notify_sse, includes traceId in event data)
  → Relay Proxy (span: apply_snapshot)
  → SDK (span: update_snapshot)
```

This enables answering: "A flag was changed at 10:05 — when did each SDK instance pick it up?"

---

## 8. Explainability

### 8.1 Questions the System Must Answer

| Question | Source |
|----------|--------|
| Is a feature flag enabled? | SDK `isEnabled()` |
| For which users? | Rule conditions + rollout percentage |
| In which region? | Condition on `region` attribute |
| Which release is it associated with? | `releaseId` on the feature |
| Why did user X get this value? | SDK `evaluate(explain=true)` or Debug API |
| Why did user X NOT get this value? | Same, with per-condition match details |
| Which rule matched? | `ruleId` in evaluation result |
| What's the rollout bucket? | `rollout.bucket` + `rollout.threshold` |
| Which snapshot version? | `snapshotVersion` in result |
| Who changed it and when? | Audit log query |

### 8.2 Explain Metadata (generated by Rule Compiler)

The compiler emits human-readable metadata alongside each compiled rule:

```json
{
  "ruleId": "rule_001",
  "description": "CN/SG beta users with app >= 5.3.0",
  "releaseId": "checkout-v2",
  "createdBy": "alice@example.com",
  "conditionsText": [
    "region in [CN, SG]",
    "appVersion >= 5.3.0",
    "userSegment == beta"
  ]
}
```

### 8.3 SDK Explain Result

When `explain=true` is passed to `evaluate()`:

```json
{
  "flagKey": "checkout_new_ui",
  "value": true,
  "reason": "RULE_MATCH",
  "ruleId": "rule_001",
  "ruleDescription": "CN/SG beta users with app >= 5.3.0",
  "matchedConditions": [
    {
      "attribute": "region",
      "operator": "in",
      "expected": ["CN", "SG"],
      "actual": "CN",
      "matched": true
    },
    {
      "attribute": "appVersion",
      "operator": ">=",
      "expected": "5.3.0",
      "actual": "5.4.1",
      "matched": true
    },
    {
      "attribute": "userSegment",
      "operator": "eq",
      "expected": "beta",
      "actual": "beta",
      "matched": true
    }
  ],
  "rollout": {
    "key": "user_123",
    "bucket": 14321,
    "threshold": 30000,
    "bucketSize": 100000,
    "matched": true
  },
  "releaseId": "checkout-v2",
  "snapshotVersion": "2026.05.12-002",
  "sdkVersion": "2.5.1",
  "evaluatedAt": "2026-05-12T10:15:00Z"
}
```

### 8.4 Standard Reason Codes

| Reason | Description |
|--------|------------|
| `RULE_MATCH` | A targeting rule matched (conditions + rollout) |
| `DEFAULT` | No rule matched; returned flag's default value |
| `FLAG_NOT_FOUND` | Flag key does not exist in the snapshot |
| `FLAG_DISABLED` | Flag exists but is disabled |
| `PREREQUISITE_FAILED` | A prerequisite flag check failed |
| `CONDITION_NOT_MATCH` | Conditions evaluated but did not match (explain detail) |
| `ROLLOUT_NOT_MATCH` | Conditions matched but user fell outside rollout percentage |
| `ERROR` | Evaluation encountered an error |
| `TYPE_MISMATCH` | Requested type doesn't match flag type |
| `STALE_SNAPSHOT` | Snapshot is older than configured staleness threshold |

### 8.5 Debug Evaluate API

The Management API exposes a server-side debug endpoint that evaluates against the **latest published snapshot** (not the DB), matching exactly what an SDK would compute:

```http
POST /api/v1/debug/evaluate
Authorization: Bearer {admin_token}

{
  "flagKey": "checkout_new_ui",
  "appId": "checkout-web",
  "env": "prod",
  "context": {
    "userId": "user_123",
    "region": "CN",
    "appVersion": "5.4.1",
    "userSegment": "beta"
  }
}
```

Response: same structure as SDK Explain Result above.

This enables product managers and support engineers to debug flag evaluations through the Admin UI without accessing SDK logs.

### 8.6 Explainability in Admin UI

The Admin UI provides a **Debug Explain** page where users can:

1. Select a flag and environment
2. Enter user context attributes (userId, region, appVersion, etc.)
3. Click "Evaluate" to see the full explain result
4. See which rule matched, which conditions passed/failed, and the rollout bucket
5. See the associated release and snapshot version

---

## 9. Security

### 9.1 Authentication & Authorization

| Layer | Mechanism |
|-------|-----------|
| Admin UI → Management API | OAuth2 / OIDC / SSO with RBAC |
| SDK → Relay Proxy | SDK Key (environment-scoped, rotatable) |
| Relay Proxy → SSE Gateway / CDN | Service credential or mTLS |

### 9.2 RBAC Model

| Role | Permissions |
|------|------------|
| Viewer | Read features, rules, audit logs |
| Editor | Create/update features and rules |
| Publisher | Trigger snapshot builds and rollbacks |
| Admin | Manage SDK keys, RBAC, system configuration |

### 9.3 Snapshot Security

The server/client snapshot split is a core security boundary:

| Aspect | Server Snapshot | Client Snapshot |
|--------|----------------|----------------|
| Contains full conditions | Yes | Sensitive conditions stripped |
| Contains rollout internals | Yes (salt, threshold) | Yes (needed for client-side rollout) |
| Contains `serverOnly` flags | Yes | No — omitted entirely |
| Distributed via | Relay Proxy (internal) | Public CDN |
| Accessible to | Backend services only | End-user browsers and devices |
| Risk if leaked | Medium (targeting logic exposed) | Low (no sensitive rules) |

**Flags with `serverOnly: true`** (e.g., internal-employee-only features, fraud detection flags) are completely excluded from client snapshots. They are evaluated only in backend SDKs.

### 9.4 SDK Key Model

| Key Type | Scope | Used By | Capabilities |
|----------|-------|---------|-------------|
| Server SDK Key | Per app × per env | Backend API SDKs via Relay | Fetch server snapshots, SSE, heartbeat |
| Client SDK Key | Per app × per env | Web and Mobile SDKs | Fetch client snapshots (CDN), client SSE, heartbeat |
| Admin Token | Per user | Admin UI, CI/CD | Full management API access |

Keys are rotatable without downtime (dual-key overlap period). Client SDK Keys are safe to embed in browser bundles and mobile binaries because they only grant access to client snapshots.

### 9.5 Additional Security Controls

- All audit logs are append-only with actor, timestamp, and before/after state
- Debug Explain API is restricted to admin/publisher roles
- Metrics avoid high-cardinality user identifiers (no userId in metric labels)
- Log sanitization removes PII from structured logs
- Relay Proxy supports mTLS for upstream connections
- Client snapshots on CDN are signed (HMAC) so SDKs can verify integrity
- Mobile SDK validates snapshot signature before applying
- Web SDK uses Subresource Integrity (SRI) or signature verification

---

## 10. High Availability & Fault Tolerance

### 10.1 Graceful Degradation Chain

Each layer has an explicit fallback:

```
SSE event fails         → Polling latest.json (30s–5min interval)
Delta download fails    → Full snapshot download
CDN unavailable         → Relay Proxy serves from memory cache
Relay Proxy unavailable → SDK continues with in-memory snapshot
SDK process restarts    → SDK loads from disk cache
Disk cache corrupt      → SDK downloads full snapshot from relay/CDN
Everything unavailable  → Code-level default value (never null)
```

**Guarantee:** Configuration updates may be delayed, but business requests never fail due to the feature management system.

### 10.2 Component Failure Impact

| Component Down | Impact | Recovery |
|---------------|--------|----------|
| Management API | Cannot modify flags; evaluation unaffected | Restart; stateless |
| Feature DB | Cannot modify flags; evaluation unaffected | DB failover |
| Rule Compiler / Snapshot Builder | Cannot publish new snapshots; current config stays active | Restart; retry job |
| CDN | Relay/SDK cannot fetch new snapshots; cached version continues | CDN failover |
| SSE Gateway | No real-time notifications; polling takes over | Reconnect; Last-Event-ID |
| Relay Proxy | SDKs use in-memory snapshot; new pods read disk cache | Restart; re-sync |
| SDK crash | App restarts → disk cache → full snapshot | Automatic |

### 10.3 Deployment Recommendations

| Component | Replicas | Notes |
|-----------|---------|-------|
| Management API | 2–3 behind LB | Stateless; DB handles consistency |
| Feature DB | Primary + replica | Standard PostgreSQL HA |
| SSE Gateway | 2+ behind LB | Stateless; clients reconnect |
| Relay Proxy | 2–3 per K8s cluster | Local to business services |
| CDN | Multi-region edge | Managed service |

---

## 11. MVP Phasing

### Phase 1: Basic End-to-End Loop

> Goal: Feature created in Admin → published as snapshot → SDK evaluates locally.

| Component | Scope |
|-----------|-------|
| Feature DB | `features`, `feature_rules`, `rule_conditions`, `audit_logs` tables |
| Management API | Feature CRUD, Rule CRUD, basic auth, validation, audit log |
| Admin UI | Feature list, feature detail, simple rule editor, publish button |
| Rule Compiler | Operators: `eq`, `in`, `gte`, `lte`, `semver_gte`; percentage rollout |
| Snapshot Builder | Server full snapshot, gzip, SHA-256 checksum, `latest.json` |
| Object Storage | Full snapshot hosting, `latest.json` hosting |
| Java SDK (Backend) | Initialize, fetch server snapshot via Relay, in-memory HashMap, `isEnabled()`, default fallback |

**Outcome:** `Admin creates flag → publish → Backend SDK pulls → isEnabled() returns true/false`

### Phase 1b: Client Snapshot & Web/Mobile SDK

> Goal: Extend to web portal and mobile client evaluation.

| Component | Scope |
|-----------|-------|
| Snapshot Builder | Add client snapshot generation (strip sensitive rules, omit `serverOnly` flags) |
| CDN | Host client snapshots on public CDN with proper cache headers |
| Web SDK (JS) | Initialize from CDN, `localStorage` cache, `isEnabled()`, SSR bootstrap support |
| Mobile SDK (iOS/Android) | Initialize from CDN, bundled snapshot, disk cache, `isEnabled()` |
| SDK Key model | Separate server SDK keys and client SDK keys |

**Outcome:** `All three client types (backend API, web portal, mobile) can evaluate flags locally`

### Phase 2: Observability & Audit

> Goal: Basic system troubleshooting capability.

Audit log enrichment, Spring Actuator + Micrometer metrics, publish job tracking (PENDING → RUNNING → SUCCEEDED → FAILED), snapshot metadata, SDK metrics (version, update status, evaluation count).

### Phase 3: Explainability

> Goal: Answer "why did user X get this value?"

Compiler generates explain metadata, SDK `evaluate(explain=true)`, standard reason codes, Debug Evaluate API, Admin UI debug page.

### Phase 4: Polling & Disk Cache

> Goal: Reliable updates and cold-start resilience.

SDK periodic polling with ETag/304, disk cache for cold-start fallback, atomic snapshot swap, SDK health indicator.

### Phase 5: SSE Streaming

> Goal: Near-real-time configuration propagation.

SSE Gateway, `snapshot_published` events, SDK streaming client, heartbeat, reconnect with `Last-Event-ID`, polling as fallback.

### Phase 6: Delta Snapshots

> Goal: Reduce bandwidth for incremental changes.

Delta generation (upsert/delete operations), `fromVersion`/`toVersion` validation, delta + result checksum verification, SDK delta applier, fallback to full on failure.

### Phase 7: Relay Proxy

> Goal: Enterprise-scale deployment with local caching.

Upstream SSE client, snapshot caching (memory + disk), downstream latest/full/delta APIs, downstream SSE, SDK `relay-url` support, health metrics.

### Phase 8: Advanced Observability

> Goal: Production debugging, rollback analysis, release correlation.

Diff reports, publish/distribution/SDK health dashboards, trace correlation, evaluation event sampling, alerting.

### Phase 9: Rust Evaluator Core

> Goal: Cross-language evaluation consistency guarantee.

Shared Rust core library, JNI/N-API/PyO3 bindings, golden test suite, conformance CI pipeline.
