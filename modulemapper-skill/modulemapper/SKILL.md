---
name: modulemapper
description: >
  Use this skill to research any university course and produce a structured student review verdict.
  Triggers when a user asks things like: "what is CS2103T like at NUS?", "review BT1101",
  "is this course worth taking?", "how hard is MATH101 at MIT?", "what do students say about X course?",
  "should I take [course code]?", or any request involving course reviews, module feedback,
  workload, grading, or student opinions about a university course. Always use this skill — don't
  just answer from training data, actually run the research workflow.
---

# ModuleMapper Skill

Research any university course by scraping live student reviews from Reddit, RateMyProfessors,
the university's course platform, and student blogs — then synthesise them into a structured verdict.

## Prerequisites

You need a **TinyFish API key** set as `TINYFISH_API_KEY` in your environment.  
Get one free at: https://agent.tinyfish.ai/api-keys

---

## Workflow (always follow in order)

```
1. DISCOVER  →  resolve university profile (subreddits, platform URL, RMP query)
2. SCRAPE    →  run TinyFish agents in parallel across all sources
3. SYNTHESISE →  analyse all raw data and produce the final verdict
4. PRESENT   →  display the verdict clearly to the user
```

---

## Step 1 — Discover

Given `{COURSE_CODE}` and `{UNIVERSITY}`, you need to figure out the right sources to scrape.
**Do this in real-time — do not rely on hardcoded URLs.** Universities change platforms, and
new ones are always being added.

### 1a. Find the relevant subreddits
Web search: `"{UNIVERSITY} reddit"` or `"site:reddit.com {UNIVERSITY} courses"`
Look for the university's own subreddit (e.g. r/nus, r/berkeley) plus any regional academic
subreddits (e.g. r/SGExams, r/UniUK). Pick the 2 most relevant.

### 1b. Find the course review platform
Web search: `"{UNIVERSITY} course review platform"` or `"{UNIVERSITY} module review"`
You're looking for a student-run or university-run site where students rate and review courses —
things like NUSMods, Bruinwalk (UCLA), Carta (Stanford), Course Evals (UofT), etc.
If you find one, construct the direct URL for this specific course code if possible.

### 1c. Find the official course page
Web search: `"{UNIVERSITY} {COURSE_CODE} official course page"` or `"{UNIVERSITY} course catalog {COURSE_CODE}"`
Look for the university's own catalog or handbook page for this specific course.

### 1d. Set the standard queries
- `rmpQuery` = `"{COURSE_CODE} {UNIVERSITY}"`
- `blogQuery` = `"{COURSE_CODE} {UNIVERSITY} course review"`

> **Key principle:** a quick web search at discover time is much better than a stale hardcoded
> table. If you can't find a platform or official page, just skip those agents — the Reddit +
> RateMyProfessors + blogs agents will still give good coverage.

---

## Step 2 — Scrape

Run **all agents concurrently** using TinyFish. Always use `browser_profile: "stealth"`.

**Always run these agents:**

### Agent: Rate My Professors
```
URL: https://www.ratemyprofessors.com/search/professors?q={rmpQuery}

GOAL:
Search Rate My Professors for professors teaching {CODE} at {UNIVERSITY}.

STEP 1: Find professors associated with this course or department.
STEP 2: Click into each professor's profile and read ALL written reviews, not just ratings.
STEP 3: For each professor extract:
- Overall rating and difficulty rating
- Every written student review (full text)
- Recurring themes (fair grader? hard exams? engaging lectures?)

Return as JSON:
{
  "professors": [
    {
      "name": "Prof name",
      "overallRating": 4.2,
      "difficultyRating": 3.1,
      "reviews": ["Full text of review 1", ...]
    }
  ]
}
```

