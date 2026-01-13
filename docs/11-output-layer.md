# 📄 Output Layer

> **Purpose**: Final presentation layer. Formats and delivers three structured outputs: Filtered Timeline, Impact Summary, and Mitigation Workflow.

---

## What It Does

The Output Layer receives validated outputs and formats them into their final deliverable form. It is a **formatting and delivery layer** — no AI processing.

## Three Outputs

| Output | Source | Format |
|---|---|---|
| **Noise Workflow Outputs — Filtered Timeline** | WF-9 Noise Agent → Evaluator | Chronological timeline document |
| **Impact Summary — Factual Report** | WF-10 Impact Agent → Evaluator | Structured impact report |
| **Mitigation Workflow — Action Plan** | WF-25 Mitigation Agent (direct) | Step-by-step action plan |

## Delivery Channels

| Channel | Format | Use Case |
|---|---|---|
| REST API | JSON response | Programmatic consumption |
| ICM Portal | Rendered HTML/Markdown | Human review in IcM |
| Email notification | HTML email | Stakeholder communication |
| Teams/Slack | Adaptive Card / Block Kit | Real-time incident channel |

## Connection to Deployment Model

The Deployment Model has an **Overlays** connection to the Output Layer, meaning the deployment configuration (dev/staging/prod) controls which delivery channels are active, API endpoints, and formatting templates.
