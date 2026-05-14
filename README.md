# 🛰️ Daily SEO Discussion Radar

> **A production n8n workflow that collects, enriches, and delivers daily SEO/Content/AI-search discussions from LinkedIn, Reddit, and Stack Exchange — straight to your inbox, Google Sheets, and Notion.**

---

## 🧠 What It Does

This workflow runs **daily** (or on-demand) and:

1. **Scrapes LinkedIn** — profile activity pages, company posts, and direct post URLs via Firecrawl
2. **Falls back to Apify** for LinkedIn posts when Firecrawl hits a login wall
3. **Discovers more LinkedIn content** via SearXNG search engine
4. **Pulls Reddit discussions** via Apify's Reddit scraper actor
5. **Fetches Stack Exchange questions** for implementation-level pain points
6. **Hydrates content** with Firecrawl for items that have thin text
7. **Enriches everything with AI** (OpenRouter) — generates summaries, counterpoints, content hooks, video angles, debate angles, and more
8. **Stores to Google Sheets** — append/update structured rows
9. **Upserts to Notion** — creates or updates database pages with AI-generated insights
10. **Sends a daily email digest** with branch diagnostics and creator recommendations
11. **Creates a Notion founder follow-up task** to review the day's signals

---

## 🏗️ Architecture

```
Schedule Trigger (daily at 8 AM)
        │
Manual Trigger (for testing)
        │
        ▼
┌─────────────────────────────────────────┐
│         Build Source Requests            │  ← Code node: builds API request configs
└─────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────┐
│          Fetch Source Request            │  ← HTTP Request node: executes all API calls
└─────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────┐
│   Normalize, Dedupe, Score, Digest      │  ← Main Code node: the brain of the workflow
│   ┌─────────────────────────────────┐   │
│   │  Normalize raw API responses    │   │
│   │  Deduplicate by URL/key         │   │
│   │  Score by engagement + niche    │   │
│   │  Hydrate thin content (Firecrawl)│   │
│   │  Enrich with OpenRouter AI      │   │
│   │  Build email digest text        │   │
│   └─────────────────────────────────┘   │
└─────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────┐
│  IF - Real Items Collected?         │
│  (yes)                    (no)      │
│    │                       │        │
│    ▼                       ▼        │
│  Notion Upsert          Empty-Run   │
│    │                   Email Digest │
│    ▼                       │        │
│  Google Sheets             │        │
│    │                       │        │
│    ▼                       ▼        │
│  Create Founder          Send Email │
│  Follow-up Task                     │
│    │                                │
│    ▼                                │
│  Prepare Email Digest               │
│    │                                │
│    ▼                                │
│  Send Email ────────────────────────┘
```

---

## 📡 Data Sources

### 1. LinkedIn — Firecrawl Scrape 🟦
- **What**: Scrapes LinkedIn profile activity pages, company post pages, and direct post URLs
- **How**: Uses Firecrawl's `/v2/scrape` endpoint with markdown extraction
- **Smart logic**: Extracts up to 3 recent direct post URLs from activity/profile pages, then scrapes each post individually for richer content
- **Login detection**: Rejects LinkedIn signup/login boilerplate before it reaches storage

### 2. LinkedIn — Apify Fallback 🟦
- **What**: Fallback for when Firecrawl only sees LinkedIn's login wall
- **Actor**: `supreme_coder/linkedin-post`
- **How**: Sends LinkedIn profile/company URLs to Apify's sync dataset API
- **No cookies required**: Works without authenticated LinkedIn cookies

### 3. LinkedIn — SearXNG Discovery 🔍
- **What**: Discovers new LinkedIn posts via self-hosted search
- **How**: Queries SearXNG with `site:linkedin.com` + keyword combinations
- **Resilient**: Empty results or rate-limiting don't break the run — logged as warnings

### 4. Reddit — Apify Scraper 🧡
- **What**: Collects Reddit posts from targeted subreddits
- **Actor**: `spry_wholemeal/reddit-scraper`
- **How**: Searches subreddits by topic keywords, fetches posts with top comments
- **Cost**: ~$0.001 per 50 items (extremely cheap)

### 5. Stack Exchange — Official API 💬
- **What**: Fetches implementation-level questions and pain points
- **How**: Uses the free Stack Exchange API with `intitle` search
- **Why**: Surfaces the "yes, but in reality..." side of SEO discussions

---

## 🤖 AI Enrichment (OpenRouter)

Each collected item gets enriched with:

