# WAR ROOM — Execution Plan

> One-Click Digest Engine: From Brief to Production
> Last Updated: March 10, 2026

---

## Original Requirements Brief

> **WAR ROOM | Feature Development Brief**
> Confidential — Internal Use Only | March 2026 | Version 1.0

### 1. Background & Problem Statement

The War Room platform currently requires multiple clicks and manual steps for a user to generate an intelligence digest from the live news feeds. This creates friction and slows down the workflow — especially during fast-moving political situations when speed matters most.

Inspiration is drawn from the **No News app**, which aggregates multiple RSS feeds and delivers a clean, ready-to-read digest in just a few clicks. The goal is to replicate that simplicity inside War Room, tailored specifically to Guyana's five political intelligence pillars.

### 2. The Five Intelligence Pillars

War Room tracks five core topic areas, referred to as the **Five Pillars**. Each pillar will have its own dedicated digest:

| Pillar | Description |
|--------|-------------|
| **Pillar 1 — President Ali** | News, statements, and coverage of the President of Guyana |
| **Pillar 2 — VP Jagdeo** | News, statements, and coverage of the Vice President |
| **Pillar 3 — Azruddin Mohamed** | Entity-filtered feed tracking this specific political figure |
| **Pillar 4 — Opposition** | Aggregated coverage of all opposition channels and figures |
| **Pillar 5 — Live Guyana News + International** | Domestic breaking news and foreign coverage of Guyana |

### 3. Feature Request: One-Click Digest Engine

**3.1 What It Should Do**

For each of the Five Pillars, the platform should automatically:
- Scrape the relevant RSS feeds and news sources assigned to that pillar
- Filter articles by the pillar topic (entity, keyword, or source group)
- Summarise and compile the top stories into a clean digest
- Format and export the digest as a downloadable document (Word / PDF)
- Make the digest available in as few clicks as possible — ideally one button per pillar

**3.2 Current Problem (Before)**

Currently, to get a digest the user must:
1. Navigate to the correct feed tab
2. Wait for articles to load and scrape
3. Manually select articles
4. Click to generate the brief
5. Export or copy the output

This is too many steps. It creates delays and increases the chance of missing important stories.

**3.3 Desired Experience (After)**

1. User opens War Room dashboard
2. User sees five Digest buttons — one per pillar
3. User clicks one button (e.g. "Ali Digest" or "Opposition Digest")
4. Platform auto-fetches, filters, and summarises the latest stories for that pillar
5. A formatted digest document is generated and available to download or email

### 4. Digest Document Format

Each digest should follow this standard structure:

| Section | Content |
|---------|---------|
| **Header** | Pillar name, date, time of generation |
| **Top Stories** | 5–10 most recent and relevant articles with headline, source, time, and 2–3 sentence AI summary |
| **Key Quotes** | Any notable quotes extracted from the articles |
| **Sentiment Note** | Brief AI-generated note on overall tone (positive / neutral / negative coverage) |
| **Export Options** | Download as Word (.docx), PDF, or send via email |

### 5. Technical Implementation Notes

- Each pillar should map to a pre-configured set of RSS sources and keyword/entity filters already defined in the Admin Settings
- The digest generation should call the existing AI summarisation logic (used in Talking Points / Narratives) but scoped to the filtered article set
- Digest generation should run server-side and return a pre-formatted document — not require the user to wait for real-time scraping
- Consider a background refresh every 1–6 hours so digests are near-instant when the user clicks
- The five digest buttons should be prominently placed on the main dashboard — not buried in a sub-menu

### 6. Priority & Impact

| Feature | Priority | Impact |
|---------|----------|--------|
| One-click digest per pillar | **HIGH** | Saves 5+ minutes per digest session |
| Auto-refresh digest cache | **HIGH** | Near-instant generation on click |
| Export to Word/PDF | **HIGH** | Ready to share immediately |
| Email digest direct from platform | **MEDIUM** | Reduces copy/paste workflow |
| Sentiment indicator per digest | **MEDIUM** | Quick read on media tone |

