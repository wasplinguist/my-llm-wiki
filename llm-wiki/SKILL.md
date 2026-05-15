---
name: llm-wiki
description: Build and maintain a personal LLM-powered knowledge wiki using Karpathy's pattern. Use when the user invokes `/llm-wiki <subcommand>` (build, ingest, query, daily, lint, delete), or asks to bootstrap, ingest sources into, query, summarize a day for, or health-check a personal wiki / second brain / research notes / Obsidian-compatible knowledge base built from documents.
---

# LLM Wiki

A pattern for building a persistent, self-maintaining knowledge base from documents. The LLM reads sources, builds a structured markdown wiki, and keeps it current — instead of re-deriving knowledge from scratch on every query (RAG).

Based on Karpathy's [llm-wiki.md](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f).

## Subcommands

The skill is invoked as `/llm-wiki <subcommand> [args]`:

| Subcommand | Purpose |
|---|---|
| `build [scenario]` | Bootstrap wiki structure in current directory |
| `ingest <path>` | Read a new source and integrate it into the wiki |
| `query <question>` | Answer using wiki content, with citations |
| `daily [date]` | Build a daily synthesis from a day's raw inputs |
| `lint` | Health-check the wiki for contradictions, orphans, gaps |
| `delete <path>` | Remove a raw source and cascade-clean the wiki |

If the user types `/llm-wiki` with no subcommand, ask which they want.

---

## `build` — Bootstrap

Sets up the three-layer architecture in the current working directory.

**Steps:**

1. **Confirm directory.** Ask: "Bootstrap a wiki in `<pwd>`? Or somewhere else?"
2. **Pick a scenario** (if not passed as arg). Offer:
   - `research` — going deep on a topic over weeks
   - `reading` — companion wiki for a book or course
   - `personal` — journals, health, goals, self-knowledge
   - `business` — team/internal knowledge
   - `general` — generic / unspecified
3. **Create the skeleton:**
   ```
   raw/                    # immutable source documents (you drop files here)
   raw/assets/             # downloaded images
   wiki/                   # LLM-generated pages
   wiki/index.md           # catalog of all wiki pages
   wiki/overview.md        # global summary of current wiki state (auto-updated)
   wiki/log.md             # chronological operation log
   wiki/entities/          # people, teams, organizations, products
   wiki/concepts/          # patterns, methods, technical terms, theories
   wiki/sources/           # one summary page per ingested raw source
   wiki/work-items/        # Jira tickets, GitHub issues/PRs, other tracked work
   wiki/synthesis/         # synthesized pages from queries
   wiki/synthesis/daily/   # one digest page per day
   purpose.md              # WHY this wiki exists — goals, key questions, thesis
   CLAUDE.md               # the schema — conventions and workflows
   ```
   Pre-create the empty `entities/`, `concepts/`, `sources/`, and `work-items/` directories at build time. Fixing the namespace upfront prevents ingest from inventing buckets ad hoc (e.g. filing Jira tickets under `entities/`).
4. **Populate `purpose.md`.** Have a short conversation to fill in: domain, primary questions, scope, success criteria. Keep it 1-2 screens of markdown.
5. **Populate `CLAUDE.md`** with the schema (see template below).
6. **Seed `index.md`** as an empty catalog with sections (Entities, Concepts, Sources, Work Items, Synthesis, Daily).
7. **Seed `overview.md`** with a placeholder section structure: `## Current Scope`, `## Major Threads`, `## Open Questions`. These will be filled as ingests happen.
8. **Seed `log.md`** with a `## [YYYY-MM-DD] init` entry.
9. **Tell the user the next step:** drop a source into `raw/` and run `/llm-wiki ingest <path>`.

### `CLAUDE.md` schema template

