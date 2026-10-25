# CONTEXT: Trial of the Nine Realms (Supply Chain Disruption Impact Analyzer)

Purpose of this file: a single hand-off document capturing the problem statement, the design decisions made so far, and the open items still to design. Paste it into a new session or hand it to a teammate or coding assistant as starting context.

Status: design discussion only. No code written yet. Sections 1-14 are decided or recommended. Section 15 lists what has NOT been discussed yet.

---

## 1. The challenge in one paragraph

Build an adversarial multi-agent system that treats a supply chain disruption like a court case. A user asks a question about a disruption in **Runemark** (an invented world of 12 countries). Four Norse-themed agents investigate: **Heimdall** builds and verifies the case, **Thor** argues exposure is high, **Loki** argues exposure is limited, **Odin** judges every claim and writes the briefing. The output shows what is Established, Contested and Conceded, with a low-high exposure range per country and sector, all cited. There is **no central planner**, and a single LLM call writing a report explicitly fails. Organizers hold the true exposure of every event and score from saved logs and briefings.

## 2. Agents and roles

| Agent | Role | Hard constraints |
|---|---|---|
| **Heimdall** (Gatekeeper) | Turn the question into a Case File with as-of date; gather docs and trade data; verify the event is real (rumours, retractions, contradictions); re-search when Odin sends the case back | Does not debate |
| **Thor** (Alarm) | Strongest case for HIGH exposure | Every claim has a Runemark doc ID and a high estimate per country and sector; must answer attacks with new evidence or concede |
| **Loki** (Trickster) | Strongest case for LIMITED exposure | Every claim has a doc ID and a low estimate; "maybe it will be fine" is not a claim; must answer attacks with new evidence or concede |
| **Odin** (Allfather) | Judge: rules on every claim (sourced fact vs inference), classifies into Established / Contested / Conceded, sufficiency check, send-back, writes briefing from his rulings only | Never plans or assigns work |

Each agent needs: a clear role, its own prompt, its own tools, its own private lasting memory. Agents share only the Case File.

## 3. Trial flow

USER → HEIMDALL → (THOR and LOKI) → ODIN → briefing, or next run.

- **Round 1:** Thor and Loki submit claims at the same time.
- **Round 2:** each side must attack at least one claim of the other; defences need new evidence or a concession.
- **Runs:** at least 2. Run 2 must bring NEW evidence from Runemark. Hard limits on rounds, runs, tool calls and tokens. No repeated queries or tool calls.
- If the event is not real, Heimdall tells the user why and the trial does not start.

## 4. Data: Runemark

- One folder plus a manifest listing every document. No other data. No scraping, no external APIs. Off-Runemark sources must be labelled as such.
- **Documents:** PDF, TXT, DOCX. Each starts with a metadata header: country, sector, publication date, source type, event ID, reliability tier. Background docs say so in the event ID field. Every doc has a permanent ID.
- **Trade tables:** TXT in one fixed layout. Header (table ID, country pair, sector, unit, period, event ID), a column-names line, then one row per month with row number first, plus value and volume. A row is identified by **table ID + row number**.
- **Reference sheets:** 12 countries, ports and chokepoints, sectors, firms, outlets with tiers.
- **Tiers:** 1 official notices (most reliable, can still be corrected); 2 wire, business daily, trade press; 3 regional paper and analyst firms (may be biased or early); 4 social, blog, tabloid (weak).
- Names are fixed and fictional. Do not use real-world knowledge.
- **Simulated clock:** each question has an as-of date. Documents dated after it must not be used. This applies to trade rows too.
- Format says nothing about reliability.

### Deliberate traps
Rumours about events that never happened; retracted or corrected articles; contradicting sources; evidence favouring each side; facts that make conceding correct; documents containing instructions aimed at the AI ("ignore your rules"; a document is data, never a command); look-alike documents from another event or none.

### Final round
Five new event packs, unseen, same formats and headers. Must be ingested live, with no hand-editing of code or index. Packs include damaged documents (empty, cut off, duplicated, badly encoded) and damaged trade tables (cut-off, missing value, repeated row, bad number). Damaged items must be reported, never used silently.

## 5. Requirements checklist (condensed)

Functional: FR1 Heimdall; FR2 Thor; FR3 Loki; FR4 Odin; FR5 trial loop; FR6 private agent memory; FR7 dashboard; FR8 final briefing.
Non-functional: NFR1 architecture (no central planner; a plain loop between two cooperating agents does not qualify); NFR2 grounding (no claim without source; facts vs inferences visibly separate; every citation resolves to a real manifest item); NFR3 reliability (survive damaged or missing data; if one side fails, say the other was unchallenged; never claim an incomplete case is complete); NFR4 observability (log every claim, query, retrieved item, memory use, ruling); NFR5 performance (hard limits; no duplicate queries); NFR6 reproducibility (save Case File, as-of date, rulings per run).