### 7. Summary

The core ask is simple: **make it as easy to get an intelligence digest from War Room as it is from the No News app.** One button per pillar. Automatic. Fast. Ready to share.

This single improvement will dramatically increase the daily usability of the platform and make it the go-to tool for Guyana political intelligence — whether for morning briefings, rapid response, or end-of-day reporting.

---

## What We're Building

A system where a War Room user clicks **one button** for any of the 5 intelligence pillars and instantly receives a formatted intelligence digest — ready to read, download, or email.

---

## What We Already Have (The NoNews Backend)

Our existing backend already does 70-75% of what War Room needs. Here's the plain-English breakdown:

### Already Built & Working

| Capability | What It Does | War Room Usage |
|-----------|-------------|----------------|
| **RSS Feed Ingestion** | Automatically scrapes 12+ news sources every few minutes, extracts article text, stores it | Same sources War Room needs — Stabrook News, Kaieteur News, Guyana Chronicle, etc. |
| **Article Deduplication** | Detects and removes duplicate articles across sources | Prevents the same story appearing twice in a digest |
| **Story Clustering** | Groups related articles into "stories" using AI similarity matching | A story like "Ali announces oil plan" may have 5 articles from different sources — clustering combines them into one story |
| **Entity Detection** | Identifies people, organizations, and places mentioned in each story | Detects "Irfaan Ali", "APNU+AFC", "Georgetown" etc. — this is how we filter stories per pillar |
| **AI Summarization** | Generates concise summaries, detailed reports, and debate-style analysis | Reused for digest story summaries |
| **Quote Extraction** | Pulls direct quotes with speaker attribution from articles | Reused for "Key Quotes" section in each digest |
| **Semantic Search** | Find stories by meaning, not just keywords | Powers the search bar and "related stories" |
| **User Authentication** | JWT-based login system via Supabase | Same auth for War Room users |
| **Rate Limiting & Security** | Input validation, XSS protection, rate limits per user | Protects War Room endpoints |

### NOT Built Yet (What We Need to Add)

| Capability | Why It's Needed | Effort |
|-----------|----------------|--------|
| **Pillar Configuration** | Define which entities + sources belong to each of the 5 pillars | Small |
| **Digest Generation Engine** | The "one click" logic — fetch stories for a pillar, compile the digest | Medium |
| **Document Export** | Generate downloadable Word (.docx) and PDF files | Medium |
| **Background Auto-Refresh** | Pre-generate digests every few hours so button click is instant | Small |
| **Sentiment Analysis** | Classify each story and overall digest tone (positive/neutral/negative) | Small |
| **Email Delivery** | Send digest as email with PDF/Word attachment | Medium |
| **Source-Based Filtering** | Filter stories by which news source published them (needed for Pillar 5) | Small |

---

## How the Existing Pipeline Feeds War Room

```
        WHAT WE HAVE                              WHAT WE'RE ADDING
        (NoNews Backend)                          (War Room Layer)

   ┌─────────────────────┐
   │  12 Guyana RSS Feeds │
   │  (Stabrook, Kaieteur,│
   │   Chronicle, etc.)   │
   └──────────┬──────────┘
              ↓
   ┌─────────────────────┐
   │  Article Processor   │  Runs automatically
   │  Scrape → Extract    │  every few minutes
   │  → Store in database │
   └──────────┬──────────┘
              ↓
   ┌─────────────────────┐
   │  Story Clustering    │  Groups articles
   │  AI Similarity +     │  into stories
   │  Entity Matching     │  (e.g., 5 articles
   └──────────┬──────────┘   = 1 story)
              ↓
   ┌─────────────────────┐
   │  Stories Database    │
   │  With entities,      │      ┌──────────────────────────┐
   │  summaries,          │ ───→ │  NEW: Pillar Filter       │
   │  source info         │      │  "Give me all stories     │
   └──────────────────────┘      │   mentioning Irfaan Ali"  │
                                 └────────────┬─────────────┘
                                              ↓
                                 ┌──────────────────────────┐
                                 │  NEW: Digest Compiler     │
                                 │  Summaries + Quotes +     │
                                 │  Sentiment + Formatting   │
                                 └────────────┬─────────────┘
                                              ↓
                                 ┌──────────────────────────┐
                                 │  NEW: Export Engine       │
                                 │  → Word document          │
                                 │  → PDF document           │
                                 │  → Email delivery         │
                                 └──────────────────────────┘
```

