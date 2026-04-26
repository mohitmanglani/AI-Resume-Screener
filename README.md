# AI Resume Screener

An n8n workflow that watches a Google Drive folder, scores incoming resumes against a job description with Groq, and routes a shortlist or rejection email through Gmail — with structured results logged back to Google Sheets.

![Architecture](assets/architecture.svg)

---

## What it does

- **Watches a Drive folder** for new resume PDFs and triggers automatically on upload.
- **Extracts and parses each resume** into a structured candidate profile (name, contact, experience, skills, tools, strengths, achievements, red flags) using Groq's `gpt-oss-20b`.
- **Pulls the active job description** from a Google Sheet and scores the candidate against it (0–100, with `SHORTLIST` / `MAYBE` / `REJECT`).
- **Routes the outcome:** shortlisted candidates get a personalised email and a row appended to the results sheet; rejected candidates get a courteous decline.

## The problem this solves

Manual resume screening has two problems: it's slow, and it's inconsistent. A recruiter looking at fifty resumes on a Friday afternoon will not score the fiftieth resume by the same yardstick they used on the first. And for early-stage teams without a dedicated recruiter, the bottleneck is worse — resumes pile up in inboxes and the best candidates get a slow response.

This workflow handles the first-pass filter: it converts a free-text resume and a free-text job description into a structured comparison, applies a consistent rubric, and surfaces the candidates worth a human's attention. The recruiter still makes the call — but on a clean shortlist, not a hundred PDFs.

## How it works

The workflow is fourteen nodes arranged in a linear pipeline:

**Trigger and extract.** A Google Drive trigger watches a designated folder. When a new file is created, the workflow downloads it and runs the file through n8n's PDF extractor to get the raw resume text.

**First LLM call — Resume Analyser.** A LangChain node sends the extracted text to Groq with a prompt that asks for a strict JSON response: name, email, phone, location, years of experience, education, skills, tools, strengths, weaknesses, achievements, and red flags. The model returns a JSON string.

**Defensive parser.** A small JavaScript node parses the LLM's response. It scans for the outermost `{...}` block (in case the model wraps its output in prose), strips escaped characters, and throws a `PARSE_ERROR` if it can't find valid JSON. This makes the pipeline resilient to the model occasionally adding preamble or markdown fencing it was told not to.

**Pull the job description.** A Google Sheets node reads the active JD from a `job_descriptions` sheet — meaning the JD is dynamic and can be edited without touching the workflow.

**Second LLM call — HR Scorer.** A second LangChain node feeds both the parsed candidate profile and the JD to Groq, asking for a score, a recommendation, the matched and missing skills, and a one-line summary. Same structured-JSON pattern.

**Parse and merge.** A second JavaScript node parses the scorer's output and merges it with the original candidate data, attaching the source file name and a timestamp. The final object has everything the downstream nodes need: identity, score, recommendation, skills breakdown, and audit metadata.

**Branch on score.** An `If` node checks whether the score is at least 70. The threshold is the only configurable rule in the workflow, and it sits at the boundary between scoring (machine) and outreach (human-facing).

**Outputs.** Shortlisted candidates trigger two actions in parallel: a personalised Gmail message to the candidate, and an append to the `Resume_Screener_Results` sheet logging the full evaluation. Rejected candidates trigger a single rejection email — no row is logged, since the HR team only needs to see who made it through.

## Stack

| Layer | Tool | Why |
|---|---|---|
| Orchestration | n8n (self-hosted on Docker) | Visual workflow editor, native nodes for every integration this needs. |
| LLM inference | Groq (`gpt-oss-20b`) | Fast and cheap for structured-JSON tasks. The 20B parameter model is more than adequate for resume parsing and scoring; saves cost vs. larger models for a high-volume use case. |
| File source | Google Drive | Where resumes already live. Trigger fires on file creation in a watched folder. |
| Job descriptions | Google Sheets | Recruiters can edit JDs without touching the workflow. |
| Notifications | Gmail | Already in use by the recruiting team; no new tool to learn. |
| Logging | Google Sheets | Same — recruiters live in spreadsheets. |

## Notable design decisions

**Two LLM calls instead of one.** A single call could in theory both parse and score — but separating them buys reliability. The parser only needs to extract; the scorer only needs to compare. Each prompt is shorter, more focused, and easier to debug when something drifts.

**Defensive JSON parsing.** LLMs occasionally ignore "return ONLY JSON" instructions and wrap output in prose or markdown fences. The parser nodes scan for the outermost JSON object rather than calling `JSON.parse` directly, and throw a clear `PARSE_ERROR` when the response really is malformed. This is the difference between a workflow that runs in production and one that breaks once a week.

**Dynamic JD via Sheets.** Hardcoding the JD in the prompt would mean redeploying the workflow every time the role changes. Pulling it from Sheets keeps the recruiter in control.

**Threshold at the boundary.** The 70/100 cutoff is the only "rule" the workflow encodes. Everything before it is reasoning about a candidate; everything after it is action. Keeping the threshold isolated means it's easy to tune — and easy to swap for a different rule (e.g., recommendation must be `SHORTLIST`, regardless of score) without touching the rest of the pipeline.

## What I'd build next

This is a working v1, not a finished product. Things I'd add for a production deployment:

- **DOCX support.** Currently PDF only. A `Switch` node by file extension and a second extractor would handle this.
- **Duplicate detection.** Re-uploads of the same resume currently get processed twice. A Sheets lookup by file name before the LLM calls would skip duplicates and save inference cost.
- **Bias redaction.** The HR Scorer currently sees the candidate's name and location. Stripping these before scoring (and re-attaching afterward for the email and log) would reduce demographic bias in scoring.
- **Eval loop.** Feed the workflow a curated set of resumes with known correct outcomes and measure agreement. Without this, "the prompts are good enough" is a gut feeling rather than a number.
- **Cost and latency tracking.** Per-resume inference cost is small but adds up at scale. Logging tokens-in/tokens-out per run would make it easy to spot regressions.

## Setup

If you want to run this yourself:

1. Import `workflow/resume-screener.json` into your n8n instance.
2. Reconnect credentials: Google Drive (OAuth), Google Sheets (OAuth), Gmail (OAuth), Groq (API key).
3. Replace the placeholder IDs:
   - `YOUR_DRIVE_FOLDER_ID` — the Drive folder to watch.
   - `YOUR_JD_SHEET_ID` — sheet with one row per active JD; the workflow reads the `Description` column.
   - `YOUR_RESULTS_SHEET_ID` — sheet to log shortlisted candidates.
4. Make sure the results sheet has headers: `Name`, `Score`, `Recommendation`, `Summary`, `Skills Matched`, `Skills Missing`, `Experience Years`, `Red Flags`, `Strengths`, `Weaknesses`, `Status`, `Education`.
5. Activate the workflow. Drop a resume PDF into the Drive folder.

## Links

- Author: [Mohit Manglani](https://linkedin.com/in/mohitmanglani-data)
- More projects: [github.com/mohitmanglani](https://github.com/mohitmanglani)
