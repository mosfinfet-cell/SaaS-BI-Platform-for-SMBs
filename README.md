# Bharatvantage — SMB Business Intelligence Platform

> A Multi-tenant SaaS analytics platform built for small and medium businesses. Connect your data sources, monitor every metric that matters, forecast what's coming, and get alerted via WhatsApp, SMS, or email before problems become crises.

[![Python](https://img.shields.io/badge/Python-3.11+-3776AB?style=flat-square&logo=python&logoColor=white)](https://python.org)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.111-009688?style=flat-square&logo=fastapi&logoColor=white)](https://fastapi.tiangolo.com)
[![React](https://img.shields.io/badge/React-18.3-61DAFB?style=flat-square&logo=react&logoColor=black)](https://react.dev)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-4169E1?style=flat-square&logo=postgresql&logoColor=white)](https://postgresql.org)
[![Redis](https://img.shields.io/badge/Redis-7-DC382D?style=flat-square&logo=redis&logoColor=white)](https://redis.io)
[![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?style=flat-square&logo=docker&logoColor=white)](https://docker.com)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow?style=flat-square)](LICENSE)

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Features](#features)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Getting Started](#getting-started)
  - [Prerequisites](#prerequisites)
  - [Local Development Setup](#local-development-setup)
  - [Environment Variables](#environment-variables)
  - [Running with Docker Compose](#running-with-docker-compose)
  - [Running Without Docker](#running-without-docker)
- [Module Documentation](#module-documentation)
  - [Data Ingestion Engine](#1-data-ingestion-engine)
  - [Analytics Engine](#2-analytics-engine)
  - [Forecasting Service](#3-forecasting-service)
  - [Dashboard Framework](#4-dashboard-framework)
  - [Alert & Notification Engine](#5-alert--notification-engine)
  - [Multi-Tenant Architecture](#6-multi-tenant-architecture)
  - [Audit & Observability](#7-audit--observability)
- [API Reference](#api-reference)
- [Database Schema](#database-schema)
- [Forecasting Models](#forecasting-models)
- [Notification Channels](#notification-channels)
- [Dashboard Widgets](#dashboard-widgets)
- [Background Workers](#background-workers)
- [Testing](#testing)
- [Deployment](#deployment)
  - [Docker Compose (Production)](#docker-compose-production)
  - [Kubernetes](#kubernetes)
  - [Environment Checklist](#production-environment-checklist)
- [Subscription Tiers](#subscription-tiers)
- [Contributing](#contributing)
- [Roadmap](#roadmap)
- [License](#license)

---

## Overview

Bharatvantage is a full-stack SaaS platform that gives SMB owners and operators a single place to understand their business. It ingests data from Shopify, Stripe, QuickBooks, Xero, WooCommerce, and custom CSV/Excel uploads, then computes hundreds of metrics across revenue, customers, operations, and finance — and surfaces them through configurable dashboards.

The platform is built around four core capabilities:

| Capability | What it does |
|-----------|-------------|
| **Analytical Engine** | Pre-computes and caches revenue, customer (RFM/CLV/churn), operational, and financial metrics on a configurable schedule |
| **Forecasting Service** | Auto-selects the best statistical model (Prophet, SARIMA, Holt-Winters, or Linear) and produces forecasts with 80% and 95% confidence intervals |
| **Alert Engine** | Evaluates rules against live metrics and fires configurable alerts — with anomaly detection, quiet hours, digest mode, and retry logic |
| **Notification Delivery** | Delivers alerts via WhatsApp (Twilio), SMS, Email (SendGrid), and Firebase push — with delivery tracking and exponential-backoff retry |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                          Client Layer                               │
│   React 18 + TypeScript  ·  Recharts  ·  @dnd-kit  ·  Zustand      │
└───────────────────────────────┬─────────────────────────────────────┘
                                │ HTTPS / WebSocket
┌───────────────────────────────▼─────────────────────────────────────┐
│                         API Layer (FastAPI)                         │
│   JWT Auth  ·  Tenant Middleware  ·  Rate Limiter  ·  Audit Log     │
│   /auth  /datasources  /metrics  /forecasts  /alerts  /dashboards   │
└──────┬────────────────────┬──────────────────────┬──────────────────┘
       │                    │                      │
┌──────▼──────┐    ┌────────▼────────┐    ┌───────▼──────────────────┐
│  PostgreSQL │    │   Redis Cache   │    │    Celery Workers        │
│  16 + RLS   │    │   + Task Queue  │    │  Sync · Forecast · Alert │
│  Row-level  │    │   TTL: 5m–1hr   │    │  Celery Beat scheduler   │
│  security   │    └─────────────────┘    └──────────────────────────┘
└─────────────┘                                      │
                                           ┌─────────▼──────────────┐
                                           │  Notification Channels  │
                                           │  WhatsApp · SMS · Email │
                                           │  Firebase Push          │
                                           └────────────────────────┘
```

**Multi-tenancy model:** Every request is scoped to a `tenant_id` extracted from the JWT. The FastAPI tenant middleware sets a PostgreSQL session variable (`app.tenant_id`) which activates row-level security policies on all tenant-scoped tables. No query in the application can accidentally return cross-tenant data.

**Caching strategy:**
- **Layer 1 (hot):** Redis — 5-minute TTL for real-time metrics, 1-hour TTL for daily aggregates
- **Layer 2 (warm):** `metrics_cache` table — pre-computed at midnight by Celery Beat, used as fallback when Redis is cold

---

## Features

### Data Sources
- ✅ Shopify (orders, products, customers, inventory)
- ✅ WooCommerce (orders, products, customers)
- ✅ Stripe (payments, subscriptions, refunds, disputes)
- ✅ QuickBooks (invoices, expenses, payments, P&L)
- ✅ Xero (invoices, bank transactions, contacts)
- ✅ CSV / Excel upload with auto schema detection
- ✅ Webhook listeners for real-time event ingestion
- ✅ Incremental sync with change-data-capture (`last_synced_at`)

### Analytics
- ✅ Revenue: daily/weekly/monthly/YTD with period-over-period comparisons
- ✅ Revenue segmented by product, category, channel, geography, customer segment
- ✅ RFM segmentation (7 segments: Champions → Churned)
- ✅ Customer Lifetime Value (historical + predicted)
- ✅ Cohort retention matrix with configurable cohort definitions
- ✅ Churn probability scoring per customer (GBM, retrained weekly)
- ✅ Inventory turnover, stockout frequency, reorder point alerts
- ✅ Fulfilment rate, delivery time, return rate
- ✅ Cash flow projection (rolling 13-week)
- ✅ Accounts receivable aging buckets
- ✅ Budget vs actual variance

### Forecasting
- ✅ Prophet (seasonality, holiday effects, configurable per country)
- ✅ SARIMA (for periodic time series)
- ✅ Holt-Winters (short history fallback)
- ✅ Linear regression with feature engineering
- ✅ Auto model selection based on cross-validated MAE
- ✅ 7, 30, and 90-day horizons with 80% and 95% confidence intervals

### Dashboards
- ✅ 9 widget types: KPI card, line, bar, area, scatter, heatmap, table, gauge, funnel
- ✅ Drag-and-drop dashboard builder
- ✅ 5 pre-built templates: Executive Overview, Revenue, Customer Health, Inventory, Finance
- ✅ Global date range filter cascading to all widgets
- ✅ Comparison periods: vs last period, vs last year
- ✅ PDF export (full dashboard) and CSV export (per widget)

### Alerts & Notifications
- ✅ Configurable rules: `IF [metric] [operator] [threshold] FOR [N periods]`
- ✅ Anomaly detection: IQR-based and Isolation Forest
- ✅ WhatsApp (Twilio Business API)
- ✅ SMS (Twilio)
- ✅ Email (SendGrid)
- ✅ Firebase push notifications
- ✅ Quiet hours, digest mode, delivery tracking, retry with backoff
- ✅ 4 built-in default alerts (revenue drop, churn threshold, cash runway, stockout)

---

## Tech Stack

| Layer | Technology | Version |
|-------|-----------|---------|
| Backend API | FastAPI | 0.111.0 |
| Database | PostgreSQL | 16 |
| ORM | SQLAlchemy (async) | 2.0.30 |
| Migrations | Alembic | 1.13.1 |
| Task Queue | Celery + Redis | 5.4.0 |
| Caching | Redis | 7 |
| Auth | python-jose + passlib | 3.3.0 / 1.7.4 |
| Validation | Pydantic v2 | 2.7.1 |
| Logging | structlog | 24.1.0 |
| Analytics | pandas, polars, scikit-learn | 2.2.2 / 0.20.21 / 1.4.2 |
| Forecasting | Prophet, statsmodels | 1.1.5 / 0.14.2 |
| HTTP client | httpx | 0.27.0 |
| WhatsApp/SMS | Twilio | 9.0.4 |
| Email | SendGrid | 6.11.0 |
| Push | firebase-admin | 6.5.0 |
| Frontend | React + TypeScript | 18.3 / 5.4 |
| Charts | Recharts | 2.12 |
| Drag-and-drop | @dnd-kit/core | 6.1 |
| Server state | @tanstack/react-query | 5.40 |
| Client state | Zustand | 4.5 |
| Styling | TailwindCSS | 3.4 |
| Container | Docker + Docker Compose | — |
| Orchestration | Kubernetes | — |

---

## Project Structure

```
bizpulse/
├── backend/
│   ├── app/
│   │   ├── main.py                      # FastAPI app factory, lifespan, middleware registration
│   │   ├── config.py                    # pydantic-settings with validation
│   │   ├── dependencies.py              # get_db, get_current_user, get_current_tenant
│   │   ├── middleware/
│   │   │   ├── tenant.py                # Extract tenant_id from JWT, set RLS session var
│   │   │   ├── logging.py               # Attach correlation_id to every request
│   │   │   └── rate_limit.py            # Redis sliding window (100 req/min per tenant)
│   │   ├── auth/
│   │   │   ├── router.py                # /auth/login, /auth/refresh, /auth/logout
│   │   │   ├── service.py               # Auth business logic
│   │   │   ├── schemas.py               # LoginRequest, TokenResponse, RefreshRequest
│   │   │   └── security.py              # JWT encode/decode, Fernet encryption, bcrypt
│   │   ├── tenants/
│   │   │   ├── router.py                # Tenant CRUD, branding, user management
│   │   │   ├── service.py
│   │   │   ├── schemas.py
│   │   │   └── models.py
│   │   ├── datasources/
│   │   │   ├── router.py                # CRUD + sync trigger + webhook receiver
│   │   │   ├── service.py
│   │   │   ├── schemas.py
│   │   │   ├── models.py
│   │   │   └── connectors/
│   │   │       ├── base.py              # Abstract BaseConnector (test, schema, sync, webhooks)
│   │   │       ├── shopify.py           # Reference implementation
│   │   │       ├── woocommerce.py
│   │   │       ├── stripe.py
│   │   │       ├── quickbooks.py
│   │   │       ├── xero.py
│   │   │       └── csv_upload.py        # Encoding detect, polars parse, quality report
│   │   ├── analytics/
│   │   │   ├── router.py                # GET /metrics/{metric_name}
│   │   │   ├── service.py               # Orchestrates cache + compute
│   │   │   ├── schemas.py               # MetricResult, SeriesData, SummaryStats
│   │   │   ├── revenue.py               # Revenue trend, AOV, margin, concentration
│   │   │   ├── customers.py             # RFM, CLV, cohort matrix, churn scoring
│   │   │   ├── operations.py            # Inventory, fulfilment, returns
│   │   │   ├── financial.py             # Cash flow, AR aging, burn rate, budget vs actual
│   │   │   └── cache.py                 # Redis + metrics_cache read/write helpers
│   │   ├── forecasting/
│   │   │   ├── router.py                # GET /forecasts/{metric_name}
│   │   │   ├── service.py               # Model selection, training, inference
│   │   │   ├── schemas.py               # ForecastResult, ConfidenceInterval
│   │   │   ├── selector.py              # Auto-select logic (FFT seasonality test + MAE eval)
│   │   │   └── models/
│   │   │       ├── base.py              # Abstract ForecastModel
│   │   │       ├── prophet_model.py     # Prophet with country holidays
│   │   │       ├── sarima_model.py      # SARIMA with auto-detected order
│   │   │       ├── holt_winters.py      # Exponential smoothing fallback
│   │   │       └── linear_model.py      # Linear regression with feature engineering
│   │   ├── alerts/
│   │   │   ├── router.py                # CRUD alert rules + manual test trigger
│   │   │   ├── service.py
│   │   │   ├── schemas.py               # AlertRule, AlertEvent, AnomalyResult
│   │   │   ├── models.py
│   │   │   ├── evaluator.py             # Rule evaluation: threshold, period check, cooldown
│   │   │   └── anomaly.py               # IQR detector + Isolation Forest detector
│   │   ├── notifications/
│   │   │   ├── router.py                # GET /notifications/log
│   │   │   ├── service.py               # Dispatch, retry, digest batching
│   │   │   ├── schemas.py
│   │   │   ├── channels/
│   │   │   │   ├── base.py              # Abstract NotificationChannel
│   │   │   │   ├── whatsapp.py          # Twilio WhatsApp Business API
│   │   │   │   ├── sms.py               # Twilio SMS with 160-char truncation
│   │   │   │   ├── email.py             # SendGrid with HTML + text templates
│   │   │   │   └── push.py              # Firebase Cloud Messaging
│   │   │   └── templates/               # Jinja2 .j2 templates per alert type
│   │   ├── dashboards/
│   │   │   ├── router.py                # CRUD + export (PDF/CSV)
│   │   │   ├── service.py
│   │   │   ├── schemas.py               # DashboardConfig, WidgetConfig, ExportRequest
│   │   │   └── models.py
│   │   ├── workers/
│   │   │   ├── celery_app.py            # App factory + Beat schedule
│   │   │   ├── sync_tasks.py            # Per-source incremental sync
│   │   │   ├── forecast_tasks.py        # Weekly retraining per tenant per metric
│   │   │   ├── alert_tasks.py           # Every-15-min rule evaluation sweep
│   │   │   └── metrics_tasks.py         # Midnight pre-compute to metrics_cache
│   │   └── db/
│   │       ├── session.py               # Async engine + session factory + RLS setup
│   │       ├── base.py                  # DeclarativeBase + TenantMixin
│   │       └── migrations/
│   │           ├── env.py               # Alembic env (async)
│   │           └── versions/            # Migration files (auto-generated)
│   ├── tests/
│   │   ├── conftest.py                  # Pytest fixtures: test DB, auth headers, tenant
│   │   ├── test_auth.py
│   │   ├── test_analytics.py            # Pure unit tests — no DB, fast
│   │   ├── test_forecasting.py          # Model fit + output shape tests
│   │   ├── test_alerts.py               # Evaluator logic + anomaly detection
│   │   └── test_api/                    # FastAPI TestClient integration tests
│   ├── requirements.txt
│   ├── requirements-dev.txt
│   └── Dockerfile
├── frontend/
│   ├── src/
│   │   ├── app/                         # Next.js app router pages
│   │   │   ├── dashboard/[id]/
│   │   │   ├── analytics/
│   │   │   ├── forecasts/
│   │   │   ├── alerts/
│   │   │   ├── datasources/
│   │   │   └── settings/
│   │   ├── components/
│   │   │   ├── dashboard/
│   │   │   │   ├── DashboardCanvas.tsx  # @dnd-kit drag-and-drop layout
│   │   │   │   ├── WidgetWrapper.tsx    # Loading, error, resize states
│   │   │   │   └── DashboardToolbar.tsx # Date range, export, add widget
│   │   │   ├── widgets/
│   │   │   │   ├── KPICard.tsx
│   │   │   │   ├── LineChartWidget.tsx  # Includes forecast overlay
│   │   │   │   ├── BarChartWidget.tsx
│   │   │   │   ├── AreaChartWidget.tsx
│   │   │   │   ├── ScatterWidget.tsx
│   │   │   │   ├── HeatmapWidget.tsx    # Cohort retention matrix
│   │   │   │   ├── DataTableWidget.tsx  # Sort, filter, paginate, CSV export
│   │   │   │   ├── GaugeWidget.tsx
│   │   │   │   └── FunnelWidget.tsx
│   │   │   ├── alerts/
│   │   │   │   ├── AlertRuleBuilder.tsx # IF/THEN rule form
│   │   │   │   └── NotificationLog.tsx
│   │   │   └── shared/
│   │   │       ├── DateRangePicker.tsx
│   │   │       ├── SegmentSelector.tsx
│   │   │       ├── GranularityToggle.tsx
│   │   │       └── TenantBranding.tsx
│   │   ├── hooks/
│   │   │   ├── useMetric.ts             # Wraps React Query for /metrics/*
│   │   │   ├── useForecast.ts
│   │   │   ├── useAlerts.ts
│   │   │   └── useExport.ts
│   │   ├── stores/
│   │   │   ├── authStore.ts             # JWT token, user, tenant
│   │   │   ├── dashboardStore.ts        # Active date range, widget layout state
│   │   │   └── notificationStore.ts     # In-app notification queue
│   │   ├── api/
│   │   │   ├── client.ts                # Axios instance with JWT interceptor
│   │   │   ├── metrics.ts
│   │   │   ├── forecasts.ts
│   │   │   ├── alerts.ts
│   │   │   └── dashboards.ts
│   │   └── types/
│   │       ├── metrics.ts               # Mirrors MetricResponse schema
│   │       ├── forecasts.ts
│   │       ├── alerts.ts
│   │       └── dashboard.ts
│   ├── public/
│   ├── package.json
│   ├── tsconfig.json
│   ├── tailwind.config.ts
│   └── Dockerfile
├── k8s/
│   ├── namespace.yaml
│   ├── configmap.yaml
│   ├── secret.yaml                      # Template — never commit with real values
│   ├── postgres/
│   │   ├── statefulset.yaml
│   │   ├── service.yaml
│   │   └── pvc.yaml
│   ├── redis/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   ├── api/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── hpa.yaml                     # Horizontal pod autoscaler
│   ├── worker/
│   │   └── deployment.yaml
│   ├── beat/
│   │   └── deployment.yaml              # Single replica — beat must not run in parallel
│   └── ingress.yaml
├── docker-compose.yml                   # Local development
├── docker-compose.prod.yml              # Production override
├── .env.example
├── .gitignore
├── Makefile                             # Common commands
└── README.md
```

---

## Getting Started

### Prerequisites

| Tool | Minimum Version | Install |
|------|----------------|---------|
| Docker Desktop | 4.x | [docker.com](https://docker.com) |
| Docker Compose | 2.x | Included with Docker Desktop |
| Python | 3.11 | [python.org](https://python.org) |
| Node.js | 20 LTS | [nodejs.org](https://nodejs.org) |
| Make | any | `brew install make` (macOS) |

External accounts required:
- **Twilio** — for WhatsApp and SMS (free trial available). Obtain `ACCOUNT_SID`, `AUTH_TOKEN`, `WHATSAPP_FROM`, `SMS_FROM`.
- **SendGrid** — for email (free tier: 100 emails/day). Obtain `API_KEY`.
- **Firebase** — for push notifications (free). Download service account JSON.
- At least one data source account (Shopify, Stripe, etc.) for end-to-end testing. CSV upload works with no external accounts.

---

### Local Development Setup

```bash
# 1. Clone the repository
git clone https://github.com/your-org/bizpulse.git
cd bizpulse

# 2. Copy and configure environment variables
cp .env.example .env
# Edit .env — see Environment Variables section below

# 3. Generate required secrets
# Fernet key (for API key encryption):
python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"
# Paste output into ENCRYPTION_KEY in .env

# JWT secret (32+ character random string):
python -c "import secrets; print(secrets.token_hex(32))"
# Paste output into SECRET_KEY in .env

# 4. Start all services
make dev
# Equivalent to: docker-compose up --build

# 5. Run database migrations
make migrate
# Equivalent to: docker-compose exec api alembic upgrade head

# 6. Seed initial data (optional — creates demo tenant + sample data)
make seed

# 7. Open the application
# Frontend:  http://localhost:3000
# API docs:  http://localhost:8000/docs
# Flower:    http://localhost:5555  (Celery monitoring)
# pgAdmin:   docker-compose --profile tools up  →  http://localhost:5050
```

---

### Environment Variables

Copy `.env.example` to `.env` and fill in every value. Never commit `.env` to version control.

```bash
# ── Application ────────────────────────────────────────────────────────────
ENVIRONMENT=development                  # development | staging | production
SECRET_KEY=your-32-char-minimum-secret  # JWT signing key — generate with secrets.token_hex(32)
ACCESS_TOKEN_EXPIRE_MINUTES=30
REFRESH_TOKEN_EXPIRE_DAYS=7
ENCRYPTION_KEY=                          # Fernet key — generate with Fernet.generate_key()

# ── Database ───────────────────────────────────────────────────────────────
DATABASE_URL=postgresql+asyncpg://bizpulse:bizpulse@localhost:5432/bizpulse
# For Docker: postgresql+asyncpg://bizpulse:bizpulse@postgres:5432/bizpulse

# ── Redis ─────────────────────────────────────────────────────────────────
REDIS_URL=redis://localhost:6379/0
# For Docker: redis://redis:6379/0

# ── Twilio (WhatsApp + SMS) ────────────────────────────────────────────────
TWILIO_ACCOUNT_SID=ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
TWILIO_AUTH_TOKEN=your_auth_token
TWILIO_WHATSAPP_FROM=whatsapp:+14155238886   # Sandbox number for testing
TWILIO_SMS_FROM=+1XXXXXXXXXX                 # Your Twilio phone number

# ── SendGrid (Email) ──────────────────────────────────────────────────────
SENDGRID_API_KEY=SG.xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
SENDGRID_FROM_EMAIL=alerts@yourdomain.com
SENDGRID_FROM_NAME=BizPulse Alerts

# ── Firebase (Push Notifications) ─────────────────────────────────────────
FIREBASE_CREDENTIALS_PATH=/app/firebase-credentials.json
# Place your Firebase service account JSON at this path (not committed to git)

# ── Celery ────────────────────────────────────────────────────────────────
CELERY_BROKER_URL=redis://localhost:6379/1
CELERY_RESULT_BACKEND=redis://localhost:6379/2

# ── Rate Limiting ─────────────────────────────────────────────────────────
RATE_LIMIT_PER_MINUTE=100

# ── Forecasting ───────────────────────────────────────────────────────────
FORECAST_RETRAIN_DAY_OF_WEEK=1            # 1=Monday, runs weekly
FORECAST_MIN_HISTORY_DAYS=60             # Minimum days required to attempt forecasting

# ── File Upload ───────────────────────────────────────────────────────────
MAX_UPLOAD_SIZE_MB=50
UPLOAD_STORAGE_PATH=/tmp/uploads          # Use S3 path in production

# ── Flower (Celery Monitoring) ────────────────────────────────────────────
FLOWER_USER=admin
FLOWER_PASSWORD=change-me-in-production
```

> **WhatsApp sandbox:** For development and testing, use the Twilio WhatsApp Sandbox. The `FROM` number `whatsapp:+14155238886` is Twilio's shared sandbox. Recipients must opt in by sending a join code. For production, you need an approved WhatsApp Business account.

---

### Running with Docker Compose

```bash
# Start all services (builds images on first run)
docker-compose up --build

# Start in background
docker-compose up -d

# View logs
docker-compose logs -f api
docker-compose logs -f worker

# Stop all services
docker-compose down

# Stop and remove volumes (⚠️ deletes all data)
docker-compose down -v

# Run with optional tools (pgAdmin)
docker-compose --profile tools up

# Scale workers (when load increases)
docker-compose up -d --scale worker=3
```

---

### Running Without Docker

```bash
# ── Backend ────────────────────────────────────────────────────────────────

# Create virtual environment
cd backend
python -m venv venv
source venv/bin/activate   # Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
pip install -r requirements-dev.txt   # Development only

# Start PostgreSQL and Redis locally (via Homebrew or native install)
# Then update DATABASE_URL and REDIS_URL in .env to use localhost

# Run migrations
alembic upgrade head

# Start API server
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000

# Start Celery worker (separate terminal)
celery -A app.workers.celery_app worker --loglevel=info --concurrency=4

# Start Celery Beat scheduler (separate terminal — never run with worker in production)
celery -A app.workers.celery_app beat --loglevel=info

# ── Frontend ───────────────────────────────────────────────────────────────

cd frontend
npm install
npm run dev   # Starts on http://localhost:3000
```

---

## Module Documentation

### 1. Data Ingestion Engine

Handles connecting to external data sources, syncing data incrementally, and processing file uploads.

**How syncing works:**

1. User adds a data source (e.g. Shopify) and provides credentials via the UI
2. Credentials are encrypted with Fernet and stored in `api_keys`
3. A Celery task triggers an initial full sync, then incremental syncs every configurable interval
4. Each connector yields records in batches of 500, which are upserted into a per-tenant raw data table
5. After sync, `sync_jobs` is updated with status, row count, and any errors

**Incremental sync logic:**

```
On each sync run:
  1. Read last_synced_at from data_sources table
  2. Call connector.sync_incremental(table, last_synced_at)
  3. Connector fetches only records modified/created since last_synced_at
  4. Upsert records using ON CONFLICT (external_id) DO UPDATE
  5. Update last_synced_at to sync start time (not end time — prevents gaps)
```

**CSV upload pipeline:**

```
Upload → Detect encoding (chardet) → Parse with polars
       → Infer column types on first 1000 rows
       → Run quality checks:
           - Null rates per column
           - Outlier detection (Z-score > 3) per numeric column
           - Duplicate row detection
           - Referential integrity checks (if FK columns detected)
       → Return quality report to frontend
       → User confirms → Insert to raw_data table → Store schema
```

**Adding a new connector:**

Subclass `BaseConnector` in `app/datasources/connectors/base.py` and implement four methods:

```python
class MyConnector(BaseConnector):
    async def test_connection(self) -> ConnectionTestResult: ...
    async def fetch_schema(self) -> list[TableSchema]: ...
    async def sync_incremental(self, table, last_synced_at, batch_size) -> AsyncIterator[list[dict]]: ...
    async def get_webhook_events(self) -> list[WebhookEventType]: ...
```

Register it in `app/datasources/service.py` and add it to the `ConnectorType` enum.

---

### 2. Analytics Engine

The analytics engine computes metrics on demand (via the API) and on schedule (via Celery Beat). All computed metrics are cached in Redis and pre-computed to `metrics_cache` at midnight.

**Available metrics:**

| Category | Metric | Endpoint |
|----------|--------|----------|
| Revenue | Daily / weekly / monthly trend | `revenue_trend` |
| Revenue | Average order value | `aov_trend` |
| Revenue | By segment (product/category/channel/geo) | `revenue_by_segment` |
| Revenue | Gross margin by SKU | `gross_margin` |
| Revenue | Top-10 customer concentration | `revenue_concentration` |
| Customer | RFM segment distribution | `rfm_segments` |
| Customer | Customer lifetime value | `clv` |
| Customer | Cohort retention matrix | `cohort_retention` |
| Customer | Churn probability per customer | `churn_scores` |
| Customer | New vs returning ratio | `new_vs_returning` |
| Operations | Inventory turnover | `inventory_turnover` |
| Operations | Fulfilment rate | `fulfilment_rate` |
| Operations | Average delivery time | `delivery_time` |
| Operations | Return rate | `return_rate` |
| Financial | Cash flow projection (13-week) | `cash_flow` |
| Financial | AR aging buckets | `ar_aging` |
| Financial | Burn rate + runway | `burn_rate` |
| Financial | Budget vs actual | `budget_variance` |

**RFM Segmentation logic:**

```
Segments (first matching rule wins):
  Champions:            R >= 4 AND F >= 4
  Loyal:                R >= 3 AND F >= 3
  Potential Loyalists:  R >= 3 AND F <= 2
  New Customers:        R >= 4 AND F == 1
  At Risk:              R <= 2 AND F >= 3
  Hibernating:          R == 2 AND F == 2
  Churned:              R <= 2 AND F <= 2  (catch-all)

Scoring:
  Recency:   quintile with reversed labels [5,4,3,2,1] (lower days = better)
  Frequency: rank(method='first') then qcut — handles ties correctly
  Monetary:  standard quintile [1,2,3,4,5]
```

**Querying a metric:**

```bash
GET /api/v1/metrics/revenue_trend
  ?start_date=2024-01-01
  &end_date=2024-03-31
  &granularity=weekly
  &segment_by=category
  &compare_to=previous_period
```

```json
{
  "metric_name": "revenue_trend",
  "granularity": "weekly",
  "series": [
    {
      "label": "Electronics",
      "data": [
        {"date": "2024-01-07", "value": 12450.00},
        {"date": "2024-01-14", "value": 13820.00}
      ]
    }
  ],
  "comparison": {
    "label": "Previous Period",
    "data": [...],
    "change_pct": 11.0
  },
  "summary": {
    "total": 284300.00,
    "average": 9476.67,
    "trend": "up",
    "trend_pct": 11.0
  },
  "cache_info": {
    "cached": true,
    "computed_at": "2024-03-31T00:02:14Z",
    "ttl_seconds": 3298
  }
}
```

---

### 3. Forecasting Service

Automatically selects the best model for each metric and tenant based on data characteristics and cross-validation performance.

**Model selection decision tree:**

```
IF history_length < 60 days
    → Holt-Winters (insufficient data for seasonal decomposition)

ELSE IF FFT test detects strong weekly periodicity
    IF history_length >= 365 days
        → Prophet (enough data for yearly seasonality + holiday effects)
    ELSE
        → SARIMA(1,1,1)(1,1,1,7)

ELSE
    → Linear Regression with engineered features

Final check:
    Evaluate all fitted models on last-30-days holdout
    If alternative model has >15% better MAE → override selection
    Log all MAE values to model_evaluation_log
```

**Retraining schedule:**

Models retrain weekly (Monday by default) via Celery Beat. Each retraining run:
1. Fetches latest data for the metric
2. Fits all candidate models
3. Evaluates on 30-day holdout
4. Saves best model artifact (joblib) to `forecast_models` table (versioned)
5. Generates new forecasts for 7, 30, and 90-day horizons
6. Stores all forecast points + confidence intervals in `forecast_results`

**Confidence intervals:**

All forecasts include 80% and 95% confidence intervals. For non-negative metrics (revenue, orders, inventory), lower bounds are clipped to 0.

---

### 4. Dashboard Framework

**Widget configuration:**

Every widget is configured as a JSON object stored in `widget_configs`:

```json
{
  "widget_id": "uuid",
  "type": "line_chart",
  "title": "Weekly Revenue",
  "metric_name": "revenue_trend",
  "config": {
    "granularity": "weekly",
    "compare_to": "previous_year",
    "show_forecast": true,
    "forecast_horizon_days": 30
  },
  "layout": {
    "x": 0, "y": 0,
    "w": 8, "h": 4
  }
}
```

**Pre-built templates:**

| Template | Description | Widget count |
|----------|------------|-------------|
| Executive Overview | KPIs, revenue trend, segment health, churn | 6 |
| Revenue Deep Dive | Category breakdown, AOV, margin, concentration, forecast | 6 |
| Customer Health | RFM distribution, cohort heatmap, CLV, churn scores | 5 |
| Inventory Status | Turnover, stockouts, reorder alerts, return rate | 5 |
| Financial Summary | Cash flow, AR aging, burn rate, budget vs actual | 5 |

**Exporting:**

```bash
# Export full dashboard to PDF
POST /api/v1/dashboards/{id}/export
{"format": "pdf", "date_range": {"start": "2024-01-01", "end": "2024-03-31"}}

# Export individual widget data to CSV
GET /api/v1/metrics/revenue_trend?...&format=csv
```

---

### 5. Alert & Notification Engine

**Creating an alert rule:**

```bash
POST /api/v1/alerts/rules
```

```json
{
  "name": "Revenue Drop Alert",
  "metric_name": "revenue_trend",
  "operator": "pct_change_lt",
  "threshold": -20,
  "consecutive_periods": 1,
  "cooldown_minutes": 60,
  "recipients": ["user-uuid-1", "user-uuid-2"],
  "channels": ["whatsapp", "email"],
  "message_template": "revenue_drop"
}
```

**Available operators:**

| Operator | Meaning |
|----------|---------|
| `gt` | Greater than threshold |
| `lt` | Less than threshold |
| `gte` | Greater than or equal |
| `lte` | Less than or equal |
| `pct_change_gt` | % change greater than threshold |
| `pct_change_lt` | % change less than threshold |
| `anomaly` | Value flagged as anomaly by detector |

**Default alerts (enabled automatically on tenant creation):**

| Alert | Condition | Channels |
|-------|-----------|---------|
| Revenue Drop | Week-over-week change < -20% | WhatsApp + Email |
| High Churn Rate | Churn rate > configurable threshold | Email |
| Low Cash Runway | Runway < 8 weeks | WhatsApp + SMS + Email |
| Stockout Imminent | Days of inventory < reorder point | SMS + Email |

**Notification delivery flow:**

```
Alert fires
  → Check quiet hours for each recipient (convert to their timezone)
  → If digest_mode: add to hourly digest queue
  → Else: render Jinja2 template with metric values
     → Dispatch to each channel concurrently
     → Log to notification_log with status
     → If delivery fails: retry up to 3 times with exponential backoff
       (1 min → 5 min → 25 min)
```

---

### 6. Multi-Tenant Architecture

**Tenant isolation model:**

- Every database table has a `tenant_id` UUID foreign key
- Row-level security (RLS) policies are enabled on all tenant-scoped tables
- The FastAPI middleware sets `SET LOCAL app.tenant_id = '{id}'` on every connection checkout, activating the RLS policies automatically
- No application-level query needs to include a `WHERE tenant_id = ?` clause — the database enforces it

**User roles and permissions:**

| Role | Permissions |
|------|------------|
| Owner | Full access including billing, tenant settings, user deletion |
| Admin | Manage users, data sources, alert rules. Cannot manage billing. |
| Analyst | View all dashboards, create/edit alerts, export data |
| Viewer | View dashboards only. Cannot export, create alerts, or access settings. |

**Subscription tiers:**

See [Subscription Tiers](#subscription-tiers) section.

---

### 7. Audit & Observability

**Audit log entries are created for:**
- User login / logout / failed login
- Data source created, updated, deleted, sync triggered
- Alert rule created, updated, deleted, triggered
- Dashboard exported
- API key created or rotated
- User role changed
- Tenant settings changed

**Accessing audit logs:**

```bash
GET /api/v1/audit/log?start_date=2024-01-01&end_date=2024-03-31&action=alert.triggered
```

**Health check endpoints:**

```bash
GET /health/live   # Liveness: returns 200 if process is running
GET /health/ready  # Readiness: checks DB connection, Redis connection, worker reachability
GET /health/connectors  # Per-connector status for configured data sources
```

**Structured log format (structlog):**

Every log line includes:
```json
{
  "timestamp": "2024-03-31T14:22:01.234Z",
  "level": "info",
  "event": "metric_computed",
  "correlation_id": "550e8400-e29b-41d4-a716-446655440000",
  "tenant_id": "tenant-uuid",
  "user_id": "user-uuid",
  "metric_name": "revenue_trend",
  "duration_ms": 142,
  "cached": false
}
```

---

## API Reference

Full interactive documentation is available at `http://localhost:8000/docs` (Swagger UI) and `http://localhost:8000/redoc` (ReDoc).

### Authentication

```bash
# Login
POST /api/v1/auth/login
Body: {"email": "user@example.com", "password": "secret"}
Response: {"access_token": "...", "token_type": "bearer"}
# Refresh token is set as httpOnly cookie

# Refresh access token
POST /api/v1/auth/refresh
# Uses refresh token from cookie

# Logout
POST /api/v1/auth/logout
# Invalidates refresh token in Redis
```

### Core Endpoints

```
POST   /api/v1/auth/login
POST   /api/v1/auth/refresh
POST   /api/v1/auth/logout

GET    /api/v1/datasources
POST   /api/v1/datasources
GET    /api/v1/datasources/{id}
PUT    /api/v1/datasources/{id}
DELETE /api/v1/datasources/{id}
POST   /api/v1/datasources/{id}/sync
GET    /api/v1/datasources/{id}/status
POST   /api/v1/datasources/upload          # CSV/Excel upload

GET    /api/v1/metrics/{metric_name}        # Query any metric

GET    /api/v1/forecasts/{metric_name}      # Latest forecast + confidence intervals

GET    /api/v1/alerts/rules
POST   /api/v1/alerts/rules
PUT    /api/v1/alerts/rules/{id}
DELETE /api/v1/alerts/rules/{id}
POST   /api/v1/alerts/rules/{id}/test       # Manually trigger alert for testing
GET    /api/v1/alerts/events               # Alert fire history

GET    /api/v1/notifications/log

GET    /api/v1/dashboards
POST   /api/v1/dashboards
GET    /api/v1/dashboards/{id}
PUT    /api/v1/dashboards/{id}
DELETE /api/v1/dashboards/{id}
POST   /api/v1/dashboards/{id}/export

GET    /api/v1/audit/log
GET    /health/live
GET    /health/ready
GET    /health/connectors
```

All endpoints (except `/health/*`):
- Require `Authorization: Bearer {access_token}` header
- Are rate-limited to 100 requests/minute per tenant
- Return a `X-Correlation-ID` header for request tracing
- Return structured errors on failure: `{"error": {"code": "...", "message": "...", "correlation_id": "..."}}`

---

## Database Schema

Key tables and their purpose:

```sql
-- Core tenancy
tenants             -- Tenant record, branding, country_code
users               -- User accounts (scoped to tenant)
user_roles          -- Role assignments (owner/admin/analyst/viewer)
api_keys            -- Encrypted external API credentials
subscription_plans  -- Tier definitions (limits, features)
tenant_usage        -- Daily API call + storage snapshots for billing

-- Data ingestion
data_sources        -- Configured connectors per tenant
connector_schemas   -- Inferred column types per source
sync_jobs           -- Sync run history (status, rows, errors)
sync_errors         -- Per-row errors from failed syncs

-- Analytics
metrics_cache       -- Pre-computed daily metrics (partitioned by tenant_id)
customer_segments   -- RFM scores + segment assignments + churn scores

-- Forecasting
forecast_models     -- Serialised model artifacts (joblib), versioned
forecast_results    -- Forecast points + confidence intervals per metric
model_evaluation_log -- MAE/RMSE/MAPE per model per training run

-- Alerts
alert_rules         -- Rule definitions (metric, operator, threshold, recipients)
alert_events        -- Alert fire history
notification_log    -- Every notification attempt (status, channel, retry count)
notification_preferences -- Per-user channel, quiet hours, digest mode

-- Dashboards
dashboard_configs   -- Dashboard metadata + layout
widget_configs      -- Per-widget type, metric, config, position

-- Observability
audit_log           -- Every significant action (partitioned by created_at monthly)
query_performance   -- p95 query times per widget type
```

All tables include: `id UUID PRIMARY KEY`, `tenant_id UUID NOT NULL`, `created_at TIMESTAMPTZ`, `updated_at TIMESTAMPTZ`.

Row-level security is enabled on all tenant-scoped tables. `metrics_cache` and `audit_log` are range/list-partitioned for query performance.

---

## Forecasting Models

| Model | Best for | Minimum history | Handles seasonality |
|-------|---------|----------------|-------------------|
| **Prophet** | Revenue, demand with holiday effects | 365 days | Yes (weekly + yearly) |
| **SARIMA** | Periodic data with clear cycles | 90 days | Yes (configurable period) |
| **Holt-Winters** | Short history, simple trends | 14 days | Yes (additive/multiplicative) |
| **Linear Regression** | Linear trends, low-variance data | 30 days | Via engineered features |

Model artifacts are stored in the database (serialised with joblib) and versioned by `created_at`. The API always serves the most recent model for each metric.

---

## Notification Channels

| Channel | Provider | Use case | Character limit |
|---------|----------|---------|----------------|
| **WhatsApp** | Twilio Business API | High-priority alerts — high open rates | ~1600 chars |
| **SMS** | Twilio | Critical alerts when WhatsApp unavailable | 160 chars |
| **Email** | SendGrid | Detailed alerts with charts and context | Unlimited |
| **Push** | Firebase FCM | In-app and mobile alerts | ~240 chars |

**WhatsApp setup:**

For development, use the Twilio WhatsApp Sandbox. For production, you need a Facebook Business account and an approved WhatsApp Business API account. The `FROM` number format must be `whatsapp:+[country_code][number]`.

**SMS truncation:**

SMS messages are automatically truncated to 160 characters. Dashboard links are shortened to fit. The full message is always sent via email regardless.

---

## Dashboard Widgets

| Widget | Best for | Config options |
|--------|---------|---------------|
| **KPI Card** | Single metric at a glance | Comparison period, trend indicator, colour threshold |
| **Line Chart** | Trends over time, forecasts | Multiple series, forecast overlay, CI bands |
| **Bar Chart** | Category comparisons | Stacked, grouped, horizontal |
| **Area Chart** | Cumulative trends, part-of-whole | Stacked area |
| **Scatter Plot** | Correlation between two metrics | Colour and size by third dimension |
| **Heatmap** | Cohort retention matrix | Custom colour scale, annotation toggle |
| **Data Table** | Ranked lists (top products, customers) | Sort, filter, pagination, CSV export |
| **Gauge** | Progress toward a target | Min, max, target value, colour zones |
| **Funnel** | Conversion rates across stages | Stage labels, conversion % display |

---

## Background Workers

Celery workers handle four categories of async work:

| Task | Schedule | Description |
|------|---------|-------------|
| `sync_all_datasources` | Every 1 hour | Triggers incremental sync for all active connectors |
| `precompute_metrics` | Daily at midnight | Computes and caches daily metrics for all tenants |
| `retrain_forecast_models` | Weekly (Monday) | Refits models for all active metrics per tenant |
| `evaluate_alert_rules` | Every 15 minutes | Checks all rules against current metric values |
| `send_notification_digests` | Every hour | Sends batched alerts for users in digest mode |
| `cleanup_old_forecasts` | Weekly | Deletes forecast results older than 90 days |
| `snapshot_tenant_usage` | Daily at 23:55 | Snapshots API calls + storage for billing |

**Celery Beat runs as a single dedicated container.** Never run `beat` and `worker` in the same process in production — this can cause duplicate task execution.

**Task idempotency:** Every task acquires a Redis lock (`SET NX EX`) before execution. If the lock is held, the task exits immediately. This prevents duplicate runs when workers are scaled horizontally.

---

## Testing

```bash
# Run all tests
make test

# Backend only
cd backend
pytest tests/ -v

# With coverage report
pytest tests/ --cov=app --cov-report=html
open htmlcov/index.html

# Run specific module tests
pytest tests/test_analytics.py -v
pytest tests/test_forecasting.py -v

# API integration tests (requires running PostgreSQL and Redis)
pytest tests/test_api/ -v --integration

# Frontend tests
cd frontend
npm run test           # Vitest unit tests
npm run test:e2e       # Playwright end-to-end tests
```

**Test strategy:**

- `test_analytics.py` — pure unit tests for all metric calculations. No database, no external services. Fast (~2 seconds for full suite).
- `test_forecasting.py` — model fit tests using synthetic time series. Validates output shape, confidence interval validity (lower ≤ predicted ≤ upper), and non-negative clipping.
- `test_alerts.py` — evaluator logic tests: threshold operators, consecutive period counting, cooldown enforcement, quiet hours.
- `test_api/` — FastAPI TestClient tests for all endpoints. Uses a separate test database seeded with fixtures per test.

**Target coverage:** >80% on `app/analytics/` and `app/forecasting/`. Coverage report is generated on every CI run.

---

## Deployment

### Docker Compose (Production)

```bash
# Build production images
docker-compose -f docker-compose.prod.yml build

# Deploy (ensure .env.prod is configured)
docker-compose -f docker-compose.prod.yml up -d

# Run migrations on deployment
docker-compose -f docker-compose.prod.yml exec api alembic upgrade head

# View logs
docker-compose -f docker-compose.prod.yml logs -f --tail=100
```

Production Docker Compose differences from development:
- No `--reload` on Uvicorn (uses Gunicorn with Uvicorn workers: `4 × CPU cores`)
- Node.js builds static frontend, served by Nginx
- Flower is behind Nginx basic auth
- Redis has AOF persistence enabled
- No pgAdmin service

---

### Kubernetes

```bash
# Apply all manifests
kubectl apply -f k8s/ --recursive

# Check deployment status
kubectl rollout status deployment/bizpulse-api -n bizpulse
kubectl rollout status deployment/bizpulse-worker -n bizpulse

# View logs
kubectl logs -f deployment/bizpulse-api -n bizpulse

# Scale workers
kubectl scale deployment bizpulse-worker --replicas=5 -n bizpulse

# Run migrations (one-off job)
kubectl create job --from=cronjob/bizpulse-migrations migrate-$(date +%s) -n bizpulse
```

**Important Kubernetes notes:**

- `beat/deployment.yaml` is configured with `replicas: 1`. Never increase this — Celery Beat must run as a singleton.
- `api/hpa.yaml` configures horizontal pod autoscaling: min 2 replicas, max 10, target 70% CPU utilisation.
- Secrets are managed via `k8s/secret.yaml` — this file is a template only. Populate it with actual base64-encoded values and never commit it to the repository. Use a secrets manager (AWS Secrets Manager, Vault, Kubernetes External Secrets) in production.
- PostgreSQL runs as a StatefulSet with a persistent volume claim. Backups must be configured separately via your cloud provider's managed DB service or pg_basebackup.

---

### Production Environment Checklist

Before going live, verify all of the following:

**Security**
- [ ] `SECRET_KEY` is at least 32 characters of cryptographically random bytes
- [ ] `ENCRYPTION_KEY` is a valid Fernet key, stored in a secrets manager (not `.env`)
- [ ] All API keys for external services (Twilio, SendGrid, Firebase) are rotated from development values
- [ ] PostgreSQL is not publicly accessible — API connects via private network only
- [ ] Redis is password-protected (`requirepass` in redis.conf)
- [ ] Flower is behind authentication and not publicly accessible
- [ ] HTTPS is enforced on all endpoints (TLS termination at load balancer or Nginx)
- [ ] CORS is restricted to your frontend domain only
- [ ] Webhook endpoints verify HMAC signatures for all connectors

**Operations**
- [ ] Database backups are configured and tested (verify restore procedure)
- [ ] Log aggregation is set up (e.g. Datadog, CloudWatch, Loki)
- [ ] Alerting is configured for pod crashes, high error rates, worker queue depth
- [ ] Celery Beat is running as a single replica
- [ ] Database connection pooling is configured (`pool_size=20, max_overflow=10`)
- [ ] Redis maxmemory and eviction policy are set (`maxmemory-policy allkeys-lru`)
- [ ] Alembic migration has been run (`alembic upgrade head`)

**Notifications**
- [ ] Twilio WhatsApp Business account is approved (not sandbox)
- [ ] SendGrid sender domain is verified
- [ ] Firebase service account credentials are in place
- [ ] Default alert rules are configured per tenant on signup

---

## Subscription Tiers

| Feature | Starter | Growth | Enterprise |
|---------|---------|--------|------------|
| Data sources | 1 | 5 | Unlimited |
| Users | 3 | 10 | Unlimited |
| Forecasting | ✗ | ✓ | ✓ |
| Custom alert rules | 3 | 20 | Unlimited |
| Dashboard templates | Pre-built only | Pre-built + custom | All + white-label |
| Data history | 12 months | 24 months | Unlimited |
| API access | ✗ | ✓ | ✓ |
| Notification channels | Email only | All channels | All channels + priority |
| Support | Community | Email | Dedicated SLA |

Tier limits are enforced in `app/tenants/service.py` by checking `subscription_plans` before creating data sources, users, or alert rules.

---

## Makefile Reference

```bash
make dev          # Start all services with Docker Compose (hot reload)
make build        # Build Docker images
make migrate      # Run Alembic migrations
make seed         # Seed demo tenant and sample data
make test         # Run full test suite (backend + frontend)
make test-backend # Run backend tests only
make lint         # Run ruff (Python) + eslint (TypeScript)
make typecheck    # Run mypy (Python) + tsc (TypeScript)
make format       # Run black + prettier
make logs         # Tail all service logs
make shell        # Open bash shell in api container
make psql         # Open psql in postgres container
make flower       # Open Celery Flower in browser
make clean        # Stop containers and remove volumes
```

---

## Contributing

1. Fork the repository and create a feature branch from `main`

```bash
git checkout -b feature/your-feature-name
```

2. Make your changes. Run the full test suite before opening a PR:

```bash
make lint
make typecheck
make test
```

3. Ensure test coverage does not decrease below the current baseline:

```bash
cd backend && pytest --cov=app --cov-fail-under=80
```

4. Write a clear PR description including:
   - What the change does
   - Which module(s) it affects
   - How to test it manually
   - Any new environment variables introduced

5. One approval from a maintainer is required to merge.

**Code style:**
- Python: `ruff` for linting, `black` for formatting, `mypy --strict` for type checking
- TypeScript: `eslint` with the project config, `prettier` for formatting
- All new Python functions require docstrings and full type hints
- All new API endpoints require corresponding tests in `tests/test_api/`

---

## Roadmap

**v1.1**
- [ ] Stripe billing integration (usage-based billing from `tenant_usage` table)
- [ ] Multi-currency support (store amounts in smallest unit, display in tenant currency)
- [ ] Bulk CSV export of all metric data per tenant

**v1.2**
- [ ] Google Analytics connector
- [ ] Meta Ads connector
- [ ] AI-generated insight summaries (natural language explanation of metric changes)

**v1.3**
- [ ] Mobile app (React Native) with push notifications
- [ ] Slack notification channel
- [ ] White-label support (custom domain per tenant)

**v2.0**
- [ ] Real-time streaming metrics (PostgreSQL LISTEN/NOTIFY → WebSocket)
- [ ] Customer-level drill-down: click any RFM segment → see individual customers
- [ ] Automated A/B test tracking and result analysis

---

## License



---

## Acknowledgements

Built with:
- [FastAPI](https://fastapi.tiangolo.com) — The API framework
- [Prophet](https://facebook.github.io/prophet/) — Time series forecasting (Meta)
- [Recharts](https://recharts.org) — React charting library
- [Twilio](https://twilio.com) — WhatsApp and SMS delivery
- [@dnd-kit](https://dndkit.com) — Accessible drag-and-drop

---



