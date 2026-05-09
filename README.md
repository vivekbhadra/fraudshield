# FraudShield

A real-time payment fraud detection platform built as two independent
microservices communicating asynchronously via Apache Kafka.

---

## System Architecture

```
                        ┌─────────────────────────────────┐
                        │         Kubernetes Cluster       │
                        │          (Minikube / Cloud)      │
                        │                                  │
  REST API              │  ┌──────────────────────┐        │
 ──────────────────────►│  │  Transaction Service │        │
 POST /transactions/    │  │  (FastAPI + Postgres) │        │
                        │  └──────────┬───────────┘        │
                        │             │ TransactionInitiated│
                        │             ▼                     │
                        │       ┌──────────┐               │
                        │       │  Kafka   │               │
                        │       └──────────┘               │
                        │             │ FraudVerdict        │
                        │             ▼                     │
                        │  ┌──────────────────────┐        │
                        │  │ Fraud Detection Svc  │        │
                        │  │ (FastAPI + Redis +   │        │
                        │  │  Postgres + Rules)   │        │
                        │  └──────────────────────┘        │
                        └─────────────────────────────────┘
```

### Services

| Service | Repo | Port | Responsibility |
|---------|------|------|----------------|
| Transaction Service | [fraudshield-transaction-service](https://github.com/vivekbhadra/fraudshield-transaction-service) | 8003 | Accept payments, publish events, update status |
| Fraud Detection Service | [fraudshield-fraud-detection-service](https://github.com/vivekbhadra/fraudshield-fraud-detection-service) | 8004 | Score transactions, publish verdicts |

### Infrastructure (shared, managed by transaction-service repo)

| Component | Purpose |
|-----------|---------|
| Apache Kafka + Zookeeper | Async event bus between services |
| PostgreSQL (×2) | Separate DB per service |
| Redis | Merchant blacklist + velocity cache |

### Kafka Topics

| Topic | Producer | Consumer |
|-------|----------|----------|
| `transactions.initiated` | Transaction Service | Fraud Detection Service |
| `fraud.verdict` | Fraud Detection Service | Transaction Service |

---

## Fraud Scoring Rules

| Rule | Trigger | Score |
|------|---------|-------|
| Blacklist | Merchant in Redis blacklist | 100 → instant BLOCK |
| High Amount | Transaction > 3× user's rolling average | 40 |
| Velocity | > 5 transactions within 60s | 40 |
| New Merchant | First transaction with this merchant | 20 |
| Off Hours | Outside 06:00–22:00 | 15 |

Verdict thresholds: `< 30` → **PASS** · `30–69` → **REVIEW** · `≥ 70` → **BLOCK**

---

## Quick Start

### Prerequisites
- [minikube](https://minikube.sigs.k8s.io/docs/start/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [docker](https://docs.docker.com/get-docker/)
- [git](https://git-scm.com/)

### 1. Clone with submodules

```bash
git clone --recurse-submodules https://github.com/vivekbhadra/fraudshield
cd fraudshield
```

Or if already cloned:

```bash
git submodule update --init --recursive
```

### 2. Deploy to Minikube

```bash
cd fraudshield-transaction-service
chmod +x deploy.sh
./deploy.sh
```

The script will:
- Start Minikube if not already running
- Build both Docker images inside Minikube's daemon
- Apply all Kubernetes manifests in dependency order
- Wait for every pod to become healthy
- Run a port-forward health check

### 3. Access the services

```bash
# Transaction Service (stays open in a terminal)
kubectl port-forward service/transaction-service 18003:8003 -n fraudshield

# Fraud Detection Service (in another terminal)
kubectl port-forward service/fraud-detection-service 18004:8004 -n fraudshield
```

| Service | Swagger UI |
|---------|-----------|
| Transaction Service | http://localhost:18003/docs |
| Fraud Detection Service | http://localhost:18004/docs |

### 4. Run the end-to-end smoke test

```bash
# From fraudshield-transaction-service/
./scripts/test_fraud_block.sh
```

Expected output: `FraudShield fraud-block smoke test PASSED.`

---

## Local Development (Docker Compose)

Runs the full stack locally without Kubernetes:

```bash
cd fraudshield-transaction-service
docker compose up --build

# Transaction Service → http://localhost:8003/docs
# Fraud Detection     → http://localhost:8004/docs
```

See `fraudshield-transaction-service/LOCAL_TESTING_GUIDE.md` for the full
test scenario walkthrough including the Postman collection.

---

## Running Tests

```bash
# Transaction Service
cd fraudshield-transaction-service
pip install -r requirements.txt
pytest tests/ -v

# Fraud Detection Service
cd ../fraudshield-fraud-detection-service
pip install -r requirements.txt
pytest tests/ -v   # 18 tests
```

---

## Useful Commands

```bash
# Cluster overview
kubectl get all -n fraudshield

# Live logs
kubectl logs -n fraudshield deployment/transaction-service -f
kubectl logs -n fraudshield deployment/fraud-detection-service -f

# Kubernetes dashboard
minikube dashboard

# Tear down
kubectl delete namespace fraudshield
minikube stop
```

---

## Repository Structure

```
fraudshield/                              ← this repo (umbrella)
├── fraudshield-transaction-service/      ← git submodule
│   ├── app/                              # FastAPI application
│   ├── k8s/                              # Kubernetes manifests
│   ├── scripts/                          # deploy.sh, smoke test
│   ├── tests/
│   ├── Dockerfile
│   └── docker-compose.yml
├── fraudshield-fraud-detection-service/  ← git submodule
│   ├── app/
│   │   └── scoring/rules/               # 5 independent fraud rules
│   ├── k8s/
│   ├── tests/                           # 18 unit tests
│   └── Dockerfile
└── README.md                            ← you are here
```
