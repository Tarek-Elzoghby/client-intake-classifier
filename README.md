# AI-Powered Client Intake Classifier

An n8n workflow that reads a client's project description on submission, classifies it with an LLM, and routes it — tailored confirmation email, internal notification, and a logged row in Google Sheets, all varying by category.

This was built as a personal project to learn AI-integrated automation properly, not for a specific client. The scenario (an automation agency's intake form) is realistic on purpose, and the workflow is built to a standard I'd be comfortable handing to a real client if the opportunity came up.

## What it does

A submission comes in through a form (name, email, project description). Gemini classifies the description into one of five categories, with a short reasoning field attached to every classification:

- **simple_automation** — a trigger with one clear action, no branching or AI judgment needed
- **ai_workflow** — the value is in AI reasoning itself (summarization, sentiment, drafting, judgment calls)
- **custom_integration** — real conditional logic and/or multi-system orchestration, driven by fixed rules rather than AI reasoning
- **ambiguous** — a real project, but genuinely unclear between categories (human review needed)
- **invalid** — not a workable project description (gibberish, out of scope, or missing critical information)

`invalid` carries a second field, `has_partial_intent`, distinguishing a vague-but-real attempt from something that isn't a lead at all — each gets a different client email.

If classification itself fails (network issue, rate limit, bad response), the workflow retries three times before falling back to a `system_error` path — distinguishable in the data from `ambiguous`, and routed to the same human-review flow without ever crashing the run.

Every outcome — including a failed or invalid one — gets logged to Google Sheets. Logging doesn't depend on any email succeeding.

## How it's built

- **Trigger:** n8n Form Trigger (name, email, description)
- **Classification:** Basic LLM Chain → Gemini Chat Model → Structured Output Parser, enforcing a fixed JSON schema (`reasoning`, `category`, `has_partial_intent`)
- **Routing:** a Switch node with six rules (one per category, plus `system_error` converging on the same path as `ambiguous`)
- **Outputs:** category-specific Gmail nodes (client + internal) and one shared Google Sheets logging node all branches converge on

Full reasoning behind the design decisions, the bugs hit along the way, and what changed and why: [PROCESS.md](./PROCESS.md)

## Setup

You'll need:
- An n8n instance (cloud or self-hosted)
- A Google Gemini API key
- A Google Cloud project with the Gmail API and Google Sheets API enabled, and OAuth credentials set up for both (register your n8n callback URL as an authorized redirect URI, and add yourself as a test user if the app is unverified)
- A Google Sheet to log submissions into

To run it:
1. Import `workflow.json` into n8n
2. Connect your Gemini, Gmail, and Google Sheets credentials to the relevant nodes
3. Point the Google Sheets node at your own sheet
4. Activate the workflow and submit the form to test

## Status

Built and tested end to end, including deliberate tests against the boundaries between categories (not just each category in isolation), a forced classification failure to verify the retry/fallback path, and real inbox checks to confirm each branch sends the right email to the right recipient.

Not deployed for a live client. Open to adapting it for one.
