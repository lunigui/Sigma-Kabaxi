# LLM Wiki Agent — Schema & Workflow Instructions

This wiki is maintained entirely by Claude Code. No API key or Python scripts needed — just open this repo in Claude Code and talk to it.

## Slash Commands (Claude Code)

| Command | What to say |
|---|---|
| `/wiki-ingest` | `ingest raw/my-article.md` |
| `/wiki-query` | `query: what are the main themes?` |
| `/wiki-health` | `health` (fast, every session) |
| `/wiki-lint` | `lint the wiki` (expensive, periodic) |
| `/wiki-graph` | `build the knowledge graph` |

Or just describe what you want in plain English:
- *"Ingest this file: raw/papers/attention-is-all-you-need.md"*
- *"What does the wiki say about transformer models?"*
- *"Check the wiki for orphan pages and contradictions"*
- *"Build the graph and show me what's connected to RAG"*

Claude Code reads this file automatically and follows the workflows below.

---

## Directory Layout

```
raw/          # Immutable source documents — never modify these
wiki/         # Claude owns this layer entirely
  index.md    # Catalog of all pages — update on every ingest
  log.md      # Append-only chronological record
  overview.md # Living synthesis across all sources
  sources/    # One summary page per source document
  entities/   # People, companies, projects, products
  concepts/   # Ideas, frameworks, methods, theories
  syntheses/  # Saved query answers
graph/        # Auto-generated graph data
tools/        # Standalone Python scripts
  health.py   # Structural checks (deterministic, no LLM calls)
  lint.py     # Content quality checks (uses LLM for semantic analysis)
  build_graph.py  # Knowledge graph generation
```

---

## Page Format

Every wiki page uses this frontmatter:

```yaml
---
title: "Page Title"
type: source | entity | concept | synthesis
tags: []
sources: []       # list of source slugs that inform this page
last_updated: YYYY-MM-DD
# concept pages only:
concept_kind: simple | teaching
prerequisites: [] # teaching concepts: ["[[PrereqOne]]"] in learning order
---
```

Use `[[PageName]]` wikilinks to link to other wiki pages.

---

## Ingest Workflow

Triggered by: *"ingest <file>"* or `/wiki-ingest`

