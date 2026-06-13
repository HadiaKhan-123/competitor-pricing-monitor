# 🧠 AI Sales Intelligence Agent — Complete Project Documentation

> **Project Type:** Portfolio Project — Automation & AI Development
> **Builder:** MBA Student (Fresher), Delhi
> **Status:** ✅ Complete and Working
> **Build Duration:** ~1 day (across multiple sessions)

---

## 📌 Table of Contents

1. [Project Overview](#project-overview)
2. [Final Architecture](#final-architecture)
3. [Tech Stack & Why Each Tool Was Chosen](#tech-stack)
4. [Complete Workflow Diagram](#workflow-diagram)
5. [Step-by-Step Build Log](#build-log)
   - [Step 1: Google Sheets Setup](#step-1-google-sheets-setup)
   - [Step 2: Gemini API Key](#step-2-gemini-api-key)
   - [Step 3: n8n Workflow — Schedule Trigger](#step-3-schedule-trigger)
   - [Step 4: Jina Scrapers (3 Nodes)](#step-4-jina-scrapers)
   - [Step 5: Google OAuth + Sheets Read Nodes](#step-5-google-oauth-and-sheets)
   - [Step 6: Compare Nodes (JavaScript Logic)](#step-6-compare-nodes)
   - [Step 7: Summarize Nodes (AI via OpenRouter)](#step-7-summarize-nodes)
   - [Step 8: Log to Google Sheets](#step-8-log-to-google-sheets)
   - [Step 9: Update Snapshot Nodes](#step-9-update-snapshot)
   - [Step 10: Merge + Slack Notification](#step-10-merge-and-slack)
6. [All JavaScript Code Used](#all-javascript-code)
7. [All Errors Encountered & How They Were Solved](#errors-and-solutions)
8. [Google Sheets Structure](#google-sheets-structure)
9. [Known Limitations](#known-limitations)
10. [How to Demo This in Interviews](#interview-demo-guide)
11. [Resume Description](#resume-description)
12. [Future Improvements](#future-improvements)

---

## Project Overview

### What It Does
An automated system that:
1. Scrapes 3 competitor pricing/product pages **every day at 8am**
2. Compares today's content with yesterday's to detect changes
3. Sends both versions to an **AI model** which summarizes what changed and why it matters
4. Logs everything to **Google Sheets** with timestamps
5. Sends a **Slack notification** when the daily run completes

### Problem It Solves
Sales teams waste hours manually checking competitor websites. This agent automates the entire process and delivers intelligence directly to Slack — every morning, automatically, for free.

### The One-Line Pitch (For Interviews)
> *"I built an AI agent that monitors competitor websites daily, detects pricing changes, summarizes them using an LLM, and delivers intelligence briefings to Slack — fully automated, zero cost to run."*

---

## Final Architecture

```
[Schedule Trigger: 8am Daily]
         ↓
[Scrape 3 Competitor URLs via Jina API] ← parallel, simultaneous
         ↓
[Read Yesterday's Content from Google Sheets]
         ↓
[Compare: Has content changed?] ← JavaScript logic
         ↓
[AI Summarize via OpenRouter (LLM)] ← free tier
         ↓
[Log change + summary to Google Sheets change_log]
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
| **n8n** (localhost:5678) | Automation orchestration | Already installed, visual workflow builder, free self-hosted |
| **Jina Reader API** | Web scraping | Completely free, no signup, converts any URL to clean text, handles JS-rendered pages |
| **OpenRouter** (free tier) | AI summarization | No daily quota exhaustion like Gemini free tier, multiple free models available |
| **Google Sheets** | Data storage | Free, no database setup needed, good enough for portfolio project |
| **Slack Webhook** | Notifications | Easier than Gmail OAuth (2 min setup vs 45 min), more professional for sales tools |
| **Google OAuth** | Google Sheets authentication | Required to connect n8n to Google Sheets |

### Tools Considered But Rejected
- **Gemini API (free tier):** Exhausted daily quota quickly during testing — switched to OpenRouter
- **Gmail:** OAuth setup was complex (30-45 min), Slack webhook was 2 minutes
- **Railway for deployment:** Free tier discontinued in 2023 — kept on localhost for demo
- **Word-based diff (JavaScript):** Too many false positives (e.g. "Buy Now" → "Get Started" flagged as change) — switched to AI-based comparison

---

## Workflow Diagram

```
Schedule Trigger (8am)
│
├──────────────────────────────────────────────────────────┐
│                           │                              │
▼                           ▼                              ▼
Scrape Zapier          Scrape Make               Scrape n8n
(Jina API)             (Jina API)                (Jina API)
│                           │                              │
▼                           ▼                              ▼
Read Yesterday         Read Yesterday            Read Yesterday
Zapier                 Make                      n8n
(Google Sheets)        (Google Sheets)           (Google Sheets)
│                           │                              │
▼                           ▼                              ▼
Compare Zapier         Compare Make              Compare n8n
(JS Code Node)         (JS Code Node)            (JS Code Node)
│                           │                              │
▼                           ▼                              ▼
Summarize Zapier       Summarize Make            Summarize n8n
(OpenRouter API)       (OpenRouter API)          (OpenRouter API)
│                           │                              │
▼                           ▼                              ▼
Log Zapier Change      Log Make Change           Log n8n Change
(Sheets append)        (Sheets append)           (Sheets append)
│                           │                              │
▼                           ▼                              ▼
Update Snapshot        Update Snapshot           Update Snapshot
Zapier                 Make                      n8n
(Sheets update)        (Sheets update)           (Sheets update)
│                           │                              │
└───────────────────────────┼──────────────────────────────┘
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

**What we built:** Two tabs in a single Google Sheet to store data.

**Sheet Name:** `Sales Intelligence Agent`
**Sheet ID:** `1uEGDjJBp5eBDXe2wnXvVCvIPYetE2XdPktCkYCg-Ma8`
**URL:** https://docs.google.com/spreadsheets/d/1uEGDjJBp5eBDXe2wnXvVCvIPYetE2XdPktCkYCg-Ma8

#### Tab 1: `competitor_snapshots`

| Column | Purpose |
|--------|---------|
| `competitor_name` | Identifier (Competitor1, Competitor2, Competitor3) |
| `url` | The pricing page URL being monitored |
| `content` | Full scraped text content — updated every run |
| `last_scraped` | Timestamp of last successful scrape |

**Initial data loaded:**
- Row 2: Competitor1 → https://zapier.com/pricing
- Row 3: ActivePieces → https://r.jina.ai/https://www.activepieces.com/pricing
- Row 4: Competitor3 → https://n8n.io/pricing

#### Tab 2: `change_log`

| Column | Purpose |
|--------|---------|
| `timestamp` | When this entry was logged |
| `competitor_name` | Which competitor |
| `change_detected` | true/false |
| `change_type` | pricing / product / messaging / none |
| `summary` | AI-generated summary |
| `sales_action` | What the sales team should do |

---

### Step 2: Gemini API Key

**What happened:** Created account at `console.anthropic.com` → required credit card purchase → switched to Google Gemini free tier at `aistudio.google.com`.

**Project name created:** `sales-intelligence-agent`

**Key issue later discovered:** Gemini free tier has a daily quota that exhausted quickly during testing. Switched to OpenRouter (see Step 7 for resolution).

---

### Step 3: Schedule Trigger

**Node type:** Schedule Trigger (native n8n node)

**Settings:**
- Trigger Interval: Days
- Days Between Triggers: 1
- Trigger at Hour: 8am
- Trigger at Minute: 0

**What it does:** Fires the entire workflow once every day at 8am. During testing, we used the orange "Execute workflow" button to trigger manually.

---

### Step 4: Jina Scrapers

**Why Jina?** Jina Reader API (`r.jina.ai`) is completely free, requires no API key, handles JavaScript-rendered pages, and returns clean readable markdown text from any URL.

**How it works:** You prepend `https://r.jina.ai/` to any URL. Jina fetches the page, strips HTML, and returns clean text.

**Three HTTP Request nodes created:**

#### Node: `Scrape Zapier`
- Method: GET
- URL: `https://r.jina.ai/https://zapier.com/pricing`
- Authentication: None
- Output field: `data` (contains full page text)

#### Node: `Scrape Make`
- Method: GET
- URL: `https://r.jina.ai/https://make.com/en/pricing`
- **Note:** make.com returns CAPTCHA block content — permanent limitation of their anti-bot protection

#### Node: `Scrape n8n`
- Method: GET
- URL: `https://r.jina.ai/https://n8n.io/pricing`

**Connection:** All 3 scrapers are connected in **parallel** from the Schedule Trigger (not in a chain). This means they run simultaneously.

**Test output from Scrape Zapier (successful):**
```
Title: Plans & Pricing | Zapier
URL Source: https://zapier.com/pricing
Markdown Content:
New: Zaps, Tables, Forms, and Zapier MCP are now all available in one unified plan...
## Free
$0 /mo — Free forever
## Professional
Starting from $19.99 /mo — Billed annually
## Team
Starting from $69 /mo — Billed annually
## Enterprise
Contact for pricing
```

---

### Step 5: Google OAuth and Sheets

#### Google Cloud Setup

**Project used:** "My First Project" (existing project)

**APIs Enabled:**
1. Google Sheets API — `sheets.googleapis.com`
2. Google Drive API — `drive.googleapis.com`

**OAuth Consent Screen:**
- App name: `n8n Intelligence Agent`
- User type: External
- Test user added: `letv1s7456@gmail.com`

**OAuth Client ID created:**
- Application type: Web application
- Name: `n8n-intelligence-agent`
- Authorized redirect URI: `http://localhost:5678/rest/oauth2-credential/callback`

**Credentials saved:**
- Client ID: `672424144660-noadeokrkb4vm3bl0040t1o6kfeqh5f5.apps.googleusercontent.com`
- Client Secret: (saved in Notepad — do not share)

#### n8n Google Sheets Credential
- Type: Google Sheets OAuth2 API
- Connected via "Sign in with Google" button
- Status: ✅ Account connected

#### Three "Read Yesterday" Nodes Created

Each node reads one row from `competitor_snapshots`:

**Node: `Read Yesterday Zapier`**
- Operation: Get Row(s)
- Document: Sales Intelligence Agent
- Sheet: competitor_snapshots
- Filter: `competitor_name` = `Competitor1`

**Node: `Read Yesterday Make`**
- Same settings, Filter: `competitor_name` = `Competitor2`

**Node: `Read Yesterday n8n`**
- Same settings, Filter: `competitor_name` = `Competitor3`

**Test output from Read Yesterday Zapier:**
```json
{
  "row_number": 2,
  "competitor_name": "Competitor1",
  "url": "https://zapier.com/pricing",
  "content": "",
  "last_scraped": ""
}
```
Content was empty on first run — expected behavior (no previous data).

---

### Step 6: Compare Nodes

**Purpose:** Compare today's scraped content with yesterday's stored content. Determine if this is the first run (no previous data) or if content has changed.

**Node type:** Code (JavaScript)

**Three nodes created:** `Compare Zapier`, `Compare Make`, `Compare n8n`

#### Final Working Code — Compare Zapier

```javascript
// Get today's content from Jina scraper
// Note: Jina returns data in the "data" field, not "content"
const todayContent = $('Scrape Zapier').first().json.data || 
                     $('Scrape Zapier').first().json.content || 
                     $('Scrape Zapier').first().json.body || "";

// Get yesterday's content from Google Sheets
const yesterdayContent = $('Read Yesterday Zapier').first().json.content || "";

// First run check — if no previous content stored
const isFirstRun = yesterdayContent === "";

// Change detection — only true if not first run AND content differs
const hasChanged = !isFirstRun && (todayContent !== yesterdayContent);

return [{
  json: {
    competitor: "Zapier",
    url: "https://zapier.com/pricing",
    hasChanged: hasChanged,
    isFirstRun: isFirstRun,
    todayContent: todayContent.substring(0, 2000),
    yesterdayContent: yesterdayContent.substring(0, 2000),
    timestamp: new Date().toISOString()
  }
}];
```

#### Compare Make Code

```javascript
const todayContent = $('Scrape Make').first().json.data || 
                     $('Scrape Make').first().json.content || "";
const yesterdayContent = $('Read Yesterday Make').first().json.content || "";

const isFirstRun = yesterdayContent === "";
const hasChanged = !isFirstRun && (todayContent !== yesterdayContent);

return [{
  json: {
    competitor: "Make",
    url: "https://make.com/en/pricing",
    hasChanged: hasChanged,
    isFirstRun: isFirstRun,
    todayContent: todayContent.substring(0, 2000),
    yesterdayContent: yesterdayContent.substring(0, 2000),
    timestamp: new Date().toISOString()
  }
}];
```

#### Compare n8n Code

```javascript
const todayContent = $('Scrape n8n').first().json.data || 
                     $('Scrape n8n').first().json.content || "";
const yesterdayContent = $('Read Yesterday n8n').first().json.content || "";

const isFirstRun = yesterdayContent === "";
const hasChanged = !isFirstRun && (todayContent !== yesterdayContent);

return [{
  json: {
    competitor: "n8n",
    url: "https://n8n.io/pricing",
    hasChanged: hasChanged,
    isFirstRun: isFirstRun,
    todayContent: todayContent.substring(0, 2000),
    yesterdayContent: yesterdayContent.substring(0, 2000),
    timestamp: new Date().toISOString()
  }
}];
```

**What each field means:**
- `hasChanged`: false = no change since last run, true = pricing page was modified
- `isFirstRun`: true = no previous data in Sheets (expected on Day 1)
- `todayContent`: Truncated to 2000 chars to avoid overwhelming the AI
- `yesterdayContent`: What was stored last time — empty string on first run

**Successful test output:**
```json
{
  "competitor": "Zapier",
  "url": "https://zapier.com/pricing",
  "hasChanged": false,
  "isFirstRun": true,
  "todayContent": "Title: Plans & Pricing | Zapier\n\nURL Source...",
  "yesterdayContent": "",
  "timestamp": "2026-06-11T08:13:49.229Z"
}
```

---

### Step 7: Summarize Nodes

**Purpose:** Send competitor content to an AI model and get a human-readable summary.

**Node type:** HTTP Request (POST)

**Why not a Code node for the API call?** This n8n version (2.21.7) does not support `fetch()` or `$http` in Code nodes. HTTP Request node must be used for external API calls.

**The Two-Step Solution used:**
1. **Code node (Compare/Prep):** Builds a `cleanPrompt` string — a single line with no newlines
2. **HTTP Request node (Summarize):** Sends the `cleanPrompt` to OpenRouter

#### Why cleanPrompt was necessary

The scraped content contains `\n` newline characters. When these get injected directly into a JSON body via `{{ $json.todayContent }}`, the JSON becomes invalid. Solution: build the prompt string in JavaScript first, then inject the pre-built string.

#### Prep Code added to Compare nodes

```javascript
const data = $input.first().json;

// Build a clean single-line prompt — no newlines allowed in JSON strings
const cleanPrompt = "Competitor: " + data.competitor + 
  ". Pricing: " + data.todayContent.substring(0, 500) + 
  ". Summarize pricing tiers for sales team in 80 words.";

return [{json: {
  competitor: data.competitor,
  url: data.url,
  hasChanged: data.hasChanged,
  isFirstRun: data.isFirstRun,
  timestamp: data.timestamp,
  cleanPrompt: cleanPrompt
}}];
```

#### Summarize Nodes — HTTP Request Settings

**Node: `Summarize Zapier`** (same pattern for Make and n8n)

- Method: POST
- URL: `https://openrouter.ai/api/v1/chat/completions`
- Authentication: None
- Send Headers: ON
  - Header Name: `Authorization`
  - Header Value: `Bearer sk-or-v1-YOUR_OPENROUTER_KEY`
- Body Content Type: JSON
- Specify Body: Using JSON

**JSON Body:**
```json
{
  "model": "google/gemma-4-31b-it:free",
  "messages": [{
    "role": "user",
    "content": "{{ 'Summarize this competitor pricing page in 80 words: ' + $json.todayContent.substring(0, 800).replace(/\n/g, ' ').replace(/\"/g, '') }}"
  }]
}
```

**OpenRouter Response Structure:**
```json
{
  "choices": [{
    "message": {
      "content": "Zapier offers four pricing tiers..."
    }
  }]
}
```

To extract the summary in subsequent nodes: `$json.choices[0].message.content`

---

### Step 8: Log to Google Sheets

**Three nodes:** `Log Zapier Change`, `Log Make Change`, `Log n8n Change`

**Node type:** Google Sheets — Append Row

**Settings for each:**
- Operation: Append Row
- Document: Sales Intelligence Agent
- Sheet: change_log
- Values mapped to columns:
  - `timestamp` → `{{ $now.toISO() }}`
  - `competitor_name` → `{{ $json.competitor }}`
  - `change_detected` → `{{ $json.hasChanged }}`
  - `change_type` → (left as static "pricing" for now)
  - `summary` → `{{ $('Summarize Zapier').first().json.choices[0].message.content }}`
  - `sales_action` → (left blank)

---

### Step 9: Update Snapshot

**Three nodes:** `Update Snapshot Zapier`, `Update Snapshot Make`, `Update Snapshot n8n`

**Node type:** Google Sheets — Update Row

**Purpose:** Replace yesterday's content with today's content so tomorrow's run has fresh "yesterday" data to compare against.

**Settings:**
- Operation: Update Row
- Document: Sales Intelligence Agent
- Sheet: competitor_snapshots
- Row to update: matched by `competitor_name`
- Update columns:
  - `content` → today's scraped text
  - `last_scraped` → current timestamp

---

### Step 10: Merge and Slack

#### Merge Node

**Purpose:** The 3 branches run in parallel and finish at different times. The Merge node waits for all 3 to complete before triggering Slack. Without Merge, Slack would fire 3 times (once per branch).

**Settings:**
- Mode: Combine → Append
- Inputs: Update Snapshot Zapier (Input 1), Update Snapshot Make (Input 2), Update Snapshot n8n (Input 3)

#### Send Slack Briefing Node

**Node type:** HTTP Request

**How Slack Webhook works:** Slack gives you a special URL. When you POST JSON to it with a `text` field, the message appears in your chosen Slack channel. No authentication needed beyond the URL itself.

**Setup steps:**
1. Go to `api.slack.com/apps`
2. Create App → From Scratch
3. Enable Incoming Webhooks
4. Add to Workspace → pick channel
5. Copy webhook URL: `https://hooks.slack.com/services/XXX/YYY/ZZZ`

**Node settings:**
- Method: POST
- URL: Your Slack webhook URL
- Body Content Type: JSON
- Specify Body: Using JSON

**JSON Body (final working version — simplified to avoid JSON injection issues):**
```json
{
  "text": "Daily Competitor Intelligence report has been logged. Check the change_log tab in your Google Sheet for today's AI summaries."
}
```

**Why simplified?** Attempting to inject `$json.choices[0].message.content` from the summarize nodes caused JSON to break because the AI output contains quotes and newlines. The important data (AI summaries) is already saved to Google Sheets — Slack is just a trigger notification.

---

## All JavaScript Code Used

### Code Node 1: Compare Zapier (final version)

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

### Code Node 2: Compare Make (final version)

```javascript
const todayContent = $('Scrape Make').first().json.data || 
                     $('Scrape Make').first().json.content || "";
const yesterdayContent = $('Read Yesterday Make').first().json.content || "";

const isFirstRun = yesterdayContent === "";
const hasChanged = !isFirstRun && (todayContent !== yesterdayContent);

const cleanPrompt = "Competitor: Make. Pricing: " + 
  todayContent.substring(0, 500) + 
  ". Summarize pricing tiers for sales team in 80 words.";

return [{
  json: {
    competitor: "Make",
    url: "https://make.com/en/pricing",
    hasChanged: hasChanged,
    isFirstRun: isFirstRun,
    todayContent: todayContent.substring(0, 2000),
    yesterdayContent: yesterdayContent.substring(0, 2000),
    timestamp: new Date().toISOString(),
    cleanPrompt: cleanPrompt
  }
}];
```

### Code Node 3: Compare n8n (final version)

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

### Error 1: Anthropic/Claude API requires paid credits upfront
**Error:** `platform.claude.com/create/credits` — payment screen with no free tier
**Root cause:** Anthropic removed the free $5 credit
**Solution:** Switched to Google Gemini API free tier at `aistudio.google.com`
**Lesson:** Always have a free fallback AI API option identified before starting

---

### Error 2: Gemini API quota exhausted
**Error:**
```
Quota exceeded for metric: generativelanguage.googleapis.com/generate_content_free_tier_requests
limit: 0, model: gemini-2.0-flash
```
**Root cause:** Too many test calls during development burned through daily free quota
**Solution:** Switched to OpenRouter.ai which has more lenient free tier limits
**Lesson:** Free AI APIs have hard daily limits — don't test more than 5-10 times per day

---

### Error 3: JSON Body invalid — bad control character
**Error:**
```
Bad control character in string literal in JSON at position 225
```
**Root cause:** Scraped content contains `\n` newline characters. When injected directly into a JSON string via `{{ $json.todayContent }}`, these break the JSON syntax.
**Solution:** Built a `cleanPrompt` string in the Code node first (JavaScript handles the escaping), then injected the clean string into the JSON body
**Lesson:** Never inject raw scraped content directly into JSON templates. Always sanitize first.

---

### Error 4: fetch is not defined
**Error:**
```
ReferenceError: fetch is not defined [line 8]
```
**Root cause:** n8n version 2.21.7 Code nodes do not support the browser `fetch()` API
**Solution:** Used HTTP Request node instead of Code node for all external API calls
**Lesson:** n8n Code nodes are for data transformation only — use dedicated nodes for external calls

---

### Error 5: $http is not defined
**Error:**
```
ReferenceError: $http is not defined
```
**Root cause:** `$http` was a built-in in older n8n versions but not in 2.21.7
**Solution:** Confirmed HTTP Request node is the correct approach — no workarounds needed
**Lesson:** Always check n8n version compatibility for built-in helpers

---

### Error 6: SyntaxError — Unexpected token 'const'
**Error:**
```
SyntaxError: Unexpected token 'const'
...module.exports = async function VmCodeWrapper() {const const data...
```
**Root cause:** When pasting new code into the Code node, old code was not fully deleted — resulted in `const const data` (duplicate keyword)
**Solution:** Always press `Ctrl+A` to select all code in the editor, then Delete, before pasting new code
**Lesson:** n8n's Code editor doesn't auto-clear when you paste — fully select and delete first

---

### Error 7: OpenRouter free model not found
**Error:**
```
No endpoints found for mistralai/mistral-7b-instruct:free
```
**Root cause:** OpenRouter's free model catalog changes frequently — the suggested model ID was outdated
**Solution:** Tried multiple model IDs until finding one that worked:
- ❌ `mistralai/mistral-7b-instruct:free` — discontinued
- ❌ `meta-llama/llama-3.2-3b-instruct:free` — rate limited
- ❌ `google/gemma-3-4b-it:free` — unavailable for free
- ✅ `google/gemma-4-31b-it:free` — working
**Lesson:** For free models, always verify current availability at `openrouter.ai/models?q=free`

---

### Error 8: OpenRouter 429 rate limit
**Error:**
```
The service is receiving too many requests from you
Provider returned error
```
**Root cause:** Multiple rapid test calls during debugging
**Solution:** Waited 60 minutes for rate limit to reset, then ran full workflow once (not individual nodes)
**Lesson:** Test the whole workflow once rather than individual nodes repeatedly — saves API calls

---

### Error 9: Authorization failed — Missing Authentication header
**Error:**
```
Missing Authentication header
```
**Root cause:** OpenRouter key was entered without the `Bearer ` prefix
**Solution:** Changed header value from `sk-or-v1-...` to `Bearer sk-or-v1-...`
**Lesson:** OpenRouter (and most APIs) require `Bearer ` prefix before the key in the Authorization header

---

### Error 10: Slack node "Node hasn't been executed" cross-branch error
**Error:**
```
ExpressionError: Node 'Summarize Make' hasn't been executed
```
**Root cause:** Slack node was only connected to the Zapier branch. When it tried to reference `$('Summarize Make')`, that branch hadn't run from Slack's perspective
**Solution:** Added a Merge node before Slack that collects output from all 3 branches. Simplified the Slack message to plain text (no cross-branch references)
**Lesson:** In parallel branch workflows, always merge branches before referencing data from multiple branches

---

### Error 11: Parallel nodes connected in chain (architecture bug)
**Issue:** All 3 scraper nodes were connected in a line: Zapier → Make → n8n, meaning they ran sequentially
**Solution:** Deleted the connections between scraper nodes. Connected all 3 directly from Schedule Trigger using parallel connections
**How to connect parallel:** Hover over Schedule Trigger → drag from the dot on its right edge → drop onto each scraper node separately
**Lesson:** In n8n, parallel execution = multiple connections from same source node, not nodes chained together

---

## Google Sheets Structure

### Sheet: competitor_snapshots (current data)

| competitor_name | url | content | last_scraped |
|---|---|---|---|
| Competitor1 | https://zapier.com/pricing | [full Zapier pricing text] | 2026-06-11T... |
| Competitor2 | https://make.com/en/pricing | [CAPTCHA block content] | 2026-06-11T... |
| Competitor3 | https://n8n.io/pricing | [full n8n pricing text] | 2026-06-11T... |

### Sheet: change_log (running log)

Each row = one daily scan per competitor.

| timestamp | competitor_name | change_detected | change_type | summary | sales_action |
|---|---|---|---|---|---|
| 2026-06-11T... | Competitor1 | false | pricing | [AI summary] | |
| 2026-06-11T... | Competitor2 | false | pricing | [AI summary] | |
| 2026-06-11T... | Competitor3 | false | pricing | [AI summary] | |

---

## Known Limitations

### 1. Make.com returns CAPTCHA content
**Issue:** make.com actively blocks scrapers. Their anti-bot protection returns a CAPTCHA page instead of pricing content.
**Impact:** Competitor2 always has unreliable data
**Fix:** Replace make.com with a different competitor that doesn't block scrapers, or use a paid scraping proxy service

### 2. change_detected is always FALSE on Day 1
**Why:** On the first run, `yesterdayContent` is empty. The logic correctly returns `isFirstRun: true` and `hasChanged: false`. This is correct behavior — there's nothing to compare against on Day 1.
**When it works:** Day 2 onwards, the system compares today vs yesterday and correctly detects real changes.

### 3. AI summaries are truncated
**Why:** The `cleanPrompt` only sends first 500-800 characters of the scraped page to avoid rate limits and long prompts
**Impact:** Summaries may miss details from lower sections of pricing pages
**Fix:** Increase substring limit to 1500-2000 once on a paid API tier

### 4. Workflow only runs when laptop is on
**Why:** n8n is running on localhost — if the laptop is off at 8am, the schedule trigger won't fire
**Fix:** Deploy to Oracle Cloud Free Tier VM (always-on, always free) or n8n Cloud ($20/mo)

### 5. OpenRouter free tier rate limits
**Issue:** Free models have per-minute and per-day rate limits
**Fix:** Add $5-10 credit to OpenRouter account — removes all rate limits, very cheap at 3 calls/day

---

## Interview Demo Guide

### What to say when presenting this project

> "I built an AI agent that monitors competitor pricing pages. Every morning it scrapes three competitors, compares the content against yesterday's snapshot, sends any changes to an LLM which summarizes what changed and what it means for our sales team, then logs everything to Google Sheets and notifies Slack. The whole thing runs automatically, costs essentially zero, and took about a day to build."

### Technical questions you should be able to answer

**Q: Why did you use Jina instead of BeautifulSoup or Puppeteer?**
> "For a portfolio project running on a schedule, I needed something zero-cost and zero-maintenance. Jina Reader API converts any URL to clean text with a single GET request — no browser automation, no dependencies, no API key. For production with anti-bot pages like Make.com, I'd upgrade to ScrapingBee or Browserless."

**Q: How does the change detection work?**
> "Simple string comparison — if today's scraped text differs from yesterday's stored text, `hasChanged` returns true. I deliberately avoided word-level diffing because it creates false positives on minor text changes. Instead I send both versions to the LLM and let it decide what's meaningfully different."

**Q: Why OpenRouter instead of OpenAI?**
> "Cost and availability. OpenAI charges from the first call. OpenRouter aggregates multiple providers and has free tiers. For a daily job running 3 calls, I can run for months on the free tier. In production I'd use Claude Haiku or GPT-4o-mini which are both under $1/month at this volume."

**Q: What would you do differently in production?**
> "Three things: First, deploy on a cloud VM so it's not dependent on my laptop. Second, use semantic similarity (embeddings) instead of string comparison for smarter change detection. Third, add a Slack message that includes the actual AI summary inline, not just a notification to check Sheets."

### Things to show in the demo
1. n8n canvas — the full workflow with all 17 nodes
2. Google Sheets — competitor_snapshots tab with real content
3. Google Sheets — change_log tab with timestamps and summaries
4. Slack channel — the notification message
5. Run the workflow live — click Execute Workflow and show nodes turning green

---

## Resume Description

### Option 1 (one line)
Built an AI-powered competitor intelligence agent using n8n, Jina.ai, and OpenRouter LLM that monitors pricing pages daily, detects changes, and delivers Slack briefings automatically.

### Option 2 (three bullets)
- Designed and built an end-to-end automation agent that scrapes 3 competitor websites daily, compares content snapshots, and generates AI summaries using OpenRouter's LLM API
- Architected a parallel-branch n8n workflow with schedule triggers, web scraping (Jina Reader API), JavaScript data transformation, Google Sheets logging, and Slack webhook delivery
- Implemented intelligent change detection logic distinguishing first-run initialization from actual pricing changes, with full error handling and data persistence

### Skills demonstrated by this project
- API integration (OpenRouter, Jina, Slack, Google Sheets)
- JavaScript (n8n Code nodes — data transformation, comparison logic)
- Workflow automation (n8n — parallel branches, merge nodes, schedule triggers)
- Prompt engineering (structuring LLM prompts for structured business output)
- OAuth setup (Google Cloud Console, API enablement, credentials)
- Data architecture (schema design, snapshot vs log pattern)
- Debugging (resolved 11 distinct errors across APIs, code, and workflow wiring)

---

## Future Improvements

| Improvement | Effort | Impact |
|---|---|---|
| Deploy to Oracle Cloud Free VM | High | Workflow runs even when laptop is off |
| Semantic diff using embeddings | Medium | Smarter change detection, fewer false positives |
| Include AI summary in Slack message | Low | More useful notifications |
| Add email delivery (Gmail) | Medium | Stakeholders without Slack can receive briefings |
| Monitor 10+ competitors | Low | Scale by duplicating branch pattern |
| Add Competitor2 fix (replace make.com) | Low | All 3 branches produce reliable data |
| Add retry logic for rate limits | Medium | More resilient in production |
| GitHub README with architecture diagram | Low | Better portfolio presentation |

---

## Credentials & Keys Reference

> ⚠️ Never share these publicly. Store only in a local Notepad file.

| Item | Where to find it |
|---|---|
| Google Sheet ID | URL of the sheet between `/d/` and `/edit` |
| Gemini API Key | aistudio.google.com/apikey |
| OpenRouter API Key | openrouter.ai → Keys section |
| Google OAuth Client ID | Google Cloud Console → APIs & Services → Credentials |
| Google OAuth Client Secret | Same location (only visible at creation time) |
| Slack Webhook URL | api.slack.com/apps → your app → Incoming Webhooks |

---

*Documentation created: June 12, 2026*
*Project built by: MBA Student, Delhi*
*Total build time: ~1 day across multiple sessions*
