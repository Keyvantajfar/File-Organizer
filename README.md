# File Organizer: an adaptive, LLM-assisted file organizer

> **Status:** planning. This README is the design doc and roadmap. Nothing here is implemented yet.

A program that **understands your files** (by content, not just by name), **proposes a better directory structure**, lets you **steer it by chatting**, and then applies the result **without ever putting your data at risk**.

You bring your own model (recommended: a self-hosted llama.cpp/vLLM/whatever OpenAI-Compatible-API server). Nothing leaves your machine unless you point it somewhere else.

---

## Table of contents

1. [Why](#1-why)
2. [Principles](#2-principles)
3. [How it thinks: the escalation ladder](#3-how-it-thinks-the-escalation-ladder)
4. [Model roles](#4-model-roles)
5. [Architecture](#5-architecture)
6. [Module design](#6-module-design)
7. [Data models](#7-data-models)
8. [Modes and the "doubt" system](#8-modes-and-the-doubt-system)
9. [Safety and privacy spec](#9-safety-and-privacy-spec)
10. [Scope: subdirectory vs `/home`](#10-scope-subdirectory-vs-home)
11. [Roadmap and todo list](#11-roadmap-and-todo-list)
12. [Evaluation](#12-evaluation)
13. [Post-v1: small embedded model](#13-post-v1-small-embedded-model)
14. [Risks](#14-risks)
15. [Open decisions](#15-open-decisions)
16. [Prior work this builds on](#16-prior-work-this-builds-on)

---

## 1. Why

Real file collections are messy in a specific way: names are useless (`untitled`, `test`, `test3final_final`, `IMG_4821`), one folder holds Python, Rust, half-written ideas and screenshots side by side, and the same document exists in six versions. Rule-based organizers (by extension, by date) can't fix that, because the information is *inside* the files.

Goal: an organizer that reads enough of each file to know what it is, builds a structure that fits **this user's** life and profession, and adapts when the user says "no, put that over there".

## 2. Principles

1. **Non-destructive by default.** The default strategy builds the new tree next to the old one using links. The original is untouched; the user deletes whichever side they don't want.
2. **Never delete, never overwrite.** The tool has no delete operation. Cleanup is the user's call.
3. **Plan as data.** The LLM never touches the filesystem. It edits a *plan* (a structured list of proposed operations). A separate, dumb, heavily tested executor applies it.
4. **Everything is journaled and reversible.** Write-ahead journal for every operation; one command undoes a run.
5. **Local-first.** Any OpenAI-compatible endpoint works; local is the recommended default. Remote endpoints need explicit opt-in.
6. **Cheap signals first, but never trust filenames.** Escalate from free signals to expensive ones only as needed, and force content analysis when the name is uninformative.
7. **Human in the loop, tunable.** Propose-and-review by default; auto mode that only interrupts on doubt.
8. **Adaptive.** User corrections become persistent rules, not one-off chat messages.
9. **Efficient.** Incremental scans, cached summaries, batched LLM calls, token budgets. A second run on the same tree should be fast.

## 3. How it thinks: the escalation ladder

The pipeline gathers signals in tiers. **This is not "LLM as a last resort".** For any file whose name and metadata don't identify it (most of a messy collection), the small worker LLM on file content is the *main* path. Tiers exist so that files that *are* well-identified (properly tagged music, EXIF-dated photos, a git repo) don't burn model calls.

| Tier | Signal | Cost | Used for |
|------|--------|------|----------|
| 0 | path, name, size, timestamps, MIME/type sniffing, content hash | ~free | ignore rules, exact dedup, atomic-unit detection, junk-name detection |
| 1 | format metadata and structural extractors (ID3, EXIF, PDF info, `pyproject.toml`, `.git`, Obsidian vault markers) | cheap | domain plugins, dates, authors, project detection |
| 2 | embeddings of text, captions, images | cheap (small model) | clustering, near-duplicates, destination matching |
| 3 | **Worker LLM** on a content sample (or VLM on image/keyframes) producing a `FileCard` | moderate | the main path for ambiguous files |
| 4 | **Architect LLM** | expensive, rare | taxonomy design, conflicts, low-confidence placements, rule interpretation, chat |

**Skip rule:** a file skips Tier 3 only when Tier 0-2 already give a high-confidence classification *and* the name is informative. A **junk-name detector** (regex plus a tiny classifier: `untitled*`, `test*`, `final*`, `copy of`, `IMG_####`, `New Document`, etc.) forces content analysis regardless of anything else.

**Agentic reading:** the worker gets a per-file token budget and a `read_more(offset, length)` tool. It starts with head/middle/tail samples and can ask for more only when the card is still ambiguous (e.g. a text file that could be Rust or a draft essay).

## 4. Model roles

One model can fill every role in a simple setup, or each role can point at a different endpoint.

| Role | Job | Size guidance | Notes |
|------|-----|---------------|-------|
| **Embedder** | vectors for clustering, near-dup, destination matching | ~100M params | runs on CPU; `sentence-transformers`-class models |
| **Worker** | per-file `FileCard` from content; per-file placement into an existing taxonomy | small (1-4B instruct) | high volume, structured JSON output, must be fast |
| **Vision** | image captions, OCR-ish text, keyframe description | small VLM, or the same multimodal model | can be skipped for images with rich metadata |
| **Architect** | designs the top-level structure from compact summaries; resolves hard cases | large (7B+ or API) | low volume, runs on summaries not files |
| **Agent** | the chat loop: tool-calling to edit plan and rules | architect-class preferred | needs reliable tool use |

Why this split: most of the knowledge a worker needs is *in the prompt* (the file content), so small models can do per-file work. The part that needs broad world knowledge (inventing a good taxonomy for a chemist vs. a carpenter vs. a programmer) happens once, on compact summaries, where a bigger model is affordable.

All LLM calls go through one **OpenAI-compatible client** (llama.cpp server, Ollama, vLLM, LM Studio, or a hosted API). Use JSON-schema / grammar-constrained output wherever the backend supports it.

## 5. Architecture

```
                         ┌────────────────────────────┐
                         │      config + rules        │
                         └─────────────┬──────────────┘
                                       │
 filesystem ──► scanner ──► index (SQLite) ◄──────────────────────┐
   (roots)      │  ignore rules          ▲                        │
                │  atomic units          │ cache by path+mtime    │
                ▼                        │                        │
            extractors ──► cards ───────►┤                        │
        (tiers 0-3; plugins)             │                        │
                                         ▼                        │
                              embed + cluster + dedup             │
                                         │                        │
                                         ▼                        │
                        planner (architect: taxonomy,             │
                                 worker: placement,               │
                                 rules applied) ──► PLAN ◄──┐     │
                                         ▲                  │     │
                                         │          agent/chat    │
                                         │       (edits plan+rules)
                                         ▼                  │     │
                               review UI (CLI/TUI) ─────────┘     │
                                         │                        │
                                         ▼                        │
                               executor (stage / in-place)        │
                                 journal + verify + undo ─────────┘
                                         │
                                         ▼
                                  feedback log (local)
```

### Run lifecycle

1. **Scan** the chosen roots; apply ignore rules; detect atomic units; hash; store in the index.
2. **Extract** signals tier by tier; produce `FileCard`s and `FolderCard`s; cache them.
3. **Group:** embeddings, clustering, exact and near-duplicate detection, version families (`report`, `report_v2`, `report_final_final`).
4. **Propose taxonomy** (architect, from compact cluster summaries and any existing good structure). User discusses and approves it **before** any per-file work is committed.
5. **Place** each file into the taxonomy (worker, plus rules, plus plugins); attach confidence and reason.
6. **Review:** chunked accept/reject in propose mode; auto mode only surfaces doubtful items.
7. **Apply** via the executor into a staging tree (default) or in place (explicit flag). Verify. Write the journal.
8. **Undo** at any time from the journal.

## 6. Module design

Proposed layout (Python 3.11+, `src` layout):

```
.
├── README.md
├── docs/                 architecture.md, safety.md, plugins.md, prompts.md
├── src/<pkg>/
│   ├── cli/              entrypoints (init, scan, plan, review, chat, apply, undo, auto)
│   ├── config/           settings schema, model endpoints, roots, policies
│   ├── scanner/          walker, ignore engine, atomic-unit detection, hashing
│   ├── index/            SQLite schema, migrations, cache, incremental rescan
│   ├── extractors/       per-type signal extraction (text, code, pdf, office, image, video, audio, archive)
│   ├── cards/            FileCard / FolderCard models, junk-name detector, summarizer
│   ├── llm/              OpenAI-compatible client, structured output, prompts, batching, token accounting
│   ├── embed/            embedding backend, vector store, clustering, near-dup, version families
│   ├── planner/          taxonomy proposal, placement, rename proposals, confidence
│   ├── rules/            persistent user rules, matching, conflict resolution
│   ├── plan/             Plan / PlanItem models, diffing, tree rendering, validation
│   ├── review/           interactive review (chunked accept/reject), tree diff view
│   ├── agent/            chat loop, tools that edit plan/rules, session memory
│   ├── executor/         link strategies, journal, verification, undo
│   ├── plugins/          domain organizers (music, photos, code projects, obsidian)
│   ├── safety/           protected paths, sensitive-file detection, policy checks
│   └── feedback/         local accept/reject log (future training data)
├── tests/
├── bench/                synthetic messy-tree generator, labeled fixtures, metrics
└── configs/              example configs
```

### 6.1 `scanner`
- Walk roots without following symlinks by default. Record path, size, mtime, inode/device, type.
- **Ignore engine** (gitignore-style, layered): built-in defaults (`__pycache__`, `site-packages`, `node_modules`, `.venv`, `target/`, `build/`, caches, `.git` internals, system dirs), user-extensible, plus optional LLM-decided skips for unknown directories (based on a sampled listing).
- **Atomic-unit detection:** a directory that is a git repo, Python/Rust/Node project, installed app, Obsidian vault, photo library bundle, etc. is a unit. It moves whole or not at all; its insides are never reorganized unless the user opts in.
- Hash files (size-gated; partial hash first, full hash on collision candidates).
- Incremental: skip anything whose `(path, size, mtime)` matches the index.

### 6.2 `index`
SQLite (WAL mode). Tables: `files`, `units`, `cards`, `embeddings` (or a separate vector store), `runs`, `plans`, `plan_items`, `journal`, `rules`. Resumable: every stage checkpoints so a crash or Ctrl-C loses nothing.

### 6.3 `extractors`
Pluggable by type; each returns a normalized `Signals` object.
- **Type sniffing:** content-based (`magika` or libmagic), not extension-based.
- **Text/code/markup:** head/middle/tail sampling, language detection (Pygments or tree-sitter), docstring/heading/title extraction, shebang, imports.
- **PDF/Office/eBook:** text, title, author, page count (PyMuPDF/pypdf, python-docx, etc.).
- **Images:** EXIF, dimensions, perceptual hash, optional OCR, VLM caption, optional CLIP-style embedding.
- **Video:** `ffprobe` metadata; sparse keyframes via scene detection (not every frame); optional audio transcript (whisper.cpp / faster-whisper).
- **Audio:** tags (mutagen); optional transcript or fingerprint for untagged files.
- **Archives:** list contents without extracting; classify by listing.
- **Binaries / unknown:** metadata only. Never read contents into a model.

### 6.4 `cards`
`FileCard`: the compact, model-friendly summary of one file (see [7](#7-data-models)). `FolderCard`: aggregate summary of a directory (dominant types, topics, how well-organized it already looks). The planner and architect only ever see cards, never raw files.

### 6.5 `llm`
- One client interface (`chat`, `chat_json(schema)`, `embed`, `vision`), configured per role.
- Batching and concurrency tuned for llama.cpp parallel slots; retries; strict JSON validation with repair.
- Prompt library versioned in `docs/prompts.md`; prompts are tested against the benchmark.
- Token accounting and a **pre-run cost/time estimate** shown to the user.
- Response cache keyed by `(prompt_hash, model)`.

### 6.6 `embed`
Vectors of card text (and image/caption embeddings). Clustering (HDBSCAN-class) proposes groups; the architect names them. Near-duplicate detection (text: MinHash/SimHash; images: perceptual hash; exact: content hash). **Version families:** group `final`, `final2`, `copy of` variants; propose keeping them together with the newest marked as primary. Never delete.

### 6.7 `planner`
- **Architect step:** input = cluster summaries plus folder cards plus user hints; output = a proposed taxonomy (tree with descriptions per folder). Reused existing structure is preferred when it looks good, to avoid churn.
- **Placement step:** worker (plus embedding similarity to folder descriptions) assigns each file to a destination with a reason.
- **Rename proposals:** optional, separate toggle from moves (renaming is more invasive). Junk names get descriptive suggestions.
- **Confidence:** see [8](#8-modes-and-the-doubt-system).

### 6.8 `rules`
Persistent, declarative, user-editable (YAML). Created from chat or by hand. Rules mix hard constraints (extension, path glob, size) with a **semantic matcher** (a short natural-language description matched via embeddings and verified by the worker). Rules apply to the current plan, trigger re-planning of affected items, and persist for future runs. Priorities and conflict resolution are explicit; the user can list, edit, disable.

### 6.9 `plan`
`Plan` = tree of proposed folders plus a list of `PlanItem`s. Operations: `link` / `move` / `copy` / `edit-metadata` / `skip`. Validation before apply: no collisions, no path escapes, no writes outside allowed roots, every source accounted for exactly once.

### 6.10 `review`
CLI first, TUI (Textual) once the flow is stable. Shows the proposed tree, then chunks of items grouped by destination with reasons. Accept / reject / edit destination / "make this a rule". Taxonomy approval comes first; per-file review follows.

### 6.11 `agent`
Chat loop with tool calls: `show_plan`, `move_item`, `move_group`, `create_rule`, `rename_folder`, `explain_item`, `search_files`, `reread_file`, `replan(scope)`. Can be invoked at the start (shape the taxonomy), mid-run (redirect a category), or after. It edits plan and rules only; it has no filesystem access of its own.

### 6.12 `executor`
- **Strategies:** `stage` (default): build the new tree in a separate output dir using the best available link type: reflink, then hard link, then symlink (preview only), then copy. `inplace` (explicit flag): moves with the journal.
- **Write-ahead journal** (every op recorded before it runs), post-apply verification (every source has exactly one target; sizes/hashes match), and `undo <run-id>`.
- Hard links only work within one filesystem and not for directories; the executor detects this and falls back. Symlinked staging must be "materialized" before the user deletes the original.
- **Edits to content (e.g. adding frontmatter to notes) always operate on a real copy, never through a hard link**, or the original would change too.
- The output dir must be outside every scan root (or excluded from scanning).

### 6.13 `plugins`
Deterministic domain organizers that take over when they recognize a collection. Interface: `detect(unit_or_dir) -> confidence`, `propose(items) -> PlanItems`.
- **music:** metadata to `Artist/Album/NN - Title` (port of the existing no-LLM scripts).
- **photos:** EXIF date/location to `Year/Month` or event clusters.
- **code-projects:** detect projects and standalone scripts; group by language/purpose.
- **obsidian:** vault-aware; YAML header/tagging; chunked accept/reject (port of the existing demo).
Plugins can call the worker LLM for sub-decisions but don't have to.

### 6.14 `safety`
Protected-path list, sensitive-file detection (keys, credentials, IDs, tax/bank documents: handled by name and metadata only, never sent to a model unless explicitly allowed), per-folder `no-read` marker, policy enforcement before any model call and any filesystem op. See [9](#9-safety-and-privacy-spec).

### 6.15 `feedback`
Local log of proposals, accepts, rejects, edits and rule creations. Used for calibration now, and (opt-in) as training data later.

## 7. Data models

### FileCard (sketch)
```json
{
  "path": "/home/u/Downloads/test3final_final.txt",
  "hash": "blake3:...",
  "type": {"mime": "text/plain", "detected": "rust-source", "confidence": 0.93},
  "name_quality": "junk",
  "summary": "Rust CLI argument parser using clap; incomplete, has TODO comments.",
  "topics": ["rust", "cli", "argument-parsing"],
  "kind": "code/source",
  "project_hint": null,
  "dates": {"modified": "2025-11-02", "content_date": null},
  "version_family": "vf_0192",
  "sensitivity": "none",
  "signals": {"tier_reached": 3, "tokens_read": 1200, "plugin": null},
  "embedding_ref": "emb_88213"
}
```

### PlanItem (sketch)
```json
{
  "id": "mv_00421",
  "src": "/home/u/Downloads/untitled (3).png",
  "action": "link",
  "dst": "Pictures/Screenshots/2025/terminal-error-traceback.png",
  "rename": {"from": "untitled (3).png", "to": "terminal-error-traceback.png"},
  "reason": "Screenshot of a Python traceback; no EXIF; similar to 14 other screenshots.",
  "confidence": 0.82,
  "signals": {"embedding_margin": 0.21, "rule": null, "new_folder": false},
  "status": "proposed"
}
```

### Rule (sketch)
```yaml
- id: rule_phd_thesis
  description: "Anything about my PhD thesis goes to the PhD folder at home"
  match:
    semantic: "PhD thesis drafts, notes, references, figures"
    extensions: [md, tex, pdf, docx]
  action:
    dest: "~/PhD/Thesis"
  priority: 100
  scope: persistent        # or: session
```

### Config (sketch)
```toml
[roots]
include = ["~/Downloads"]
output  = "~/Organized"          # must be outside include roots

[llm.worker]
base_url = "http://localhost:8080/v1"
model    = "local-worker"

[llm.architect]
base_url = "http://localhost:8081/v1"
model    = "local-architect"

[embed]
backend = "local"

[policy]
mode            = "propose"       # propose | auto | dry-run
strategy        = "stage"         # stage | inplace
allow_remote    = false
rename_files    = false
auto_threshold  = 0.80
```

### CLI (sketch)
```
<cmd> init                    # create config, test endpoints
<cmd> scan <root>...          # scan + extract + cards (resumable)
<cmd> plan                    # propose taxonomy, then placements
<cmd> review                  # interactive accept/reject
<cmd> chat                    # steer with natural language (also available inside review)
<cmd> apply --strategy stage --out ~/Organized
<cmd> undo <run-id>
<cmd> auto <root>             # auto mode: only asks when in doubt
<cmd> rules list|edit|disable
```

## 8. Modes and the "doubt" system

| Mode | Behavior |
|------|----------|
| `dry-run` | produce and print a plan; touch nothing |
| `propose` (default) | taxonomy approval, then chunked per-item review, then apply |
| `auto` | apply high-confidence items; queue the rest for review. The **taxonomy** is still approved once on the first run of any root |

**Doubt is computed, not self-reported.** Models are poor at stating their own confidence. Signals:
- embedding **margin** between the best and second-best destination
- **agreement** across multiple samples or between worker and similarity ranking
- whether a **rule** or plugin matched (strong evidence)
- destination is a **new folder** (lower confidence)
- file is **sensitive**, an atomic unit, or part of a version family with unclear primary
- content read was **truncated** before an informative part

The threshold is configurable and calibrated against logged accept/reject data over time.

## 9. Safety and privacy spec

**Invariants (each has automated tests):**
1. **No data loss:** every source file appears exactly once in the plan, the journal, and the post-apply verification.
2. **No mutation of originals** in `stage` mode.
3. **No delete operation exists.**
4. **No writes outside** the configured output dir (stage) or scan roots (inplace).
5. **Write-ahead journal;** every applied operation is reversible via `undo`.
6. **Atomic units are never split.**
7. **No content leaves configured endpoints;** remote endpoints require `allow_remote = true` and a visible warning.
8. **Sensitive files** are never sent to a model unless explicitly allowed per file or folder.
9. **Content edits** (frontmatter etc.) run on copies and keep a backup.
10. **Collisions** never overwrite; they get a suffix or are flagged.

**Protected by default:** system directories, dot-directories in home (`.ssh`, `.config`, `.local`, `.cache`, ...), application data, package managers' trees, mounted network shares (unless opted in).

**First run on any new root is always propose-only/dry-run**, even if auto mode is configured.

## 10. Scope: subdirectory vs `/home`

Both are supported in v1; the difference is policy, not code.

- **Subdirectory mode** (`~/Downloads`, `~/Documents`, an external drive): full pipeline, normal defaults.
- **`/home` mode:** treated as system-like. Dot-directories, application data and atomic units are protected by default; only "user content" zones are reorganized. Requires an explicit confirmation flag, a pre-run size/time estimate, and the mandatory first-run propose-only rule. Staging output must live on a separate path (ideally the same filesystem for link efficiency, but excluded from the scan).

## 11. Roadmap and todo list

**Everything below is in scope for v1.0.** The order is a build order, not a priority order: it puts the safety-critical, non-LLM spine first and proves it end-to-end with the music plugin before any model is involved.

### M0: Foundations
- [ ] Repo, `src` layout, packaging (`uv`/`pip`), lint + format + type-check, pytest, CI
- [ ] Config schema and loader; `init` command with endpoint health check
- [ ] `docs/safety.md` with the invariants above
- [ ] Synthetic messy-tree generator (`bench/`): junk names, mixed languages, version families, duplicates, fake projects, fake vaults
- [ ] Logging and run IDs

### M1: Scanner and index
- [ ] Walker (no symlink following), ignore engine with defaults and user layers
- [ ] Atomic-unit detectors (git, Python, Rust, Node, app bundles, Obsidian vault)
- [ ] Content-based type sniffing
- [ ] Hashing (partial then full), SQLite index, incremental rescans, resumable stages
- [ ] Protected-path policy

### M2: Plan, executor and journal (walking skeleton, no LLM)
- [ ] `Plan` / `PlanItem` models and validation
- [ ] Executor: reflink / hardlink / symlink / copy strategies, collision handling, output-dir-outside-roots check
- [ ] Write-ahead journal, post-apply verification, `undo`
- [ ] **Music plugin** (port existing scripts) driving the whole thing end-to-end
- [ ] Property tests for the no-data-loss invariants

### M3: Extractors and cards
- [ ] Text/code extractor with sampling, language detection, title/heading extraction
- [ ] PDF, Office, eBook extractors
- [ ] Junk-name detector
- [ ] `FileCard` / `FolderCard` models and storage
- [ ] Sensitivity detector
- [ ] Archive listing; binaries = metadata only

### M4: LLM layer
- [ ] OpenAI-compatible client (chat, JSON-schema output, embeddings, vision)
- [ ] Worker prompt: content sample to `FileCard` (benchmarked)
- [ ] `read_more` tool and per-file token budgets
- [ ] Batching, concurrency, retries, caching, token accounting, cost estimate
- [ ] Test against llama.cpp, Ollama and one hosted API

### M5: Embeddings, clustering, dedup
- [ ] Embedding backend and vector store
- [ ] Clustering and cluster summaries
- [ ] Exact + near-duplicate detection (text, image)
- [ ] Version-family grouping

### M6: Planner
- [ ] Architect prompt: cluster summaries to taxonomy proposal
- [ ] Reuse of already-good existing structure
- [ ] Worker placement into the taxonomy with reasons
- [ ] Rename proposals (optional toggle)
- [ ] Confidence signals (embedding margin, agreement, new-folder, etc.)

### M7: Review UI
- [ ] Tree rendering and taxonomy approval step
- [ ] Chunked per-item accept / reject / edit destination
- [ ] "Make this a rule" action
- [ ] TUI (Textual) once the CLI flow is stable

### M8: Rules and chat agent
- [ ] Rule schema, storage, matching (hard constraints + semantic matcher)
- [ ] Conflict resolution, list/edit/disable
- [ ] Agent loop with tool calls that edit plan and rules
- [ ] Targeted re-planning of only the affected items
- [ ] Chat at start, mid-run and post-run

### M9: Auto mode
- [ ] Threshold-based auto-apply with a review queue for doubtful items
- [ ] Calibration from the feedback log
- [ ] First-run-on-new-root guard

### M10: Multimodal
- [ ] Images: EXIF, perceptual hash, VLM caption, optional OCR and embedding
- [ ] Video: `ffprobe`, scene-based keyframes, optional transcript
- [ ] Audio: tags, optional transcript/fingerprint
- [ ] Per-modality token/time budgets

### M11: Plugins
- [ ] Plugin interface and docs
- [ ] Photos plugin
- [ ] Code-projects plugin
- [ ] Obsidian plugin (port of the existing demo: frontmatter tagging on copies, chunked accept/reject)

### M12: Hardening and release
- [ ] `/home` mode policies and confirmation flow
- [ ] Benchmarks on the synthetic and hand-labeled sets (see below)
- [ ] Performance pass (incremental rescan, parallelism, memory)
- [ ] Docs: quickstart, model setup guides (llama.cpp first), safety, plugin authoring
- [ ] v1.0 release

## 12. Evaluation

Build this **early** (M0 generator, grown through M12). Without it, prompt changes and any future fine-tune can't be measured.

- **Datasets:** synthetic messy trees with known target structure; a few hand-labeled real-world-style trees (different professions: software, science, construction, finance, student).
- **Metrics:**
  - placement agreement with reference structure
  - user-edit rate (fraction of proposals changed in review)
  - tokens and wall-clock per 1k files
  - second-run speedup (cache effectiveness)
  - junk-name files correctly identified from content
  - **safety: zero data-loss violations** across fuzzed runs
- **Model comparison:** same benchmark across candidate worker models (size vs. quality vs. speed) to decide what to recommend, and whether a fine-tune pays off.

## 13. Post-v1: small embedded model

Goal: a ~1B-class model, fine-tuned for the worker role, bundled so average users get fast CPU / low-end GPU operation with no setup, while API use stays available.

- **Scope the small model to the worker role** (card extraction, placement into a *given* taxonomy). The architect role keeps using a larger model (user's API or a bigger local model) or an optional "taxonomy templates by profession" fallback.
- **Training data:** synthetic labeled data generated with a large model; opt-in, locally logged accept/reject/edit data from real runs; benchmark held out.
- **Quantization and packaging** for llama.cpp-class runtimes.
- **Decision gate:** only pursue if the benchmark shows a clear gap between off-the-shelf small models and the target quality.

## 14. Risks

| Risk | Mitigation |
|------|-----------|
| Data loss from bugs | executor isolated from LLM; invariants + property/fuzz tests; stage-by-default; journal |
| Breaking projects by moving parts of them | atomic units; protected paths |
| Privacy leak via remote API | local-first; explicit opt-in; sensitive-file policy; per-folder `no-read` |
| Slow on large trees | incremental index, tiered signals, batching, estimates before running |
| Model hallucinated folder structures / unstable naming | schema-constrained output, taxonomy approval step, rules, benchmark-tested prompts |
| Hard-link surprises (shared edits, cross-device) | detect and fall back; edits on copies only; documented |
| User fatigue in review | chunking by destination, "make this a rule", auto mode with calibrated doubt |
| Scope creep ("everything in v1") | milestone order above; each milestone ships a usable vertical slice |

## 15. Open decisions

Defaults in bold; change them when you start coding.

- **Project name** and **license** (MIT/Apache-2.0)
- **Language:** **Python** for everything; consider a Rust scanner later only if profiling demands it
- **UI:** **CLI first, then Textual TUI**; GUI is post-v1
- **Vector store:** **SQLite + a local ANN index (e.g. `sqlite-vec` or `hnswlib`)** vs. a heavier DB
- **Default recommended models** (decide after the M12 benchmarks; document in the quickstart)
- **Renames:** **off by default**, opt-in
- **Telemetry:** **none**; feedback log is local only; sharing is a manual opt-in export
- **Windows support:** **Linux/macOS first**; Windows needs link-strategy and path-policy work

## 16. Prior work this builds on

- **Music organizer (no LLM):** metadata to `Artist/Album/Track`, output via hard links so the original is untouched. Becomes the `music` plugin and the template for the `stage` strategy.
- **Obsidian notes organizer (LLM demo):** read everything, propose a structure, talk to the user to finalize it, then tag and place notes chunk by chunk with accept/reject. Becomes the `obsidian` plugin and the template for the review loop.
