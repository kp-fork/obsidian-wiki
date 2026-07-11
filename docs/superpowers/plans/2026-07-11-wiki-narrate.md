# wiki-narrate Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a standalone `wiki-narrate` skill that renders a topic from an Obsidian vault as citation-bound Markdown in one of three canonical voices.

**Architecture:** The deliverable is declarative skill content, not a new application runtime. `.skills/wiki-narrate/SKILL.md` owns command validation, retrieval, claim-ledger construction, citation auditing, saving, and failure behavior; `references/voices.md` owns only the three voice skeletons. A focused stdlib test guards the skill's contractual phrases and the two routing documents expose the new entrypoint.

**Tech Stack:** Markdown Agent Skills, YAML frontmatter, Python 3 standard-library `unittest`, `rg`, and Git.

## Global Constraints

- Support only `/wiki-narrate <topic> [--voice briefing|plain-language|lecturer] [--save]`.
- Default to `briefing`; voice identifiers are canonical and case-sensitive; do not implement aliases.
- Generate Markdown only; do not add HTML, PDF, slides, renderer handoffs, tag/page-list selection, or prior-query input modes.
- Treat vault pages as the complete fact set: cite every factual sentence adjacently, mark interpretations `^[inferred]`, and mark unresolved conflicts `^[ambiguous]`.
- Resolve configuration through the shared Config Resolution Protocol, honor visibility filtering and `OBSIDIAN_LINK_FORMAT`, and exclude `_readouts/` from retrieval.
- Persist only after `--save`, to `_readouts/<slug>.md`; never add a readout to `index.md`, `.manifest.json`, or the knowledge graph.
- With `--save`, update `log.md` and `hot.md`; without it, append only a narration event to `log.md`.

---

## File Structure

| File | Responsibility |
| --- | --- |
| `.skills/wiki-narrate/SKILL.md` | Command contract, vault retrieval, claim ledger, drafting/audit, persistence, reporting, and failure rules. |
| `.skills/wiki-narrate/references/voices.md` | Section skeletons and prose boundaries for the three canonical voices. |
| `AGENTS.md` | Natural-language routing entry for narration requests. |
| `README.md` | Public skills-table entry and canonical command synopsis. |
| `tests/test_wiki_narrate_docs.py` | Regression tests for the required public contract and documentation wiring. |

## Task 1: Lock the public contract with regression tests

**Files:**
- Create: `tests/test_wiki_narrate_docs.py`

**Interfaces:**
- Consumes: repository-root Markdown files through `Path.read_text()`.
- Produces: `WikiNarrateDocsTest`, a stdlib test class that rejects missing canonical commands, citation markers, derived-view isolation, voices, or routing documentation.

- [ ] **Step 1: Write the failing test**

Create `tests/test_wiki_narrate_docs.py` with this complete content:

