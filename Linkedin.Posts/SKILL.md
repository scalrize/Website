# LinkedIn Automation — Claude Code Skill
**Project:** Automated LinkedIn post generation for Matthieu Jammers
**Repo:** https://github.com/scalrize/linkedin-automation (PRIVATE)
**Local folder:** `~/Desktop/linkedin-automation/`

---

## WHAT THIS SKILL DOES

When invoked, this skill helps maintain, debug, and improve a fully automated LinkedIn content system that:
- Runs every Monday at 00:00 UTC (08:00 AM Bali time) via GitHub Actions
- Scrapes Bali real estate news + Matthieu's LinkedIn profile via Firecrawl
- Generates 4 LinkedIn post options via Google Gemini AI
- Emails matthieu.jammers@gmail.com with 2 options for Tuesday + 2 for Thursday
- Matthieu manually posts his preferred option at 11:00 AM Bali time

---

## SYSTEM ARCHITECTURE

```
GitHub Actions (cron: every Monday 00:00 UTC)
    │
    ▼
scraper.py          → Firecrawl scrapes LinkedIn profile + industry news
    │                  Falls back to verified news sources if LinkedIn blocks
    ▼
generate_post.py    → Gemini AI generates 4 posts (2 Tuesday + 2 Thursday)
    │                  Rotation based on ISO week number (no file storage needed)
    │                  Validates: hook <200 chars, total 1200-1600 chars, no banned phrases
    ▼
send_email.py       → Gmail API sends ONE formatted email
    │                  Subject: "Your LinkedIn Posts for the Week — [date]"
    ▼
matthieu.jammers@gmail.com receives the email
```

---

## FILE STRUCTURE

```
linkedin-automation/
├── .github/workflows/linkedin_poster.yml  ← GitHub Actions (cron + manual trigger)
├── generate_post.py                        ← Gemini post generator
├── send_email.py                           ← Gmail API sender
├── scraper.py                              ← Firecrawl scraper with fallback
├── main.py                                 ← Pipeline orchestrator
├── authorize_gmail.py                      ← One-time local OAuth setup (do not deploy)
├── linkedin_prompt.md                      ← Full AI content brief (voice, themes, rules)
├── post_log.txt                            ← Legacy log (no longer used for rotation)
└── requirements.txt                        ← Python dependencies
```

---

## GITHUB SECRETS (already configured)

| Secret | Description |
|---|---|
| `GEMINI_API_KEY` | Google Gemini API key — account: hello@scalrize.com |
| `GMAIL_TOKEN` | Full JSON of OAuth2 token.json — account: matthieu.jammers@gmail.com |
| `FIRECRAWL_API_KEY` | Firecrawl API key for web scraping |
| `GITHUB_TOKEN` | Auto-provided by GitHub Actions |

**To refresh GMAIL_TOKEN:** run `python3 authorize_gmail.py` locally, copy token.json content to the secret.

**Gemini quota:** Free tier has daily limits. If 429 error appears, either wait until next day or enable billing at console.cloud.google.com for the hello@scalrize.com project.

---

## CONTENT SYSTEM

### 4 Themes (rotate by ISO week number)
1. Sales Techniques in Real Estate
2. Bali, Lombok & Surrounding Islands Real Estate Market
3. Legal Framework for Foreign Buyers in Indonesia
4. Construction Due Diligence in Bali

### 4 Content Pillars (rotate by ISO week number)
1. The Educator — practical how-to, step-by-step guide
2. The Perspective — contrarian take, challenges assumptions
3. The Human — personal story, behind-the-scenes, client case
4. The Proof — data-backed insight, case study, specific results

### Post Rules (from linkedin_prompt.md)
- Hook: under 200 characters, no emojis on line 1
- Total: 1,200 to 1,600 characters
- No banned phrases: "Exciting news", "Thrilled to announce", "Game-changer", "Synergy", "Leverage", "Innovative", "Passionate", "Thought leader", "Best practices", "In today's fast-paced world", "I am pleased to share"
- End with low-friction question CTA (never "Like and share!")
- Max 5 hashtags at the bottom
- No external links in post body
- Active voice throughout
- Legal posts must include notary disclaimer
- Construction posts must close with professional agent recommendation

---

## ROTATION LOGIC

Rotation is calculated from the ISO week number — **no persistent state needed**.

```python
week_num = datetime.utcnow().isocalendar()[1]  # 1-53

tuesday_theme  = THEMES[week_num % 4]
thursday_theme = THEMES[(week_num + 1) % 4]

tuesday_pillar1  = PILLARS[(week_num * 2) % 4]
tuesday_pillar2  = PILLARS[(week_num * 2 + 1) % 4]
thursday_pillar1 = PILLARS[(week_num * 2 + 2) % 4]
thursday_pillar2 = PILLARS[(week_num * 2 + 3) % 4]
```

