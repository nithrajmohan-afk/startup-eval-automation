# Startup Evaluation Automation

An n8n workflow that automates end-to-end startup evaluation for corporate innovation and venture programs.

Built by [Nithin Rajmohan](https://www.linkedin.com/in/nithin-rajmohan/) — originally designed to replace 150+ hours of manual analyst work per evaluation cohort at a Fortune 100 consulting engagement.

---

## What It Does

Takes a startup name + website URL as input. Returns a structured evaluation brief in under 6 minutes, including:

- 5-dimension scoring rubric (Team / Market / Product / Traction / Fit)
- 3-line AI-generated summary
- Recommendation flag: Strong Consider / Consider / Pass
- 3 tailored demo call questions
- Red flag identification
- Auto-routing to Shortlist / Watch / Archive
- Real-time write to Microsoft Excel (SharePoint)
- Slack alert for Shortlist startups

---

## Architecture

```
Webhook Trigger
    ├── Crunchbase API (funding, investors, founding date)
    ├── LinkedIn via Proxycurl (founder background, team size)
    └── Website Scrape + HTML Extract (product positioning)
            │
        Merge Node
            │
        Build AI Prompt (Code Node)
            │
        Claude API (claude-3-5-sonnet) → structured JSON evaluation
            │
        Parse Score + Route (Code Node)
            │
        IF Node (score ≥ 70 → Shortlist, 50-69 → Watch, <50 → Archive)
            ├── Slack Notification (Shortlist only)
            └── Microsoft Excel (SharePoint) Write (all)
                    │
            Respond to Webhook
```

---

## Setup

### 1. Prerequisites

- n8n cloud or self-hosted instance (v1.0+)
- API keys: Crunchbase, Proxycurl, Anthropic
- Microsoft Excel (SharePoint) with the column schema below
- Slack workspace with a `#startup-pipeline` channel

### 2. Import Workflow

1. In n8n, go to **Workflows → Import from File**
2. Upload `startup_eval_workflow.json`
3. The workflow will appear with nodes disconnected from credentials — wire them up in the next step

### 3. Configure Credentials

Set up the following in n8n's **Credentials** panel:

| Credential | Type | Used By |
|-----------|------|---------|
| `CRUNCHBASE_API_KEY` | Header Auth | Crunchbase Enrichment node |
| `PROXYCURL_API_KEY` | Bearer Token | LinkedIn Enrichment node |
| `ANTHROPIC_API_KEY` | Header Auth | Claude AI Evaluation node |
| Microsoft 365 OAuth | Microsoft OAuth2 | Write to Excel via SharePoint node |
| Slack OAuth | Slack App | Slack Notification node |

You can also set environment variables directly in n8n's settings panel and reference them as `{{$env.KEY_NAME}}`.

### 4. Microsoft Excel (SharePoint) Schema

Create an Excel worksheet named `Pipeline` in a SharePoint-hosted workbook with these columns:

```
Startup Name | Website | Total Score | Team | Market Size | Product Differentiation |
Traction Signals | Strategic Fit | Summary | Recommendation | Demo Q1 | Demo Q2 | Demo Q3 |
Red Flags | Bucket | Processed At
```

Set `SHAREPOINT_WORKBOOK_ID` in your environment variables to the sheet's ID (found in the URL).

### 5. Activate + Test

1. Activate the workflow in n8n
2. Copy the webhook URL from the **Webhook Trigger** node
3. Send a test POST request:

```bash
curl -X POST https://your-n8n-instance.com/webhook/startup-eval \
  -H "Content-Type: application/json" \
  -d '{"startup_name": "Acme AI", "website": "https://acme.ai"}'
```

Expected response:
```json
{
  "status": "processed",
  "startup": "Acme AI",
  "score": 74,
  "recommendation": "Strong Consider"
}
```

---

## Sample Output

```json
{
  "startup_name": "Acme AI",
  "summary": "Acme AI replaces manual contract review with an LLM-powered extraction engine. Their product ingests any contract format and outputs structured data in under 90 seconds. The differentiation is their pre-trained legal-domain model, which outperforms general LLMs on clause classification by 34%.",
  "scores": {
    "team": 16,
    "market_size": 18,
    "product_differentiation": 15,
    "traction_signals": 14,
    "strategic_fit": 11
  },
  "total_score": 74,
  "recommendation": "Strong Consider",
  "demo_questions": [
    "What's your accuracy rate on non-standard clause formats, and how do you handle edge cases?",
    "How does your model stay current as contract law evolves across jurisdictions?",
    "What does your enterprise onboarding look like — and what's your current ACV?"
  ],
  "red_flags": "Limited public traction data; pre-revenue stage means strategic fit validation depends on pilot terms.",
  "bucket": "Shortlist"
}
```

---

## Scoring Rubric

| Dimension | Max Score | What the AI evaluates |
|----------|-----------|----------------------|
| Team | 20 | Founder pedigree, domain expertise, prior exits or relevant experience |
| Market Size | 20 | TAM signals, growth rate, enterprise vs SMB fit |
| Product Differentiation | 20 | Defensibility, moat, clarity of problem-solution fit |
| Traction Signals | 20 | Funding, customers named, press, growth indicators |
| Strategic Fit | 20 | Relevance to client's innovation agenda, partnership potential |

**Routing thresholds:** Shortlist ≥ 70 | Watch 50–69 | Archive < 50

---

## Prompt Template

The system prompt used in the Claude AI Evaluation node is in `/prompts/evaluation_prompt.txt`. Key design decisions:

- Instructs the model to return JSON only — no prose wrapping
- Calibration examples are embedded in the prompt for the scoring dimensions
- Red flags are only surfaced if there is a specific signal, not as a generic placeholder

---

## Files

```
/
├── startup_eval_workflow.json     # Import this into n8n
├── prompts/
│   └── evaluation_prompt.txt     # Full system prompt for Claude
├── sample_outputs/
│   └── sample_eval_output.json   # Example of a processed evaluation
└── README.md
```

---

## Built With

- [n8n](https://n8n.io) — workflow automation
- [Anthropic Claude](https://anthropic.com) — AI evaluation synthesis
- [Crunchbase API](https://data.crunchbase.com/docs) — company enrichment
- [Proxycurl](https://nubela.co/proxycurl) — LinkedIn data
- Microsoft Excel (SharePoint) API — output storage
- Slack API — real-time alerts

---

## License

MIT
