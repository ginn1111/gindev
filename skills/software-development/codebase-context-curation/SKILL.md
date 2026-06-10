---
name: codebase-context-curation
description: Two-way context management — intake (exploring codebase → curating into ByteRover) and export (materializing curated context as project docs). Covers knowledge provenance, storage placement, and extraction workflow.
version: 1.1.0
---

# Codebase Context Curation

## Overview

Two-directional knowledge management for project work:

**Intake (explore → curate):** Systematic approach to exploring a new codebase and distilling its conventions, structure, and patterns into durable context. Prevents shallow understanding and ensures future sessions can skip re-discovery.

**Export (curated → project docs):** Materialize agent-curated context as version-controlled documentation files in the project. Keeps knowledge human-accessible, team-readable, and independent of any agent's profile storage.

## When to Use

- First time encountering a project's codebase
- Need to understand conventions before making changes
- User asks to "curate context," "document the codebase," or "explore and organize knowledge"
- Before starting significant feature work on an unfamiliar project
- **Export mode:** user asks where knowledge comes from, to move knowledge into the project, to verify source of truth, or says "don't put it in my profile"
- **Export mode:** agent's ByteRover context tree has accumulated project knowledge that should be team-readable

## Workflow

### Phase 1 — Foundation (Config files first)

Read root config files to establish the tech stack and tooling:

```
eslint / prettier / tsconfig    → code-style rules and strictness
package.json                     → dependencies, scripts, package manager
next.config / postcss / tailwind → build and styling pipeline
AGENTS.md / CONTRIBUTING.md     → project-specific conventions (always start here)
```

These define the rules. Read them before any source code.

### Phase 2 — Documentation scan

Open every `docs/` file and per-directory `AGENTS.md` that governs a layer:

| Layer | Doc |
|-------|-----|
| Component conventions | `src/components/AGENTS.md` (if exists) |
| Data fetching pipeline | `src/hooks/AGENTS.md`, `src/services/AGENTS.md` |
| Type/schema rules | `src/types/AGENTS.md` |
| Styling | `docs/widget-layout-spacing.md`, `docs/liquid-best-practices.md` |
| Structure | `docs/structure.md` |

These docs encode the team's hard-won conventions. Don't reverse-engineer from source when the doc exists.

### Phase 3 — Structure survey

Map the directory tree with `codegraph_files(format='flat')` or `codegraph_files(format='tree')`. Identify:

- Route segments in `src/app/`
- Widget directory at `src/components/widget/`
- UI primitives at `src/components/ui/`
- Store slices at `src/store/`
- Hook/type/service organization patterns

Note the pattern: `components/` owns rendering, `hooks/` owns data fetching, `types/` owns schemas, `store/` owns state.

### Phase 4 — Representative source reads

Read one or two exemplar files per layer:

- A widget's `index.tsx` (shows the 4-region anatomy pattern)
- A hook's `useInfinity<Domain>.ts` (shows the generated-options + select + safeParse pattern)
- A store slice (Redux Toolkit createSlice pattern)
- A type file (Zod-from-generated-DTO pattern)
- A service file (class-based singleton pattern, if used)

Pick the most heavily referenced widget first — it will be the canonical example.

### Cross-layer applicability audits

When the task is not "how is X implemented" but **"which layers should get X"** (for example: interval filters, semantic badges, clustering, legend treatment), do a three-way scan instead of looking at only current UI wiring:

1. **Inventory / docs** — start from the canonical layer inventory in `docs/` to understand what each layer means to the user (event stream, live tracker, static infrastructure, sensor snapshot, etc.).
2. **Current registry / UI wiring** — inspect registries like `MAP_FILTER_LAYERS_REGISTRY` to see what is already exposed. Treat this as implementation status, not product truth.
3. **Hooks / query shapes** — inspect the map layer hooks and generated query types for time-bearing params (`since`, `until`, `date_from`, `date_to`, `year`, etc.) and note whether the hook actually transforms UI state into API params.

Use those three views to build a matrix:

- `layer`
- `semantic type` (event / live current-state / static reference / infrastructure)
- `should apply feature?`
- `why from user meaning`
- `backend/query support`
- `current implementation status`

This prevents a common mistake: assuming "currently wired" means "semantically correct," or assuming a backend time param means the feature should appear in the UI.

### When user asks for a decision doc or spec

If the task is "which layers / modules should get X" and the user wants docs in the repo, produce **two companion docs** rather than only one narrative answer:

1. **Decision matrix doc** — concise per-layer table covering semantic type, whether the feature should apply, user-facing meaning/label copy, current implementation status, and notes.
2. **Implementation/product spec doc** — goals, non-goals, product principles, scope (in / conditional / out), UX behavior, data/query behavior, rollout order, and acceptance criteria.

Recommended pattern:

- Matrix doc answers **what / where / why**.
- Spec doc answers **how it should behave and how to ship it**.
- Cross-link both docs and cite the canonical inventory / implementation docs they derive from.

This is especially useful for semantic UI features like interval filters, legends, clustering rules, badges, and layer-specific controls.

### Phase 5 — Curate into knowledge store

Organize curated context by domain, not by file:

| Domain | Content |
|--------|---------|
| Code Style & Quality | Linting rules, TS strictness, error handling, testing conventions |
| Styling & Design | CSS framework, typography system, Liquid UI, widget anatomy, chart patterns |
| Naming Conventions | File naming, type/hook/service naming, import conventions |
| Project Structure & Dependencies | Directory layout, provider tree, Redux shape, widget registration flow |