**Supported formats:** Markdown (`.md`) ingested directly. Non-markdown files (`.pdf`, `.docx`, `.pptx`, `.xlsx`, `.html`, `.txt`, `.csv`, `.json`, `.xml`, `.rst`, `.rtf`, `.epub`, `.ipynb`, `.yaml`, `.yml`, `.tsv`, `.wav`, `.mp3`) auto-converted to markdown via [markitdown](https://github.com/microsoft/markitdown) before ingestion. Use `--no-convert` to skip auto-conversion.

Steps (in order):
1. Read the source document fully using the Read tool (auto-convert if non-markdown)
2. Read `wiki/index.md` and `wiki/overview.md` for current wiki context
3. Write `wiki/sources/<slug>.md` — use the source page format below
4. Update `wiki/index.md` — add entry under Sources section
5. Update `wiki/overview.md` — revise synthesis if warranted; keep the **Learning Paths** section current (see Concept Classification)
6. Update/create entity pages for key people, companies, projects mentioned
7. Update/create concept pages for key ideas and frameworks discussed — classify each concept as `simple` or `teaching` per the Concept Classification rules and use the matching template; re-check existing simple concepts touched by this source for promotion triggers
8. Flag any contradictions with existing wiki content
9. Append to `wiki/log.md`: `## [YYYY-MM-DD] ingest | <Title>`
10. **Post-ingest validation** — check for broken `[[wikilinks]]`, verify all new pages are in `index.md`, print a change summary

### Source Page Format

```markdown
---
title: "Source Title"
type: source
tags: []
date: YYYY-MM-DD
source_file: raw/...
---

## Summary
2–4 sentence summary.

## Key Claims
- Claim 1
- Claim 2

## Key Quotes
> "Quote here" — context

## Connections
- [[EntityName]] — how they relate
- [[ConceptName]] — how it connects

## Contradictions
- Contradicts [[OtherPage]] on: ...
```

### Domain-Specific Templates

If the source falls into a specific domain (e.g., personal diary, meeting notes), the agent should use a specialized template instead of the default generic one above:

#### Diary / Journal Template
```markdown
---
title: "YYYY-MM-DD Diary"
type: source
tags: [diary]
date: YYYY-MM-DD
---
## Event Summary
...
## Key Decisions
...
## Energy & Mood
...
## Connections
...
## Shifts & Contradictions
...
```

#### Meeting Notes Template
```markdown
---
title: "Meeting Title"
type: source
tags: [meeting]
date: YYYY-MM-DD
---
## Goal
...
## Key Discussions
...
## Decisions Made
...
## Action Items
...
```

---

## Concept Classification: Simple vs Teaching

Concept pages come in two kinds, set in frontmatter via `concept_kind`. They serve different consumers: **simple** concepts keep the knowledge graph connected; **teaching** concepts double as lesson plans for the Sigma tutor skill (`.agents/skills/sigma/`), which turns them into one-on-one tutoring sessions.

### Which kind? (decide at ingest, per concept)

Classify as **teaching** when most of these hold:
- The source dedicates substantial space to it (a section or more), not a passing mention
- It has prerequisites — understanding it requires other concepts first
- The source provides teachable material: definitions, worked examples, contrasts, common errors
- It is a learning goal of the source (something a reader would want to *master*), not just vocabulary

Classify as **simple** when:
- It is mentioned as support/background for other ideas and one paragraph can capture it
- The source gives no examples, contrasts, or failure modes to teach from
- Its main job is being a node other pages link to

**When unsure, classify as simple.** Simple → teaching promotion is cheap and expected; a bloated teaching page for a trivial term is waste.

### Simple Concept Template

```markdown
---
title: "Concept Title"
type: concept
concept_kind: simple
tags: []
sources: []
last_updated: YYYY-MM-DD
---

## Definition
One paragraph. What it is, precisely.

## Connections
- [[OtherConcept]] — how it relates

## Sources
- [[source-slug]] — what this source contributed
```

### Teaching Concept Template

A teaching concept is a **lesson plan, not a textbook chapter**. Keep each section tight; its consumer is the Sigma tutor, which teaches by questioning, never by lecturing. Sections map 1:1 onto Sigma's tutor loop (diagnose → question → mastery check → practice).

```markdown
---
title: "Concept Title"
type: concept
concept_kind: teaching
tags: []
sources: []
prerequisites: ["[[PrereqOne]]", "[[PrereqTwo]]"]
last_updated: YYYY-MM-DD
---

## Definition
One paragraph. Precise and self-contained.

## Why It Exists
The problem it solves / what breaks without it. Fuel for the opening question.

## Key Ideas
- **Idea 1** — stated concretely enough to be turned into a question
- **Idea 2** — ...
(2–5 items)

## Prerequisites
- [[PrereqOne]] — which aspects must already be mastered
(may be "None" only for genuine entry points)

## Common Misconceptions
| Misconception | Root cause | Counter-example |
|---|---|---|
| "wrong belief" | why it feels right | scenario where it predicts the wrong outcome |

## Diagnostic Questions
1. Question probing whether the learner already knows this
2. ...

## Mastery Check Questions
1. Question (tests: Accurate + Explained)
2. Question on a novel scenario (tests: Novel application)
3. Question contrasting with [[SimilarConcept]] (tests: Discrimination)
(≥3 questions covering all four rubric criteria: Accurate, Explained, Novel application, Discrimination)

## Practice Tasks
- A 2–5 minute task with a checkable output

## Connections
- [[RelatedConcept]] — how it relates

## Sources
- [[source-slug]] — what this source contributed
```

### Promotion: Simple → Teaching

Re-check the classification of **every existing simple concept the current source substantively touches**. Promote when any trigger fires:
- The new source discusses it in depth (definition + examples/contrasts) and it now has 2+ such sources
- It appears in overview's Learning Paths as a step a learner must master
- The user asks for it explicitly (*"promote X to a teaching concept"*)

Promotion steps:
1. Flip `concept_kind` to `teaching` and add `prerequisites`
2. Expand the page to the teaching template, drawing material from **all** sources that discuss it
3. Move the entry to the **Teaching** subsection of Concepts in `wiki/index.md`
4. Append to `wiki/log.md`: `## [YYYY-MM-DD] promote | <Concept Title>`

### Overview: Learning Paths

`wiki/overview.md` carries a `## Learning Paths` section — teaching concepts ordered by dependency (a topological read of `prerequisites`), grouped per coherent path:

```markdown
## Learning Paths
1. [[ConceptA]] → [[ConceptB]] → [[ConceptC]]
2. [[ConceptD]] → [[ConceptE]]
```

Update it whenever teaching concepts or their prerequisites change. The Sigma tutor reads this section to build learning roadmaps.

---

## Query Workflow

Triggered by: *"query: <question>"* or `/wiki-query`

Steps:
1. Read `wiki/index.md` to identify relevant pages
2. Read those pages with the Read tool
3. Synthesize an answer with inline citations as `[[PageName]]` wikilinks
4. Ask the user if they want the answer filed as `wiki/syntheses/<slug>.md`

---

## Lint Workflow

Triggered by: *"lint the wiki"* or `/wiki-lint`

Use Grep and Read tools to check for:
- **Orphan pages** — wiki pages with no inbound `[[links]]` from other pages
- **Broken links** — `[[WikiLinks]]` pointing to pages that don't exist
- **Contradictions** — claims that conflict across pages
- **Stale summaries** — pages not updated after newer sources
- **Missing entity pages** — entities mentioned in 3+ pages but lacking their own page
- **Upgrade candidates** — simple concepts that 2+ sources treat as a core topic (definition + examples), candidates for `simple → teaching` promotion
- **Data gaps** — questions the wiki can't answer; suggest new sources

Output a lint report and ask if the user wants it saved to `wiki/lint-report.md`.

---

## Health Workflow

Triggered by: *"health"* or `/wiki-health`

Run: `python tools/health.py` (or `python tools/health.py --json` for machine-readable output)

Fast structural integrity checks — **zero LLM calls**, safe to run every session:
- **Empty / stub files** — pages with no content beyond frontmatter (rate-limit damage)
- **Index sync** — `wiki/index.md` entries vs actual files on disk
- **Log coverage** — source pages missing a corresponding `ingest` entry in `wiki/log.md`

Output a health report. Use `--save` to write to `wiki/health-report.md`.

### Health vs Lint Boundary

| Dimension | `health` | `lint` |
|---|---|---|
| **Scope** | Structural integrity | Content quality |
| **LLM calls** | Zero | Yes (semantic analysis) |
| **Cost** | Free | Tokens |
| **Frequency** | Every session, before other work | Every 10-15 ingests |
| **Checks** | Empty files, index sync, log sync | Orphans, broken links, contradictions, gaps |
| **Tool** | `tools/health.py` | `tools/lint.py` |
| **Run order** | First (pre-flight) | After health passes |

> Run `health` first — linting an empty file wastes tokens.

---

## Graph Workflow

Triggered by: *"build the knowledge graph"* or `/wiki-graph`

When the user asks to build the graph, run `tools/build_graph.py` which:
- Pass 1: Parses all `[[wikilinks]]` → deterministic `EXTRACTED` edges
- Pass 2: Infers implicit relationships → `INFERRED` edges with confidence scores
- Runs Louvain community detection
- Outputs `graph/graph.json` + `graph/graph.html`

If the user doesn't have Python/dependencies set up, instead generate the graph data manually:
1. Use Grep to find all `[[wikilinks]]` across wiki pages
2. Build a node/edge list
3. Write `graph/graph.json` directly
4. Write `graph/graph.html` using the vis.js template

---

## Naming Conventions

- Source slugs: `kebab-case` matching source filename
- Entity pages: `TitleCase.md` (e.g. `OpenAI.md`, `SamAltman.md`)
- Concept pages: `TitleCase.md` (e.g. `ReinforcementLearning.md`, `RAG.md`)
- Source pages: `kebab-case.md`

## Index Format

```markdown
# Wiki Index

## Overview
- [Overview](overview.md) — living synthesis

## Sources
- [Source Title](sources/slug.md) — one-line summary

## Entities
- [Entity Name](entities/EntityName.md) — one-line description

## Concepts

### Teaching
- [Concept Name](concepts/ConceptName.md) — one-line description · prereqs: [[A]] → [[B]]

### Simple
- [Concept Name](concepts/ConceptName.md) — one-line description

## Syntheses
- [Analysis Title](syntheses/slug.md) — what question it answers
```

## Log Format

Each entry starts with `## [YYYY-MM-DD] <operation> | <title>` so it's grep-parseable:

```
grep "^## \[" wiki/log.md | tail -10
```

Operations: `ingest`, `query`, `health`, `lint`, `graph`, `promote`
