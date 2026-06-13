# AI Sales Intelligence Agent — Project Documentation

> **Project Type:** Portfolio Project — Automation & AI  
> **Status:** Complete and Working  
> **Build Duration:** ~1 day across multiple sessions

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Final Architecture](#final-architecture)
3. [Tech Stack](#tech-stack)
4. [Workflow Diagram](#workflow-diagram)
5. [Step-by-Step Build Log](#build-log)
6. [All JavaScript Code](#all-javascript-code)
7. [Errors Encountered and How They Were Solved](#errors-and-solutions)
8. [Google Sheets Structure](#google-sheets-structure)
9. [Known Limitations](#known-limitations)
10. [Interview Demo Guide](#interview-demo-guide)
11. [Resume Description](#resume-description)
12. [Future Improvements](#future-improvements)

---

## Project Overview

### What It Does

An automated system that:
1. Scrapes 3 competitor pricing pages every day at 8am
2. Compares today's content with yesterday's to detect changes
3. Sends the content to an AI model which summarizes pricing strategy and recommends a sales action
4. Logs everything to Google Sheets with timestamps
5. Sends a Slack notification when the daily run completes

### Competitors Monitored

| Competitor | URL |
|-----------|-----|
| Zapier | https://zapier.com/pricing |
| Activepieces | https://www.activepieces.com/pricing |
| n8n | https://n8n.io/pricing |

> Note: The original second competitor was Make.com, which was replaced by Activepieces during the build. Make.com's anti-bot protection returns a CAPTCHA page instead of pricing content, making it unscrapable with a free tool like Jina.

### Problem It Solves

Sales teams waste hours manually checking competitor websites. This agent automates the entire process and delivers intelligence directly to Slack — every morning, automatically, at zero cost.

### One-Line Pitch

> "I built an AI agent that monitors competitor pricing pages daily, detects changes, generates AI summaries with sales recommendations, and delivers briefings to Slack — fully automated, zero cost to run."

---

## Final Architecture

```
[Schedule Trigger: 8am Daily]
         ↓
[Scrape 3 Competitor URLs via Jina API]  ← parallel, simultaneous
         ↓
[Read Yesterday's Content from Google Sheets]
         ↓
[Compare: Has content changed?]  ← JavaScript logic
         ↓
[AI Summarize via OpenRouter LLM]  ← free tier
         ↓
[Log change + summary + sales_action to Google Sheets]
         ↓
[Update competitor_snapshots with today's content]
         ↓
[Merge all 3 branches]
         ↓
[Send Slack briefing notification]
```

---

## Tech Stack

| Tool | Purpose | Why Chosen |
|------|---------|------------|
| **n8n** (self-hosted, localhost:5678) | Workflow orchestration | Free, visual, already installed |
| **Jina Reader API** | Web scraping | Free, no API key, handles JS-rendered pages, single GET request |
| **OpenRouter** (free tier) | AI summarization | No daily quota like Gemini, multiple free models available |
| **Google Sheets** | Data storage | Free, no database setup, easy to demo |
| **Slack Webhook** | Notifications | 2-minute setup vs 45-minute Gmail OAuth |
| **Google OAuth** | Google Sheets authentication | Required by Google to connect external apps |

### Tools Considered and Rejected

| Tool | Reason Rejected |
|------|----------------|
| Anthropic/Claude API | Requires paid credits upfront, no free tier |
| Gemini API (free tier) | Daily quota exhausted quickly during testing |
| Make.com (competitor) | Anti-bot protection returns CAPTCHA, not scrapable for free |
| Gmail notifications | OAuth setup too complex for a notification-only use case |
| Word-based JavaScript diff | Too many false positives on minor text changes |

---

## Workflow Diagram

```
Schedule Trigger (8am)
│
├─────────────────────────────────────────────────┐
│                        │                         │
▼                        ▼                         ▼
Scrape Zapier       Scrape Activepieces        Scrape n8n
(Jina API)          (Jina API)                 (Jina API)
│                        │                         │
▼                        ▼                         ▼
Read Yesterday      Read Yesterday             Read Yesterday
Zapier              Activepieces               n8n
(Google Sheets)     (Google Sheets)            (Google Sheets)
│                        │                         │
▼                        ▼                         ▼
Compare Zapier      Compare Activepieces        Compare n8n
(JS Code Node)      (JS Code Node)             (JS Code Node)
│                        │                         │
▼                        ▼                         ▼
Summarize Zapier    Summarize Activepieces      Summarize n8n
(OpenRouter API)    (OpenRouter API)            (OpenRouter API)
│                        │                         │
▼                        ▼                         ▼
Log Zapier Change   Log Activepieces Change     Log n8n Change
(Sheets append)     (Sheets append)             (Sheets append)
│                        │                         │
▼                        ▼                         ▼
Update Snapshot     Update Snapshot             Update Snapshot
Zapier              Activepieces                n8n
(Sheets update)     (Sheets update)             (Sheets update)
│                        │                         │
└────────────────────────┼─────────────────────────┘
                         ▼
                    Merge Node
                  (append mode)
                         │
                         ▼
               Send Slack Briefing
               (HTTP POST webhook)
```

---

## Build Log

### Step 1: Google Sheets Setup

**Sheet Name:** `Sales Intelligence Agent`

Two tabs created:

#### Tab 1: `competitor_snapshots`

Stores the most recent scraped content per competitor. Updated on every run so tomorrow's comparison has fresh data.

| Column | Purpose |
|--------|---------|
| `competitor_name` | Identifier — Zapier, Activepieces, n8n |
| `url` | The pricing page URL being monitored |
| `content` | Full scraped text — overwritten on each run |
| `last_scraped` | Timestamp of last successful scrape |

**Initial rows loaded:**
- Row 2: Zapier → https://zapier.com/pricing
- Row 3: Activepieces → https://www.activepieces.com/pricing
- Row 4: n8n → https://n8n.io/pricing

#### Tab 2: `change_log`

Append-only log. Every run adds one row per competitor. Never overwritten.

| Column | Purpose |
|--------|---------|
| `timestamp` | When this entry was logged |
| `competitor_name` | Which competitor |
| `change_detected` | true / false |
| `change_type` | pricing / product / messaging / none |
| `summary` | AI-generated summary of current pricing |
| `sales_action` | AI-generated one-sentence sales recommendation |

---

### Step 2: API Key Setup

**What happened:**
- Tried Anthropic/Claude API — requires upfront payment, no free tier. Skipped.
- Tried Google Gemini free tier — daily quota exhausted quickly during testing.
- Switched to OpenRouter.ai — aggregates multiple free AI models, no daily hard quota.

**Working model found:** `google/gemma-4-31b-it:free`

---

### Step 3: Schedule Trigger

**Node type:** Schedule Trigger (native n8n node)

**Settings:**
- Trigger Interval: Days
- Days Between Triggers: 1
- Trigger at Hour: 8
- Trigger at Minute: 0

During testing, the orange "Execute workflow" button was used to trigger manually instead of waiting for 8am.

---

### Step 4: Jina Scrapers

**Why Jina?**
Jina Reader API (`r.jina.ai`) is free, requires no API key, handles JavaScript-rendered pages, and returns clean readable text from any URL. You prepend `https://r.jina.ai/` to any URL and it returns the page as clean markdown.

**Three HTTP Request nodes:**

#### Scrape Zapier
- Method: GET
- URL: `https://r.jina.ai/https://zapier.com/pricing`

#### Scrape Activepieces
- Method: GET
- URL: `https://r.jina.ai/https://www.activepieces.com/pricing`

#### Scrape n8n
- Method: GET
- URL: `https://r.jina.ai/https://n8n.io/pricing`

**Important:** All 3 scrapers connect in **parallel** from the Schedule Trigger — not in a chain. This means they run simultaneously, not one after another.

**Sample output from Scrape Zapier:**
```
Title: Plans & Pricing | Zapier
URL Source: https://zapier.com/pricing
Markdown Content:
New: Zaps, Tables, Forms, and Zapier MCP are now all available in one unified plan...
## Free — $0/mo
## Professional — Starting from $19.99/mo
## Team — Starting from $69/mo
## Enterprise — Contact for pricing
```

---

### Step 5: Google OAuth and Sheets Read Nodes

#### Google Cloud Setup

**APIs Enabled:**
- Google Sheets API
- Google Drive API

**OAuth Consent Screen:**
- App name: `n8n Intelligence Agent`
- User type: External

**OAuth Client settings:**
- Application type: Web application
- Authorized redirect URI: `http://localhost:5678/rest/oauth2-credential/callback`

#### Three "Read Yesterday" Nodes

Each reads one row from `competitor_snapshots` to get yesterday's stored content for comparison.

**Node: `Read Yesterday Zapier`**
- Operation: Get Row(s)
- Sheet: competitor_snapshots
- Filter: `competitor_name` = `Zapier`

**Node: `Read Yesterday Activepieces`**
- Same settings, Filter: `competitor_name` = `Activepieces`

**Node: `Read Yesterday n8n`**
- Same settings, Filter: `competitor_name` = `n8n`

On the first run, `content` returns empty — this is expected and handled by the Compare node logic.

---

### Step 6: Compare Nodes

**Purpose:** Compare today's scraped content with yesterday's stored content. Detect whether this is the first run or whether content has actually changed.

**Node type:** Code (JavaScript)

**Three nodes:** `Compare Zapier`, `Compare Activepieces`, `Compare n8n`

The logic is the same across all three — only the competitor name and source node references differ.

**Key fields returned:**
- `hasChanged` — false = no change, true = content was modified since last run
- `isFirstRun` — true = no previous data exists (expected on Day 1)
- `todayContent` — truncated to 2000 chars to avoid overwhelming the AI
- `yesterdayContent` — empty string on first run
- `cleanPrompt` — pre-sanitized prompt string (newlines removed) safe for JSON injection

---

### Step 7: Summarize Nodes

**Purpose:** Send competitor content to an LLM and get a structured summary + sales recommendation.

**Node type:** HTTP Request (POST)

**Why not a Code node?** n8n version 2.21.7 does not support `fetch()` or `$http` in Code nodes. All external API calls must use the HTTP Request node.

**Why cleanPrompt?** Scraped content contains `\n` characters. Injecting raw content directly into a JSON body breaks the JSON. The solution: build a sanitized single-line string in the Compare (Code) node, then inject the clean string into the HTTP Request body.

#### Summarize Node Settings (same pattern for all 3)

- Method: POST
- URL: `https://openrouter.ai/api/v1/chat/completions`
- Header: `Authorization: Bearer YOUR_OPENROUTER_KEY`
- Body Content Type: JSON

**JSON Body (final working version):**
```json
{
  "model": "google/gemma-4-31b-it:free",
  "messages": [{
    "role": "user",
    "content": "{{ 'Analyze this competitor pricing. Reply in this exact format - SUMMARY: [80 words on pricing tiers] SALES_ACTION: [one sentence for sales team] - Content: ' + $json.todayContent.substring(0, 600).replace(/\n/g, ' ').replace(/\"/g, '') }}"
  }]
}
```

**OpenRouter response structure:**
```json
{
  "choices": [{
    "message": {
      "content": "SUMMARY: Zapier has transitioned... SALES_ACTION: Emphasize our standalone flexibility..."
    }
  }]
}
```

---

### Step 8: Log to Google Sheets

**Three nodes:** `Log Zapier Change`, `Log Activepieces Change`, `Log n8n Change`

**Node type:** Google Sheets — Append Row

**Values mapped:**
- `timestamp` → `{{ $now.toISO() }}`
- `competitor_name` → `{{ $('Compare Zapier').first().json.competitor }}`
- `change_detected` → `{{ $('Compare Zapier').first().json.hasChanged }}`
- `summary` → `{{ $json.choices[0].message.content.split('SALES_ACTION:')[0].replace('SUMMARY:', '').trim() }}`
- `sales_action` → `{{ $json.choices[0].message.content.split('SALES_ACTION:')[1].trim() }}`

The `summary` and `sales_action` fields are split from the AI response using the `SALES_ACTION:` delimiter. This is why the prompt must ask the AI to respond in that exact format.

---

### Step 9: Update Snapshot Nodes

**Three nodes:** `Update Snapshot Zapier`, `Update Snapshot Activepieces`, `Update Snapshot n8n`

**Node type:** Google Sheets — Update Row

**Purpose:** Overwrite yesterday's content with today's scraped content so tomorrow's run has fresh data to compare against.

**Updated columns per run:**
- `content` → today's scraped text
- `last_scraped` → current timestamp

Row matched by `competitor_name` value.

---

### Step 10: Merge and Slack

#### Merge Node

**Why needed:** The 3 branches run in parallel and finish at different times. Without a Merge node, Slack would fire 3 separate times — once when each branch finishes. The Merge node waits for all 3 to complete before triggering Slack.

**Settings:**
- Mode: Combine → Append
- Inputs: Update Snapshot Zapier, Update Snapshot Activepieces, Update Snapshot n8n

#### Send Slack Briefing Node

**Node type:** HTTP Request

**How Slack Webhook works:** Slack provides a special URL. POST JSON to it with a `text` field and the message appears in your chosen Slack channel. No authentication beyond the URL itself.

**Setup:**
1. Go to `api.slack.com/apps`
2. Create App → From Scratch
3. Enable Incoming Webhooks
4. Add to Workspace → pick your channel
5. Copy the webhook URL

**JSON Body (final working version):**
```json
{
  "text": "Daily Competitor Intelligence report has been logged. Check the change_log tab in your Google Sheet for today's AI summaries."
}
```

**Why simplified?** Injecting `$json.choices[0].message.content` from the Summarize nodes into the Slack JSON body caused the JSON to break — AI output contains quotes and newlines that corrupt the JSON string. The full AI summaries are already saved to Google Sheets. Slack is a trigger notification only.

---

## All JavaScript Code

### Compare Zapier (final version)

```javascript
const todayContent = $('Scrape Zapier').first().json.data || 
                     $('Scrape Zapier').first().json.content || 
                     $('Scrape Zapier').first().json.body || "";

const yesterdayContent = $('Read Yesterday Zapier').first().json.content || "";

const isFirstRun = yesterdayContent === "";
const hasChanged = !isFirstRun && (todayContent !== yesterdayContent);

const cleanPrompt = "Competitor: Zapier. Pricing: " + 
  todayContent.substring(0, 500) + 
  ". Summarize pricing tiers for sales team in 80 words.";

return [{
  json: {
    competitor: "Zapier",
    url: "https://zapier.com/pricing",
    hasChanged: hasChanged,
    isFirstRun: isFirstRun,
    todayContent: todayContent.substring(0, 2000),
    yesterdayContent: yesterdayContent.substring(0, 2000),
    timestamp: new Date().toISOString(),
    cleanPrompt: cleanPrompt
  }
}];
```

### Compare Activepieces (final version)

```javascript
const todayContent = $('Scrape Make').first().json.data || 
                     $('Scrape Make').first().json.content || "";

const yesterdayContent = $('Read Yesterday Make').first().json.content || "";

const isFirstRun = yesterdayContent === "";
const hasChanged = !isFirstRun && (todayContent !== yesterdayContent);

const cleanPrompt = "Competitor: Activepieces. Pricing: " + 
  todayContent.substring(0, 500) + 
  ". Summarize pricing tiers for sales team in 80 words.";

return [{
  json: {
    competitor: "Activepieces",
    url: "https://www.activepieces.com/pricing",
    hasChanged: hasChanged,
    isFirstRun: isFirstRun,
    todayContent: todayContent.substring(0, 2000),
    yesterdayContent: yesterdayContent.substring(0, 2000),
    timestamp: new Date().toISOString(),
    cleanPrompt: cleanPrompt
  }
}];
```

### Compare n8n (final version)

```javascript
const todayContent = $('Scrape n8n').first().json.data || 
                     $('Scrape n8n').first().json.content || "";

const yesterdayContent = $('Read Yesterday n8n').first().json.content || "";

const isFirstRun = yesterdayContent === "";
const hasChanged = !isFirstRun && (todayContent !== yesterdayContent);

const cleanPrompt = "Competitor: n8n. Pricing: " + 
  todayContent.substring(0, 500) + 
  ". Summarize pricing tiers for sales team in 80 words.";

return [{
  json: {
    competitor: "n8n",
    url: "https://n8n.io/pricing",
    hasChanged: hasChanged,
    isFirstRun: isFirstRun,
    todayContent: todayContent.substring(0, 2000),
    yesterdayContent: yesterdayContent.substring(0, 2000),
    timestamp: new Date().toISOString(),
    cleanPrompt: cleanPrompt
  }
}];
```

---

## Errors and Solutions

### Error 1: Anthropic API requires paid credits upfront
**Error:** Payment screen with no free tier option  
**Root cause:** Anthropic removed the free credit for new accounts  
**Solution:** Switched to Google Gemini free tier  
**Lesson:** Always identify a free fallback AI API before starting

---

### Error 2: Gemini API quota exhausted
**Error:** `Quota exceeded for metric: generativelanguage.googleapis.com/generate_content_free_tier_requests`  
**Root cause:** Too many test calls during development burned through the daily free quota  
**Solution:** Switched to OpenRouter.ai — more lenient free tier, multiple models available  
**Lesson:** Free AI APIs have hard daily limits. Don't run more than 5-10 test calls per day.

---

### Error 3: JSON Body invalid — bad control character
**Error:** `Bad control character in string literal in JSON at position 225`  
**Root cause:** Scraped content contains `\n` characters. When injected directly into a JSON string via `{{ $json.todayContent }}`, these break the JSON syntax.  
**Solution:** Built a `cleanPrompt` string in the Code node first (JavaScript handles the escaping), then injected the clean string  
**Lesson:** Never inject raw scraped content into JSON templates. Always sanitize first.

---

### Error 4: fetch is not defined
**Error:** `ReferenceError: fetch is not defined [line 8]`  
**Root cause:** n8n version 2.21.7 Code nodes do not support the browser `fetch()` API  
**Solution:** Used HTTP Request node for all external API calls  
**Lesson:** n8n Code nodes are for data transformation only — use dedicated nodes for external calls

---

### Error 5: $http is not defined
**Error:** `ReferenceError: $http is not defined`  
**Root cause:** `$http` was a built-in in older n8n versions but removed in 2.21.7  
**Solution:** Confirmed HTTP Request node is the correct approach — no workarounds needed  
**Lesson:** Always check n8n version compatibility for built-in helpers

---

### Error 6: SyntaxError — Unexpected token 'const'
**Error:** `SyntaxError: Unexpected token 'const' ...const const data`  
**Root cause:** Pasting new code into the Code node without fully clearing old code first — resulted in duplicate keywords  
**Solution:** Always press `Ctrl+A` to select all, then Delete, before pasting new code  
**Lesson:** n8n's code editor doesn't auto-clear on paste

---

### Error 7: OpenRouter free model not found
**Error:** `No endpoints found for mistralai/mistral-7b-instruct:free`  
**Root cause:** OpenRouter's free model catalog changes frequently — the suggested model ID was outdated  
**Solution:** Tried multiple models until finding one that worked:
- ❌ `mistralai/mistral-7b-instruct:free` — discontinued
- ❌ `meta-llama/llama-3.2-3b-instruct:free` — rate limited
- ❌ `google/gemma-3-4b-it:free` — unavailable
- ✅ `google/gemma-4-31b-it:free` — working

**Lesson:** For free models, always verify current availability at `openrouter.ai/models?q=free`

---

### Error 8: OpenRouter 429 rate limit
**Error:** `The service is receiving too many requests from you`  
**Root cause:** Multiple rapid test calls during debugging  
**Solution:** Waited 60 minutes for the rate limit to reset, then ran the full workflow once instead of individual nodes  
**Lesson:** Test the whole workflow once rather than individual nodes repeatedly — saves API calls

---

### Error 9: Missing Authentication header
**Error:** `Missing Authentication header`  
**Root cause:** OpenRouter key entered without the `Bearer ` prefix  
**Solution:** Changed header value from `sk-or-v1-...` to `Bearer sk-or-v1-...`  
**Lesson:** Most APIs require `Bearer ` prefix in the Authorization header

---

### Error 10: Slack node cross-branch reference error
**Error:** `ExpressionError: Node 'Summarize Make' hasn't been executed`  
**Root cause:** Slack node was only connected to the Zapier branch. When it tried to reference `$('Summarize Make')`, that branch hadn't run from Slack's perspective.  
**Solution:** Added a Merge node before Slack that collects output from all 3 branches. Simplified the Slack message to plain text.  
**Lesson:** In parallel branch workflows, always merge before referencing data from multiple branches.

---

### Error 11: Parallel nodes connected in a chain (architecture bug)
**Issue:** All 3 scraper nodes were connected in a line: Zapier → Make → n8n, so they ran sequentially instead of in parallel  
**Solution:** Deleted the connections between scraper nodes. Connected all 3 directly from Schedule Trigger.  
**How to connect parallel in n8n:** Hover over Schedule Trigger → drag from the right edge dot → drop onto each scraper separately  
**Lesson:** Parallel execution in n8n = multiple connections from the same source node, not nodes chained together

---

### Error 12: Compare Make false positive — wrong competitor logged
**Issue:** Compare Make node was logging `competitor: "Make"` but the actual competitor being monitored was Activepieces. The `competitor_name` field in the output was wrong.  
**Root cause:** The node code still had `competitor: "Make"` hardcoded after the switch from Make.com to Activepieces  
**Solution:** Updated the competitor field in the Compare node code to `"Activepieces"` and updated the URL to `https://www.activepieces.com/pricing`  
**Lesson:** When you replace a competitor mid-build, check every hardcoded reference in the code nodes — not just the scraper URL

---

## Google Sheets Structure

### Tab: competitor_snapshots

Stores the most recent scraped content. One row per competitor. Overwritten on each run.

| competitor_name | url | content | last_scraped |
|---|---|---|---|
| Zapier | https://zapier.com/pricing | [full Zapier pricing text] | 2026-06-13T... |
| Activepieces | https://www.activepieces.com/pricing | [full Activepieces pricing text] | 2026-06-13T... |
| n8n | https://n8n.io/pricing | [full n8n pricing text] | 2026-06-13T... |

### Tab: change_log

Append-only log. One new row per competitor per daily run.

| timestamp | competitor_name | change_detected | change_type | summary | sales_action |
|---|---|---|---|---|---|
| 2026-06-13T... | Zapier | false | pricing | [AI summary] | [sales recommendation] |
| 2026-06-13T... | Activepieces | false | pricing | [AI summary] | [sales recommendation] |
| 2026-06-13T... | n8n | false | pricing | [AI summary] | [sales recommendation] |

---

## Known Limitations

### 1. change_detected is always FALSE on Day 1
**Why:** On the first run, `yesterdayContent` is empty. The logic correctly returns `isFirstRun: true` and `hasChanged: false`. There is nothing to compare against.  
**When it works:** Day 2 onwards, the system compares today vs yesterday and detects real changes.

### 2. AI summaries use truncated content
**Why:** The prompt only sends the first 600-800 characters of the scraped page to stay within free tier limits.  
**Impact:** Summaries may miss details from lower sections of pricing pages.  
**Fix:** Increase the substring limit to 1500-2000 characters once on a paid API tier.

### 3. Workflow only runs when laptop is on
**Why:** n8n is running on localhost. If the laptop is off at 8am, the trigger won't fire.  
**Fix:** Deploy to Oracle Cloud Free Tier VM (always-on, free) or n8n Cloud ($20/mo).

### 4. OpenRouter free tier rate limits
**Issue:** Free models have per-minute and per-day rate limits. Running the workflow multiple times in quick succession will hit the limit.  
**Fix:** Add $5-10 credit to OpenRouter — removes all rate limits at very low cost for 3 calls/day.

### 5. Slack message contains no AI summary
**Why:** Injecting AI-generated content directly into a Slack webhook JSON body breaks the JSON due to quotes and newlines in the AI response.  
**Current state:** Slack sends a plain notification. Full summaries are in Google Sheets.  
**Fix:** Sanitize the AI response (escape quotes, remove newlines) before injecting into the Slack body, or use Slack's Block Kit format.

---

## Interview Demo Guide

### What to say when presenting this project

> "I built an AI agent that monitors competitor pricing pages. Every morning it scrapes three competitors, compares the content against yesterday's snapshot, sends the content to an LLM which produces a pricing summary and a sales recommendation, then logs everything to Google Sheets and sends a Slack notification. The whole thing runs automatically, costs essentially nothing, and took about a day to build."

### Technical questions you should be able to answer

**Q: Why Jina instead of BeautifulSoup or Puppeteer?**
> "For a portfolio project on a schedule, I needed zero cost and zero maintenance. Jina Reader API converts any URL to clean text with a single GET request — no browser automation, no dependencies, no API key needed. For production with anti-bot sites, I'd upgrade to ScrapingBee or Browserless."

**Q: How does the change detection work?**
> "String comparison — if today's scraped text differs from yesterday's stored text, `hasChanged` returns true. I deliberately avoided word-level diffing because it creates too many false positives on minor text changes like button labels. I send both versions to the LLM and let it determine what is meaningfully different."

**Q: Why OpenRouter instead of OpenAI?**
> "Cost and availability. OpenAI charges from the first call. OpenRouter aggregates multiple providers and has free tiers. For a daily job running 3 calls, I can run for months at zero cost. In production I'd use Claude Haiku or GPT-4o-mini — both under $1/month at this volume."

**Q: Why did you replace Make.com with Activepieces?**
> "Make.com has anti-bot protection that returns a CAPTCHA page instead of pricing content. Since I'm using Jina's free scraper, I can't bypass it. I switched to Activepieces, which is a direct competitor in the no-code automation space and doesn't block scrapers."

**Q: What would you do differently in production?**
> "Three things: Deploy on a cloud VM so it's not dependent on my laptop. Use semantic similarity with embeddings instead of string comparison for smarter change detection. And include the AI summary directly in the Slack message rather than requiring someone to open Google Sheets."

### Demo flow

1. Show n8n canvas — the full workflow with all nodes green
2. Show Google Sheets — `competitor_snapshots` tab with real scraped content
3. Show Google Sheets — `change_log` tab with timestamps, summaries, and sales actions
4. Show Slack channel — the notification message
5. Run the workflow live — click Execute Workflow and show nodes turning green in sequence

---

## Resume Description

### One line
Built an AI-powered competitor intelligence agent using n8n, Jina.ai, and OpenRouter that monitors pricing pages daily, detects changes, and delivers AI summaries with sales recommendations to Google Sheets and Slack automatically.

### Three bullets
- Designed and built an end-to-end automation agent that scrapes 3 competitor websites daily, compares content snapshots, and generates structured AI output (pricing summary + sales recommendation) using OpenRouter's LLM API
- Architected a parallel-branch n8n workflow with schedule triggers, web scraping (Jina Reader API), JavaScript data transformation, Google Sheets logging, and Slack webhook delivery
- Implemented change detection logic that distinguishes first-run initialization from actual pricing changes, with structured prompt engineering to extract consistent AI output for downstream processing

### Skills this project demonstrates
- API integration (OpenRouter, Jina, Slack, Google Sheets)
- JavaScript (data transformation, comparison logic, prompt sanitization)
- Workflow automation (parallel branches, merge nodes, schedule triggers)
- Prompt engineering (structured output format for reliable downstream parsing)
- OAuth setup (Google Cloud Console, API enablement, credentials)
- Data architecture (snapshot vs append-log pattern)
- Debugging (resolved 12 distinct errors across APIs, code, and workflow wiring)

---

## Future Improvements

| Improvement | Effort | Impact |
|---|---|---|
| Deploy to Oracle Cloud Free VM | High | Workflow runs even when laptop is off |
| Include AI summary in Slack message | Low | More useful notifications without opening Sheets |
| Semantic diff using embeddings | Medium | Smarter change detection, fewer false positives |
| Monitor 10+ competitors | Low | Scale by duplicating the branch pattern |
| Add retry logic for rate limits | Medium | More resilient in production |
| Add email delivery | Medium | Stakeholders without Slack can receive briefings |
| Increase content window to 1500+ chars | Low | More complete AI summaries |

---

*Documentation created: June 2026*  
*Total build time: ~1 day across multiple sessions*
