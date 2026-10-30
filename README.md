# 🚀 PRPilot — Autonomous Multi-Agent AI Code Review System

[![Python 3.11](https://img.shields.io/badge/Python-3.11-3776AB?style=for-the-badge&logo=python&logoColor=white)](https://www.python.org/)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.110+-009688?style=for-the-badge&logo=fastapi&logoColor=white)](https://fastapi.tiangolo.com/)
[![LangGraph](https://img.shields.io/badge/LangGraph-Multi--Agent-FF6F00?style=for-the-badge&logo=langchain&logoColor=white)](https://langchain-ai.github.io/langgraph/)
[![Celery](https://img.shields.io/badge/Celery-5.3+-37814A?style=for-the-badge&logo=celery&logoColor=white)](https://docs.celeryq.dev/)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-RDS-4169E1?style=for-the-badge&logo=postgresql&logoColor=white)](https://www.postgresql.org/)
[![Redis](https://img.shields.io/badge/Redis-ElastiCache-DC382D?style=for-the-badge&logo=redis&logoColor=white)](https://redis.io/)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-EKS-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![Terraform](https://img.shields.io/badge/Terraform-AWS_IaC-844FBA?style=for-the-badge&logo=terraform&logoColor=white)](https://www.terraform.io/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg?style=for-the-badge)](LICENSE)

> **PRPilot** is an enterprise-grade, distributed, multi-agent AI code review platform. It automatically reviews GitHub Pull Requests using specialized AI agents running concurrently via **LangGraph**, post line-level comments via the **GitHub App API**, and continually adapts to your repository's conventions through an asynchronous feedback loop.

---

## 📑 Table of Contents

- [System Architecture](#-system-architecture)
- [Key Features](#-key-features)
- [Multi-Agent Review Pipeline](#-multi-agent-review-pipeline)
- [Microservices Architecture](#-microservices-architecture)
- [Repository Structure](#-repository-structure)
- [Continuous Learning & Feedback Loop](#-continuous-learning--feedback-loop)
- [Infrastructure & Cloud Architecture](#-infrastructure--cloud-architecture)
- [Observability & Monitoring](#-observability--monitoring)
- [Evaluation & Benchmarking](#-evaluation--benchmarking)
- [Local Quickstart](#-local-quickstart)
- [Environment Variables](#-environment-variables)
- [Complete Production Operations Manual](#-complete-production-operations-manual)
- [License](#-license)

---

## 🏛 System Architecture

PRPilot is designed as an asynchronous, distributed microservices platform deployed on AWS EKS. Incoming pull request webhooks are cryptographically authenticated at the edge, enqueued into Redis, analyzed across 4 specialized AI agents in parallel, and posted as inline annotations.

```mermaid
flowchart TB

%% USERS / ENTRY
A[👨‍💻 Developer Creates Pull Request on GitHub] --> B[🐙 GitHub PR Event]
B --> C[🌐 Public Webhook URL]

%% INFRA ENTRY
C --> D[⚖️ AWS Application Load Balancer]

%% GATEWAY
D --> E[🚪 Gateway Service : FastAPI]
E --> E1[Verify HMAC Signature]
E --> E2[Reject Fake Requests]
E --> F[📩 Forward Verified Event]

%% WEBHOOK
F --> G[🪝 Webhook Service : FastAPI]
G --> G1[Parse PR Number / Repo / SHA]
G --> G2[Deduplicate Existing SHA]
G --> G3[Store PR Metadata]

%% DATABASE
G3 --> DB[(🗄️ PostgreSQL RDS)]

%% QUEUE
G --> H[📬 Push Job to Redis Queue]
H --> R[(⚡ Redis / ElastiCache)]

%% WORKER
R --> I[⚙️ Celery Worker]

%% ORCHESTRATOR
I --> J[🧠 Orchestrator Service]
J --> J1[Fetch Code Diff from GitHub API]
J --> J2[Load Past Repo Patterns]
J --> J3[Run LangGraph Multi-Agent Review]

%% AI AGENTS
J3 --> K1[🔍 Static Analysis Agent]
J3 --> K2[🛡️ Security Agent]
J3 --> K3[🎨 Style Agent]
J3 --> K4[🏗️ Architecture Agent]

K1 --> L[Merge Findings]
K2 --> L
K3 --> L
K4 --> L

L --> M[Remove Duplicate Comments]
M --> N[Save Findings to DB]
N --> DB

%% REVIEWER
M --> O[💬 Reviewer Service]
O --> O1[Generate GitHub App JWT]
O --> O2[Create Installation Token]
O --> O3[Post Inline Comments]
O --> O4[Post Summary Review]

O --> P[🐙 GitHub Pull Request Updated]

%% FEEDBACK LOOP
P --> Q[Developer Fixes Code & Pushes Again]
Q --> B

%% MERGE EVENT
P --> Z{PR Merged?}
Z -->|Yes| AA[📘 Learner Worker]
AA --> AB[📈 Learner Service]
AB --> AC[Extract Frequent Issues]
AC --> AD[Store Repo Patterns]
AD --> DB
AD --> AE[Future Reviews Become Smarter]

%% MONITORING
subgraph Monitoring & Observability
MO1[📊 Prometheus]
MO2[📉 Grafana]
MO3[🧾 LangFuse]
end

E --> MO1
G --> MO1
J --> MO1
O --> MO1
J --> MO3
MO1 --> MO2

%% DEPLOYMENT
subgraph DevOps CI/CD
CI1[GitHub Actions]
CI2[Docker Build]
CI3[AWS ECR]
CI4[EKS Kubernetes Cluster]
end

CI1 --> CI2 --> CI3 --> CI4
CI4 --> E
CI4 --> G
CI4 --> J
CI4 --> O
CI4 --> AB
CI4 --> I
```

---

## ✨ Key Features

- **🔐 Cryptographically Secure Edge**: Validates incoming GitHub HMAC SHA-256 signatures (`X-Hub-Signature-256`) before requests touch backend services.
- **⚡ Non-Blocking Asynchronous Processing**: Instant `200 OK` response to GitHub; review jobs are distributed via Redis and processed by dedicated Celery workers.
- **🤖 Parallel Multi-Agent Consensus**: 4 discrete LangGraph agents review the diff simultaneously with domain-specialized prompts.
- **🎯 Precise Line-Level Annotations**: Comments are placed directly on the offending diff lines via GitHub's Pull Request Review Comments API, including actionable code suggestion blocks.
- **🧹 Deduplication & Hallucination Guardrails**: Cross-agent findings are merged, deduplicated, and validated against the actual unified diff range.
- **🧠 Continuous Repository Adaptation**: A self-improving feedback worker learns from accepted vs. rejected developer feedback, tuning subsequent reviews.
- **📊 Production-Grade Observability**: End-to-end metrics scraped with Prometheus, visualized on Grafana, and LLM calls traced with LangFuse.
- **☁️ Cloud-Native & Scalable**: Production Kubernetes manifests with Horizontal Pod Autoscalers (HPA), automated Alembic DB migrations, and complete AWS Terraform IaC.

---

## 🤖 Multi-Agent Review Pipeline

The Orchestrator leverages **LangGraph** to execute a directed cyclic graph of specialized review agents:

| Agent | Focus Area | Checks Performed |
| :--- | :--- | :--- |
| **🔍 Static Analysis Agent** | Code Correctness & Hygiene | • Unused variables and imports<br>• High cyclomatic complexity<br>• Shadowed variables & dead code paths<br>• Type hinting and return signatures |
| **🛡️ Security Agent** | Vulnerabilities & Exploits | • OWASP Top 10 vulnerabilities<br>• SQL injection & command injection<br>• Hardcoded secrets, tokens, and private keys<br>• Insecure deserialization & SSRF |
| **🎨 Code Style Agent** | Conventions & Maintainability | • PEP8 & language-specific style guides<br>• Naming conventions (snake_case vs. camelCase)<br>• Comment accuracy and docstrings<br>• Code readability and cognitive overhead |
| **🏗️ Architecture Agent** | Design & Reliability | • Separation of concerns & modularity<br>• Missing exception handling & edge cases<br>• Database N+1 query patterns<br>• API design contracts and breaking changes |

```mermaid
graph LR
    Diff[Unified Git Diff] --> SA[Static Analysis Agent]
    Diff --> SEC[Security Agent]
    Diff --> STY[Code Style Agent]
    Diff --> ARC[Architecture Agent]
    
    Patterns[(Learned Repo Patterns)] -.-> SA
    Patterns -.-> SEC
    Patterns -.-> STY
    Patterns -.-> ARC

    SA --> Aggregator[Findings Aggregator & Deduplicator]
    SEC --> Aggregator
    STY --> Aggregator
    ARC --> Aggregator

    Aggregator --> Filter[Diff Range Validator]
    Filter --> ReviewPayload[Structured Inline Review]
```

---

## 🧩 Microservices Architecture

PRPilot is decomposed into 5 independently scalable microservices:

```
services/
├── gateway/          # Port 8000: Edge HMAC verification & authentication proxy
├── webhook/          # Port 8001: Webhook ingestion, DB logging & Celery task dispatch
├── orchestrator/     # Port 8002: LangGraph multi-agent review graph execution
├── reviewer/         # Port 8003: GitHub App JWT generation & inline comment publisher
└── learner/          # Port 8004: Developer feedback aggregator & repo pattern extractor
```

| Service | Port | Tech Stack | Responsibilities |
| :--- | :--- | :--- | :--- |
| **Gateway** | `8000` | FastAPI, Cryptography | Computes HMAC SHA-256 against `GITHUB_WEBHOOK_SECRET`. Rejects forged payloads. |
| **Webhook** | `8001` | FastAPI, Celery, SQLAlchemy | Parses pull request events (`opened`, `synchronize`). Deduplicates run requests by SHA. Pushes jobs to Redis. |
| **Orchestrator** | `8002` | FastAPI, LangGraph, OpenAI | Downloads pull request diff chunks from GitHub. Executes the 4-agent LangGraph workflow. Deduplicates findings. |
| **Reviewer** | `8003` | FastAPI, PyJWT, Requests | Authenticates as GitHub App via private key PEM. Creates installation tokens and posts batch review comments. |
| **Learner** | `8004` | FastAPI, Celery, SQLAlchemy | Triggers upon PR merge. Analyses accepted vs. dismissed review comments. Stores learned rules in PostgreSQL. |

---

## 📂 Repository Structure

```
.
├── .github/
│   └── workflows/              # GitHub Actions CI/CD workflows for all services
│       ├── service-ci.yml      # Automated linting & unit testing on PRs
│       ├── gateway.yml         # Gateway Docker build & ECR deployment
│       ├── webhook.yml         # Webhook service build & deployment
│       ├── orchestrator.yml    # Orchestrator build & deployment
│       ├── reviewer.yml        # Reviewer service build & deployment
│       ├── learner.yml         # Learner service build & deployment
│       └── evaluate.yml        # Weekly scheduled model accuracy evaluation
├── db/
│   ├── alembic.ini             # Alembic migration configuration
│   └── migrations/             # Database versions (PRs, Findings, Feedback, Rules)
├── infra/
│   ├── terraform/              # AWS IaC (VPC, EKS, RDS, ElastiCache, ALB, IAM)
│   └── k8s/                    # Kubernetes manifests (Deployments, Services, HPA, Ingress)
├── monitoring/
│   ├── prometheus.yml          # Prometheus scrape targets
│   └── grafana-dashboard.json  # Prebuilt Grafana dashboard definitions
├── scripts/
│   ├── evaluate.py             # Accuracy, precision, recall benchmarking suite
│   └── Dockerfile              # Container for evaluation Kubernetes CronJob
├── services/                   # All 5 microservices (FastAPI + Dockerfiles)
├── docs/
│   └── PRODUCTION_SETUP.md     # Comprehensive 16-phase production operations runbook
├── final-diagram.mmd           # Source Mermaid architecture diagram
├── LICENSE                     # MIT License
└── README.md                   # Project documentation
```

---

## 🔁 Continuous Learning & Feedback Loop

Unlike generic static review bots, PRPilot gets smarter as your engineering team uses it:

1. **Inline Interaction**: When PRPilot leaves a comment on a PR, the author can accept the fix, push a new commit, or reply.
2. **Merge Event Trigger**: When the PR is merged into `main`, a GitHub webhook triggers the `learner-worker`.
3. **Pattern Extraction**: The learner service compares initial review comments against the final merged code.
4. **Knowledge Store**: Patterns accepted by senior maintainers are persisted to PostgreSQL as repository-specific guidelines (e.g., *"This repo prefers custom Result[T] types instead of throwing exceptions"*).
5. **Context Injection**: Future runs of the Orchestrator load these patterns directly into the LangGraph agent prompts.

---

## ☁️ Infrastructure & Cloud Architecture

PRPilot is provisioned on AWS using modular Terraform and deployed on Amazon EKS:

```
AWS Cloud
├── VPC (Public & Private Subnets across 2 AZs)
├── Internet Gateway & NAT Gateways
├── Application Load Balancer (ALB) -> Ingress Controller
├── Amazon EKS Cluster (v1.29)
│   ├── Managed Node Group (m5.large instances)
│   ├── Horizontal Pod Autoscaler (CPU/Memory targets)
│   └── Namespaced Workloads (Gateway, Webhook, Orchestrator, Reviewer, Learner)
├── Amazon RDS (PostgreSQL 15 Multi-AZ)
└── Amazon ElastiCache (Redis 7 Cluster Mode)
```

- **Terraform IaC**: Deploy VPC, EKS, RDS, and ElastiCache in a single command (`terraform apply`).
- **Zero-Downtime Rolling Updates**: Kubernetes Deployments with liveness and readiness probes.
- **Automated Database Migrations**: Run via Kubernetes Jobs before new application versions are routed traffic.

---

## 📊 Observability & Monitoring

PRPilot includes out-of-the-box observability:

- **Prometheus Scrapes**: Every FastAPI microservice exports standard `/metrics` via `prometheus-fastapi-instrumentator`.
- **Preconfigured Grafana Dashboard**:
  - Webhook processing latency (P50, P95, P99)
  - Celery queue depth and active worker counts
  - Token consumption by agent type (OpenAI API usage)
  - Review generation time and error rates
- **LangFuse Tracing**: Deep observability for LLM token usage, prompt versions, agent execution chains, and latencies.

---

## 🧪 Evaluation & Benchmarking

The evaluation suite (`scripts/evaluate.py`) runs periodically or via GitHub Actions to benchmark model quality against labeled PR datasets:

$$\text{Precision} = \frac{\text{True Actionable Comments}}{\text{Total Comments Posted}}$$

$$\text{Recall} = \frac{\text{Detected Vulnerabilities}}{\text{Total Known Vulnerabilities}}$$

$$\text{Noise Ratio} = \frac{\text{Dismissed Comments}}{\text{Total Comments}}$$

To run the evaluation locally:
```bash
python scripts/evaluate.py --dataset test_prs.json --threshold 0.85
```

---

## ⚡ Local Quickstart

### Prerequisites
- Python 3.11+
- Docker & Docker Compose
- PostgreSQL & Redis (or running via Docker)

### 1. Clone the Repository
```bash
git clone https://github.com/PalakshiVerma/PRPilot.git
cd PRPilot
```

### 2. Set Up Environment Variables
Create a `.env` file in the root directory:
```env
GITHUB_APP_ID=123456
GITHUB_APP_PRIVATE_KEY="-----BEGIN RSA PRIVATE KEY-----\n..."
GITHUB_WEBHOOK_SECRET=your_webhook_secret_here
OPENAI_API_KEY=sk-...
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/prpilot
REDIS_URL=redis://localhost:6379/0
LANGFUSE_PUBLIC_KEY=pk-...
LANGFUSE_SECRET_KEY=sk-...
```

### 3. Run Database Migrations
```bash
cd db
pip install alembic psycopg2-binary
alembic upgrade head
cd ..
```

### 4. Run Services Locally
```bash
# Terminal 1: Webhook Gateway
cd services/gateway && uvicorn main:app --port 8000 --reload

# Terminal 2: Webhook Ingest
cd services/webhook && uvicorn main:app --port 8001 --reload

# Terminal 3: Celery Worker
cd services/webhook && celery -A worker.celery worker --loglevel=info

# Terminal 4: AI Orchestrator
cd services/orchestrator && uvicorn main:app --port 8002 --reload

# Terminal 5: Reviewer Service
cd services/reviewer && uvicorn main:app --port 8003 --reload

# Terminal 6: Learner Service
cd services/learner && uvicorn main:app --port 8004 --reload
```

---

## ⚙️ Environment Variables

| Variable | Description | Services |
| :--- | :--- | :--- |
| `GITHUB_APP_ID` | The App ID generated in GitHub App settings | `reviewer`, `orchestrator` |
| `GITHUB_APP_PRIVATE_KEY` | Private key PEM string for authenticating as GitHub App | `reviewer`, `orchestrator` |
| `GITHUB_WEBHOOK_SECRET` | Secret key used to generate and verify HMAC SHA-256 signatures | `gateway` |
| `OPENAI_API_KEY` | OpenAI API key for LangGraph agents (GPT-4o) | `orchestrator` |
| `DATABASE_URL` | PostgreSQL connection URI | `webhook`, `learner`, `db` |
| `REDIS_URL` | Redis connection URI for task queues | `webhook`, `learner` |
| `LANGFUSE_PUBLIC_KEY` | Public key for LangFuse LLM tracing | `orchestrator` |
| `LANGFUSE_SECRET_KEY` | Secret key for LangFuse LLM tracing | `orchestrator` |

---

## 📖 Complete Production Operations Manual

For a complete, zero-to-production 16-phase runbook including:
- Creating and configuring your GitHub App
- Provisioning AWS VPC, EKS, RDS, and ElastiCache with Terraform
- Setting up GitHub Actions deployment secrets
- Connecting `kubectl` to EKS and deploying manifests
- Configuring Prometheus, Grafana, and LangFuse dashboards
- End-to-end integration testing and cost management

👉 **See the full [Production Setup Guide](docs/PRODUCTION_SETUP.md)**.

---

## 📄 License

Distributed under the MIT License. See [`LICENSE`](LICENSE) for details.

---

<p align="center">
  Built with ❤️ for modern software engineering teams.
</p>