**Critical: split large payloads.** `brv curate` has a ~120s timeout. If the curated content is large (2000+ chars), split by domain into separate curate calls. Each call should be a self-contained domain chunk.

### When brv curate times out

The brv curate tool times out after 120s on large payloads. Workarounds:

1. **Split by domain** — each chunk covers one of the 4 domains above. Smaller payloads complete reliably.
2. **Split by sub-domain** — if a domain is still too large, split further (e.g. "Provider Tree + Redux" separate from "Directory Layout").
3. **Check success** — after timeout, check which entries succeeded by running `brv curate view` and re-submitting any that failed.

## Phase 6 — Export to project docs

Curated context in ByteRover/agent storage is useful for the agent but invisible to the human and fragile (tied to profile storage). Materialize key knowledge as project documentation files.

### When to export

- User explicitly asks to move knowledge into the project
- Context tree has accumulated multiple sessions of project-specific knowledge
- Knowledge is mature (refined through iteration) and ready for team consumption
- User says "don't put it in my profile" — this is a clear signal

### Extraction procedure

1. **Locate the source context tree** — brv stores data at `~/.hermes/profiles/<profile>/byterover/.brv/context-tree/`. The project's `.brv/` may be empty or broken — always check the profile location.

2. **Identify content vs metadata files** — brv generates 4 file types per node:
   - `<topic>.md` — the actual content (this is what you want)
   - `<topic>.abstract.md` — brv summary metadata (skip)
   - `<topic>.overview.md` — brv overview metadata (skip)
   - `<context.md>` — brv context metadata (skip)
   - `<index.md>` — brv index metadata (skip)

3. **Copy to `docs/`** — flatten with `category--topic.md` naming:
   ```bash
   SRC="~/.hermes/profiles/<profile>/byterover/.brv/context-tree/<project>/"
   DST="<project-root>/docs/architecture/"
   mkdir -p "$DST"
   find "$SRC" -name "*.md" | while read f; do
     basename=$(basename "$f")
     case "$basename" in
       *.abstract.md|*.overview.md|_index.md|context.md) continue ;;
     esac
     rel="${f#$SRC/}"
     category=$(dirname "$rel")
     leaf="${basename%.md}"
     dest_name="${category}--${leaf}.md"
     cp "$f" "$DST/$dest_name"
   done
   ```

4. **Create a README index** — map every file to what it covers so future agents can navigate without reading all files.

5. **Organize by domain** — group files under `docs/architecture/` with consistent naming. Consider subdirectories if the knowledge base grows large.

### Post-extraction: version control

After copying files, immediately commit and push:

```bash
cd <project-root>
git add docs/architecture/
git commit -m "docs: migrate brv knowledge into project docs/architecture/"
git pull --rebase
git push
```

This makes the extracted knowledge durable. Without this step, files exist only in the working tree and can be lost to tool ephemerality.

### Knowledge provenance

When answering questions using curated knowledge, always distinguish:
- **AGENTS.md / docs/** — project-owned, canonical, authoritative
- **ByteRover context tree** — agent-curated transcripts of the above (may be stale or derivative)
- **Code analysis** — reverse-engineered from source (may miss documented intent)

**Rule:** AGENTS.md is the primary source. ByteRover is a local cache. Code analysis is fallback. Say which you're using.

## Pitfalls

- **Skipping config files** — reading only source code misses the rules that govern style, imports, and naming. Always start with eslint/prettier/tsconfig/package.json.
- **Reading every file** — don't. Read the docs first, then exemplar files. Most knowledge lives in conventions, not individual source files.
- **One big curate call** — leads to timeout. Split by domain.
- **Skipping AGENTS.md** — project-specific AGENTS.md files contain concise rules that are the Single Source of Truth for each layer. Not reading them means guessing what the team already documented.
- **Curating without structure** — dumping everything as one blob makes retrieval harder later. Organize by domain and include concrete examples (file paths, code patterns).
- **Memory vs. skill vs. project docs confusion** — three different stores with different purposes:
  - **Hermes memory**: preferences, environment facts, user corrections (durable per-user, session-crossing)
  - **ByteRover context tree**: project-specific knowledge for the agent (agent-facing, fragile)
  - **Project docs (docs/)**: team-readable knowledge, version-controlled with the code
  - **Skills**: reusable workflows and class-level task guidance
  - Project-specific knowledge belongs as `docs/` in the project, not in ByteRover/profile storage. If it's knowledge the human should see or that survives a profile reset, it goes in `docs/`.
- **Assuming project `.brv/` is the source** — the project's `.brv/context-tree/` may be empty or detached. The real data often lives at `~/.hermes/profiles/<profile>/byterover/.brv/context-tree/`. Always check both locations before reporting "nothing found" or "only skeleton files."
- **Leaking profile storage references into answers** — when answering "what is the architecture", cite AGENTS.md or project docs, not "ByteRover says..." or "memory says...". The human wants to know the source they can trust and update.
- **Not flattening on export** — exporting with nested directories (`project_structure/feature_modules.md`) requires the reader to guess the hierarchy. Flatten with `category--topic.md` so a single `ls` shows everything.
- **Tool ephemerality on export** — `write_file()` and `terminal cp` can report success but the files may not survive to the next turn. Always re-verify immediately in the same call chain. If files vanished, re-execute the copy. Only commit confirms durability. Do not trust a "bytes_written" return from a previous turn — that state is gone.
