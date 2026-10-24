# ReachInbox – Email Job Scheduler

Full-stack distributed email scheduling service and dashboard built for the **ReachInbox Hiring Assignment**.

- **Candidate**: Vansh Arora
- **Reviewers**: Mitrajit, Yadav036

---

## ⚡ Quick Start

### 1. Start Database & Redis
```bash
docker compose up -d
```
*(Starts PostgreSQL on port 5433, Redis on port 6379 with AOF persistence, and Elasticsearch on 9200).*

### 2. Run Backend (Express + BullMQ Worker)
```bash
cd backend
npm install
npm run dev
```
- API & Worker run on: `http://localhost:4000`
- BullMQ Monitor (Bull-Board): `http://localhost:4000/admin/queues`

### 3. Run Frontend (React + Vite)
```bash
cd ../frontend
npm install
npm run dev
```
- Dashboard opens at: `http://localhost:5173`

---

## 🔑 Environment & Ethereal SMTP Setup

In `backend/.env`:
- `PORT=4000`
- `DATABASE_URL=postgres://reachinbox@localhost:5433/reachinbox`
- `REDIS_URL=redis://localhost:6379`
- `MIN_DELAY_BETWEEN_EMAILS_MS=2000` (min 2-second spacing)
- `MAX_EMAILS_PER_HOUR_PER_SENDER=200`
- `WORKER_CONCURRENCY=5`

**Ethereal Email Setup**:
No manual setup needed! On first launch, the backend automatically generates 3 real Ethereal test accounts via `nodemailer.createTestAccount()`. Every sent email in the dashboard has an instant Ethereal web preview link.

---

## 🏗️ Architecture Overview

```
Frontend (React) ──► Express API (:4000) ──► PostgreSQL (Source of Truth)
                                      └──► BullMQ Delayed Jobs (Redis AOF)
                                                  │
                                                  ▼
                                       BullMQ Worker (Concurrency: 5)
                                         ├─ Atomic SQL Claim
                                         ├─ Redis Lua Rate-Limit Check
                                         └─ Nodemailer Ethereal SMTP
```

### 1. How Scheduling Works (No Cron)
Every email is scheduled as a BullMQ **delayed job** (`delay = scheduledAt - now`) with `jobId = emailId`. BullMQ utilizes Redis sorted sets (`ZSET`) where the timestamp is the score. Zero cron jobs or polling intervals are used.

### 2. Persistence on Restart (Crash Resilience)
- **Redis AOF**: Redis is run with `--appendonly yes` so all timers survive restarts.
- **Boot Reconciliation**: On startup, `reconcile()` queries PostgreSQL for any unsent scheduled emails and ensures they exist in BullMQ without creating duplicates.
- **Absolute Timestamps**: Each email has an absolute UTC timestamp; restarting the server never resets progress.

### 3. Idempotency & Concurrency
- **Zero Duplicate Sends**: Workers run an atomic SQL query `UPDATE emails SET status='processing' WHERE id=$1 AND status='scheduled' RETURNING id`. If already claimed or sent, 0 rows return and the job is skipped.
- **Concurrency**: Set to `5` parallel jobs per worker.

### 4. Rate Limiting Under Load
- **Global Inter-Email Delay**: BullMQ limiter enforces a strict 2-second delay (`MIN_DELAY_BETWEEN_EMAILS_MS=2000`) between outgoing emails.
- **Hourly Quota per Sender**: Tracked atomically via a Redis Lua script. If a sender reaches the hourly limit, the job is not dropped—it is rescheduled to the start of the next hour window, and a Slack alert is dispatched if connected.

---

## 📦 Features Implemented

### Backend
- **Scheduler**: BullMQ delayed jobs with millisecond precision, zero cron jobs.
- **Persistence**: PostgreSQL relational storage + Redis AOF + boot reconciliation.
- **Idempotency**: Atomic SQL conditional claims + deterministic `jobId = emailId`.
- **Rate Limiting**: 2-second inter-message limiter + Redis Lua hourly quotas.
- **Multi-Sender**: Automatic round-robin distribution across Ethereal accounts.
- **Search**: Elasticsearch indexing for fuzzy search on recipients, subjects, and bodies.
- **Slack Alerts**: Automated notification when hourly rate limits are hit.
- **Auth**: Google OAuth 2.0 + Instant One-Click Demo Login for evaluators.

### Frontend
- **Modern Dark UI**: Glassmorphic theme with Plus Jakarta Sans and high-contrast typography.
- **1-Click Demo Sign-in**: Instant login button for testing without OAuth credentials.
- **Dashboard**: Live tabs for **Scheduled** and **Sent** emails with counters and status badges.
- **Composer**: Email modal with CSV recipient upload, subject, rich-text body editor, and delay/limit controls.
- **Search & Filters**: Real-time filtering and status filtering.

---

## ⚖️ Trade-offs & Assumptions

- **Fixed Window Rate Limiting**: Simple and O(1) performance via Redis Lua. Global 2-second limiter prevents boundary bursts.
- **At-Least-Once Delivery**: Guaranteed via atomic database claims and unique `Message-ID` headers.
- **Ethereal Mail**: Used for zero-cost, immediate end-to-end SMTP testing with web preview URLs.