---

## The 5 Pillars — How Each One Works

Every pillar is essentially a **pre-set filter** on our story database:

### Pillar 1: President Ali
- **Filter**: Show stories where entities include "Irfaan Ali", "President Ali", or "Dr. Ali"
- **Sources**: All — any source that mentions him
- **Expected volume**: 5-15 stories per day

### Pillar 2: VP Jagdeo
- **Filter**: Show stories where entities include "Bharrat Jagdeo", "VP Jagdeo"
- **Sources**: All
- **Expected volume**: 3-10 stories per day

### Pillar 3: Azruddin Mohamed
- **Filter**: Show stories where entities include "Azruddin Mohamed"
- **Sources**: All
- **Expected volume**: 1-5 stories per day (lower volume)

### Pillar 4: Opposition
- **Filter**: Show stories mentioning "Aubrey Norton", "APNU+AFC", "APNU", "AFC", "PNC"
- **Sources**: All
- **Expected volume**: 5-15 stories per day

### Pillar 5: Live Guyana + International
- **Filter**: NOT entity-based — instead filters by **source** (all domestic + international outlets)
- **Sources**: All 12+ configured sources
- **Expected volume**: 20-40 stories per day (highest volume)

---

## What a Digest Looks Like

When a user clicks "Ali Digest", they get:

```
╔══════════════════════════════════════════════╗
║  WAR ROOM — PRESIDENT ALI DIGEST            ║
║  March 10, 2026 | 1:15 PM                   ║
╠══════════════════════════════════════════════╣
║                                              ║
║  EXECUTIVE SUMMARY                           ║
║  Today's coverage of President Ali focuses   ║
║  on the oil revenue framework announcement   ║
║  and regional infrastructure plans...        ║
║                                              ║
╠══════════════════════════════════════════════╣
║                                              ║
║  TOP STORIES (8 stories)                     ║
║                                              ║
║  1. President Ali announces new oil          ║
║     revenue plan                             ║
║     📰 Stabrook News | ⏰ 2h ago | 5 articles║
║     Summary: President Irfaan Ali unveiled   ║
║     a new framework for distributing oil...  ║
║                                              ║
║  2. Infrastructure spending targets          ║
║     announced for Regions 3 and 4            ║
║     📰 Kaieteur News | ⏰ 4h ago | 3 articles║
║     Summary: The government outlined new...  ║
║                                              ║
║  [... 6 more stories ...]                    ║
║                                              ║
╠══════════════════════════════════════════════╣
║                                              ║
║  KEY QUOTES                                  ║
║                                              ║
║  "This is a historic moment for our nation   ║
║   and its people."                           ║
║     — President Irfaan Ali (Stabrook News)   ║
║                                              ║
║  "The revenue sharing model ensures every    ║
║   region benefits equally."                  ║
║     — Finance Minister (Guyana Chronicle)    ║
║                                              ║
╠══════════════════════════════════════════════╣
║                                              ║
║  SENTIMENT NOTE                              ║
║  Overall: POSITIVE                           ║
║  Coverage is predominantly positive, focused ║
║  on economic development and infrastructure. ║
║  Some neutral coverage from opposition-      ║
║  leaning sources provides balance.           ║
║                                              ║
╠══════════════════════════════════════════════╣
║  📥 Download: [Word] [PDF] | 📧 [Email]     ║
╚══════════════════════════════════════════════╝
```

---

## Delivery Plan

