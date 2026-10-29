# Automated Log Anomaly Detector & Incident Correlation AI (LAD)

> **Enterprise-grade, real-time log anomaly detection, multi-service incident correlation, and operational intelligence platform powered by Serverless Cloud Architecture, Machine Learning, and Automated CI/CD.**

---

## 🌟 Executive Summary

**Incident Correlation AI** is an end-to-end, high-throughput observability platform designed to solve alert fatigue and reduce Mean Time to Resolution (MTTR) in distributed systems. It continuously collects raw operating system and service logs from edge servers, normalizes and redacts them, extracts multidimensional statistical features, detects anomalous operational patterns using Machine Learning and Robust Statistics, and automatically correlates cascading multi-service failures into unified, actionable incident briefs with root-cause hypotheses and diagnostic checklists.

The system is deployed on an **enterprise serverless cloud infrastructure** across **Amazon Web Services (AWS)** and **Vercel**, achieving sub-second ingestion, asynchronous decoupled ML processing, zero-loss buffering, and real-time interactive incident triage.

---

## 🏗️ High-Level System Architecture & End-to-End Flow

```
[ Edge Server / Hosts ]
  └─ lad-collector (Systemd Journald / Log Agent)
        │
        │ 1. HTTPS POST /api/v1/events/batch (Bearer Token)
        ▼
┌────────────────────────────────────────────────────────────────────────┐
│                   Amazon API Gateway (HTTP API)                        │
│   - Global HTTPS endpoint & CORS management                            │
│   - Invokes IncidentAuthorizerFn for sub-millisecond token auth        │
└────────────────────────────────────────────────────────────────────────┘
        │                                         ▲
        │ 2. Authenticate                         │ Validated
        ▼                                         │
┌───────────────────────────┐                     │
│   IncidentAuthorizerFn    │─────────────────────┘
│ (Token & Identity Guard)  │
└───────────────────────────┘
        │
        │ 3. Forward Validated Batch
        ▼
┌───────────────────────────┐      Deduplication Check
│     IncidentIngestFn      │───────────────────────────────┐
│ (Schema Validate & Dedup) │                               │
└───────────────────────────┘                               │
        │                                                   │
        │ 4. Enqueue Batches (Zero Log Loss)                │
        ▼                                                   │
┌───────────────────────────┐                               │
│      Amazon SQS           │                               │
│  (IncidentMainQueue)      │                               │
└───────────────────────────┘                               │
        │                                                   │
        │ 5. EventSourceMapping (Batches up to 50 / 5s)     │
        ▼                                                   │
┌────────────────────────────────────────────────────┐      │
│                IncidentWorkerFn                    │      │
│  ════════════════════════════════════════════════  │      ▼
│  🧠 AI & MACHINE LEARNING DETECTION ENGINE         │  ┌──────────────────────┐
│  - Multidimensional Windowing (39 Metrics)         │  │   Amazon DynamoDB    │
│  - Robust Statistics: Median Absolute Deviation    │  │   (IncidentTable)    │
│  - Isolation Forest & Statistical Baselines        │  │                      │
│  - Cross-Service Temporal Proximity Correlation    │─▶│ - DEDUP# records     │
│  - Root-Cause Hypothesis & Diagnostic Synthesis    │  │ - INC# incidents     │
└────────────────────────────────────────────────────┘  │ - Audit trail logs   │
        │                                               └──────────────────────┘
        │ (Fault-Tolerant Dead-Letter Redrive)                      ▲
        ▼                                                           │
┌───────────────────────────┐                                       │
│          SQS DLQ          │                                       │
│       (IncidentDLQ)       │                                       │
└───────────────────────────┘                                       │
                                                                    │ Fetch & Update
[ Vercel Cloud ]                                                    │
  └─ Next.js Operator Dashboard                                     │
        │                                                           │
        │ 6. HTTPS GET/POST /api/v1/incidents                       │
        ▼                                                           │
┌───────────────────────────┐                                       │
│      IncidentReadFn       │───────────────────────────────────────┘
│  (Incident Serving & API) │
└───────────────────────────┘
```

---

## 🧠 Artificial Intelligence & Machine Learning Architecture

The AI/ML engine is deployed directly inside **`IncidentWorkerFn`**, executing intelligent anomaly detection and graph correlation in real time without human intervention.

