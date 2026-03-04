# 📬 Inbox Personal Assistant

An AI-powered email management system built on **n8n**, **Microsoft Graph API**, and **OpenAI**. Automatically categorizes, PII-masks, summarizes, and delivers daily email digests to Microsoft Teams — supporting multiple users with full OAuth onboarding, self-healing webhook subscriptions, and automatic token renewal.

![Architecture]

<img width="1536" height="1024" alt="ChatGPT Image Mar 1, 2026 at 05_01_25 PM" src="https://github.com/user-attachments/assets/3bb7e99b-d237-4f84-9b9b-9791ffc55b0f" />


## Features

- **Microsoft OAuth2 Onboarding** — Self-serve sign-in flow with CSRF-protected state, automatic category creation, and Graph webhook subscription setup
- **Real-time Email Categorization** — Incoming emails are classified by AI and tagged with Outlook categories
- **PII Guardrails** — 18-category regex masking engine strips sensitive data (SSNs, credit cards, addresses, passwords, etc.) before any AI processing
- **AI Email Summarization** — Per-email summaries with action items, urgency scores, and response flags delivered as HTML digests
- **Daily Teams Digest** — Formatted HTML summary card sent each morning via Microsoft Teams DM
- **Self-Healing Subscriptions** — Automatic renewal/recreation of Graph webhooks with a 24–60h renewal window strategy
- **Token Refresh Logic** — Proactive access token refresh (triggers at <5 min remaining) with mutex-safe multi-user support
- **User Preference Selection** — Action-First vs. Standard summary preference selectable during onboarding

---

## Architecture

```
User Sign-In
    │
    ▼
OAuth Authentication  ──►  Microsoft Graph API
    │
    ▼
Email Subscription (Graph Webhook)
    │
    ▼
Secure Data Processing (PII Masking)
    │
    ├──► Email Guardrails ──► Daily Summarization ──► Database Storage
    │                              │
    │                              ▼
    │                        Teams HTML Digest
    │
    └──► Subscription Management ◄── Token Renewal Logic
```

The system uses **n8n** as the orchestration layer with a queue-mode deployment (Redis + PostgreSQL). All secrets are stored as n8n environment variables — no credentials are hardcoded.

## Workflow Files

| File | Description | Nodes |
|------|-------------|-------|
| `Onboarding_Page.json` | Serves the OAuth sign-in landing page, generates CSRF state and Microsoft login URL | 5 |
| `Outlook_Onboarding_with_Categories___Subscription.json` | Handles OAuth callback, token exchange, user upsert, category creation, and Graph subscription setup | 32 |
| `Select_Preference.json` | Serves the summary preference selection UI after onboarding | 3 |
| `Save_User_Preferences.json` | Receives and validates user preference selection, updates database | 4 |
| `Multi_User_Email_Categorization.json` | Processes incoming webhook events, applies PII masking and AI categorization per user | 28 |
| `Email_Summarization.json` | Fetches unread emails, runs through guardrails + AI summarization, sends Teams digest | 36 |
| `Subscription_Renewal_Workflow.json` | Scheduled job to check token/subscription health and renew/recreate as needed | 38 |


## Tech Stack

- **Orchestration**: [n8n](https://n8n.io) (queue mode)
- **Email & Calendar**: Microsoft Graph API (OAuth2)
- **AI**: OpenAI / LiteLLM (via LangChain nodes)
- **PII Masking**: Custom regex engine (18 categories) + optional Azure AI Language / Presidio
- **Database**: n8n DataTables (PostgreSQL-backed)
- **Cache / Queue**: Redis
- **Notifications**: Microsoft Teams (Graph API)


## Setup

### Prerequisites

- n8n instance (self-hosted, queue mode recommended)
- Microsoft Azure app registration with the following scopes:
  ```
  openid profile email User.Read Calendars.ReadWrite
  offline_access Calendars.ReadWrite.Shared User.ReadBasic.All
  ```
- OpenAI-compatible API (OpenAI or LiteLLM proxy)

### Environment Variables

Set the following in your n8n instance under **Settings → Variables**:

| Variable | Description |
|----------|-------------|
| `Inbox_Assistant_Microsoft_Client_Id` | Azure app client ID |
| `Inbox_Assistant_Microsoft_Client_Secret` | Azure app client secret |
| `Inbox_Assistant_Microsoft_Tenant_Id` | Azure tenant ID |
| `Inbox_Assistant_Prod_Redirect_URI` | OAuth redirect URI (your n8n webhook URL) |
| `Email_Categorization_Webhook_URL` | n8n webhook URL for incoming email events |

### Importing Workflows

1. In n8n, go to **Workflows → Import**
2. Import each JSON file in this order:
   1. `Onboarding_Page.json`
   2. `Outlook_Onboarding_with_Categories___Subscription.json`
   3. `Select_Preference.json`
   4. `Save_User_Preferences.json`
   5. `Multi_User_Email_Categorization.json`
   6. `Email_Summarization.json`
   7. `Subscription_Renewal_Workflow.json`
3. In each workflow, update the **credential references** in any OpenAI or Outlook nodes to point to your own configured credentials
4. Activate all workflows

### Onboarding a User

Direct users to your `Onboarding_Page` webhook URL:
```
https://<your-n8n-host>/webhook/onboarding
```
The flow handles everything from there: Microsoft sign-in → token exchange → category creation → webhook subscription → preference selection.


## Security Notes

- All Microsoft credentials are stored as n8n environment variables, never hardcoded
- PII is masked before being sent to any AI model
- OAuth state parameter uses CSRF tokens + nonce to prevent replay attacks
- Token refresh uses a mutex pattern to prevent race conditions in multi-user scenarios
- Webhook subscriptions are scoped per-user and cleaned up on re-onboarding


## 📄 License

MIT — see [LICENSE](./LICENSE) for details.