### 🟢 Phase 1 — Foundation (Days 1-3)
**Goal**: Database ready, sources connected, pillars configured.

| Task | What Happens | Who Validates |
|------|-------------|---------------|
| Create pillar configuration database table | 5 pillars stored with their entity filters and settings | Engineer runs query, confirms 5 pillars exist |
| Create digest storage table | Place to store generated digests | Engineer confirms table created |
| Add 12 Guyana RSS feed sources | Stabrook News, Kaieteur News, etc. added to system | PM verifies source names match War Room UI |
| Generate AI embeddings for each pillar | System learns what each pillar "means" semantically | Engineer confirms all 5 have embeddings |
| Create story matching function | SQL function that finds stories per pillar | Engineer tests: "ali-digest" returns Ali-related stories |

**Milestone**: ✅ System can find and return stories for each pillar

---

### 🟡 Phase 2 — Core Digest Engine (Days 4-7)
**Goal**: Can generate a complete digest for any pillar.

| Task | What Happens | Who Validates |
|------|-------------|---------------|
| Build digest generation service | Core logic: fetch stories → summarize → extract quotes → analyze sentiment → compile | Engineer tests all 5 pillars |
| Add sentiment analysis AI prompt | New AI prompt classifies tone as positive/neutral/negative | PM reviews sentiment accuracy on 10 sample stories |
| Add digest introduction AI prompt | AI writes 2-3 sentence executive summary per digest | PM reviews quality of introductions |
| Integrate with existing AI content | Reuse pre-computed summaries and quotes (already in database) | Engineer confirms no duplicate AI calls |

**Milestone**: ✅ Running a function produces a complete digest JSON with stories, quotes, sentiment

**Performance target**: Each digest generated in under 15 seconds

---

### 🔵 Phase 3 — API & Frontend Integration (Days 8-10)
**Goal**: Frontend can call backend and display digests.

| Task | What Happens | Who Validates |
|------|-------------|---------------|
| Build "List Pillars" endpoint | Frontend calls → gets list of 5 pillars with status | Frontend engineer confirms data renders |
| Build "Generate Digest" endpoint | **THE ONE-CLICK BUTTON** — frontend calls → gets full digest | PM clicks button in War Room UI, sees digest |
| Build "Get Latest Digest" endpoint | Returns pre-generated digest instantly (no wait) | PM confirms instant response |
| Build "Digest History" endpoint | View past digests by date | PM can browse previous days |
| Add War Room domain to allowed origins | CORS security — allows War Room frontend to talk to backend | Frontend engineer confirms no CORS errors |

**Milestone**: ✅ PM can open War Room, click a pillar button, and see a digest

---

### 🟣 Phase 4 — Document Export (Days 11-13)
**Goal**: Digests downloadable as Word and PDF.

| Task | What Happens | Who Validates |
|------|-------------|---------------|
| Build Word (.docx) generator | Converts digest into formatted Word document | PM downloads and opens in Word/Google Docs |
| Build PDF generator | Converts digest into formatted PDF | PM downloads and opens in browser |
| Build CSV export | Tabular export of story data | PM downloads and opens in Excel |
| Upload documents to cloud storage | Files stored securely with 24-hour download links | Engineer confirms links expire correctly |
| Build "Export" API endpoint | Frontend calls → gets download link | PM clicks export button in UI, download starts |

**Milestone**: ✅ PM can download a professional-looking digest document

**Document format matches**: Header → Executive Summary → Top Stories → Key Quotes → Sentiment Note → Footer

---

### 🟠 Phase 5 — Auto-Refresh (Days 14-15)
**Goal**: Digests are always fresh — no waiting when user clicks.

| Task | What Happens | Who Validates |
|------|-------------|---------------|
| Set up scheduled job (every 3 hours) | System automatically regenerates all 5 pillar digests | Engineer confirms job runs in cloud console |
| Wire refresh endpoint | Scheduled job calls backend → all pillars refresh | Engineer checks logs for successful refreshes |
| Verify instant response | When user clicks button, they get pre-generated digest (< 1 second) | PM clicks button → digest appears instantly |

