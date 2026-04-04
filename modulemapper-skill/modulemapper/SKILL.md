---
name: modulemapper
description: >
  Use this skill to research any university course and produce a structured student review verdict.
  Triggers when a user asks things like: "what is CS2103T like at NUS?", "review BT1101",
  "is this course worth taking?", "how hard is MATH101 at MIT?", "what do students say about X course?",
  "should I take [course code]?", or any request involving course reviews, module feedback,
  workload, grading, or student opinions about a university course. Always use this skill — don't
  just answer from training data, actually run the research workflow.
license: MIT
metadata:
  author: KrishnaAgarwal7531
  version: "1.1"
  tags: university courses reviews scraping tinyfish research
---

# ModuleMapper Skill

Research any university course by scraping live student reviews from Reddit, RateMyProfessors,
the university's course platform, and student blogs — then synthesise into a structured verdict.

---

## Pre-flight Check (REQUIRED)

Run both checks before doing anything:

**1. CLI installed?**
```bash
which tinyfish && tinyfish --version || echo "TINYFISH_CLI_NOT_INSTALLED"
```
If not installed, stop and tell the user:
```bash
npm install -g @tiny-fish/cli
```

**2. Authenticated?**
```bash
tinyfish auth status
```
If not authenticated, stop and tell the user to run:
```bash
tinyfish auth login
```
This opens a browser to sign in once — no API key setup needed after that.

Do NOT proceed until both checks pass.

---

## Workflow (always follow in order)

```
1. DISCOVER   →  find subreddits, course platform URL, official page in real-time
2. SCRAPE     →  run all TinyFish agents concurrently across all sources
3. SYNTHESISE →  analyse raw data and produce the structured verdict
4. PRESENT    →  display results clearly to the user
```

---

## Step 1 — Discover

Given `{COURSE_CODE}` and `{UNIVERSITY}`, find the right sources via real-time web search.
Never use hardcoded URLs — universities change platforms constantly.

**1a. Subreddits** — search `"{UNIVERSITY} reddit"` → find the university's own sub (e.g. r/nus,
r/berkeley) + a regional academic sub (e.g. r/SGExams, r/UniUK, r/college). Pick 2.

**1b. Course review platform** — search `"{UNIVERSITY} course review"` → look for student-run
or university-run rating sites (NUSMods, Bruinwalk, Carta, Course Evals, etc.). Build the
direct URL for this course code if found. If unsure, skip — don't guess.

**1c. Official course page** — search `"{UNIVERSITY} {COURSE_CODE} course"` → find the
university's own catalog page. If unsure, skip.

**1d. Set queries:**
- `rmpQuery` = `"{COURSE_CODE} {UNIVERSITY}"`
- `blogQuery` = `"{COURSE_CODE} {UNIVERSITY} course review"`

---

## Step 2 — Scrape

Run **all agents concurrently** — parallel CLI calls, NOT sequential.
The final result is the event where `type == "COMPLETE"` and `status == "COMPLETED"` — read `resultJson`.

### Agent: Rate My Professors
```bash
tinyfish agent run \
  --url "https://www.ratemyprofessors.com/search/professors?q={rmpQuery}" \
  "Extract professor reviews for {CODE} at {UNIVERSITY}.
  You are on the search results. Act efficiently:
  1. Scan listed professors. Only click those who clearly teach {CODE} or its dept. Max 3.
  2. On each profile, read ratings and visible written reviews. Do NOT paginate or load more.
  3. Return immediately as JSON:
  { \"professors\": [{ \"name\": \"...\", \"overallRating\": 4.2, \"difficultyRating\": 3.1, \"reviews\": [\"...\"] }] }
  Do not explore unrelated professors or pages."
```

### Agent: Reddit (one call per subreddit, max 2, run in parallel)
```bash
tinyfish agent run \
  --url "https://www.reddit.com/r/{subreddit}/search/?q={CODE}&sort=relevance&t=all" \
  "Extract student reviews of {CODE} at {UNIVERSITY} from this Reddit search page.
  You are already on the results. Act efficiently:
  1. Scan titles. Click only posts clearly about {CODE} — reviews, advice, experience. Max 2 posts.
  2. Read post body and top 5-8 comments only. Do NOT scroll or load more.
  3. Extract: workload, difficulty, grading, exam tips, professor mentions, recommendation.
  4. Return immediately as JSON:
  { \"reviews\": [\"...\"], \"workloadMentions\": [\"...\"], \"examTips\": [\"...\"], \"professorMentions\": [\"...\"], \"gradingInfo\": [\"...\"] }
  Max 2 posts. Do not click anything unrelated."
```