```markdown
# Wiki Schema

## Layers
- **raw/** — immutable source documents. Read only. Never modify.
- **wiki/** — LLM-maintained markdown pages. Read and write.
- **purpose.md** — why this wiki exists. Read every ingest/query/daily.
- **CLAUDE.md** — this file. Conventions and workflows.

## Page conventions
- Every page has YAML frontmatter:
  - `title`
  - `type` (entity|concept|source|work-item|synthesis)
  - `created`
  - `updated`
  - `sources` — list of `raw/...` paths that contributed to this page. **Required on every page, not just source pages.** Entity and concept pages accumulate this list across ingests.
  - For source pages only: `sha256` — content hash of the raw file at ingest time, used to skip unchanged re-ingests.
- Cross-references use `[[wikilink]]` syntax (Obsidian compatible).
- One page per entity / concept / source / work-item. Don't merge unrelated topics.
- Page filenames are kebab-case: `wiki/concepts/vector-embeddings.md`.
- Daily digests live under `wiki/synthesis/daily/<YYYY-MM-DD>.md`, kept separate from durable synthesis pages.

## Page taxonomy (which bucket does this go in?)
- **`wiki/entities/`** — proper nouns with continuity: people, teams, organizations, products, services. Filename = canonical name slug (e.g. `payment-platform-team.md`).
- **`wiki/concepts/`** — reusable ideas: patterns, methods, architectures, technical terms, theories. Filename = the concept (e.g. `circuit-breaker.md`).
- **`wiki/sources/`** — one summary page per ingested raw file. Filename = source slug (e.g. `2026-05-02-design-doc.md`).
- **`wiki/work-items/`** — Jira tickets, GitHub issues/PRs, support tickets — anything with a tracker ID and a lifecycle. Filename = the ID, lowercased (e.g. `pay-1234.md`, `gh-567.md`). Link the people who own them via `[[entity]]` and the concepts they touch via `[[concept]]`.
- **`wiki/synthesis/`** — cross-source pages produced from `query` or promoted decisions.

A Jira ticket is **not** an entity. An entity is a thing that persists across many tickets. Conflating them collapses the graph.

## Ingest workflow (two-step chain-of-thought)
1. Read the source in raw/. Compute its SHA256 hash; if a `wiki/sources/<slug>.md` already exists with the same `sha256`, skip with a notice.
2. **Step 1 — Analysis (no writes yet).** Produce a structured analysis note in chat:
   - Key entities, concepts, arguments, claims
   - Mapping of each item to its target bucket (entity / concept / work-item)
   - Connections to existing wiki pages (read `wiki/index.md` and relevant pages)
   - **Contradictions & tensions** with existing claims — explicit list
   - Recommended new pages vs. updates to existing pages
   - Folder-path hint: treat the raw file's parent directory (e.g. `raw/papers/energy/`) as a user-provided classification signal
3. Discuss key takeaways with the user (3-5 bullets). User can steer emphasis before writes.
4. **Step 2 — Generation.** Using the analysis as input, write/update pages:
   - `wiki/sources/<source-slug>.md` (summary + key claims + citations + `sha256` + `sources: [raw/...]`)
   - Each touched entity / concept / work-item page, with `sources[]` extended to include this raw file
   - Cross-references in both directions
   - Contradictions surfaced as `> [!warning] Contradicts ...` callouts on the older page (never overwrite silently)
   - Pre-generated web search queries for any item flagged as needing external research, stored inline in the source page under `## Review queue`
5. Update `wiki/index.md` with new entries.
6. Update `wiki/overview.md`: refresh the global summary (current scope, major threads, open questions) based on the new state. Keep it short — one screen.
7. Append `## [YYYY-MM-DD] ingest | <source title>` to `wiki/log.md`.

## Query workflow (4-signal candidate selection)
1. Read `wiki/index.md` and `wiki/overview.md` first to find candidate pages.
2. Expand the candidate set using four signals (markdown-only, no vector search needed):
   - **Direct link (×3.0)** — pages linked via `[[wikilink]]` from any candidate
   - **Source overlap (×4.0)** — pages whose frontmatter `sources[]` overlaps with a candidate's `sources[]`
   - **Common neighbors (×1.5)** — pages that share linked pages with a candidate (Adamic-Adar style)
   - **Type affinity (×1.0)** — small bonus for matching the question's expected type (entity question → entity pages, etc.)
3. Read the top candidate pages (usually 3-8).
4. Synthesize an answer with `[[wikilink]]` citations.
5. Offer to file the answer back into the wiki as a synthesis page.

## Daily workflow
1. Collect every raw file matching the date.
2. Classify and dedup events that appear in multiple sources.
3. Discuss takeaways with the user (3-5 bullets).
4. Apply digest rules: extract decisions, in-progress, open items, people, external context.
5. Write `wiki/synthesis/daily/<date>.md` with structured frontmatter `events`.
6. Update `wiki/index.md` (`## Daily`) and `wiki/log.md`.
7. Optionally propose promoting decisions to durable synthesis pages.