**Milestone**: ✅ Digests are always pre-generated and ready — one-click feels instant

**Schedule**: Every 3 hours (configurable to 1-6 hours based on need)

---

### 🔴 Phase 6 — Email & Polish (Days 16-22)
**Goal**: Full feature set — email delivery, story filtering, refinements.

| Task | What Happens | Who Validates |
|------|-------------|---------------|
| Set up email service (SendGrid) | Backend can send emails with attachments | Engineer sends test email |
| Build "Email Digest" endpoint | User enters email → digest sent as PDF/Word attachment | PM receives digest email |
| Build story filtering API | Powers the source/entity/tone filter dropdowns in War Room UI | PM tests each filter dropdown |
| Refine sentiment accuracy | Iterate on AI prompts based on PM feedback | PM reviews 20+ sentiment classifications |
| Polish document templates | Refine Word/PDF formatting based on PM/client feedback | PM approves final document design |
| End-to-end testing | All flows tested together | PM runs full walkthrough |

**Milestone**: ✅ All features from the brief are working

---

## Timeline Visual

```
Week 1                    Week 2                    Week 3
─────────────────────     ─────────────────────     ─────────────
│ Phase 1 │ Phase 2 │     │ Phase 3 │ Phase 4 │     │Ph 5│ Ph 6 │
│ DB +    │ Digest  │     │ API +   │ Export  │     │Auto│ Email│
│ Sources │ Engine  │     │ Frontend│ Docs    │     │    │Polish│
│         │         │     │         │         │     │    │      │
│ Day 1-3 │ Day 4-7 │     │ Day 8-10│Day 11-13│     │14  │16-22 │
─────────────────────     ─────────────────────     ─────────────
         ↑                         ↑                       ↑
    First stories            PM sees digest          Full launch
    matched to pillars       in War Room UI          ready
```

---

## What the User Experience Looks Like

### Before (Current — from the Brief)
```
Step 1: Navigate to correct feed tab
Step 2: Wait for articles to load
Step 3: Manually select articles
Step 4: Click to generate brief
Step 5: Export or copy output
→ 5+ minutes per digest, risk of missing stories
```

### After (With Digest Engine)
```
Step 1: Click "Ali Digest" button
→ Digest appears instantly (pre-generated)
→ Click "Download PDF" or "Email"
→ Done in under 10 seconds
```

---

## Key Decisions for PM to Confirm

### 1. Pillar Names & Structure

Are these correct? Any changes needed?

| # | Current Name | Sidebar Label (from UI) | Separate or Combined? |
|---|-------------|------------------------|----------------------|
| 1 | President Ali | Ali & Jagdeo | ⚠️ UI shows combined — should digest be combined or separate? |
| 2 | VP Jagdeo | Ali & Jagdeo | ⚠️ Same question |
| 3 | Azruddin Mohamed | Azruddin Mohamed | ✅ Matches |
| 4 | Opposition | Opposition: All | ✅ Matches |
| 5 | Live Guyana + International | Live: Guyana + Foreign: Guyana | ⚠️ UI shows as 2 items — one digest or two? |

**Recommendation**: Generate separately (5 individual digests), let frontend combine display if needed. More flexible.

### 2. Refresh Frequency
- Brief says "1-6 hours"
- **Recommendation**: Start with every 3 hours, adjust based on how fast Guyana news moves
- Can be changed without code changes (just update the schedule)

### 3. Max Stories Per Digest
- Brief says "5-10 most recent and relevant"
- **Recommendation**: Default 10, configurable per pillar
- Azruddin (lower volume) might only have 2-3 some days — that's OK

### 4. Email Provider
- **Option A**: SendGrid — easier setup, good deliverability
- **Option B**: Amazon SES — cheaper at scale
- **Recommendation**: SendGrid (faster to integrate, can switch later)

