# 🎫 AI-Powered IT Helpdesk Triage & Auto-Response System

An end-to-end IT support automation built with **n8n** and **Anthropic Claude**. Incoming employee tickets are analyzed by an LLM, classified by category and priority, and either **auto-resolved with a drafted email reply** or **escalated to the IT team** — with a fail-safe design that escalates rather than guesses when anything is uncertain.

![n8n](https://img.shields.io/badge/n8n-Workflow_Automation-EA4B71)
![Claude](https://img.shields.io/badge/Anthropic-Claude_API-d97706)
![LLM](https://img.shields.io/badge/Pattern-Human--in--the--loop-blue)

<!-- Demo GIF: record the n8n execution view processing a ticket -->
<!-- ![Demo](docs/demo.gif) -->

---

## 🏗️ Architecture

```
 Employee Ticket (POST)
         │
         ▼
 ┌───────────────┐     ┌──────────────────────┐     ┌─────────────┐
 │    Webhook    │ ──▶ │  LLM Analysis        │ ──▶ │ JSON Parser │
 │  /it-ticket   │     │  (Claude via n8n     │     │ + Fail-safe │
 └───────────────┘     │   Basic LLM Chain)   │     └──────┬──────┘
                       └──────────────────────┘            │
                                                           ▼
                                                  ┌─────────────────┐
                                                  │  needs_human?   │
                                                  └────┬───────┬────┘
                                                 true  │       │  false
                                                       ▼       ▼
                                        ┌──────────────────┐ ┌──────────────────────┐
                                        │ Escalate to IT   │ │ Auto-reply to        │
                                        │ Team (email)     │ │ employee (email)     │
                                        └────────┬─────────┘ └──────────┬───────────┘
                                                 └───────┬──────────────┘
                                                         ▼
                                              ┌────────────────────┐
                                              │ Respond to Webhook │
                                              │  (JSON result)     │
                                              └────────────────────┘
```

## ✨ What It Does

For every incoming ticket, the LLM returns **strict structured JSON**:

```json
{
  "category": "VPN",
  "priority": "High",
  "solution": "Restart the VPN client and reconnect. If the issue persists, restart your machine and try a different network.",
  "needs_human": false,
  "email": "Hi, thank you for contacting IT support..."
}
```

| Field | Purpose |
|---|---|
| `category` | One of 8 fixed categories (Hardware, Software, Network, VPN, Email, Printer, Password, Other) |
| `priority` | High / Medium / Low — work-blocking issues are always High |
| `solution` | Short, safe troubleshooting steps |
| `needs_human` | Escalation flag — `true` whenever the model is unsure |
| `email` | Professionally drafted reply, sent automatically if no escalation |

## 🛡️ Key Design Decisions

**1. Escalate on uncertainty, never guess.**
The system prompt instructs the model to set `needs_human: true` whenever it isn't confident in a safe solution. Vague tickets ("nothing works properly") and physical hardware issues route straight to humans.

**2. Fail-safe JSON parsing.**
LLMs occasionally break format. The parsing layer strips markdown fences, attempts a strict `JSON.parse`, and — on any failure — **defaults to human escalation** instead of dropping the ticket:

```javascript
try {
  parsed = JSON.parse(raw);
} catch (e) {
  parsed = { category: 'Other', priority: 'Medium', solution: '',
             needs_human: true, email: '' };
}
```

No ticket is ever lost or mishandled due to malformed model output.

**3. Priority tied to business impact.**
Any issue that prevents an employee from working is forced to `High`, regardless of category.

**4. Constrained outputs.**
Fixed category and priority vocabularies keep downstream routing deterministic — no free-text classification drift.

## 🚀 Setup

1. **Import the workflow** — copy the contents of [`it-helpdesk-workflow.json`](it-helpdesk-workflow.json) and paste directly onto an empty n8n canvas (Ctrl+V / Cmd+V), or use *Import from File*.
2. **Add credentials:**
   - **Anthropic Chat Model** node → your Anthropic API key (or swap in any chat model node — OpenAI, Gemini, etc.)
   - Both **Send Email** nodes → SMTP credentials (or replace with Gmail/Outlook nodes)
3. **Update addresses** — replace the placeholder `helpdesk@yourcompany.com` and `it-team@yourcompany.com`.
4. **Activate** the workflow. Production endpoint: `POST /webhook/it-ticket`

## 🧪 Testing

Three payloads in [`tests/`](tests/) cover the full routing matrix:

| Test case | Expected route |
|---|---|
| **Clear escalation** — laptop shutting down after a drop, burning smell | `needs_human: true` → IT team |
| **Ambiguous escalation** — "something is wrong, nothing works" | `needs_human: true` → IT team |
| **Happy path** — forgotten Windows password | `needs_human: false` → auto-reply to employee |

Example (test mode — click *Listen for test event* first):

```bash
curl -X POST "https://YOUR-N8N-URL/webhook-test/it-ticket" \
  -H "Content-Type: application/json" \
  -d '{
    "employee_email": "user@company.com",
    "subject": "Forgot my password",
    "message": "I forgot my Windows login password and need to reset it."
  }'
```

## 📦 Request Format

```json
POST /webhook/it-ticket
{
  "employee_email": "user@company.com",
  "subject": "Can't connect to VPN",
  "message": "My VPN keeps disconnecting and I can't access the shared drive."
}
```

## 🔮 Possible Extensions

- Structured Output Parser node to replace the manual JSON parsing layer
- Ticket logging to PostgreSQL / Google Sheets for metrics (auto-resolution rate, category distribution)
- RAG over an internal IT knowledge base for company-specific solutions
- Slack/Teams notification on High-priority escalations

---