### 1. Multidimensional Feature Engineering (39 Canonical Metrics)
Incoming canonical event streams are windowed into epoch-aligned observation buckets across hosts and services. The engine extracts **39 operational telemetry features**, including:
* **Error Dynamics:** Error counts, total error ratios, and error-eligible denominators.
* **HTTP Protocols:** Request frequency, HTTP 5xx server error bursts, 4xx client errors, and status eligibility.
* **Database Telemetry:** DB event volumes, timeout counts, timeout ratios, connection failure counts, and connection failure ratios.
* **Authentication & Identity:** Failure counts, success counts, failure ratios, distinct source IP counts, and account diversity.
* **Latency Distribution:** Mean latency, $p_{50}$, $p_{95}$, $p_{99}$, and maximum latency samples.
* **Service Lifecycle:** Process crash events, service restart surges, and telemetry gap tracking.

### 2. Robust Outlier Detection with Median Absolute Deviation (MAD)
To eliminate false positives caused by skewed distributions or historical attack outliers, the system uses **Median Absolute Deviation (MAD)** for rolling causal baseline deviations:

$$\text{Center} = \text{median}(X)$$
$$\text{MAD} = \text{median}\left(|X_i - \text{Center}|\right)$$
$$\text{Scale} = 1.4826 \times \text{MAD}$$
$$\text{Robust } z\text{-score} = \text{clip}\left(\frac{x - \text{Center}}{\text{Scale}}, -10.0, 10.0\right)$$

This scale factor ensures consistency with normal distributions while preventing extreme spikes from shifting the baseline.

### 3. Isolation Forest & Statistical Anomaly Scoring
* **Isolation Forest Engine:** Leverages multidimensional decision trees to isolate anomalous observations. Anomalies map to shorter path lengths, generating negative anomaly scores ($raw\_score = -E(h(x))$) evaluated against calibrated threshold boundaries.
* **Domain Baseline Engine:** Applies strict volume safeguards to prevent false alarms on low-traffic servers. Validates signal strength across HTTP error surges, database connection drops, credential brute-forcing, latency degradation, and service crashes.

### 4. Deterministic Cross-Service Evidence Correlation
When an incident occurs (e.g. database connection pool exhaustion causing downstream web API 500 errors), the correlation engine groups anomalies using an **online sliding-window graph algorithm**:
* **Temporal Proximity Clustering:** Merges related anomaly candidates across services occurring within configurable correlation windows ($120\text{s}$).
* **Deterministic Incident ID (UUIDv5):** Employs SHA-1 based namespace UUIDv5 over sorted window identities (`uuid.uuid5(NAMESPACE_URL, "lad-correlation|" + window_ids)`), ensuring replay-safe, idempotent incident creation.
* **Automated Hypothesis & Runbook Generation:** Synthesizes human-readable root-cause statements, confidence scores, supporting/contradicting event IDs, and actionable diagnostic investigation checklists for SRE teams.

---

## ☁️ Cloud Architecture & AWS Services

The backend is built natively on **Amazon Web Services (AWS)** in the `eu-north-1` region, utilizing serverless primitives for maximum elasticity, high availability, and zero idle costs:

### 1. Amazon API Gateway (HTTP API v2)
* Serves as the ultra-low-latency public HTTPS ingress (`d6l60no0kk.execute-api.eu-north-1.amazonaws.com`).
* Enforces strict CORS handling for the Vercel web dashboard.
* Integrates directly with `IncidentAuthorizerFn` for sub-millisecond edge authentication.

### 2. AWS Lambda Functions (Serverless Microservices)
* **`IncidentAuthorizerFn`:** High-speed security authorizer executing constant-time cryptographic token verification (`hmac.compare_digest`) on every request.
* **`IncidentIngestFn`:** High-throughput entry point validating JSON schemas, performing atomic deduplication, and streaming batches to SQS in $<30\text{ ms}$.
* **`IncidentWorkerFn`:** Asynchronous compute engine running the **AI/ML detection model**, MAD statistics, and incident correlation pipeline.
* **`IncidentReadFn`:** Query and lifecycle engine serving incidents with cursor pagination and atomic state updates (`open` $\rightarrow$ `acknowledged` $\rightarrow$ `resolved`).

### 3. Amazon Simple Queue Service (SQS) & Dead-Letter Queue (DLQ)
* **Main Queue (`IncidentMainQueue`):** Absorbs massive log bursts during distributed attacks or outages, smoothing consumption for workers without dropping data.
* **Event Source Mapping:** Configured with a 5-second batching window and batch size of 50 messages to assemble multi-event contexts for the ML feature extractors.
* **Dead-Letter Queue (`IncidentDLQ`):** Redrive policy isolates malformed messages after repeated delivery failures, ensuring the pipeline never stalls.

