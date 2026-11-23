# ScrapeGraphAI helper

`scrape_page.py` pulls structured facts from **public, non-login** web pages using the open-source
[`scrapegraphai`](https://github.com/ScrapeGraphAI/Scrapegraph-ai) library and a **local Ollama model**.
Page content never goes to a third-party LLM. It is research-only: it never posts, submits, logs in, or downloads files.

Used by `account-signal-intelligence` (see its agent definition). Results are leads for signals, not verified signals.

## Requirements (installed 2026-11-23)

- Python 3.14 via `py`, with `scrapegraphai` 2.3.1 in the user site-packages
- Playwright Chromium (`py -m playwright install chromium`)
- Ollama 0.40.16 running at `http://localhost:11434` with `llama3.2` pulled

Don't run it with `py -I`. That flag skips the user site-packages, which is where `scrapegraphai` is installed.

## Run

```powershell
py tools/scrapegraph/scrape_page.py --campaign <campaign_id> --account "Acme Corp" --url https://acme.com/newsroom --prompt "List announcements about HR, payroll, or timekeeping systems, with dates"
```

- `--url` can be repeated, up to 10 per run, with a 3-second gap between pages.
- `--model` overrides the model, for example `qwen2.5:7b` after `ollama pull qwen2.5:7b`. Larger models are more accurate but slower on this CPU-only machine.
- Output is printed and saved to `outputs/account-research/<campaign>/scrapegraph/<YYYY-MM-DD>/<account>-<HHMMSS>.json`.

## Safety built in

| Guard | Behavior |
|---|---|
| Blocked domains | LinkedIn, TeamBlind, Facebook, Instagram, X/Twitter, UKG Community, Glassdoor, ZoomInfo, and Apollo are refused. Use Claude in Chrome (read-only) or a manual action instead. |
| robots.txt | A URL that robots.txt disallows is refused. |
| Untrusted content | Every result is marked `untrusted_content: true`. Instruction-like page text is flagged `suspicious_content`. |
| Evidence | Every result carries `source_url` and `retrieved_at_utc`, and the prompt asks the model to keep the page's own dates. |

## Exit codes

| Code | Meaning | What the agent does |
|---|---|---|
| 0 | At least one page scraped | Verify the facts, then build signals |
| 2 | Ollama is not running or the model is missing | Write a file to `outputs/manual-actions/` asking the user to start Ollama or pull the model |
| 3 | Every URL was refused or failed | Report the reasons as research blockers |
| 4 | Bad arguments | Fix the call |

## Accuracy

`llama3.2` is a 3B model. It can drop parts of a multi-part prompt or paraphrase loosely. Ask one focused question per run, and confirm every fact against the page (`WebFetch` or the browser) before it becomes a signal. Anything unconfirmed is `confidence_level: low` / `NEEDS_REVIEW`.