## Delete workflow
1. Identify the raw file to remove.
2. Find the source page (`wiki/sources/<slug>.md`) and all pages whose `sources[]` includes this raw file.
3. For each touched page:
   - If `sources[]` becomes empty after removal → delete the page.
   - Otherwise → remove only the entry from `sources[]`, page itself is preserved.
4. Strip dead `[[wikilink]]` references to deleted pages from all remaining pages.
5. Clean up `wiki/index.md`.
6. Refresh `wiki/overview.md`.
7. Append `## [YYYY-MM-DD] delete | <source title> | removed: N pages, trimmed: M pages` to `wiki/log.md`.

## Lint workflow
Look for: contradictions across pages, stale claims, orphan pages,
missing concept pages, missing cross-references, dead wikilinks,
isolated nodes, sparse clusters, bridge nodes, data gaps.
Suggest new questions or sources rather than auto-fixing.
```

---

## `ingest <path>` — Process a source

`<path>` points to a file in `raw/` (or to be moved into `raw/`).

**Steps:**

1. If the path is outside `raw/`, ask whether to copy or move it in. Sources must live in `raw/` to be tracked.
2. **Hash check.** Compute SHA256 of the raw file (`shasum -a 256 <path>` or equivalent). If `wiki/sources/<slug>.md` already exists with the same `sha256` in its frontmatter, report "unchanged, skipping" and stop. The user can pass `--force` to override.
3. Read the source. For PDFs, extract text. For images, describe them factually.
4. **Step 1 — Analysis (no writes).** Read `purpose.md` and `wiki/index.md` (and `wiki/overview.md` if it exists), then produce a structured analysis note in chat covering:
   - **Key items:** entities, concepts, arguments, claims, work-item IDs
   - **Bucket routing:** for each item, which folder it belongs in (entity / concept / work-item)
   - **Existing connections:** which current wiki pages relate, via what
   - **Contradictions & tensions:** explicit list of claims that disagree with existing pages
   - **Page plan:** which pages to create new, which to update, which `sources[]` arrays to extend
   - **Folder hint:** note the raw file's parent directory as a classification signal (e.g. `raw/papers/energy/foo.pdf` → bias toward energy-related concept routing)
5. **Discuss key takeaways with the user (3-5 bullets).** Don't skip — this is the steering point.
6. **Step 2 — Generation.** Write `wiki/sources/<slug>.md` with:
   - frontmatter: `type: source`, `title`, `created`, `updated`, `sources: [raw/...]`, `sha256: <hash>`
   - 3-paragraph summary
   - Key claims (each with citation back to raw section/page)
   - `## Review queue` section listing items flagged for human judgment, each with a pre-generated web search query suggestion
7. For each item touched: create or update its page in the right bucket. Aim for 5-15 pages touched per ingest. Routing rules:
   - **Jira ticket / GitHub issue / GitHub PR / support ticket** → `wiki/work-items/<id>.md`
   - **Person, team, organization, product, service** → `wiki/entities/<slug>.md`
   - **Pattern, method, architecture, technical term, theory** → `wiki/concepts/<slug>.md`
   - **Source summary** → `wiki/sources/<slug>.md` (covered in step 6)

   Every page (not just source pages) must carry `sources: []` in frontmatter. When updating an existing entity/concept page, append this raw path to its `sources[]` if not already present. This is what makes `delete` and the source-overlap query signal work.

   When in doubt: does this thing have a tracker ID and a lifecycle (open/closed)? → work-item. Does it persist across many tickets/sources? → entity or concept.
8. Add `[[wikilink]]` cross-references in both directions.
9. If a new claim contradicts an existing one (already surfaced in step 4 analysis), add a `> [!warning]` callout on the older page pointing to the new source. Do NOT silently overwrite.
10. Update `wiki/index.md`.
11. **Update `wiki/overview.md`.** Refresh `## Current Scope`, `## Major Threads`, `## Open Questions` to reflect the new state. Keep it ≤1 screen — overview is a snapshot, not an accumulator.
12. Append to `wiki/log.md`: `## [YYYY-MM-DD] ingest | <title> | touched: N pages`.
13. Report back: which pages were created, which updated, any contradictions flagged, any review-queue items with pre-generated search queries.

**Stay involved.** Prefer one source at a time over batch ingestion unless the user explicitly asks for batch.

---

