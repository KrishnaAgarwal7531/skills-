# ModuleMapper Skill for Claude

Research any university course using live student reviews — directly inside Claude.

## What it does

Ask Claude things like:
- *"What is BT1101 like at NUS?"*
- *"Is CS2103T worth taking?"*
- *"How hard is MATH101 at MIT?"*
- *"What do students say about CS50 at Harvard?"*

Claude will automatically scrape **Reddit**, **RateMyProfessors**, the university's **course review platform**, the **official course page**, and **student blogs** in real-time using TinyFish — then synthesise everything into a structured verdict with score, difficulty, workload, student quotes, and more.

## Requirements

- A **TinyFish API key** — get one free (500 steps, no credit card) at [tinyfish.ai](https://agent.tinyfish.ai/api-keys)
- Set it as the environment variable `TINYFISH_API_KEY`

## Install

1. Download `modulemapper.skill` from the [Releases](../../releases) page
2. Go to **Claude.ai → Settings → Skills**
3. Upload the `.skill` file
4. Done — Claude will now use this skill automatically whenever you ask about a course

## How it works

The skill follows a 4-step pipeline:

1. **Discover** — searches the web in real-time to find the right subreddits, course review platform, and official course page for any university
2. **Scrape** — runs multiple TinyFish web agents in parallel across all sources
3. **Synthesise** — analyses all raw student data and produces a structured verdict
4. **Present** — displays score, difficulty, workload, student quotes, tags, and more

Works for **any university worldwide** — not limited to a hardcoded list.

## Built with

- [TinyFish Web Agent](https://tinyfish.ai) — parallel web scraping
- [ModuleMapper](https://github.com/YOUR_USERNAME/modulemapper) — the original web app this skill is based on

## License

MIT