---

## GEMINI MODEL FALLBACK CHAIN

```python
for model_name in ["gemini-2.0-flash", "gemini-1.5-flash", "gemini-1.5-pro", "gemini-pro"]:
    try:
        candidate = genai.GenerativeModel(model_name)
        candidate.generate_content("hello")
        model = candidate
        break
    except Exception as e:
        continue
```

---

## OUTPUT FORMAT — GEMINI MARKERS

Gemini is instructed to output content between strict markers:

```
RESEARCH_SUMMARY_START ... RESEARCH_SUMMARY_END
PROFILE_CHECK_START    ... PROFILE_CHECK_END
SOURCES_START          ... SOURCES_END
OPTION1_POST_START     ... OPTION1_POST_END
OPTION1_HOOK_START     ... OPTION1_HOOK_END
OPTION1_WHY_START      ... OPTION1_WHY_END
OPTION1_VISUAL_START   ... OPTION1_VISUAL_END
OPTION2_POST_START     ... OPTION2_POST_END
OPTION2_HOOK_START     ... OPTION2_HOOK_END
OPTION2_WHY_START      ... OPTION2_WHY_END
OPTION2_VISUAL_START   ... OPTION2_VISUAL_END
RECOMMENDATION_START   ... RECOMMENDATION_END
```

Parser strips markdown code blocks and does case-insensitive fallback search.

---

## EMAIL FORMAT

**Subject:** `Your LinkedIn Posts for the Week — 9 March 2026`
**To/From:** matthieu.jammers@gmail.com

```
Hi Matthieu,

Here are your four LinkedIn post options for this week.

Tuesday post   → publish at 11:00 AM Bali time
Thursday post  → publish at 11:00 AM Bali time

════ RESEARCH SUMMARY ════
[trending topics + profile check + sources]

════ TUESDAY [date] — Theme: [theme] ════
── OPTION 1 — [pillar] ──
Why this works: [one sentence]
[full post]
Character count: X | Read time: X seconds
Suggested visual: [description]

── OPTION 2 — [pillar] ──
[same structure]

★ MY RECOMMENDATION FOR TUESDAY: Option X — [reason]

════ THURSDAY [date] — Theme: [theme] ════
[same structure]

════ NEXT DELIVERY ════
Next email: Monday [date] at 08:00 AM Bali time
Tuesday theme: [theme]
Thursday theme: [theme]
```

---

## KNOWN ISSUES & FIXES APPLIED

| Issue | Fix |
|---|---|
| git push post_log.txt conflict (recurring) | Removed git commit step entirely. Rotation uses ISO week number — no state storage needed. |
| Gemini output markers not parsed | `_extract()` strips backticks and uses case-insensitive fallback |
| Gemini model 404 errors | Fallback chain tries 4 models in sequence |
| Gemini 429 quota exceeded | Caused by too many test runs. Resets daily. Enable billing for production use. |
| Firecrawl AttributeError | `safe_scrape()` tries both `scrape_url` and `scrape` API methods |
| GMAIL_TOKEN JSON parse error | Must copy token.json via TextEdit (Cmd+A), not terminal output |
| Google OAuth "Access blocked" | Add matthieu.jammers@gmail.com as test user in OAuth consent screen |

---

## HOW TO DEPLOY A FIX

1. Edit the relevant file in `~/Desktop/linkedin-automation/`
2. Push to GitHub:
```bash
cd ~/Desktop/linkedin-automation
git add [file]
git commit -m "Fix: [description]"
git push origin main
```
3. Test: GitHub → Actions → LinkedIn Weekly Post Generator → Run workflow

---

## HOW TO REFRESH THE GMAIL TOKEN

If Gmail authentication fails (GMAIL_TOKEN expired):
```bash
cd ~/Desktop/linkedin-automation
python3 authorize_gmail.py
```
Browser will open → authorize → token.json is created.
Open token.json in TextEdit → Cmd+A → Cmd+C.
GitHub → repo → Settings → Secrets → GMAIL_TOKEN → Update → paste → Save.

---

## MATTHIEU'S LINKEDIN PROFILE

URL: https://www.linkedin.com/in/matthieu-jammers-a40a0726/
Based in Bali, Indonesia — real estate professional
Voice: confident but approachable, direct, occasionally witty, active voice, conversational

---

## REQUIREMENTS

```
google-generativeai>=0.8.0
google-auth>=2.28.0
google-auth-oauthlib>=1.2.0
google-auth-httplib2>=0.2.0
google-api-python-client>=2.120.0
firecrawl-py>=0.0.20
python-dotenv>=1.0.0
requests>=2.31.0
```
