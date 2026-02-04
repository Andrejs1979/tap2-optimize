# Technical Architecture

## Overview

Tap2 Optimize is a cloud-native SaaS platform built on Cloudflare Workers for edge processing and optimization.

---

## System Design

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              Frontend Layer                             │
│                         (React + TypeScript)                           │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                           API Gateway Layer                             │
│                    (Cloudflare Workers + Router)                        │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
        ┌───────────────────────────┼───────────────────────────┐
        ▼                           ▼                           ▼
┌───────────────┐           ┌───────────────┐           ┌───────────────┐
│ Optimization  │           │   A/B Test    │           │     3DS       │
│   Engine      │           │   Platform    │           │  Decisioner   │
│  (Python/TF)  │           │  (Node.js)    │           │ (Rules + ML)  │
└───────────────┘           └───────────────┘           └───────────────┘
        │                           │                           │
        └───────────────────────────┼───────────────────────────┘
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                           Data Layer                                    │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │
│  │    D1       │  │   R2        │  │   KV        │  │   Queue     │  │
│  │  (SQLite)   │  │  (Object)   │  │  (Cache)    │  │  (Events)   │  │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Component Details

### Frontend (Dashboard)
- **Framework**: React 18+ with TypeScript
- **Styling**: Tailwind CSS
- **State**: React Query + Zustand
- **Charts**: Recharts / Chart.js
- **Deployment**: Cloudflare Pages

### API Layer
- **Runtime**: Cloudflare Workers (V8 isolate)
- **Router**: Hono / itty-router
- **Auth**: OAuth 2.0 (merchant connections)
- **Rate Limit**: Cloudflare Rate Limiting

### Optimization Engine
- **Language**: Python (Cloudflare Workers Python support)
- **ML Framework**: TensorFlow Lite (for edge inference)
- **Models**: Pre-trained authorization prediction

### A/B Platform
- **Runtime**: Node.js on Cloudflare Workers
- **Statistics**: Bayesian A/B testing
- **Allocation**: Consistent hashing-based traffic split

### 3DS Decisioner
- **Rules Engine**: Custom rules DSL
- **ML**: Risk scoring model
- **Integration**: EMVCo 3DS 2.3

---

## Data Model

### Key Tables (D1 / SQLite)

**merchants**
```sql
CREATE TABLE merchants (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  plan TEXT NOT NULL,
  stripe_account_id TEXT,
  adyen_merchant_account TEXT
);
```

**experiments**
```sql
CREATE TABLE experiments (
  id TEXT PRIMARY KEY,
  merchant_id TEXT REFERENCES merchants(id),
  name TEXT NOT NULL,
  status TEXT NOT NULL, -- draft, running, paused, completed
  traffic_split INTEGER DEFAULT 50,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  started_at TIMESTAMP,
  ended_at TIMESTAMP,
  winner_variant TEXT
);
```

**events**
```sql
CREATE TABLE events (
  id TEXT PRIMARY KEY,
  merchant_id TEXT REFERENCES merchants(id),
  experiment_id TEXT REFERENCES experiments(id),
  event_type TEXT NOT NULL, -- impression, conversion, payment_attempt
  variant TEXT,
  payload TEXT, -- JSON
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**recommendations**
```sql
CREATE TABLE recommendations (
  id TEXT PRIMARY KEY,
  merchant_id TEXT REFERENCES merchants(id),
  type TEXT NOT NULL, -- auth_optimization, cost_reduction, etc
  status TEXT NOT NULL, -- pending, applied, dismissed
  impact_estimate REAL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

---

## API Specification

### Authentication
- Bearer token (JWT)
- Merchant-scoped tokens
- OAuth flow for processor connections

### Endpoints

#### Recommendations
```
GET    /v1/optimize/recommendations
       Query: { status, type }
       Response: [{ id, type, impact_estimate, actions }]

POST   /v1/optimize/recommendations/:id/apply
       Response: { success, applied_at }
```

#### Experiments
```
GET    /v1/optimize/experiments
       Response: [{ id, name, status, variants, results }]

POST   /v1/optimize/experiments
       Body: { name, traffic_split, variants, success_metric }
       Response: { id, status }

PATCH  /v1/optimize/experiments/:id
       Body: { status } -- pause, resume, stop

POST   /v1/optimize/experiments/:id/rollout
       Body: { variant_id, rollout_percentage }
```

#### 3DS Decisioning
```
POST   /v1/optimize/3ds/decide
       Body: { transaction_id, amount, currency, card_bin, merchant_id }
       Response: { challenge_required, exemption_type, confidence }
```

#### Webhooks
```
POST   /v1/webhooks
       Body: { url, events, secret }
       Response: { id, status }

DELETE /v1/webhooks/:id
```

---

## Security & Compliance

- **PCI DSS**: SAQ A (no card data storage)
- **SOC 2**: planned for Enterprise tier
- **GDPR**: Data processing agreements available
- **Encryption**: TLS 1.3 for all connections

---

## Deployment

### Environments
- **Development**: `dev.tap2optimize.com`
- **Staging**: `staging.tap2optimize.com`
- **Production**: `optimize.tap2.com`

### CI/CD
- GitHub Actions for deployments
- Automated tests on PR
- Staged rollouts with monitoring

---

## Monitoring & Observability

- **Metrics**: Cloudflare Analytics
- **Logs**: Cloudflare Logpush
- **Errors**: Sentry integration
- **Uptime**: Pingdom / UptimeRobot
- **Analytics**: Plausible (privacy-friendly)

---

## Technology Stack Summary

| Layer | Technology |
|-------|------------|
| Frontend | React, TypeScript, Tailwind |
| API | Cloudflare Workers, Hono |
| Backend | Python, Node.js |
| Database | D1 (SQLite), R2, KV |
| ML | TensorFlow Lite |
| Auth | OAuth 2.0, JWT |
| Deploy | GitHub Actions, Wrangler |
