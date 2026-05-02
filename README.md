# OnboardIQ — Automated B2B Customer Onboarding Orchestration Engine

> An n8n automation that monitors trial accounts, detects stall patterns, sends personalized interventions, and reports onboarding analytics — replacing $30K–$80K/year tools like Gainsight with a free-tier stack.

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Solution](#solution)
- [Tech Stack](#tech-stack)
- [Architecture](#architecture)
- [Workflow Breakdown](#workflow-breakdown)
  - [Trigger 1 — New Account Created](#trigger-1--new-account-created)
  - [Trigger 2 — Milestone Completion](#trigger-2--milestone-completion)
  - [Trigger 3 — Stall Detection](#trigger-3--stall-detection)
  - [Trigger 4 — Trial End Protocol](#trigger-4--trial-end-protocol)
  - [Trigger 5 — Weekly Analytics](#trigger-5--weekly-analytics)
- [Airtable Schema](#airtable-schema)
- [Key Design Decisions](#key-design-decisions)
- [Setup Guide](#setup-guide)
- [Testing](#testing)
- [Project Structure](#project-structure)
- [Interview Talking Points](#interview-talking-points)

---

## Overview

OnboardIQ is a fully automated B2B customer onboarding engine. It watches every new trial account from the moment it is created, tracks milestone progress, detects when a user goes silent, fires the right intervention email at the right time, escalates to sales when trials are about to expire, and delivers a weekly analytics report to CS leadership — all without any human having to manually monitor dashboards or send follow-ups.

---

## Problem Statement

SaaS companies lose 40–60% of trialists in the first 14 days. The root cause is not a bad product — it is the absence of a system that watches for failure signals in real time. Customer Success teams react to churn instead of preventing it, and manual check-ins do not scale past 50 accounts.

Traditional solutions are either:
- **Time-based drip emails** — send the same message to everyone on day 3 regardless of where the user actually is in onboarding
- **Manual CSM check-ins** — impossible to scale, dependent on individual memory and attention

OnboardIQ replaces both with a behavioral, event-driven system.

---

## Solution

Instead of sending emails based on time, every intervention is triggered by what the customer is actually doing — or not doing. The system knows exactly which onboarding step a customer is stuck on and sends a message that directly addresses that specific blocker.

Key capabilities:
- Real-time milestone tracking via webhooks
- Stall detection running every 6 hours
- Step-specific intervention emails (not generic drip campaigns)
- Idempotency checks preventing duplicate sends
- 4-tier escalation system (CSM → Manager → Sales → Rescue Call)
- AI-generated personalized conversion pitch at trial end
- Weekly analytics email to CS leadership

---

## Tech Stack

| Tool | Purpose |
|---|---|
| n8n | Workflow automation engine |
| Airtable | Database (accounts, milestones, interventions, templates) |
| Gmail API (OAuth2) | All outbound email delivery |
| OpenAI GPT-4o-mini | Personalized trial-end conversion pitch |
| Webhooks | Real-time event triggers from product |
| Cron Schedules | Periodic stall detection and reporting |

**Cost:** Effectively $0 additional — uses free tiers of all services listed above.

---

## Architecture

```
Product App
    │
    ├── POST /webhook/new-account ──────────► Create Account + 6 Milestone Records
    │
    └── POST /webhook/milestone-complete ───► Update Milestone + Recalculate Health Score
    
Cron (every 6h)
    └── Stall Detection ─────────────────────► Detect → Escalate → Intervene → Log

Cron (daily 8am)
    └── Trial End Check ─────────────────────► Summary Email + AI Pitch + Sales Escalation

Cron (Monday 9am)
    └── Weekly Analytics ────────────────────► Aggregate Metrics → Email CS Leadership
```

---

## Workflow Breakdown

### Trigger 1 — New Account Created

**Webhook:** `POST /webhook/new-account`

**Payload:**
```json
{
  "company_name": "Acme Corp",
  "domain": "acmecorp.com",
  "plan": "Growth",
  "account_id": "acc_001",
  "csm_email": "csm@yourcompany.com",
  "csm_name": "Sarah Chen"
}
```

**What happens:**
1. n8n receives the webhook and logs the raw payload to `Webhook_Log` table
2. A Code node generates trial start/end dates and sets default values (`health_score: 100`, `onboarding_status: Onboarding`)
3. Creates one record in the `Accounts` table
4. Loops through a hardcoded list of 6 milestone steps and creates one `Milestones` record per step, all set to `Pending`, all linked to the new Account

**Why milestones are normalized into their own table:** Enables rollup counts, per-step analytics, and independent metadata without changing the Account schema.

---

### Trigger 2 — Milestone Completion

**Webhook:** `POST /webhook/milestone-complete`

**Payload:**
```json
{
  "account_record_id": "recXXXXXXXXXX",
  "milestone_record_id": "recYYYYYYYYYY",
  "step_name": "Account Setup",
  "interventions_count": 0
}
```

**What happens:**
1. Raw payload logged to `Webhook_Log` (reliability pattern — enables replay on failure)
2. Updates the Milestone record status to `Completed` and timestamps it
3. Fetches all milestones for the account
4. Recalculates health score:

```
Health Score = (Milestones Completed / Total) × 100 − (Interventions Received × 10)
```

The intervention penalty is intentional — an account that needed 3 nudges to complete onboarding is less healthy than one that completed organically.

5. Writes updated health score back to the Account record

---

### Trigger 3 — Stall Detection

**Schedule:** Every 6 hours

**What happens:**
1. Queries all Accounts with `onboarding_status = Onboarding`
2. Reads `hours_since_last_activity` from each account
3. Passes each account through a cascade of IF nodes:

| Hours Stalled | Escalation Level | Notified |
|---|---|---|
| ≥ 120h | L4 — Rescue | CSM + CS Manager + Sales |
| ≥ 96h | L3 | CSM + CS Manager |
| ≥ 72h | L2 | CSM + CS Manager |
| ≥ 48h | L1 | CSM only — step-specific |
| ≥ 24h | L0 | CSM only — day 1 |
| < 24h | None | No action |

**Idempotency Check:** Before sending any intervention, the system queries the `Interventions` table for an existing record matching the same account + same stuck step. If found → skip. This prevents duplicate emails on retry runs or repeated cron executions.

**Template Selection:** Templates are stored in Airtable, not hardcoded in n8n. The system fetches the active template for the current stuck step. CS teams can update email copy without touching the workflow.

**After sending:**
- Intervention email sent via Gmail node
- CSM notification email sent to CSM's own address
- Intervention record logged in `Interventions` table
- Account status updated to `Stalled`

---

### Trigger 4 — Trial End Protocol

**Schedule:** Daily at 8am

**What happens:**
1. Finds all accounts with `onboarding_status` of `Onboarding` or `Stalled`
2. Fetches full milestone list for each account
3. Compiles completion summary — which steps done, which pending, percentage
4. Sends data to OpenAI GPT-4o-mini with a prompt requesting a 2–3 sentence personalized conversion pitch referencing exactly what the customer accomplished
5. Assembles a formatted HTML "Your Onboarding Summary" email with milestone checklist + AI pitch
6. Sends to customer via Gmail
7. If completion < 50% → sends urgent escalation email to `sales@yourcompany.com` flagging for rescue call

---

### Trigger 5 — Weekly Analytics

**Schedule:** Every Monday at 9am

**What happens:**
1. Fetches all accounts and interventions from the past 7 days
2. Aggregates: total accounts, conversion rate, churn rate, stall count, total interventions, intervention success rate, average health score
3. Sends formatted HTML report to `cs-leadership@yourcompany.com`

---

## Airtable Schema

### Accounts
| Column | Type | Description |
|---|---|---|
| company_name | Single line text | Company name |
| domain | Single line text | Company domain |
| account_id | Single line text | Product's internal account ID |
| plan | Single line text | Trial plan tier |
| trial_start | Single line text | Trial start date |
| trial_end | Single line text | Trial end date |
| account_owner | Single line text | CSM name |
| csm_email | Email | CSM email for notifications |
| onboarding_status | Single select | Onboarding / Stalled / Completed / Converted / Churned |
| health_score | Number | 0–100, recalculated on every milestone completion |
| current_stuck_step | Single line text | Step name where account is currently stalled |
| hours_since_last_activity | Number | Updated on milestone completion, read by stall detection |

### Milestones
| Column | Type | Description |
|---|---|---|
| Accounts | Linked record | Links to Accounts table |
| step_name | Single line text | Name of the onboarding step |
| step_order | Number | 1–6 ordering |
| status | Single select | Pending / Completed / Skipped |
| completed_at | Single line text | Timestamp of completion |
| hours_to_complete | Number | Time taken to complete |
| is_blocking | Checkbox | Whether step blocks downstream progress |

### Interventions
| Column | Type | Description |
|---|---|---|
| Account (Company Name) | Linked record | Links to Accounts table |
| stuck_step | Single line text | Which step triggered this intervention |
| Intervention Type | Single select | Email |
| Sent At | Single line text | Timestamp of send |
| Template Used | Single line text | Which template was used |
| outcome | Single select | Pending / Successful / No Response |
| CSM Notified | Checkbox | Whether CSM notification was sent |

### Templates
| Column | Type | Description |
|---|---|---|
| Step Name | Single line text | Matches stuck step name exactly |
| Intervention Type | Single select | Email |
| Subject Line | Single line text | Email subject with `{{placeholder}}` tokens |
| Body HTML | Long text | Full HTML email body with tokens |
| Trigger Condition (Hours Stuck) | Number | Informational — hours threshold |
| Active | Checkbox | Uncheck to disable without deleting |

### Webhook_Log
| Column | Type | Description |
|---|---|---|
| Event Type | Single line text | new_account / milestone_complete |
| Payload | Long text | Raw JSON payload string |
| Received At | Single line text | Timestamp |
| Processed | Checkbox | Whether processing completed |

### CSMs
| Column | Type | Description |
|---|---|---|
| Name | Single line text | CSM full name |
| Email | Email | CSM email |
| Manager Email | Email | Used for L3/L4 escalation routing |
| Assigned Accounts Count | Number | Portfolio size |
| Avg Onboarding Score | Number | Average health score across accounts |

---

## Key Design Decisions

### 1. Content Decoupled from Logic
Email templates live in Airtable, not hardcoded in n8n nodes. The CS team owns all messaging and can update subject lines, body copy, and links without touching the workflow. Logic and content are completely independent.

### 2. Idempotency on Every Intervention
Before any intervention email is sent, the system checks `Interventions` table for an existing record matching the same account + stuck step. Duplicate sends are impossible regardless of retry attempts, cron re-runs, or manual executions.

### 3. Webhook Reliability via Payload Logging
Every incoming webhook payload is written to `Webhook_Log` before any processing begins. If processing fails midway, the original payload is recoverable and can be replayed. Nothing is lost on downstream failure.

### 4. Behavioral Scoring with Intervention Penalty
Health score is not just completion percentage. Each intervention received subtracts 10 points. An account that completed 6/6 milestones but needed 3 emails to do it scores 70, not 100. This distinguishes organic engagement from prompted completion.

### 5. Escalation Tiers
Four escalation levels ensure the right people are notified at the right severity without alert fatigue. Low-severity stalls stay with the CSM. High-severity stalls (120h+) reach CS Manager and Sales simultaneously with a rescue call flag.

---

## Setup Guide

### Prerequisites
- n8n installed locally (`npm install -g n8n`) or cloud instance
- Airtable account with a new base
- Gmail account with OAuth2 credentials configured in Google Cloud Console
- OpenAI API key (for trial-end conversion pitch)

### Step 1 — Google Cloud Setup
1. Go to [Google Cloud Console](https://console.cloud.google.com)
2. Create a new project
3. Enable **Gmail API**
4. Create **OAuth 2.0 Client ID** credentials (Web application)
5. Add `http://localhost:5678/rest/oauth2-credential/callback` as an authorized redirect URI

### Step 2 — Airtable Setup
1. Create a new Airtable base named `Automated B2B Customer Onboarding`
2. Create these 6 tables with exact column names and types from the schema above:
   - `Accounts`
   - `Milestones`
   - `Interventions`
   - `Templates`
   - `CSMs`
   - `Webhook_Log`
3. Import the provided CSV files for `Templates` and `CSMs`
4. Add linked record fields manually:
   - `Milestones.Accounts` → links to Accounts
   - `Interventions.Account (Company Name)` → links to Accounts

### Step 3 — n8n Setup
1. Start n8n: `n8n start`
2. Open `http://localhost:5678`
3. Go to **Workflows** → **Import from File**
4. Import `onboardiq_workflow.json`
5. Set up credentials:
   - **Airtable Personal Access Token** — from Airtable account settings
   - **Gmail OAuth2** — using your Google Cloud credentials
   - **OpenAI API** — from platform.openai.com

### Step 4 — Configure Recipients
Update these email addresses in the workflow nodes:
- `sales@yourcompany.com` — in Build Sales Escalation Email node
- `cs-leadership@yourcompany.com` — in Build Analytics Email node

### Step 5 — Activate
1. Activate the two webhook nodes first
2. Then activate the three cron triggers
3. Test with the commands in the Testing section below

---

## Testing

### Test Trigger 1 — New Account

Click **Listen for test event** on the New Account Webhook node, then run:

**PowerShell:**
```powershell
Invoke-WebRequest -Uri "http://localhost:5678/webhook-test/new-account" -Method POST -ContentType "application/json" -Body '{"company_name":"Test Corp","domain":"testcorp.com","plan":"Growth","account_id":"acc_001","csm_email":"your@gmail.com","csm_name":"Your Name"}'
```

**Mac/Linux:**
```bash
curl -X POST http://localhost:5678/webhook-test/new-account \
  -H "Content-Type: application/json" \
  -d '{"company_name":"Test Corp","domain":"testcorp.com","plan":"Growth","account_id":"acc_001","csm_email":"your@gmail.com","csm_name":"Your Name"}'
```

**Expected result:** 1 Account row + 6 Milestone rows in Airtable

---

### Test Trigger 2 — Milestone Completion

Grab the Account record ID and Milestone record ID from Airtable, then:

```powershell
Invoke-WebRequest -Uri "http://localhost:5678/webhook-test/milestone-complete" -Method POST -ContentType "application/json" -Body '{"account_record_id":"recXXXXXXXXXX","milestone_record_id":"recYYYYYYYYYY","step_name":"Account Setup","interventions_count":0}'
```

**Expected result:** Milestone status → Completed, Health Score updated on Account

---

### Test Trigger 3 — Stall Detection

1. Set `hours_since_last_activity` to `50` on the test account in Airtable
2. Set `current_stuck_step` to `API Setup`
3. Open **Stall Detection Cron** node in n8n
4. Click **Execute step**

**Expected result:** Intervention email in Gmail, new row in Interventions table, account status → Stalled

---

### Test Trigger 4 — Trial End

1. Open **Trial End Check Cron** node
2. Click **Execute step**

**Expected result:** Trial summary email sent, sales escalation email if completion < 50%

---

### Test Trigger 5 — Weekly Analytics

1. Open **Weekly Analytics Cron** node
2. Click **Execute step**

**Expected result:** HTML analytics report sent to cs-leadership email

---

## Project Structure

```
onboardiq/
├── onboardiq_workflow.json     # Full n8n workflow — import this
├── accounts.csv                # Airtable Accounts table schema + sample data
├── milestones.csv              # Airtable Milestones table schema
├── interventions.csv           # Airtable Interventions table schema (empty — populated by workflow)
├── templates.csv               # Airtable Templates table — all 6 intervention email templates
├── csms.csv                    # Airtable CSMs table schema + sample data
├── webhook_log.csv             # Airtable Webhook_Log table schema (empty — populated by workflow)
└── README.md                   # This file
