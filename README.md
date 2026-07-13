# ⚡ InsightFlow — Real-Time Marketing, CRM & Scheduling Platform

**Event-driven AWS platform unifying live CRM webhooks, scheduling webhooks, and batch video analytics — validated through a 7-day continuous production run with documented fault-tolerance and idempotency tests.**

---

## 📋 Overview

Revenue teams lose visibility the moment their CRM, scheduling tool, and marketing video platform stop talking to each other. InsightFlow unifies all three — Close CRM, Calendly, and Wistia — into one event-driven AWS platform that does two things at once: triggers a **real-time Slack alert** the instant a sales-qualified lead is created and enriched, and powers **6 marketing-attribution dashboards** correlating ad spend, bookings, and lead generation.

Unlike a typical batch ELT build, this system is **fully live**. It was required to run continuously, unattended, for **7 consecutive days**, with documented proof that it handles webhook redelivery (idempotency) and component failure (fault tolerance) gracefully — not just a one-time demo.

**The numbers:**
- **2** live webhook sources (Close CRM, Calendly) + **1** paginated batch API (Wistia)
- **10-minute** mandatory SQS delay between lead creation and owner-enrichment lookup
- **7 days** continuous live run, with deliberately engineered failure-recovery and duplicate-event tests
- **6** Streamlit dashboards + **1** cross-source correlation view

---

## 🏗️ Architecture

```
   Close CRM webhook          Calendly webhook              Wistia Stats API
         │                          │                          (scheduled pull)
         ▼                          ▼                                │
   API Gateway                API Gateway                            ▼
         │                          │                       EventBridge + Lambda
         ▼                          ▼                       (paginated, incremental,
    Lambda A (receive)         Lambda (receive)               DynamoDB watermark)
         │                          │                                │
         ▼                          │                                ▼
   SQS delay queue                  ▼                          S3 Bronze (Wistia)
   (10 min)                  S3 Bronze (Calendly)
         │                          │
         ▼                          ▼
   Lambda B (enrich)          Scheduled Glue/Lambda
   ── looks up lead owner      (flatten → Silver,
      in public S3 bucket       join daily spend → Gold)
         │
         ▼
   S3 target/ (enriched)
         │
         ▼
   Lambda C → Slack "New Lead Alert"
```

| Layer | Technology | Purpose |
|---|---|---|
| **Webhook receivers** | API Gateway + Lambda | Real public endpoints for Close and Calendly to POST to |
| **Async delay** | Amazon SQS (10-min delay queue) | Gives Close time to assign a lead owner before enrichment runs |
| **State tracking** | DynamoDB | High-water-mark for incremental, paginated Wistia pulls |
| **Storage** | AWS S3 | Bronze/Silver/Gold, idempotent deterministic key naming |
| **Fault tolerance** | Step Functions retries, SQS DLQ, CloudWatch Alarms | Named deliverable — diagrammed and tested, not assumed |
| **Notifications** | Slack incoming webhook | Real-time "New Lead Alert" with 7 enriched fields |
| **Dashboard** | Streamlit | 6 Calendly-specific pages + 1 cross-source correlation view |
| **CI/CD** | GitHub Actions | Lint/test + automated Lambda deployment |

---

## 🔑 The CRM Flow — Step by Step

1. Close fires a webhook → API Gateway → **Lambda A**
2. Lambda A verifies `Close-Sig-Hash`, extracts `lead_id`, writes raw event to `s3://.../crm/source/crm_event_{lead_id}.json`
3. Lambda A enqueues `{lead_id, s3_key}` to an **SQS delay queue** (`DelaySeconds=600`)
4. After 10 minutes, SQS releases the message → **Lambda B**
5. Lambda B fetches `https://dea-lead-owner.s3.../{lead_id}.json` — handles "not yet assigned" (404) by letting SQS retry
6. Lambda B merges CRM event + lead owner data → writes to `crm/target/crm_enriched_{lead_id}.json`
7. **Lambda C** (S3-triggered) posts a Slack "New Lead Alert" with all 7 required fields
8. Any failure routes to a dead-letter queue with a CloudWatch alarm

**Idempotency:** every S3 write uses a deterministic key (`crm_event_{lead_id}.json`) — webhook redelivery overwrites rather than duplicates, and a duplicate-event test was run and documented during the 7-day live window to prove this.

---

## 📅 The Calendly Flow — Step by Step