### Agent: Reddit (run for first 2 subreddits)
```
URL: https://www.reddit.com/r/{subreddit}/search/?q={CODE}+{UNIVERSITY}&sort=relevance&t=all

GOAL:
Search Reddit for detailed student reviews of {CODE} at {UNIVERSITY}.

STEP 1: Find posts about this specific course — look for titles mentioning "{CODE}",
        module reviews, course advice.
STEP 2: Click into 3-5 of the most relevant posts and read the FULL post AND all top comments.
STEP 3: Extract:
- Full text of each post and substantial comments
- Mentions of: workload, weekly hours, exams, assignments, grading, bell curve,
  professor names, tips, difficulty, what you actually learn

Return as JSON:
{
  "reviews": ["Detailed paraphrase including context like semester, background, grade", ...],
  "workloadMentions": ["any specific mentions of hours/week or workload"],
  "examTips": ["tips about exams or assessments"],
  "professorMentions": ["mentions of specific professors"],
  "gradingInfo": ["mentions of grades, bell curve, average grade"]
}
```

### Agent: Course Platform (only if courseplatformUrl exists)
```
URL: {courseplatformUrl}

GOAL:
Extract all student reviews and ratings for {CODE}.
Get: overall rating, workload rating, difficulty rating, all written reviews.
Return as JSON: { overallRating, workloadRating, difficultyRating, reviews: [...] }
```

### Agent: Official Course Page (only if officialUrl exists)
```
URL: {officialUrl}

GOAL:
Extract the official course description for {CODE}.
Get: title, full description, learning outcomes, topics, prerequisites,
assessment breakdown (exam %, CA %, project %).
Return as JSON: { title, description, learningOutcomes, topics, prerequisites, assessmentBreakdown }
```

### Agent: Student Blogs
```
URL: https://www.google.com/search?q={blogQuery}+site:medium.com+OR+site:reddit.com+OR+site:wordpress.com

GOAL:
Find student blog posts reviewing {CODE} at {UNIVERSITY}.
STEP 1: Find blog posts that are actual student reviews.
STEP 2: Click into 2-3 most relevant results and read the FULL article.
STEP 3: Extract verdict, workload details, tips, background of the student.

Return as JSON:
{
  "reviews": ["Detailed paraphrase with experience, recommendation, and advice", ...],
  "source_urls": ["url of each article"]
}
```

---

## Step 3 — Synthesise

Collect all raw results (skip any that errored). Pass them to Claude or any LLM with this prompt:

```
You are an expert at analysing student course reviews.
Analyse the following scraped data about {CODE} at {UNIVERSITY}
and return a structured JSON verdict.

The data may include Reddit posts with fields like "reviews", "workloadMentions",
"examTips", "professorMentions", "gradingInfo" — use ALL of these.

SCRAPED DATA:
{all raw results joined with === SOURCE: {name} === headers}

Return ONLY valid JSON with this structure:
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
  "assessment": <description of components>,
  "attendance": <e.g. "Affects grade" or "Not graded">,
  "whatYouLearn": [<4-6 learning outcomes>],
  "tags": [{"label": <string>, "color": <"blue"|"green"|"amber"|"red">}],
  "bestFor": <who should take this>,
  "notGreatIf": <who should avoid>,
  "reviews": [
    {"text": <student quote>, "source": <name>, "sentiment": <"positive"|"negative"|"mixed">, "date": ""}
  ],
  "sourceCounts": { <source>: <count> }
}

Rules:
- score reflects genuine student sentiment
- Extract at least 8 real student quotes for reviews
- whatYouLearn comes from official page if available
- tags: blue=general info, green=positive, amber=warning, red=negative
- Use "Unknown" for any field with insufficient data
```

---

## Step 4 — Present

Display the verdict conversationally. Always include:

- **Score & verdict** — e.g. "7.2/10 — Generally recommended"
- **Summary paragraph**
- **Key stats** — difficulty, workload, hrs/week, exam info, average grade, attendance
- **Tags** — inline labels
- **Best for / Not great if**
- **What you'll learn** — bullet list
- **Student reviews** — at least 4–6 quotes with source labels
- **Source breakdown** — how many reviews came from each source

---

## Notes

- If TinyFish errors on a source, skip it and note it in the output
- If fewer than 2 sources return data, tell the user results may be limited
- The skill works for any university worldwide — use the fallback profile for unknown ones
- See `references/universities.md` for the full university profile table
