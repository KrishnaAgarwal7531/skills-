# Tutor Finder — Claude Skill

**Find and compare exam tutors from across the web in real time.**

Ask Claude things like:
- *"Find me a GRE tutor in London"*
- *"Best online SAT tutors under $50/hr"*
- *"Compare GMAT tutors — I'm in Singapore"*
- *"Find JEE tutors in India"*
- *"Who are the top TOEFL tutors available online?"*

Claude fires parallel TinyFish agents across 7-10 tutoring platforms simultaneously — Wyzant, Varsity Tutors, Preply, Kaplan, Princeton Review, and location-specific platforms — extracting live tutor profiles with pricing, qualifications, experience, and booking links.

## Supported exams

SAT · ACT · AP · GRE · GMAT · TOEFL/IELTS · JEE/NEET · Olympiads

## What you get

- Tutor name, qualifications, and experience
- Pricing per hour (where listed)
- Teaching mode (online / offline / hybrid)
- Past student results and score improvements
- Direct profile link or booking method
- Quick comparison table across all tutors
- A top recommendation based on your exam and location

## Requirements

- TinyFish CLI: `npm install -g tinyfish`
- Authenticated: `tinyfish auth login`

## Install

**Claude.ai:** Download `tutor-finder.skill` from Releases → upload to Settings → Skills

**CLI:**
```bash
npx skills add KrishnaAgarwal7531/skills- --skill tutor-finder
```

## Based on

The [Exam Tutor Finder web app](https://tinyfishtutorsfinder.lovable.app) — this skill brings the same functionality directly into Claude without needing the web app.

## Built with

- [TinyFish Web Agent](https://tinyfish.ai)
- Part of the [TinyFish Cookbook](https://github.com/tinyfish-io/tinyfish-cookbook)
