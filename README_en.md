# llm-wiki

A Claude Code skill for building and maintaining a personal LLM-powered knowledge wiki.

The LLM reads your raw documents, builds a structured markdown wiki, and keeps it current — instead of re-deriving knowledge from scratch on every query (RAG-style).

Based on Karpathy's [llm-wiki.md](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) pattern.

## Architecture

Three layers:

```
raw/          # source documents (immutable — drop files here)
wiki/         # LLM-generated and maintained markdown pages
purpose.md    # why this wiki exists
CLAUDE.md     # schema and workflows
```

`wiki/` is split into fixed buckets to keep the namespace tight:

- `entities/` — people, teams, organizations, products
- `concepts/` — patterns, methods, technical terms, theories
- `sources/` — one summary page per ingested source
- `synthesis/` — cross-source pages produced from queries
- `synthesis/daily/` — one digest page per day
- `index.md`, `overview.md`, `log.md` — catalog, snapshot, history

Every page carries `sources: [raw/...]` in its frontmatter. This is what makes the source-overlap query signal and the delete cascade work.

## Subcommands

Invoke as `/llm-wiki <subcommand> [args]`:

| Subcommand | Purpose |
|---|---|
| `build [scenario]` | Bootstrap wiki structure in the current directory |
| `ingest [path]` | Read a source in `raw/` and integrate it into the wiki (processes all of `raw/` when path is omitted) |
| `query <question>` | Answer from the wiki, with citations |
| `daily [date]` | Compress a day's raw inputs into a digest |
| `lint` | Health-check for contradictions, orphans, and gaps |
| `delete <path>` | Remove a source and cascade-clean the wiki |

## Quick start

1. In the directory where you want the wiki, run `/llm-wiki build`. Pick a scenario (research, reading, personal, business, general) and fill in `purpose.md`.
2. Drop one or more source files into `raw/`.
3. Run `/llm-wiki ingest` — it walks every file under `raw/` and integrates them into the wiki. To process a single file, use `/llm-wiki ingest raw/<file>`. Files already ingested (matched by SHA256 hash) are skipped automatically; each remaining file runs through analysis → 3-5 takeaway discussion → write.
4. Ask questions with `/llm-wiki query <question>`. Answers cite wiki pages and raw files.
5. At end of day, run `/llm-wiki daily` to compress the day's raw into one digest.
6. Periodically run `/llm-wiki lint` to surface contradictions, orphan pages, and coverage gaps.

## Design principles

- **Human curates, LLM maintains.** You pick sources and ask questions; the LLM handles organization and linking.
- **Raw is immutable.** `raw/` is read-only. `delete` only cleans wiki pages — it never touches raw files.
- **Two-step ingest.** Analyze first, discuss takeaways, then write. The pause between them is where contradictions get caught.
- **Cite everything.** Every wiki claim traces back to either a raw source or another wiki page.
- **Compound, don't re-derive.** File good answers back into the wiki as synthesis pages.
- **Hash before re-reading.** SHA256 on raw files; skip unchanged sources rather than re-running them through the LLM.
- **No premature tooling.** At small scale (<200 pages) an index file plus four relevance signals is enough. Don't introduce vector search until you hit a real limit.

## Query relevance (markdown-only)

`query` expands its candidate set with four signals — no embeddings required:

| Signal | Weight |
|---|---|
| Source overlap (shared `sources[]`) | ×4.0 |
| Direct `[[wikilink]]` | ×3.0 |
| Common neighbors (Adamic-Adar) | ×1.5 |
| Type affinity | ×1.0 |

## Compatibility

The `wiki/` directory is plain markdown with Obsidian-compatible `[[wikilinks]]` and YAML frontmatter. Open it in Obsidian for graph view, or treat it as a git repo and commit after each ingest.

## See also

- Full skill spec: [`.claude/llm-wiki/SKILL.md`](.claude/llm-wiki/SKILL.md)
- Other languages: [한국어](README.md) / [日本語](README_jp.md)
- Original gist: [karpathy/llm-wiki.md](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)
