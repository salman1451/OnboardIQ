# OnboardIQ

> A custom n8n automation for managing B2B customer onboarding. It tracks trial accounts, catches when they get stuck, and sends out personalized follow-ups.

---

## Table of Contents

- [What is OnboardIQ?](#what-is-onboardiq)
- [How it works](#how-it-works)
- [Tech Stack](#tech-stack)
- [Architecture](#architecture)
- [Airtable Schema](#airtable-schema)
- [Why I built it this way](#why-i-built-it-this-way)
- [How to set it up](#how-to-set-it-up)
- [Testing things out](#testing)
- [Project Structure](#project-structure)

---

## What is OnboardIQ?

I built OnboardIQ to automate the tedious parts of customer onboarding. Most SaaS companies lose a lot of trial users because they don't have a good way to see who is actually using the product and who isn't. 

Instead of just sending generic "how's it going?" emails every few days, OnboardIQ watches for specific milestones. If someone gets stuck on a step like "API Setup" for too long, the system knows exactly what happened and sends a relevant nudge. It also keeps the Customer Success (CS) team in the loop and escalates to Sales if a trial is ending without much progress.

---

## How it works

The system is built around five main triggers:

### 1. New Account Setup
When a new trial starts, a webhook triggers n8n to create an entry in Airtable. It automatically sets up a list of 6 standard milestones that every account needs to hit.

### 2. Tracking Milestones
As users complete steps in your app, your app hits a webhook. n8n updates the milestone status and recalculates a "Health Score." This score goes down if we have to send too many follow-ups, giving you a real sense of how smoothly their onboarding is going.

### 3. Catching Stalled Users
Every 6 hours, a cron job checks for any accounts that haven't had activity for a while. Depending on how long they've been stuck (24h, 48h, 72h+), it sends different levels of alerts to the CSM and the customer. 

### 4. Trial End Protocol
Every morning, the system looks for trials that are wrapping up. It uses OpenAI to write a quick, personalized summary of what they achieved during the trial and sends it to them. If they haven't finished much, it flags them to the Sales team for a "rescue call."

### 5. Weekly Analytics
Every Monday morning, it aggregates all the data from the week and sends a clean HTML report to CS leadership so they can see conversion rates and where people are getting stuck.

---

## Tech Stack

I kept the stack simple and free-tier friendly:

| Tool | What it's for |
|---|---|
| **n8n** | The "brain" that runs all the logic and workflows. |
| **Airtable** | Used as the database to store accounts, milestones, and email templates. |
| **Gmail API** | For sending out all the follow-up and notification emails. |
| **OpenAI (GPT-4o-mini)** | To write personalized trial-end summaries. |
| **Webhooks/Cron** | How the system actually gets triggered by your app or on a schedule. |

---

## Architecture

Here is a quick look at how the data flows:

```
Product App
    │
    ├── New Account Webhook ──────────► Creates Account + Milestones in Airtable
    │
    └── Milestone Webhook ───────────► Updates progress + recalculates Health Score
    
Cron (Every 6h)
    └── Stall Detection ─────────────► Checks for inactivity → Sends nudges → Logs it

Cron (Daily 8am)
    └── Trial End Protocol ──────────► AI-generated summary → Sales escalation if needed

Cron (Monday 9am)
    └── Weekly Analytics ────────────► Sends a summary report to leadership
```

---

## Airtable Schema

The system uses a few linked tables in Airtable:

### Accounts
The main table for tracking companies, their trial dates, and their current health score.

### Milestones
Every account gets 6 milestone records (e.g., "Account Setup", "API Integrated"). This lets us track exactly where someone is stuck.

### Interventions
Logs every time we send an automated email so we don't spam people with the same nudge twice.

### Templates
Where the actual email copy lives. You can change your email subjects and body text here without ever touching the n8n workflow.

### CSMs & Webhook Log
Helper tables for routing emails to the right CSM and keeping a raw log of every webhook for debugging.

---

## Why I built it this way

- **Content vs Logic:** I didn't want to hardcode email text into n8n. Keeping templates in Airtable means the CS team can update the copy themselves.
- **Don't Spam Users:** I built an "idempotency" check. Before sending any nudge, the system checks if it already sent that specific one. This prevents people from getting 5 emails if a cron job runs twice.
- **Health is more than completion:** If a customer needs 3 nudges to finish a step, they are less "healthy" than someone who did it themselves. The Health Score reflects this by subtracting points for every intervention.

---

## How to set it up

### 1. Google Cloud
You'll need a project in Google Cloud Console with the **Gmail API** enabled and **OAuth 2.0** credentials. Set your redirect URI to `http://localhost:5678/rest/oauth2-credential/callback`.

### 2. Airtable
- Create a base named `Automated B2B Customer Onboarding`.
- Set up the tables (Accounts, Milestones, etc.) as described in the schema.
- Import the provided CSV files to get the initial templates and data structure.

### 3. n8n
- Import `onboardiq_workflow.json` into n8n.
- Connect your Airtable, Gmail, and OpenAI credentials.
- Update the hardcoded email addresses (like `sales@yourcompany.com`) in the final notification nodes.

---

## Testing things out

You can test the webhooks using `curl` or PowerShell:

**New Account Test:**
```bash
curl -X POST http://localhost:5678/webhook-test/new-account \
  -H "Content-Type: application/json" \
  -d '{"company_name":"Test Corp","domain":"testcorp.com","account_id":"acc_001","csm_email":"your@email.com"}'
```

**Milestone Completion Test:**
```bash
curl -X POST http://localhost:5678/webhook-test/milestone-complete \
  -H "Content-Type: application/json" \
  -d '{"account_record_id":"recXXX","milestone_record_id":"recYYY","step_name":"Account Setup"}'
```

---

## Project Structure

```
onboardiq/
├── onboardiq_workflow.json     # The n8n workflow
├── accounts.csv                # Sample data for Airtable
├── milestones.csv              
├── interventions.csv           
├── templates.csv               # The 6 email templates
├── csms.csv                    
├── webhook_log.csv             
└── README.md                   
```
