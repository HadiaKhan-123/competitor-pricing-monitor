# Competitor Pricing Intelligence Agent

An automated agent that monitors competitor pricing pages daily, detects changes, generates AI summaries, and delivers briefings to Slack.

## What It Does

- Scrapes 3 competitor websites every morning at 8am
- Compares today's content against yesterday's snapshot
- Sends changes to an LLM which summarizes what changed and what it means for sales
- Logs everything to Google Sheets with timestamps
- Sends a Slack notification when the daily run completes

## Architecture

Schedule Trigger → Scrape (Jina API) → Compare (JavaScript) → Summarize (OpenRouter LLM) → Log (Google Sheets) → Notify (Slack)

All 3 competitors run in parallel branches.

## Tech Stack

| Tool | Purpose |
|------|---------|
| n8n | Workflow orchestration |
| Jina Reader API | Web scraping (free, no key needed) |
| OpenRouter | LLM summarization (free tier) |
| Google Sheets | Data storage and logging |
| Slack Webhook | Notifications |

## Output

Each daily run produces:
- `change_detected`: true/false per competitor
- `summary`: 80-word AI analysis of current pricing strategy
- `sales_action`: one-sentence recommendation for the sales team

## Workflow

[Workflow Canvas](https://github.com/HadiaKhan-123/competitor-pricing-monitor/commit/71408f44806af4e672d073f4f1b21bc5f2331081)

## Sample Output (Google Sheets)

[Google Sheets Output](https://github.com/HadiaKhan-123/competitor-pricing-monitor/blob/main/Sample%20Output%20(Google%20Sheets))

## Limitations

- Runs on localhost — laptop must be on at 8am (fix: deploy to Oracle Cloud Free VM)
- Make.com blocks scrapers via CAPTCHA — replaced with Activepieces
- AI summaries use first 600 chars of page content due to free tier limits

## Skills Demonstrated

API integration, JavaScript (data transformation + comparison logic), workflow automation, prompt engineering, OAuth setup, data architecture, debugging
