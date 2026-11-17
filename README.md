# Job-Hunt Automation (n8n + Gemini)

A self-hosted [n8n](https://n8n.io) workflow that, once a day, collects new job postings, scores each one against my resume with an LLM agent, writes a tailored cover letter and a list of resume edits, and logs everything to a Google Sheet.

I ran it on a locally hosted n8n instance (started from the terminal, no paid n8n cloud plan) from October to December 2025.

## Results

From the workflow's own output sheet (Oct–Dec 2025):

- **415** unique job postings processed (deduplicated by job link)
- **227** distinct companies
- **347 of 415** postings produced a complete match score and cover letter. The other 68 hit the parser's safe fallback (the model returned unparseable output), so the run kept going instead of failing.

## How it works

```
Schedule (daily, 05:00)
  -> Google Drive: download resume PDF -> Extract text from PDF
  -> Google Sheets: read filter row (Remote / Easy Apply)
  -> Code (JS): build a LinkedIn job-search URL (last 24h, Canada, DS/DA/ML roles, experience levels 1-4)
  -> HTTP + HTML: fetch the public results page, extract job links (CSS selectors)
  -> Split Out -> Loop Over Items (2s wait between jobs)
       -> HTTP + HTML: extract title, company, location, description, job ID
       -> AI Agent #1 (Gemini via n8n LangChain agent node)
            strict-schema JSON: job analysis, resume analysis,
            0-100 match score over 6 weighted categories with cited evidence,
            gaps + upskilling suggestions, 150-220 word cover letter
       -> Set (JS): robust JSON parser for the model output
       -> AI Agent #2 (Gemini): prioritized, tagged resume edits
            ([ADD] [REMOVE] [REWRITE] [ORDER] [QUANTIFY] [KEYWORDS] [FORMAT] [FOCUS])
       -> Google Sheets: append-or-update row, matched on job link
  -> Telegram: "results are ready" notification when the loop finishes
```

### Scoring weights (Agent #1)

| Category | Points |
|---|---|
| Skills / tools overlap | 40 |
| Relevant experience & seniority | 25 |
| Responsibilities alignment | 15 |
| Education / certifications | 10 |
| Domain / industry fit | 5 |
| Logistics (location, work authorization) | 5 |

Up to 10 points can be deducted for red flags; the score is clamped to 0-100.

### Making LLM output reliable

The model is asked for one JSON object in a fixed schema, ending with `END_OF_JSON`. The `Edit Fields1` node then:

1. strips Markdown code fences,
2. normalizes curly quotes that Gemini sometimes emits,
3. cuts the text at `END_OF_JSON`,
4. isolates the first complete `{...}` object by brace counting (string- and escape-aware),
5. parses it, and returns `{}` on failure so a single bad response never stops the run.

Results are upserted on the job link, so re-running the workflow updates existing rows instead of duplicating them.

## Setup

1. Run n8n locally, for example `npx n8n` or `docker run -it --rm -p 5678:5678 n8nio/n8n`, then open http://localhost:5678.
2. Import `Job-Hunt_N8N.json`.
3. Create credentials in n8n and attach them to the nodes marked `REPLACE_WITH_YOUR_CREDENTIAL`:
   - Google Drive and Google Sheets (OAuth2)
   - Google Gemini (free-tier API key from Google AI Studio)
   - Telegram bot (optional), Gmail (optional; that node is disabled)
4. Replace the placeholders:
   - `YOUR_RESUME_PDF_GOOGLE_DRIVE_URL`: share link to your resume PDF
   - `YOUR_GOOGLE_SHEET_URL` / `YOUR_GOOGLE_SHEET_ID` / `YOUR_RESULT_TAB_GID`: your tracking sheet
   - `YOUR_TELEGRAM_CHAT_ID`
5. Create the Google Sheet with two tabs:
   - **Filter**: `Keyword | Location | Experience Level | Remote | Easy Apply` (one row; the Code node currently uses only `Remote` and `Easy Apply`, and a fixed role list)
   - **Result**: `Title | Company  | Locaton | Link | Score | Cover Letter | Skills | Improvements` (the column names, including the trailing space in `Company ` and the `Locaton` spelling, must match the mapping in the workflow)
6. Activate the workflow.

## Notes and limitations

- Job collection reads LinkedIn's public job-search pages with CSS selectors, so it breaks when LinkedIn changes its markup. It is a personal tool: keep volume low and respect LinkedIn's terms of service.
- The role list (data scientist / data analyst / ML engineer) is hard-coded in the Code node; edit it there to change the search.
- Scores and cover letters are LLM output and need human review before use.
- All personal IDs, links and credential references were removed from this export and replaced with placeholders.
