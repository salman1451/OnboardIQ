# OnboardIQ — Automated B2B Customer Onboarding Orchestration Engine

> A production-ready prototype built with n8n that monitors trial accounts, detects stall patterns, sends personalized interventions, and reports onboarding analytics.

## Overview
OnboardIQ is an automated onboarding engine that tracks user progress, detects inactivity, and sends context-aware intervention emails.

## Tech Stack
- n8n (workflow automation)
- Airtable (state management)
- Gmail API (email delivery)
- OpenAI (trial-end personalization)

## Key Features
- Event-driven onboarding tracking
- Stall detection every 6 hours
- Idempotent intervention system
- Escalation logic (CSM → Manager → Sales)
- AI-generated trial-end messaging

## Architecture
- Webhooks → real-time events
- Cron jobs → periodic checks
- Airtable → data layer
- Gmail API → delivery

## Limitations & Future Improvements
- Airtable rate limits may impact scaling
- Cron-based detection introduces delay
- No distributed queue system (future: Kafka/RabbitMQ)
- Future: ML-based churn prediction, retry strategies