### 4. Amazon DynamoDB (`IncidentTable`)
* Ultra-fast NoSQL database operating with single-digit millisecond response times.
* **Deduplication Lane (`DEDUP#<event_id>`):** Uses conditional writes and automatic 7-day Time-To-Live (TTL) expiration to guarantee idempotency.
* **Incident Lane (`INC#<incident_id>`):** Stores full incident drafts, ML severity, anomaly scores, affected entity graphs, evidence arrays, and immutable audit logs.

### 5. Amazon CloudWatch Logs
* Centralized structured logging across all Lambda microservices.
* Provides real-time execution metrics, duration tracking, memory utilization, and diagnostic error tracing.

### 6. AWS Systems Manager (SSM) Parameter Store
* Centralized secure storage for encrypted secrets and credentials (`/lad/dev/admin-password-hash`, `/lad/dev/collector-token-hash`, `/lad/dev/session-secret`) backed by AWS Key Management Service (KMS).

### 7. Identity & Access Management (IAM)
* Enforces role isolation between system components:
  * **Collector Identity (`collectorAuth`):** Authorized strictly to ingest event batches (`POST /api/v1/events/batch`).
  * **Admin Identity (`adminAuth`):** Authorized strictly to inspect incidents and trigger lifecycle transitions (`GET/POST /api/v1/incidents*`).

---

## 🚀 DevOps, CI/CD & Frontend Deployment

### Continuous Integration & Continuous Deployment (CI/CD)
* **Automated GitHub Actions Pipeline (`deploy-lambdas-cd`):**
  1. **Continuous Testing:** Checks out both the core service and AI repositories, installs dependencies, and runs the automated test suite across contract, validation, and integration tests.
  2. **Automated Package Assembly:** Pre-compiles dependencies for `manylinux2014_x86_64`, packages microservices with shared contracts, bundles the `anomaly_detector` AI engine, and generates optimized zip artifacts.
  3. **Zero-Downtime Deployment:** Uses the AWS CLI to update Lambda function code in parallel, waits for successful deployment health checks, and executes live smoke tests against the public API.

### Vercel Web Dashboard
* Built with modern **Next.js**, React, and Tailwind CSS.
* Hosted on **Vercel** for global CDN edge delivery and real-time updates.
* Provides SOC/SRE teams with incident timelines, severity badges, affected entity graphs, AI diagnostic checklists, and one-click incident acknowledgement and resolution.

---

## 📡 Edge Log Collector Agent (`lad-collector`)

The edge log collector runs as a lightweight daemon on monitored Linux and Windows nodes:
* **Systemd-Journald Real-Time Streaming:** Ingests Linux system logs directly from the local journal.
* **Intelligent Rule Classification:** Maps kernel, authorization (`sshd`), database (`postgresql`, `mysql`, `redis`), web server (`nginx`, `caddy`, `apache`), and service events into a strict canonical schema.
* **Automated Privacy Redaction:** Sanitizes passwords, access tokens, API keys, and sensitive authorization headers before data leaves the host.
* **Micro-Batching:** Groups telemetry and automatically flushes batches every 3 seconds or 100 events over HTTPS with Bearer token authentication.

---

## 🏆 Summary of Platform Capabilities

| Area | Capability Highlights |
| :--- | :--- |
| **Artificial Intelligence** | Multidimensional Windowing (39 features), Isolation Forest ML Anomaly Detection, Robust Statistics (Median Absolute Deviation), Cross-Service Sliding-Window Graph Correlation. |
| **Cloud Engineering** | 100% Serverless AWS Architecture (API Gateway, 4x Lambda, SQS Main Queue + DLQ, DynamoDB with TTL, CloudWatch, SSM Parameter Store). |
| **DevOps & Delivery** | Fully automated GitHub Actions CI/CD pipeline, pre-compiled platform builds, zero-downtime deployment, Vercel edge-hosted Next.js dashboard. |
| **Reliability & Scale** | Asynchronous decoupled queuing, sub-second ingestion, atomic idempotency deduplication, and automated fault-tolerant DLQ redrives. |
| **Security** | Zero-trust role separation (`collectorAuth` vs `adminAuth`), edge token authorizers, and automatic PII/credential redaction at the collector. |