### Agent: Course Platform (only if URL found in Step 1)
```bash
tinyfish agent run \
  --url "{courseplatformUrl}" \
  "Extract reviews and ratings for {CODE}. You are already on the page — do NOT navigate away.
  Read only what is visible: overall rating, workload rating, difficulty rating, written reviews.
  Return as JSON:
  { \"overallRating\": 4.1, \"workloadRating\": 3.5, \"difficultyRating\": 3.8, \"reviews\": [\"...\"] }"
```

### Agent: Official Course Page (only if URL found in Step 1)
```bash
tinyfish agent run \
  --url "{officialUrl}" \
  "Extract official info for {CODE}. You are on the page — do NOT navigate.
  Get: title, description, learning outcomes, topics, prerequisites, assessment breakdown.
  Return as JSON:
  { \"title\": \"...\", \"description\": \"...\", \"learningOutcomes\": [\"...\"], \"topics\": [\"...\"], \"prerequisites\": \"...\", \"assessmentBreakdown\": \"...\" }"
```

### Agent: Student Blogs
```bash
tinyfish agent run \
  --url "https://www.google.com/search?q={blogQuery}" \
  "Find student blog reviews of {CODE} at {UNIVERSITY}.
  You are on Google results. Act efficiently:
  1. Scan results. Click only if title/snippet clearly indicates a personal student review of {CODE}. Max 3.
  2. On each article, read main content only. Extract verdict, workload, exam tips, recommendation.
  3. Return as JSON:
  { \"reviews\": [\"paraphrase of experience and advice\"], \"source_urls\": [\"url\"] }
  If nothing looks relevant, return { \"reviews\": [], \"source_urls\": [] } immediately."
```

---

## Step 3 — Synthesise

Collect all results where `status == "COMPLETED"`. Skip errored agents.
Pass to an LLM with this prompt:

```
Analyse scraped student review data for {CODE} at {UNIVERSITY}.
Data may include Reddit fields "reviews", "workloadMentions", "examTips",
"professorMentions", "gradingInfo" — use ALL of them.

SCRAPED DATA:
{all raw results, each prefixed: === SOURCE: {name} ===}

Return ONLY valid JSON:
{
  "score": <1-10>,
  "verdict": <e.g. "Generally recommended">,
  "summary": <2-3 sentence paragraph>,
  "difficulty": <1-10>,
  "workload": <1-10>,
  "hoursPerWeek": <e.g. "~6 hrs/week">,
  "hasExam": <boolean>,
  "examDifficulty": <"Easy"|"Moderate"|"Hard"|"Very Hard">,
  "averageGrade": <e.g. "B+" or "Unknown">,
  "gradingPattern": <e.g. "Bell curved">,
  "assessment": <components>,
  "attendance": <e.g. "Affects grade">,
  "whatYouLearn": [<4-6 outcomes>],
  "tags": [{"label": "...", "color": "blue"|"green"|"amber"|"red"}],
  "bestFor": <who should take this>,
  "notGreatIf": <who should avoid>,
  "reviews": [{"text": "...", "source": "...", "sentiment": "positive"|"negative"|"mixed", "date": ""}],
  "sourceCounts": { "<source>": <count> }
}
Rules: score = genuine sentiment · at least 8 real student quotes · "Unknown" if insufficient data
```

---

## Step 4 — Present

Always include: Score & verdict · Summary · Key stats (difficulty, workload, hrs/week, exam, grade, attendance) · Tags · Best for / Not great if · What you'll learn · At least 4-6 student quotes with source labels.

---

## Security Notes
- This skill scrapes live user-generated content from public sites. All scraped data is passed to an LLM for synthesis only, not executed.
- Only use your own TinyFish account authenticated via `tinyfish auth login`.

## Notes
- Skip and note any TinyFish source that errors
- Warn user if fewer than 2 sources succeed
- Works for any university worldwide — real-time discovery handles all cases