## `query <question>` — Answer from the wiki

**Steps:**

1. Read `purpose.md` and `wiki/overview.md` for domain context.
2. Read `wiki/index.md` to identify initial candidate pages by keyword/title match.
3. **Expand candidates using the 4-signal relevance model** (markdown-only, no vector search):
   - **Direct link (weight ×3.0):** pages reached by following `[[wikilink]]` from a candidate
   - **Source overlap (weight ×4.0):** pages whose frontmatter `sources[]` overlaps with a candidate's `sources[]` — this is why every page must carry `sources[]`
   - **Common neighbors (weight ×1.5):** pages that share linked pages with a candidate (Adamic-Adar approximation)
   - **Type affinity (weight ×1.0):** small bonus for matching the question's expected type
4. Read the top 3-8 candidate pages. Don't read all of `raw/`.
5. Synthesize an answer:
   - Use `[[wikilink]]` for citations to wiki pages.
   - Use file paths for citations to raw sources.
   - Flag any uncertainty or contradiction explicitly.
6. **Offer to file the answer back into the wiki** as `wiki/synthesis/<topic>.md`. Good answers shouldn't disappear into chat.
7. If filed: update `index.md` and `log.md` with `## [YYYY-MM-DD] query | <topic>`. Also refresh `wiki/overview.md` if the synthesis changes the global picture.

**Output formats** depending on question shape: prose answer, comparison table, timeline, or a slide outline (Marp). Don't force prose if a table is clearer.

---

## `daily [date]` — Daily digest

Compress a day's events scattered across multiple raw sources (Slack, Jira, Confluence, GitHub, Claude Code transcripts, etc.) into a single synthesis page. The goal is **decisions, events, and open items** — not an activity log.

`<date>` is ISO format (YYYY-MM-DD). Defaults to yesterday. Aliases: `yesterday`, `today`.

**Preconditions:**

- `purpose.md` and `wiki/index.md` must exist (`build` has been run).
- At least one raw file for that date must exist somewhere under `raw/`.
- Entities and sources mentioned that day **should already be in the wiki via `ingest`** for `daily` to produce rich cross-links. Running `daily` without prior `ingest` produces flat text with empty links — `daily` does **not** replace `ingest`.

**Steps:**

1. **Collect the day's raw.** Find every file under `raw/` matching the date (KST). Matching rules:
   - Filename: `<date>.jsonl`, `<date>.md`, `<date>-*.{md,jsonl}`
   - JSONL contents: `ts` field falls within the KST day window — UTC `<date-1>T15:00:00Z` ≤ `ts` < `<date>T15:00:00Z`. A naive prefix match on `<date>T...` will miss KST 00:00–09:00.
   - Single-file docs (e.g. meeting notes): frontmatter `created` matches (also KST).
2. **Classify and dedup.** When the same event appears in multiple sources (e.g. a PR notification in both `github` and `slack`), collapse to one. **Dedup is reported, not auto-applied** — ask the user how to merge. Never silently fold duplicates.
3. **Discuss takeaways with the user (3-5 bullets).** Surface the day's core before writing. *This step is mandatory*, just as in `ingest`. It prevents automated digest spam and gives the user a brief daily review ritual.
4. Read `purpose.md` to know what to emphasize in the wiki's lens.
5. Read `wiki/index.md` to check whether entities and concepts mentioned that day already have pages. For missing pages, still emit `[[wikilink]]` (a "red link") and include a "needs ingest" note in the report.
6. **Apply digest rules.** Extract content into these buckets:
   - **Decisions** — patterns like "decided / agreed / chose / handover / changed", plus explicit decision statements in meeting notes.
   - **In progress** — open PRs, In Progress tickets, drafts.
   - **Open items** — "next / TBD / pending / unanswered" patterns.
   - **People involved** — only people whose actions mattered that day. Pure notification senders are not "involved".
   - **External context** — outside dependencies, vendor updates, events that shaped the day.
   - **Chronological events** — structured into frontmatter (for visualization).

   Drop noise: simple confirmations, 1-2 turn quick questions, repeated phrasings.
7. **Write the digest** at `wiki/synthesis/daily/<date>.md`. Required structure:
   - frontmatter: `title`, `type: synthesis`, `date`, `created`, `sources`, `events`
   - `# Daily — <date> (<weekday>)`
   - `## TL;DR` (one sentence with `[[wikilinks]]`)
   - `## Decisions`
   - `## In progress`
   - `## Open items`
   - `## People involved`
   - `## Tomorrow` (filled by the user during review; may be empty)

   Every claim must carry a `[[wikilink]]` or a raw-file path citation.