| Field | Description |
|---|---|
| `summary` | 2-3 sentence synthesis of the actual claim or tension |
| `counterpoint` | Specific challenge or why the claim may be overstated |
| `operator_take` | What the founder/operator should do next |
| `hidden_assumption` | Unproven assumption underlying the claim |
| `proof_gap` | Exact missing evidence |
| `curiosity` | Non-obvious question worth exploring |
| `research_angle` | Teardown, experiment, or essay angle |
| `content_hook` | Specific hook for LinkedIn or X |
| `video_angle` | YouTube angle with audience + demonstration plan |
| `debate_angle` | How to frame the disagreement |
| `linkedin_post_angle` | Best LinkedIn post framing |
| `x_thread_angle` | Best X thread framing |
| `youtube_hook` | Opening hook for the video |
| `creator_notes` | Notes for writing or recording fast |
| `evidence_to_collect` | Proof assets to gather |
| `argument_map` | Claim → objection → proof path |
| `signal_label` | `signal` / `mixed` / `noise` |
| `why_now` | Why this matters right now |
| `interesting_points` | 3 concise bullet-like takeaways |

**Model**: `deepseek/deepseek-v4-pro` (with `openai/gpt-oss-120b` fallback)

---

## 📤 Outputs

### 📊 Google Sheets
Headers: `date`, `run_started_at`, `platform`, `collection_method`, `source_label`, `community`, `author`, `title`, `url`, `canonical_key`, `published_at`, `summary`, `counterpoint`, `operator_take`, `hidden_assumption`, `proof_gap`, `curiosity`, `research_angle`, `content_hook`, `video_angle`, `debate_angle`, `linkedin_post_angle`, `x_thread_angle`, `youtube_hook`, `creator_notes`, `evidence_to_collect`, `argument_map`, `signal_label`, `hashtags`, `raw_text_excerpt`, `score`, `comments_count`

### 📓 Notion Database
Properties: `Name` (Title), `Canonical Key`, `Source URL`, `Collected Date`, `Published At`, `Platform`, `Collection Method`, `Community`, `Author`, `Source Label`, `Summary`, `Counterpoint`, `Operator Take`, `Hidden Assumption`, `Proof Gap`, `Curiosity`, `Research Angle`, `Content Hook`, `Video Angle`, `Debate Angle`, `LinkedIn Post Angle`, `X Thread Angle`, `YouTube Hook`, `Creator Notes`, `Evidence To Collect`, `Argument Map`, `Signal Label`, `Hashtags`, `Raw Text Excerpt`, `Reddit Score`, `Comments Count`, `Run Started At`

> Upserts by **Canonical Key** — no duplicate pages.

### 📧 Email Digest
Daily email with:
- Branch counts (LinkedIn direct, LinkedIn discovery, Reddit, Stack Exchange)
- AI-generated digest with tensions, proof gaps, experiments, and product ideas
- Diagnostics (warnings, errors, empty branches)

### ✅ Notion Founder Follow-up Task
- Automatically created daily task to review insights
- Priority based on number of high-signal items
- Links to the top 5 items

---

## 🔧 Setup

### 1. Import the Workflow
1. Copy `daily-seo-discussion-radar-workflow.json`
2. In n8n: **Workflows → Add Workflow → Import from File**
3. Select the JSON file

### 2. Set Environment Variables
```
OPENROUTER_API_KEY=sk-or-v1-...
APIFY_API_TOKEN=apify_token_...
NOTION_API_KEY=ntn_...
FIRECRAWL_API_KEY=fc-...          (optional if self-hosted without auth)
```

### 3. Create n8n Credentials
- **Google Sheets** — OAuth2 service account
- **SMTP/Email** — for sending the daily digest

### 4. Configure the Workflow
Open **Build Source Requests** (Code node) and update:

```javascript
const config = {
  firecrawl_base_url: 'https://firecrawl.example.com',        // Your Firecrawl instance
  searxng_base_url: 'https://search.example.com',             // Your SearXNG instance
  apify_base_url: 'https://api.apify.com',                    // Apify API
  apify_linkedin_actor_id: 'supreme_coder/linkedin-post',
  apify_reddit_actor_id: 'spry_wholemeal/reddit-scraper',
  linkedin_recent_posts_per_source: 3,
  digest_email: 'you@example.com',                            // Where to send the digest
  digest_from_email: 'radar@example.com',                     // Sender email
  google_sheet_id: 'YOUR_GOOGLE_SHEET_ID',                    // From sheet URL
  google_sheet_tab: 'Daily Radar',                            // Tab name
  notion_database_id: 'YOUR_NOTION_DATABASE_ID',              // Database UUID
  openrouter_model: 'deepseek/deepseek-v4-pro',
  firecrawl_hydration_limit: 4,
  firecrawl_hydration_min_chars: 450
};
```