```python
from __future__ import annotations

from pathlib import Path
import re
import unittest


ROOT = Path(__file__).resolve().parents[1]


class WikiNarrateDocsTest(unittest.TestCase):
    def read(self, relpath: str) -> str:
        return (ROOT / relpath).read_text()

    def test_skill_declares_the_canonical_command_and_voices(self) -> None:
        skill = self.read(".skills/wiki-narrate/SKILL.md")

        self.assertIn("/wiki-narrate <topic> [--voice briefing|plain-language|lecturer] [--save]", skill)
        self.assertIn("default voice is `briefing`", skill)
        self.assertIn("case-sensitive", skill)
        self.assertIn("Unsupported values", skill)

    def test_skill_requires_closed_set_citations(self) -> None:
        skill = self.read(".skills/wiki-narrate/SKILL.md")

        self.assertIn("every factual sentence", skill)
        self.assertIn("^[inferred]", skill)
        self.assertIn("^[ambiguous]", skill)
        self.assertIn("web knowledge", skill)
        self.assertIn("model memory", skill)

    def test_skill_keeps_readouts_out_of_the_knowledge_graph(self) -> None:
        skill = self.read(".skills/wiki-narrate/SKILL.md")

        self.assertIn("`_readouts/<slug>.md`", skill)
        self.assertIn("must not update `index.md` or `.manifest.json`", skill)
        self.assertIn("exclude `_readouts/`", skill)

    def test_voice_reference_has_exactly_the_three_first_release_voices(self) -> None:
        voices = self.read(".skills/wiki-narrate/references/voices.md")

        headings = set(re.findall(r"^## `([^`]+)`$", voices, flags=re.MULTILINE))
        self.assertEqual(headings, {"briefing", "plain-language", "lecturer"})

    def test_routing_and_readme_expose_wiki_narrate(self) -> None:
        agents = self.read("AGENTS.md")
        readme = self.read("README.md")

        self.assertIn("`wiki-narrate`", agents)
        self.assertIn("`wiki-narrate`", readme)
        self.assertIn("`/wiki-narrate <topic>`", readme)


if __name__ == "__main__":
    unittest.main()
```

- [ ] **Step 2: Run the test to verify it fails**

Run:

```bash
python3 -m unittest tests/test_wiki_narrate_docs.py -v
```

Expected: FAIL with `FileNotFoundError` because `.skills/wiki-narrate/SKILL.md` and its voice reference do not exist.

- [ ] **Step 3: Commit the failing-contract test**

```bash
git add tests/test_wiki_narrate_docs.py
git commit -m "test: define wiki narrate contract"
```

## Task 2: Add the isolated narration skill and voice contract

**Files:**
- Create: `.skills/wiki-narrate/SKILL.md`
- Create: `.skills/wiki-narrate/references/voices.md`

**Interfaces:**
- Consumes: `<topic>`, optional `--voice`, optional `--save`, the resolved vault config, `index.md`, `hot.md`, QMD when configured, and candidate vault pages.
- Produces: a cited Markdown readout in conversation; with `--save`, a `_readouts/<slug>.md` derived file plus a `WIKI_NARRATE` log event and refreshed `hot.md`.

- [ ] **Step 1: Create the voice reference**

Create `.skills/wiki-narrate/references/voices.md`. It must begin with `# wiki-narrate Voices`, then define exactly these headings and skeletons:

```markdown
## `briefing`

Use for fast orientation and decisions. Keep the conclusion before evidence.

1. Executive Summary
2. Key Evidence
3. Implications
4. Open Questions

## `plain-language`

Use for readers unfamiliar with the topic. Define terms before using them and prefer short sentences. Do not add analogies that introduce uncited facts.

1. What This Means
2. Key Ideas
3. How the Pieces Fit
4. What We Still Do Not Know

## `lecturer`

Use for a progressive teaching readout. Build from context to mechanism, then implications and limits.

1. Context
2. Core Concepts
3. How It Works
4. Implications and Limits
5. Recap
```

- [ ] **Step 2: Create the skill procedure**

Create `.skills/wiki-narrate/SKILL.md` with YAML frontmatter naming `wiki-narrate` and a description that routes topic-based briefing, explanation, and lecture requests to it. Include these sections and exact behavior:

```markdown
# Wiki Narrate — Cited Narrative Readouts

## Command Contract

`/wiki-narrate <topic> [--voice briefing|plain-language|lecturer] [--save]`

- Require a non-empty `<topic>`.
- The default voice is `briefing`.
- Voice names are canonical and case-sensitive. Unsupported values must return an error listing `briefing`, `plain-language`, and `lecturer` without searching or writing.
- `--save` is the only persistence switch.

## Retrieval

1. Resolve configuration with the Config Resolution Protocol, then read the target vault's `AGENTS.md` when it exists.
2. Read `hot.md` and `index.md` first. Select candidates by frontmatter and summary before reading bodies.
3. When configured, use QMD before `rg`; if QMD is absent or fails, continue with the index and `rg` path.
4. Honor filtered-mode phrases and skip pages tagged `visibility/internal` or `visibility/pii` in that mode.
5. Exclude `_readouts/`, `_raw/`, `_archives/`, `_meta/`, `index.md`, `log.md`, `hot.md`, and `_insights.md` from candidates.
6. Read matching sections before full pages, and read full pages only when a factual claim cannot otherwise be established.

## Claim Ledger and Citation Audit

Draft a ledger before prose. Each item contains a claim, supporting `[[vault page]]` links, and one status: supported fact, inferred connection, or ambiguous conflict. Every factual sentence must have adjacent supporting citations. Mark inferred connections `^[inferred]`; mark unresolved conflicts `^[ambiguous]`. Never use web knowledge, model memory, or invented examples to close a gap. Omit unsupported claims and name the gap in Coverage.

## Persistence

Present the result by default. For `--save`, create `_readouts/` if necessary and write `_readouts/<slug>.md` with `title`, `topic`, `voice`, `sources`, `created`, and `updated` frontmatter. A readout is derived output: exclude `_readouts/` from retrieval and must not update `index.md` or `.manifest.json`.
```

Finish the skill with explicit output, logging, `hot.md`, and error rules from the approved design: a `## Coverage` footer; `WIKI_NARRATE` log entries; `hot.md` changes only after a successful save; safe no-match, weak-evidence, QMD-fallback, and write-failure behavior.

- [ ] **Step 3: Run the focused regression test**

Run:

```bash
python3 -m unittest tests/test_wiki_narrate_docs.py -v
```

Expected: the first four tests PASS; `test_routing_and_readme_expose_wiki_narrate` still FAILS because Task 3 has not updated public documentation.

- [ ] **Step 4: Commit the skill content**

```bash
git add .skills/wiki-narrate/SKILL.md .skills/wiki-narrate/references/voices.md
git commit -m "feat: add wiki narrate skill"
```

## Task 3: Publish the canonical entrypoint and complete verification

**Files:**
- Modify: `AGENTS.md:50-89`
- Modify: `README.md:420-438`
- Modify: `tests/test_wiki_narrate_docs.py`

**Interfaces:**
- Consumes: the canonical command and supported voice names defined by Task 2.
- Produces: discoverability through repository routing and documentation, guarded by the complete test suite.

- [ ] **Step 1: Add the routing row**

Insert this row immediately after the existing `wiki-query` routing row in `AGENTS.md`:

```markdown
| "narrate" / "briefing" / "explain this topic" / "/wiki-narrate" | `wiki-narrate` |
```

- [ ] **Step 2: Add the README skill-table entry**

Insert this row immediately after `wiki-query` in the README skills table:

```markdown
| `wiki-narrate`          | Render a cited narrative from a wiki topic        | `/wiki-narrate <topic>` |
```

- [ ] **Step 3: Run the focused contract test**

Run:

```bash
python3 -m unittest tests/test_wiki_narrate_docs.py -v
```

Expected: PASS, 5 tests.

- [ ] **Step 4: Run the repository test suite**

Run:

```bash
python3 -m unittest discover -s tests -p 'test_*.py' -v
```

Expected: PASS with no regressions.

- [ ] **Step 5: Perform manual skill-content checks**

Verify each condition with `rg` and inspect the resulting lines:

```bash
rg -n 'every factual sentence|\^\[inferred\]|\^\[ambiguous\]|web knowledge|model memory|exclude `_readouts/`|must not update `index.md` or `.manifest.json`' .skills/wiki-narrate/SKILL.md
rg -n "^## `(briefing|plain-language|lecturer)`$" .skills/wiki-narrate/references/voices.md
rg -n "wiki-narrate" AGENTS.md README.md
git diff --check HEAD~2..HEAD
```

Expected: all required contract phrases exist, the voice file has exactly three canonical headings, public routing appears once in each document, and `git diff --check` produces no output.

- [ ] **Step 6: Commit integration and tests**

```bash
git add AGENTS.md README.md tests/test_wiki_narrate_docs.py
git commit -m "docs: document wiki narrate"
```

## Plan Self-Review

- **Spec coverage:** Task 2 covers the command, the three voices, topic-only retrieval, citation rules, visibility/link-format requirements, derived-view persistence, logging, and failure paths. Task 3 covers the required routing and README exposure. Task 1 and Task 3 provide the automated and repository-wide verification required by the spec.
- **Placeholder scan:** No `TBD`, `TODO`, deferred implementation, or undefined interfaces remain.
- **Type consistency:** All tasks use the same canonical `briefing`, `plain-language`, `lecturer`, `--save`, `_readouts/<slug>.md`, and `WIKI_NARRATE` contract names.