8. Update `wiki/index.md`. Add `[[daily/<date>]]` to the `## Daily` section. If `## Daily` doesn't exist, create it under `## Synthesis`.
9. Append to `wiki/log.md`: `## [<date>] daily | decisions: N, open: M, sources: K`.
10. **Promotion prompt (optional).** Ask the user whether any decision in the digest deserves its own permanent synthesis page. Example: "Would you like to promote the 'circuit breaker rationale' decision to `wiki/synthesis/circuit-breaker-rationale.md`?" Promoted pages outlive the daily; declined ones live only in the digest.

**Output — what to report back:**

The digest path, plus always include:
- **Red links (expected)** — entities or concepts the digest references that lack pages. Candidates the user should `ingest` next. This is normal, not a lint error.
- **Promotion proposals** — from step 10, if any.

**Often paired with:**

- **If `ingest` hasn't run for the day:** `daily` can pull directly from raw, but cross-link quality drops. After `daily`, recommend that the user `ingest` the most important sources from that day — `daily` is a *summary*, `ingest` is *durable encoding*. They are not interchangeable.
- **Weekly retros:** collecting the last seven `daily` pages into a single weekly synthesis is a natural follow-up. This may become a separate `weekly` subcommand or stay as a `query` use case — defer the decision until usage patterns emerge.

---

## `delete <path>` — Cascade-clean a removed source

Remove a raw source and clean up the wiki without losing shared knowledge. Use when a source is wrong, outdated, or was ingested by mistake.

`<path>` points to a file inside `raw/`.

**Steps:**

1. **Confirm.** Show the file path and ask: "Delete this source and cascade-clean? (y/n)".
2. **Find affected pages.** Locate:
   - `wiki/sources/<slug>.md` matching this raw file
   - Every wiki page (entity/concept/work-item/synthesis) whose frontmatter `sources[]` includes this raw path
3. **For each affected page, decide:**
   - **Delete the page** if removing this entry would leave `sources[]` empty AND the page has no inbound `[[wikilinks]]` from other still-present pages. The page existed only because of this source.
   - **Trim** if `sources[]` still has other entries after removal, OR if other pages link to it. Remove just the one entry from `sources[]`; keep the page.
4. **Clean dead wikilinks.** For every page deleted in step 3, grep the wiki for `[[<deleted-page>]]` references and strip them (or convert to plain text — ask the user).
5. **Update `wiki/index.md`** — remove entries for deleted pages.
6. **Refresh `wiki/overview.md`** — the global picture may have changed.
7. **Append to `wiki/log.md`:** `## [YYYY-MM-DD] delete | <source title> | removed: N pages, trimmed: M pages`.
8. **Do not delete the raw file automatically.** Report the cleanup result and let the user remove the raw file themselves with `rm` (or `git rm`). Raw is sacred; the skill never writes to it.

**Report back:** which pages were deleted, which were trimmed, which wikilinks were cleaned.

---

## `lint` — Health check

Read `wiki/index.md` and `wiki/overview.md`, then sample pages (or read all if small wiki). Report findings, don't auto-fix.

**Checks:**

- **Contradictions** — claims on different pages that disagree. Flag both pages.
- **Stale claims** — a page asserts X, a newer source asserts not-X. The page should at least acknowledge the disagreement.
- **Orphan pages** — pages with no inbound `[[wikilink]]`. Either link them or consider deletion.
- **Isolated nodes** — pages with total degree ≤ 1 (inbound + outbound combined). Stronger than orphan; these floats outside the graph entirely.
- **Bridge nodes** — pages that link to (or are linked from) pages spanning 3+ different top-level wiki folders. These are critical junctions; flag them so the user knows they carry load.
- **Sparse clusters** — within a single folder (e.g. `wiki/concepts/`), groups of pages with very few internal `[[wikilinks]]` between them. Indicates a thin area where cross-references are missing.
- **Missing concept pages** — a term mentioned ≥3 times across pages but has no page of its own.
- **Missing cross-references** — page A mentions entity B but doesn't link to `[[B]]`.
- **Coverage gaps** — purpose.md says we care about X, but no source covers it. Suggest sources or web searches.
- **Schema drift** — pages missing frontmatter, inconsistent slugs, **dead `[[wikilinks]]`** (links to pages that no longer exist — different from `daily` red links, which are intentional pointers to not-yet-ingested entities).
- **Missing `sources[]`** — any page (other than `index.md`, `log.md`, `overview.md`) without a `sources[]` frontmatter field. This breaks the source-overlap query signal and the `delete` cascade.