### 5. Social Listening & Civic Portal
- These are visible in the War Room sidebar but are **out of scope** for this plan
- Social Listening = Facebook monitoring (needs Facebook API — separate project)
- Civic Portal = separate platform tab entirely

---

## Success Metrics

| Metric | Target | How We Measure |
|--------|--------|----------------|
| Time to get a digest | < 10 seconds (click to read) | Pre-generated digests served in < 1 second |
| Digest relevance | 90%+ of stories are on-topic for the pillar | PM spot-checks 5 digests daily for first week |
| Story coverage | No major story missed | Compare digest against manual news scan |
| Sentiment accuracy | 80%+ correct classifications | PM reviews sample of 20 stories |
| Document quality | Professional, ready to share | PM approves template before launch |
| Uptime | 99%+ | Cloud monitoring alerts |
| Export success rate | 99%+ | Error logs |

---

## Risks & Mitigations

| Risk | Likelihood | Impact | What We Do |
|------|-----------|--------|-----------|
| RSS feed URL changes or breaks | Medium | No new articles from that source | Monitoring alerts when a source hasn't published in 6+ hours |
| AI gives wrong sentiment | Medium | Misleading digest tone | Human review during Phase 6; iterate on AI prompts |
| Entity name spelled differently in articles | Medium | Stories missed for a pillar | Include multiple aliases (e.g., "Irfaan Ali" + "President Ali" + "Dr. Ali") |
| Digest generation takes too long | Low | User waits instead of instant click | Background pre-generation every 3 hours eliminates wait |
| Low story volume for some pillars | Medium | Azruddin digest might be empty some days | Show "No stories in the last 24 hours" message — not an error |
| Document formatting issues | Low | PDF/Word looks bad | Test with real Guyanese content (proper nouns, special chars) early |

---

## Team Requirements

| Role | Needed For | Phases |
|------|-----------|--------|
| **Backend Engineer** (Senior) | Database, digest engine, API endpoints, export, email | All phases |
| **Frontend Engineer** | Connect War Room UI to new API endpoints | Phase 3 onward |
| **PM** | Validate pillar configs, review digest quality, approve document templates | Phase 1 decisions + Phase 3-6 validation |
| **DevOps** (part-time) | Cloud Scheduler setup, GCS bucket, environment variables | Phase 1 + Phase 5 |

---

## Budget Considerations

| Item | Cost | Notes |
|------|------|-------|
| AI API calls (OpenRouter/Gemini) | ~$5-15/day | 5 pillars × 3-5 LLM calls per refresh × 8 refreshes/day |
| Cloud Functions compute | Existing budget | No new functions needed — extends existing API service |
| Cloud Storage (exports) | < $1/month | Documents auto-deleted after 30 days |
| SendGrid (email) | Free tier: 100 emails/day | Upgrade to $20/month for higher volume |
| Cloud Scheduler | Free | 3 jobs/month included free |

**Total estimated additional cost: $150-500/month** depending on usage volume.

---

## Appendix: Detailed Technical Docs

For the implementing engineer, these companion documents in `/warroom/` contain code-level details:

| Doc | What's Inside |
|-----|--------------|
| `EXECUTION_PLAN.md` | Line-by-line engineering execution plan with file paths and code patterns |
| `01_BACKEND_REUSE_MAP.md` | Component-by-component reuse analysis |
| `02_PILLAR_NEWSLETTER_MAPPING.md` | How pillars map to the newsletter system |
| `03_NEW_TABLES_AND_MIGRATIONS.md` | Full SQL for new database tables |
| `04_API_ENDPOINTS.md` | API endpoint specs with request/response schemas |
| `05_DIGEST_GENERATION_ENGINE.md` | Digest engine architecture and AI prompts |
| `06_DOCUMENT_EXPORT.md` | Word/PDF/email implementation details |
| `07_IMPLEMENTATION_PHASES.md` | Sprint breakdown with file change lists |
| `08_SOURCES_AND_ENTITIES.md` | Source URLs and entity alias mapping |
