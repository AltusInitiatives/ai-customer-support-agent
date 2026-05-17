# AltusShop AI Customer Support Agent
### Week 5 Capstone — 90-Day AI Automation Specialist Execution Plan

> An AI-powered customer support agent built in n8n that handles customer inquiries
> autonomously, retrieves real-time order data, searches a knowledge base, and
> escalates unresolvable issues to a human agent.

---

## What It Does

The AltusShop AI Customer Support Agent receives customer inquiries via webhook and
handles them end-to-end without human intervention. For each inquiry, the agent:

1. Reads the customer's name, email, question, and order ID from the incoming request
2. Decides which tool to use based on the nature of the question
3. Searches the FAQ knowledge base, looks up order status, or retrieves purchase history
4. Generates a professional, concise response grounded in real data
5. Rates its own confidence in the response on a scale of 1–10
6. Routes the interaction — low confidence triggers a Slack escalation alert and human
   review; high confidence logs the interaction and responds directly
7. Logs every interaction to a Google Sheet with timestamp, response, confidence score,
   and escalation status

---

## System Architecture Overview

```
Incoming Request (POST /webhook)
         │
         ▼
  AI Customer Support Agent
  ┌─────────────────────────────────┐
  │  LLM: Claude Haiku 4.5          │
  │  Memory: Window Buffer (session) │
  │                                  │
  │  Tools:                          │
  │  ├── faq_search (Google Sheets)  │
  │  ├── order_status (HTTP / n8n)   │
  │  └── customer_history (Sheets)   │
  └─────────────────────────────────┘
         │
         ▼
  Parse Confidence Score (Code Node)
         │
         ▼
  IF confidence < 7?
  ├── YES → Slack Alert → Log (escalated: true) → Respond
  └── NO  → Log (escalated: false) → Respond
```

**External services used:**
- Anthropic API (Claude Haiku 4.5) — language model
- Google Sheets — FAQ knowledge base, customer history, interaction log
- n8n webhook — mock order status API
- Slack — escalation alert notifications

---

## Tools the Agent Has Access To

### faq_search
Searches the AltusShop FAQ Google Sheet for answers to common customer questions
about returns, shipping, payments, and policies. The agent calls this tool when the
question is policy or procedure related.

### order_status
Makes an HTTP GET request to the mock order status API, passing the customer's
`order_id` as a query parameter. Returns the current status, estimated delivery
date, and carrier for that order. The agent calls this tool when the customer asks
about the whereabouts or status of a specific order.

### customer_history
Reads the AltusShop Customer History Google Sheet and returns all orders associated
with the customer's email address. The agent calls this tool when the customer asks
about their past purchases.

---

## Escalation Logic

Every response the agent generates ends with a self-rated confidence score:

```
CONFIDENCE: [1–10]
```

A **Parse Confidence Score** Code node extracts this number from the agent's output
using regex, removes it from the visible response, and routes the interaction:

| Confidence Score | Meaning | Action |
|---|---|---|
| 7 – 10 | Agent answered confidently | Log interaction, respond to customer |
| 1 – 6 | Agent could not find a satisfactory answer | Slack alert, log as escalated, respond to customer |

When escalation triggers, a Slack message is sent to the designated channel containing
the customer's name, email, order ID, question, agent response, and confidence score.
A human agent reviews the ticket and follows up directly.

---

## Setup Instructions

### Credentials Required

| Service | Credential Type | Where to Obtain |
|---|---|---|
| Anthropic | API Key | console.anthropic.com |
| Google Sheets | OAuth2 | Google Cloud Console |
| Slack | OAuth2 | api.slack.com/apps |

### Google Sheets to Create

**1. AltusShop FAQ**
Columns: `question`, `answer`
Populate with at least 10 common support questions and answers before activating.

**2. AltusShop Customer History**
Columns: `customer_email`, `order_id`, `product`, `purchase_date`, `status`
Populate with existing customer order records.

**3. AltusShop Support Log**
Columns: `timestamp`, `name`, `email`, `question`, `order_id`, `response`,
`confidence_score`, `escalated`
Leave empty — the workflow populates this automatically.

### Mock Order Status API

This project uses a second n8n workflow as a mock order status API. Set it up as
follows:

1. Create a new n8n workflow with a Webhook trigger at path `order-status`
2. Add a Code node with order status data keyed by `order_id`
3. Add a Respond to Webhook node returning the order data
4. **Activate this workflow** before running the main workflow

Update the `order_status` tool URL in the main workflow to point to your mock API's
production webhook URL.

### n8n Credentials Setup

1. Add your **Anthropic API key** under Settings → Credentials → Anthropic
2. Add your **Google Sheets OAuth2** credential and connect it to your Google account
3. Add your **Slack OAuth2** credential and select the workspace and channel for alerts
4. Update the Google Sheet IDs in the `faq_search`, `customer_history`, and both
   logging nodes to match your own sheets

---

## Example Test Payloads

Send these via Postman or curl to test each capability. Replace the webhook URL with
your own n8n instance URL.

### FAQ Query
```bash
curl -X POST https://your-n8n-url/webhook/day-35-capstone \
  -H "Content-Type: application/json" \
  -d '{
    "session_id": "s1",
    "name": "Nicole Liu",
    "email": "nicole_liu@example.com",
    "question": "What is your return policy?",
    "order_id": ""
  }'
```

### Order Status Query
```bash
curl -X POST https://your-n8n-url/webhook/day-35-capstone \
  -H "Content-Type: application/json" \
  -d '{
    "session_id": "s2",
    "name": "Samantha Wong",
    "email": "samantha_wong@example.com",
    "question": "Where is my order right now?",
    "order_id": "as002"
  }'
```

### Customer History Query
```bash
curl -X POST https://your-n8n-url/webhook/day-35-capstone \
  -H "Content-Type: application/json" \
  -d '{
    "session_id": "s3",
    "name": "Marina Puffer",
    "email": "marina_puffer@example.com",
    "question": "What have I ordered from you before?",
    "order_id": ""
  }'
```

### Escalation Trigger
```bash
curl -X POST https://your-n8n-url/webhook/day-35-capstone \
  -H "Content-Type: application/json" \
  -d '{
    "session_id": "s4",
    "name": "Samantha Wong",
    "email": "samantha_wong@example.com",
    "question": "I want to negotiate a wholesale partnership deal for 1000 units per month.",
    "order_id": ""
  }'
```

### Multi-Turn Memory Test
Send these two requests in sequence using the same `session_id`. The second message
should reference context from the first without it being re-provided.

```bash
# Message 1
curl -X POST https://your-n8n-url/webhook/day-35-capstone \
  -H "Content-Type: application/json" \
  -d '{
    "session_id": "s5",
    "name": "Marina Puffer",
    "email": "marina_puffer@example.com",
    "question": "I just want to confirm my order was delivered successfully.",
    "order_id": "as004"
  }'

# Message 2 — agent should recall the order from Message 1
curl -X POST https://your-n8n-url/webhook/day-35-capstone \
  -H "Content-Type: application/json" \
  -d '{
    "session_id": "s5",
    "name": "Marina Puffer",
    "email": "marina_puffer@example.com",
    "question": "What product did I order?",
    "order_id": ""
  }'
```

---

## Expected Response Format

Every response from the agent follows this structure:

```json
{
  "response": "Agent's reply to the customer in plain text.",
  "confidence_score": 9,
  "escalated": false
}
```

---

*Built during Phase 2, Week 5 of the 90-Day AI Automation Specialist Execution Plan*
*Stack: n8n · Claude Haiku 4.5 · Google Sheets · Slack*