### 5. Add Your Targets

**LinkedIn URLs** (profile activity pages, company posts, or direct post URLs):
```javascript
const linkedinUrls = [
  'https://www.linkedin.com/in/example-profile/recent-activity/all/',
  'https://www.linkedin.com/company/example-company/posts/',
];
```

**Reddit subreddits and search terms**:
```javascript
const redditSubreddits = ['seo', 'bigseo', 'content_marketing'];
const redditTerms = ['seo', 'ai SEO', 'Topical maps', 'content marketing'];
```

**Stack Exchange terms**:
```javascript
const stackTerms = ['seo', 'ai seo', 'marketing', 'content marketing'];
```

**Discovery keywords** (for SearXNG LinkedIn search):
```javascript
const discoveryKeywords = ['seo', 'ai SEO', 'Semantic SEO', 'content marketing'];
```

### 6. Prepare Google Sheets
Create a sheet with these exact column headers:
```
date,run_started_at,platform,collection_method,source_label,community,author,title,url,canonical_key,published_at,summary,counterpoint,operator_take,hidden_assumption,proof_gap,curiosity,research_angle,content_hook,video_angle,debate_angle,linkedin_post_angle,x_thread_angle,youtube_hook,creator_notes,evidence_to_collect,argument_map,signal_label,hashtags,raw_text_excerpt,score,comments_count
```

### 7. Prepare Notion Database
Create a database with properties as listed in the **Notion Database** section above.

### 8. Test & Schedule
1. Click **Execute Workflow** (Manual Trigger)
2. Verify Google Sheets, Notion, and email all work
3. Switch to **Schedule Trigger** for daily runs at 8 AM

---

## 🎯 Content Series Ideas

| Series | Description |
|---|---|
| 🔥 **"What SEO people are arguing about this week"** | Roundup of tensions across platforms |
| 🛑 **"The strongest counterargument to [popular SEO claim]"** | Pick one claim, dismantle it |
| ⚙️ **"What Reddit operators believe that LinkedIn marketers gloss over"** | Cross-platform tension |
| 🔧 **"What implementation pain reveals about bad SEO advice"** | Stack Exchange as signal |
| 🧪 **"One curiosity worth testing this week"** | Experiment-driven content |
| ✍️ **"A better framing for [trending discussion]"** | Content reframing |

---

## 🛡️ Resilience Features

- **Login-wall detection**: LinkedIn signup pages are rejected before storage
- **Empty-run handling**: If no items collected, sends a diagnostic email instead of silently failing
- **Graceful degradation**: Each source branch is independent — one failure doesn't block others
- **Fallback models**: OpenRouter primary + fallback model chain
- **Retry logic**: Email node retries once on failure
- **Shape mismatch detection**: Self-heals common config errors (e.g., subreddit fields with search phrases)
- **Firecrawl hydration limit**: Cap on API calls per run to control costs

---

## 🧪 Free Source Extensions

Want more data? Add these later:
- [Hacker News API](https://github.com/HackerNews/API)
- [YouTube Data API](https://developers.google.com/youtube/v3/getting-started)
- [OpenAlex](https://docs.openalex.org/)
- [Wikimedia APIs](https://api.wikimedia.org/wiki/Main_Page)

---

## 📚 References

- [n8n Workflow SDK](https://docs.n8n.io/workflows/)
- [Firecrawl Scrape API](https://docs.firecrawl.dev/api-reference/endpoint/scrape)
- [SearXNG Search API](https://docs.searxng.org/dev/search_api)
- [Apify Actor Runs API](https://docs.apify.com/api/v2/actors-actor-runs)
- [LinkedIn Post Scraper (Apify)](https://apify.com/supreme_coder/linkedin-post)
- [Reddit Scraper (Apify)](https://apify.com/spry_wholemeal/reddit-scraper)
- [OpenRouter API](https://openrouter.ai/docs/guides/best-practices/reasoning-tokens)
- [Stack Exchange API](https://api.stackexchange.com/docs/search)
- [Notion API](https://developers.notion.com/)

---

## 📄 License

MIT — use it, fork it, ship it.