1. Calendly fires an `invitee.created` webhook → API Gateway → Lambda receiver
2. Receiver verifies the `Calendly-Webhook-Signature` header (`t=<timestamp>,v1=<hmac>` format — different scheme from Close's), filters out any non-`invitee.created` event types, and writes the raw payload to `calendly/bronze/invitee_created_{booking_id}.json`
3. A scheduled Glue job flattens the nested Bronze JSON into Silver — extracting `booking_id`, `booking_date`, UTM fields, and mapping the `event_type` URI to a channel name (`facebook_paid_ads`, `youtube_paid_ads`, `tiktok_paid_ads`) via a static lookup
4. A daily job pulls the previous day's ad spend file (checking `file_index.json` first to confirm it's published) and joins it to Silver bookings on `date` + `channel` to compute Cost Per Booking in Gold

---

## 📊 Dashboards (Calendly Metrics)

| # | Dashboard | Key Visual |
|---|---|---|
| 1.1 | Daily Calls Booked by Source | Line chart, date vs. bookings |
| 1.2 | Cost Per Booking (CPB) by Channel | Bar chart + KPI tiles |
| 1.3 | Bookings Trend Over Time | Line + cumulative area chart |
| 1.4 | Channel Attribution Leaderboard | Leaderboard table + heatmap |
| 1.5 | Booking Volume by Time Slot/Day | Heatmap, hour vs. weekday |
| 1.6 | Meeting Load per Employee | Bar chart, avg meetings/week |
| 7 | **Cross-Source Relevance** | Daily leads (CRM) × bookings (Calendly) × video plays (Wistia) |

**Cost Per Booking** is computed by joining Calendly bookings to a daily spend feed:
```
CPB = Total Spend (by channel, by day) / Total Booked Calls (by channel, by day)
```

---

## ✅ 7-Day Live Run — What Was Actually Validated

This wasn't a "build it and demo it once" project. Over the 7-day continuous run:

- **CloudWatch dashboard** tracked Lambda invocation/error counts, SQS queue depth (incl. DLQ), and daily S3 object counts per flow
- **Failure-recovery test**: deliberately broke the lead-owner lookup mid-week and documented how the system recovered via SQS retry/DLQ
- **Idempotency test**: deliberately replayed a duplicate webhook and confirmed no duplicate S3 object or duplicate Slack message resulted
- **Daily spot-checks** confirmed new leads appeared in `target/` within the expected delay window with zero DLQ accumulation
---

## 🧩 Key Design Decisions

- **SQS delay queue, not a `sleep()` or cron retry loop** — purpose-built for exactly this "wait then process" pattern, fully serverless, no infrastructure to babysit.
- **Deterministic S3 keys for idempotency** — `crm_event_{lead_id}.json` and `calendly_invitee_created_{booking_id}.json` mean webhook redelivery is a safe overwrite, not a duplicate.
- **Per-source signature verification, not a shared helper** — Close and Calendly sign webhooks differently (`Close-Sig-Hash`/`Close-Sig-Timestamp` headers vs. Calendly's combined `t=...,v1=...` header), so each receiver implements its own verification rather than forcing a common abstraction that would obscure the actual scheme being checked.
- **Event-type filtering at the Calendly receiver** — a single Calendly webhook subscription can fire multiple event types (`invitee.created`, `invitee.canceled`, etc.); the receiver explicitly no-ops on anything except `invitee.created` so the Bronze layer only ever contains bookings, not cancellations.
- **DynamoDB watermark over re-scanning S3** — single-digit-millisecond reads to check "have I already processed this Wistia update" beat scanning prior Bronze files on every run.
- **Date-level join for cross-source correlation, not a forced row-level key** — CRM, Calendly, and Wistia have no natural shared ID. Rather than invent one, the cross-source dashboard correlates at the daily aggregate level and states that assumption explicitly.

---

## 📁 Repo Structure

```
insightflow-project/
├── README.md
├── .github/workflows/{ci.yml, deploy.yml}
├── architecture/ (one diagram per flow + 1 consolidated)
├── crm/
│   ├── lambda_webhook_receiver.py
│   ├── lambda_enrich_lead.py
│   └── lambda_notify_slack.py
├── calendly/
│   ├── lambda_webhook_receiver.py
│   ├── bronze_to_silver_bookings.py
│   ├── ingest_spend_data.py
│   └── build_gold_cpb.py
├── wistia/extract_stats.py
└── dashboard/app.py
```