**Output:** a short report grouped by severity, plus a suggested action list. Append a lint entry to `wiki/log.md`.

---

## Time zone

All `<date>` arguments, `wiki/log.md` headers, page frontmatter `created` / `updated` fields, and `wiki/synthesis/daily/<date>.md` filenames are **KST (UTC+9)** dates.

When matching raw files by an internal `ts` field (which may be UTC), convert: a KST date `2026-05-12` covers UTC `2026-05-11T15:00:00Z` ≤ `ts` < `2026-05-12T15:00:00Z`. The `daily` step 1 matcher must apply this window — a naive `ts.startswith("2026-05-12")` will silently miss the first 9 hours of the KST day.

## Operating principles

- **Human curates, LLM maintains.** The user picks sources and asks questions. The LLM does all bookkeeping.
- **Raw sources are immutable.** Never write to `raw/`. Read-only. `delete` removes wiki pages, not raw files.
- **Don't silently lose information.** When a new source contradicts an old claim, surface it — don't overwrite.
- **Cite everything.** Every wiki claim should trace back to either a raw source or another wiki page. Every page carries `sources[]`.
- **Compound, don't re-derive.** Filing a good answer back into the wiki is better than re-answering the same question next month.
- **Compression beats accumulation.** Raw preserves everything; synthesis must drop most of it. A digest that keeps every detail isn't a digest. `overview.md` is a snapshot, not an archive.
- **The wiki has two clocks.** `ingest` runs on the source's clock (whenever a source arrives). `daily` runs on the calendar's clock (one per day). Don't confuse them.
- **Two-step ingest beats one-step.** Analyze first, write second. The pause between them is where contradictions get caught and routing gets right.
- **Hash before re-reading.** SHA256 on the raw file is cheap; re-running an unchanged source through the LLM is not.
- **No premature tooling.** At small scale (<200 pages) the index file plus the 4 relevance signals are enough; don't introduce vector search or extra tooling unless the user hits a real limit.

## Tips

- The wiki is just a git repo of markdown. Commit after each ingest if version history matters.
- The `wiki/` directory is Obsidian-compatible — graph view is the best way to see the shape.
- For images embedded in PDFs or web pages, save them under `raw/assets/` and reference by relative path.
- Log entries with consistent prefixes (`## [YYYY-MM-DD] ingest | ...`, `## [YYYY-MM-DD] daily | ...`, `## [YYYY-MM-DD] delete | ...`) are grep-friendly: `grep "^## \[" wiki/log.md | tail -10`.
- A digest's frontmatter `events` array doubles as input for timeline visualizations — keep it well-structured even when the body is short.
- The raw file's parent folder is free classification signal — treat `raw/papers/energy/foo.pdf` differently from `raw/slack/foo.md` during ingest.

## What NOT to do

- Don't dump source text verbatim into wiki pages — extract and synthesize.
- Don't create pages for trivial mentions (one-off references).
- Don't merge unrelated entities into one page to "reduce clutter."
- Don't run `lint` as part of `ingest` — they're separate operations for a reason.
- Don't rewrite the schema (`CLAUDE.md`) without asking — it's co-evolved with the user.
- Don't auto-generate `daily` on a cron — it runs only when explicitly invoked, so the user stays in the loop.
- Don't run `daily` on a day with no raw — produce a "no inputs" notice instead of fabricating events.
- Don't merge daily digests across days into one page — each day stays its own page; cross-day synthesis is a `query` job.
- Don't auto-promote daily decisions to durable synthesis pages — promotion is a human judgment.
- Don't skip the analysis step in `ingest` to "save a call" — the two-step CoT is what catches contradictions and prevents bucket misrouting.
- Don't delete the raw file inside `delete` — the skill only cleans the wiki. Raw deletion is a separate, deliberate user action.
- Don't let `sources[]` go missing on a generated page — that field is load-bearing for both `query` (source overlap signal) and `delete` (cascade logic).