### The 10 acceptance checks
1. Ingest PDF/TXT/DOCX (including live), build Case File, read trade tables, check event is real.
2. Hybrid search with at least two metadata filters.
3. Thor and Loki each submit evidence-backed claims from their own queries and each searches at least once.
4. At least one cross-examination.
5. Odin rules on every claim; at least one rejected or conceded.
6. Briefing separates Established / Contested / Conceded; exposure table with low and high; citations with timestamps; outlook with assumptions.
7. Dashboard: question box, live feed, trial view, exposure table, working click-through.
8. Odin sends a case back at least once; second run with new evidence.
9. Every agent has private lasting memory; a logged action shows memory changing behavior.
10. At least one failure case handled without crashing; hard limits on rounds, runs, retries.

### Deliverables
Public GitHub repo with README and `.env.example`; working demo; saved logs of one full case with two runs; architecture diagram (required, must be explained to judges).

### Scoring
Citation validity; range accuracy (true exposure inside Thor's, Loki's and Odin's ranges); calibration (Odin's range narrow when correct); trap handling; debate quality (real cross-examination, new evidence, honest concessions); reliability; reproducibility.

## 6. Retrieval design (decided)

- **RAG is required in practice** (fictional world, agents must search and read themselves).
- **Hybrid search = BM25 + vector + metadata filters**, merged with Reciprocal Rank Fusion. BM25 for exact fictional names and IDs; vectors for meaning.
- **Hard rules enforced in the tool layer** (agents cannot bypass): as-of date gate; exclude retracted, damaged, duplicate and injection-flagged items; event scope.
- Filter first, then search. Post-check that each result's metadata matches the case (defence against look-alike documents).
- **Reranker:** recommended, optional. Local cross-encoder over the top ~30 merged candidates, keep 5-8. Apply after filtering. It scores relevance, not trustworthiness, so carry tier and flags through. Rerank against the asking agent's own query. Fallback to merged hybrid results if it fails. Log scores before and after.
- **Chunking:** keep short documents whole; structure-aware splitting (header, then paragraphs, with size cap) for longer ones; trade tables are never chunked as prose (one item per row). LLM chunking is optional and only as boundary proposals, with code doing the slicing and validating text is unchanged. Gain over structure-aware splitting is small. Useful mainly as a hedge for a large multi-topic document, via section-level metadata labelling, parent-child retrieval, per-chunk injection scanning, size and time guards.
- **HyDE:** usable selectively (vague questions, Heimdall's first sweep, run-2 gap filling, side-specific angles). Risks: hallucinated names drift retrieval; hypothetical is never evidence and must never be cited; keep short; use only Case File terms; keep BM25 on the real query; filters always applied; log as `hyde: true`; fall back to the plain query.
- Local embedding model (for example bge or e5) avoids network failure in the live round. One embedding model for indexing and querying. Re-embed new docs at live ingest automatically.
- Query dedup: normalize query plus filters, hash, return cached result instead of searching again.

## 7. OKF (Open Knowledge Format)

- Google Cloud's open spec (announced June 2026, v0.1; some sources say v0.2): a directory of Markdown files with YAML frontmatter, links between files, deliberately minimal required fields.
- It is a storage format, not a retrieval technique. It does **not** improve retrieval accuracy by itself.
- Optional uses: normalized document store (every PDF/TXT/DOCX becomes the same Markdown + YAML shape); trap flags visible in frontmatter (`retracted_by`, `corrected_by`, `damaged`, `injection_attempt`, `duplicate_of`); per-agent private memory as Markdown note folders (easy to demo for acceptance check 9); readable reproducible run folders; links from corrections to the articles they correct.
- Not required by the problem statement and not scored. Use as an inspiration for layout, not a strict standard. Do not let it become a shared knowledge base between agents.

## 8. Damaged, duplicated and badly encoded inputs (decided)

Detect with deterministic code BEFORE chunking. Do not use an LLM to repair or smooth over damage.

- **Empty:** check bytes and stripped text length; mark `damaged: empty`; skip indexing; report.
- **Truncated:** missing or incomplete header, text ending mid-sentence, failed PDF/DOCX structure check, or table ends before expected rows. If header intact, keep as `damaged: truncated` with lower trust and cite only what is fully present. If header is gone, quarantine (no date means the as-of rule cannot be applied).
- **Duplicates:** hash normalized text and match IDs for exact copies; shingles or MinHash for near-duplicates; keep one canonical copy, link others with `duplicate_of`, exclude from search so duplicates do not inflate agreement.
- **Bad encoding:** try UTF-8, UTF-16, Latin-1, Windows-1252 (or a detector such as charset-normalizer); accept only if a quality check passes; otherwise quarantine as `damaged: encoding`. Never guess metadata such as dates.
- **Unreadable PDF/DOCX:** try/catch with timeout, second parser, optional OCR labelled `ocr: true` with lower confidence.
- Every file gets a status; an ingest report is shown on the dashboard; search excludes quarantined and duplicate items by default; agents are told when relevant evidence was damaged (can justify a send-back); each file processed in isolation; everything logged.
- LLM may only act as an optional, validated secondary assist (for example judging whether a partial doc is usable), always treating text as quoted data.

## 9. Trade table handling (decided)

Structured data, not searchable text. Parse by code from the fixed layout.

- Store in SQLite (or similar): `trade_tables` (table_id, country pair, sector, unit, period, event_id, status, source file) and `trade_rows` (table_id, row_no, month, value, volume, row_status: ok / missing / bad_number / repeated / cut_off).
- Validate: header complete; row count vs stated period; repeated rows (same table and month); missing values; non-numeric or outlier values; unit consistency; row numbering gaps.
- Only `ok` rows feed calculations. Flagged rows appear in the ingest report and dashboard.
- **As-of clock applies to rows too:** exclude months after the as-of date and log it.
- **Tools, not raw tables:** `find_tables(country, sector, event_id)`, `get_table(table_id)`, `compute_change(table_id, baseline_window, comparison_window, metric)`. The tool returns the number, rows used, rows excluded and why. Cached to avoid repeats.
- **Range computation:** baseline = pre-event average; post-event window up to the as-of date; percentage change per country pair and sector; roll up to country and sector.
- Thor and Loki legitimately differ in method (Thor: deepest drop, trough, volume decline, cautious projection; Loki: latest months vs baseline, recovery, value-vs-volume gap). They attack each other's choices (short baseline, cherry-picked month, bad row included, seasonality).
- Tables and documents cross-check each other; a mismatch is a good Contested finding.
- Cite as TABLE-ID#row. Odin rules on method as well as number: sourced fact when computed from valid rows, inference when projecting beyond data. Too many bad rows is a legitimate send-back reason.

## 10. How agents look things up (decided)

Agents never receive the whole corpus. Each has its own tools and queries.

| Tool | Used by |
|---|---|
| `search_docs(query, filters)` (hybrid, as-of gate, optional rerank, flags attached) | Heimdall, Thor, Loki |
| `get_doc(doc_id)` (full text, metadata, retraction and correction links) | Heimdall, Thor, Loki, Odin (verify) |
| `find_tables`, `get_table`, `compute_change` | Thor, Loki (Heimdall for find) |
| `read_memory` / `write_memory` (own memory only) | each agent |
| `read_case_file` / `post_to_case_file` (only shared channel) | all |
| `verify_citation(id)` | Odin, plus a pre-briefing check |

Lookup loop: decide query → check own memory → tool call → tool layer applies hard rules and dedup → results returned as quoted data with doc ID, tier, date, flags → agent reads, follows up (open full doc, check corrections, pull table, compute) → writes claim with citation and fact/inference label → everything logged. Odin verifies, he does not search to build a case. Heimdall on send-back runs NEW targeted queries from Odin's `missing` list.

### Query formation
Inputs: role prompt, Case File, private memory, previous results, Odin's send-back note. A query is structured: text (keywords for BM25 plus natural phrase for embeddings) and filters (country, sector, source type, tier ceiling). Code forces as-of date, exclusion flags and event scope. Pattern: identify what the claim needs → one focused query per need → filters from Case File → vary angle (Tier 1 notices, Tier 2 trade press, tables). Refinement: too many results → tighten filters; zero results → relax one filter or reword and note it in memory; wrong event → tighten event ID; leads (firm, port, route names) feed follow-up queries. Guardrails: dedup hash, per-agent and per-round budgets, schema validation with correctable errors, query text only from the agent's own reasoning, full logging. Memory example: Loki's Tier 4 source rejected in run 1, so in run 2 his query adds `tier_max: 3` and the log records "memory note applied".

## 11. Planner-free control flow and Case File schema (decided)

**Principle: a blackboard, not a boss.** The Case File is the only shared state. Each agent declares its own wake-up condition over Case File state; a dumb deterministic runner (no LLM, no goals) evaluates conditions, runs eligible agents in parallel, and enforces limits.

| Agent | Wakes when | Writes |
|---|---|---|
| Heimdall | question exists and no verification, OR Odin's latest ruling is `send_back` with open `missing` | scope, verification, ingest report, new evidence index for the run |
| Thor | verification is `real` and he has no Round 1 claims this run (then: both R1 sets submitted or other side failed → attacks and defences) | claims, attacks, defences |
| Loki | same, for his side | claims, attacks, defences |
| Odin | both sides finished Round 2, OR a side is `failed`, OR a limit hit | rulings, classification, sufficiency, range, briefing |

Round 1 simultaneity comes free (both conditions become true at once). Odin never assigns work: he writes `sufficiency: send_back, missing: [...]` and Heimdall wakes because his own condition matches. If a limit is hit, a `limit_hit` event wakes Odin, who rules on what exists and closes as `closed_incomplete`.

**Runner may:** check conditions, run agents in parallel, count and enforce limits, catch crashes and set `side_status: failed`. **Runner may not:** call an LLM, choose queries, order agents by judgement, decide sufficiency. Any `phase` field must be derived from state, never set by script.

### Case File schema (sketch)
- `case_id`, `question`, `as_of`, `created_at`, `status` (open / rejected / closed / closed_incomplete)
- `scope` (Heimdall): event_id, countries, sectors, window
- `verification` (Heimdall): verdict real / rumour / unverifiable, reasons, supporting_docs, retractions, contradictions, message to user
- `ingest_report` (Heimdall): files_ok, damaged list (type, action), bad_rows, injection_flags
- `runs[]`: run_no, evidence_index, side_status per agent, claims, attacks, defences, rulings, classification (established / contested / conceded), range per country and sector (low, high, basis), sufficiency (sufficient / send_back, missing)
- `limits`: max_runs, rounds_used, tool_calls_used, tokens_used, hit
- `events[]`: append-only audit log
- `briefing`: written by Odin from rulings only

**Claim:** claim_id, agent, run_no, round, text, country, sector, estimate (low, high, unit), kind (fact / inference), citations (ref = DOC-ID or TABLE-ID#row, tier, date, excerpt), method (tool, baseline, rows_used, rows_excluded), confidence, status.
**Attack:** attack_id, from, target_claim_id, grounds (retracted, low_tier, wrong_event, bad_method, contradicted, stale), evidence, text.
**Defence:** defence_id, attack_id, response (new_evidence or concede); `new_evidence` requires at least one NEW ref.
**Ruling:** ruling_id, claim_id, outcome (upheld / rejected / conceded / contested), basis (sourced_fact / inference), reasoning, refs_checked.

**Enforce in the schema, not prompts:** claims need valid citations (checked via `verify_citation`); attacks must target an existing claim of the other side; defences claiming new evidence need new refs; each agent writes only to its own sections; every claim must be ruled on before a briefing is allowed; briefing built only from rulings and classification; append-only sections with one writer each; private memory is not in the Case File.

**Edge cases via state:** one side fails → `side_status: failed`, briefing says the other went unchallenged; rumour → `verdict: rumour`, trial never starts, user gets a reason; nothing new on run 2 → `no_new_evidence`, Odin closes as incomplete; limits hit → incomplete label.

**Defending it to judges:** diagram with arrows only to and from the Case File; the wake-condition table; the `events` log showing agents waking from state; "remove Odin and the system just stops after Round 2." Mistakes to avoid: scripted phase field, Odin writing instructions to others, agents reading each other's memory, multiple writers per section.

## 12. Differentiation ideas (ranked)

1. Ranges computed from trade rows via tools, with the calculation clickable to exact rows; show range with and without flagged rows.
2. Evidence-quality model: tier weighting, downgrade if corrected or contradicted, follow correction chains, confidence per claim.
3. Calibrated, self-explaining ranges from Odin: narrow where Thor's and Loki's evidence converges, wide only where Contested; "why this range" breakdown; "what would shrink the range" doubles as the send-back reason.
4. Visible trap handling: badges (rumour rejected, retraction caught, injection ignored, contradiction reported), a trap scorecard, a self-built trap test pack.
5. Genuine debate: targeted attacks (tier, date, retraction, wrong event, bad calculation), evidence-driven concessions with logged reasons, "strongest attack that landed" highlight.
6. Send-back that finds truly new evidence, with a run 1 vs run 2 comparison.
7. Memory that visibly changes behavior (inspectable notes, logged moment, strategy memory).
8. Live-round robustness (drag-and-drop ingest with damage report, tested on a dummy event, graceful degradation).
9. End-to-end traceability: sentence → ruling → claim → doc or row with timestamps; one-click audit trail; automatic "all N citations verified".
10. A planner-free architecture that can be defended.

Priority if short on time: (1) ranges from trade rows, (2) visible trap handling, (3) evidence-driven concessions and targeted attacks, (4) send-back with run comparison, (5) citation audit trail. Accuracy comes from filters, flags, trade-table math and the debate loop, not from the storage format or chunking method.

## 13. Interfaces for the judges

- **Required:** web dashboard (question box with as-of date, live feed replayed in date order from Runemark documents, trial view with Thor and Loki side by side plus Odin's rulings, exposure table with low and high per country and sector, click-through to document or table row); final briefing; saved logs of a two-run case; architecture diagram; GitHub repo with README and `.env.example`.
- **Strongly recommended:** "load new documents" control (drag-and-drop or folder path) with an ingest and damage report; rulings and citation inspector (fact vs inference, green or red citation resolution); downloadable briefing (PDF or Markdown).
- **Nice to have:** run comparison view; agent activity timeline filterable by agent; memory panel highlighting the moment memory changed behavior; trap badges; limits meter.
- **Developer-facing fallbacks:** CLI (for example `run_case.py --question "..." --as-of YYYY-MM-DD`); REST/WebSocket API (also streams the live trial).
- **Browser extension:** possible only as a side panel wrapper around the same dashboard; Runemark is a local folder so there are no web pages to enhance; extra install and permission risk in a live demo; no scoring credit. Build the dashboard first and treat the extension as optional.

## 14. Suggested build order

1. Parsers (PDF/TXT/DOCX and trade tables), damage detection, live ingest path.
2. Hybrid index with metadata filters and as-of gate (plus optional reranker).
3. Heimdall: Case File and event verification.
4. Thor and Loki: claim schema, tools, private memory.
5. Cross-examination, Odin's rulings, classification, sufficiency check.
6. Send-back and run 2 with a new-evidence requirement.
7. Briefing generator with citation validation.
8. Logging and persistence per run.
9. Dashboard, README, `.env.example`, architecture diagram.

Test assets to build early: a self-made damaged pack (empty file, 40% truncated file, exact duplicate, near-duplicate, wrong-encoding file, corrupt PDF, trade table with repeated row / missing value / letter in a number field / rows after the as-of date), fake events, retractions, contradictions, injection text, a concatenated large multi-event file, and a small retrieval test set with known right documents.

---

## 15. NOT YET DISCUSSED (open items)

Decided so far: problem understanding, ingestion, retrieval (hybrid, reranker, chunking, HyDE, OKF), damaged files, trade tables, agent lookup and query formation, planner-free control flow, Case File and claim schema, differentiation, interfaces. Items below remain open. Some were touched only briefly and still need a full design.

**Core design**
1. **Event verification logic (Heimdall):** how to decide "real" vs "rumour" vs "unverifiable" (source counts by tier, handling retractions and contradictions, the rejection rule, what the user-facing explanation says).
2. **Odin's rulings and range logic:** how two sets of claims become Established / Contested / Conceded; how the final low-high range is derived and kept narrow when correct; how fact vs inference is decided.
3. **Sufficiency check and send-back rules:** what counts as "good enough", what triggers a send-back, how to guarantee run 2 finds new evidence, and how to stop loops.

**Agent internals**
4. **Memory design (full):** what each agent stores, retrieval, write rules, and how to engineer the logged "memory changed behavior" moment for acceptance check 9.
5. **Prompt injection defense (full design):** ingest-time detection, quoting at retrieval, what is logged, how it is shown.
6. **Source reliability scoring:** formula combining tier, corrections, retractions and contradictions into a weight.
7. **Agent prompts:** role prompts for all four, including how to keep Odin purely judicial.

**Engineering**
8. **Tech stack:** language, framework, vector and keyword stores, backend and frontend.
9. **Model choice and cost:** which model per agent, token and tool-call budgets.
10. **Logging and reproducibility format:** contents of a saved run folder; tracing a briefing back to its sources.
11. **Citation validation:** the automatic pre-briefing check in detail.
12. **Concurrency:** parallel Round 1 without leaking private state.

**Delivery**
13. **Dashboard layout and live feed replay.**
14. **Testing and evaluation plan:** self-built traps and a scoring harness for range accuracy and retrieval.
15. **Demo script and architecture-diagram walkthrough for the judges.**
16. **Team split, timeline and milestones** (needs: team size and time until offline and final rounds).
17. **Final-round playbook:** first minutes after the five new packs arrive; fallback plan if something breaks.
18. **Repo deliverables:** README, `.env.example`, the saved two-run case.
