# AI Agent Dev Guide Restructure — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Reposition the Embabel-specific skill suite into a generic "AI Agent Dev" suite — rename the umbrella, rename the backend for naming consistency (content unchanged), and decouple the frontend into a standalone backend-agnostic `json-render-ui` skill.

**Architecture:** Three skills under one plugin. Rename three skill directories, rewrite the router skill to be framework-generic, lightly edit the backend (name + cross-refs only), reframe the frontend as standalone, then update the plugin manifests and 4 READMEs. Verification is by `grep` (zero stale skill-name tokens) and `markdownlint` (zero errors), since there is no code test framework here.

**Tech Stack:** Markdown skills, JSON plugin manifests, `markdownlint-cli2`, Git Bash (`mv`, `sed`, `grep`).

**Naming map:**

| Role | Before | After |
| ---- | ------ | ----- |
| repo / plugin / marketplace / router skill | `embabel-dev-guide` | `ai-agent-dev-guide` |
| backend skill | `embabel-spring-ai-dev` | `embabel-agent-backend` |
| frontend skill | `embabel-generative-ui` | `json-render-ui` |

**Tokens that must NOT change:** `embabel-json-render` (POC name), `learn-embabel` (course name), all code identifiers (`@Agent`, `@Action`, etc.).

> **Git note:** this working directory is **not** a git repo, so the `git add/commit` steps below cannot run. Treat each "Commit" step as a **verification checkpoint** instead (run the stated `grep`/`markdownlint` check, confirm clean). If the user runs `git init` later, the commit commands become valid as written.

---

## File Structure

- `skills/embabel-spring-ai-dev/` → `skills/embabel-agent-backend/` (rename dir; edit `SKILL.md` only)
- `skills/embabel-generative-ui/` → `skills/json-render-ui/` (rename dir; edit `SKILL.md` + `references/observability-and-interaction.md`)
- `skills/embabel-dev-guide/` → `skills/ai-agent-dev-guide/` (rename dir; full rewrite of `SKILL.md`)
- `.claude-plugin/plugin.json` (rewrite name + description)
- `.claude-plugin/marketplace.json` (rewrite name, plugin name, description, `skills[]` paths)
- `README.md`, `README.zh-TW.md`, `README.zh-CN.md`, `README.ja.md` (token rename + reposition framing)

---

## Task 1: Rename the backend skill (content unchanged)

**Files:**
- Move: `skills/embabel-spring-ai-dev/` → `skills/embabel-agent-backend/`
- Modify: `skills/embabel-agent-backend/SKILL.md` (3 lines)

- [ ] **Step 1: Move the directory**

Run:
```bash
mv skills/embabel-spring-ai-dev skills/embabel-agent-backend
```

- [ ] **Step 2: Update the frontmatter name**

In `skills/embabel-agent-backend/SKILL.md`, replace line 2:
- Old: `name: embabel-spring-ai-dev`
- New: `name: embabel-agent-backend`

- [ ] **Step 3: Update the "where to start" cross-reference**

In the same file, replace this blockquote line:
- Old: `> **Not sure where to start?** The entry point and routing for the entire Embabel skill suite is in **`embabel-dev-guide`** (which includes the sub-skill routing table and the full-stack development order). This skill focuses on the backend agent/GOAP/JVM side.`
- New: `> **Not sure where to start?** The entry point and routing for the entire AI Agent Dev suite is in **`ai-agent-dev-guide`** (which includes the routing table and the full-stack build order). This skill focuses on the backend agent/GOAP/JVM side.`

- [ ] **Step 4: Update the frontend-pairing cross-reference**

In the same file, replace this blockquote line:
- Old: `> **Pairing with frontend skills**: When this Embabel backend needs a dynamic frontend built around "natural language → agent-generated UI spec → progressive frontend rendering" (json-render / SSE streaming / progress-and-cost observability panel / click-to-drill-down / component catalog maintenance), use the companion skill **`embabel-generative-ui`** instead. This skill covers the agent/GOAP/JVM; that one covers the frontend/backend UI contract, streaming rendering, and interaction.`
- New: `> **Pairing with the frontend skill**: When this Embabel backend needs a dynamic frontend built around "natural language → agent-generated UI spec → progressive frontend rendering" (json-render / SSE streaming / progress-and-cost observability panel / click-to-drill-down / component catalog maintenance), use the companion skill **`json-render-ui`**. This skill covers the agent/GOAP/JVM; that one covers the frontend/backend UI contract, streaming rendering, and interaction.`

- [ ] **Step 5: Verify no stale tokens remain in the backend skill**

Run:
```bash
grep -rn "embabel-spring-ai-dev\|embabel-dev-guide\|embabel-generative-ui" skills/embabel-agent-backend/
```
Expected: **no output** (exit 1). Any hit is a miss to fix.

- [ ] **Step 6: Commit (checkpoint)**

```bash
git add skills/embabel-agent-backend
git commit -m "refactor: rename embabel-spring-ai-dev → embabel-agent-backend"
```
(If not a git repo: confirm Step 5 is clean and move on.)

---

## Task 2: Rename + decouple the frontend skill

**Files:**
- Move: `skills/embabel-generative-ui/` → `skills/json-render-ui/`
- Modify: `skills/json-render-ui/SKILL.md`
- Modify: `skills/json-render-ui/references/observability-and-interaction.md`

- [ ] **Step 1: Move the directory**

Run:
```bash
mv skills/embabel-generative-ui skills/json-render-ui
```

- [ ] **Step 2: Mechanical token rename across the moved skill (safe — tokens are distinct)**

Run:
```bash
grep -rl "embabel-spring-ai-dev\|embabel-dev-guide\|embabel-generative-ui" skills/json-render-ui/ \
  | xargs sed -i \
      -e 's/embabel-spring-ai-dev/embabel-agent-backend/g' \
      -e 's/embabel-generative-ui/json-render-ui/g' \
      -e 's/embabel-dev-guide/ai-agent-dev-guide/g'
```
This rewrites all navigational cross-references. `embabel-json-render` and `learn-embabel` are untouched (different tokens).

- [ ] **Step 3: Update the frontmatter name**

In `skills/json-render-ui/SKILL.md`, replace line 2:
- Old: `name: json-render-ui` _(already correct? No — the name was `embabel-generative-ui`; sed above did not touch the frontmatter `name:` value because it equals the dir token. Verify and set it explicitly.)_
- Ensure line 2 reads exactly: `name: json-render-ui`

> Note: the `sed` in Step 2 already converts `embabel-generative-ui` → `json-render-ui` everywhere including the frontmatter `name:`. This step is a confirmation, not a second edit.

- [ ] **Step 4: Rewrite the description to lead with standalone/agnostic**

In `skills/json-render-ui/SKILL.md` frontmatter, replace the entire `description:` value with:

```
description: "Use when building a FRONTEND that renders UI generated dynamically by an LLM backend — natural language → backend produces a JSON UI spec → progressively rendered dashboard. Backend-agnostic and usable on its own with any backend (LangChain, plain Spring AI, Node, Embabel, …) that produces the spec shape and emits progress/cost events. Covers json-render flat element-tree specs, a component catalog ↔ frontend registry, SSE streaming (fetch + ReadableStream, not EventSource), lenient partial-spec rendering, backend step-progress + cost observability panels, deterministic vs LLM rendering tradeoffs, click-to-drill-down interaction, and a catalog maintenance page. Worked examples use an Embabel + Spring AI backend; if your backend is Embabel, embabel-agent-backend is the companion backend skill."
```

- [ ] **Step 5: Add a standalone-usability note under the intro**

In `skills/json-render-ui/SKILL.md`, find the existing backend-agnostic blockquote that begins `> **Backend-agnostic (important)**:` and append this sentence to the end of that blockquote (same line, after the final period):
- Append: ` This skill is installable and usable **on its own** — you do not need the backend skill; any backend meeting the "Backend contract" below works.`

- [ ] **Step 6: Reframe the "Division of labor" heading to a conditional**

In `skills/json-render-ui/SKILL.md`, replace the section heading:
- Old: `## Division of labor when the backend is Embabel`
- New: `## If your backend is Embabel (optional)`

(The table and surrounding text below it already read correctly after the Step 2 token rename — they now point to `embabel-agent-backend`.)

- [ ] **Step 7: Verify no stale tokens remain in the frontend skill**

Run:
```bash
grep -rn "embabel-spring-ai-dev\|embabel-dev-guide\|embabel-generative-ui" skills/json-render-ui/
```
Expected: **no output** (exit 1).

- [ ] **Step 8: Verify the agnostic-but-kept tokens are still present**

Run:
```bash
grep -rn "embabel-agent-backend\|▼ Embabel example\|embabel-json-render" skills/json-render-ui/ | head
```
Expected: matches for all three (cross-refs retargeted, example markers kept, POC name kept).

- [ ] **Step 9: Commit (checkpoint)**

```bash
git add skills/json-render-ui
git commit -m "refactor: rename embabel-generative-ui → json-render-ui, decouple framing"
```

---

## Task 3: Rewrite the router skill

**Files:**
- Move: `skills/embabel-dev-guide/` → `skills/ai-agent-dev-guide/`
- Replace: `skills/ai-agent-dev-guide/SKILL.md` (full rewrite)

- [ ] **Step 1: Move the directory**

Run:
```bash
mv skills/embabel-dev-guide skills/ai-agent-dev-guide
```

- [ ] **Step 2: Overwrite `skills/ai-agent-dev-guide/SKILL.md` with the new generic router**

Write this exact content:

````markdown
---
name: ai-agent-dev-guide
description: "Use when starting, scoping, or reviewing any AI agent application — backend agent logic and/or an LLM-generates-UI frontend — and you are not yet sure which skill to open. This is the entry-point / router for the AI Agent Dev suite. It dispatches to backend framework skills (currently Embabel via embabel-agent-backend — GOAP / @Agent / @Action / Spring AI / JVM) and to the backend-agnostic generative-UI frontend (json-render-ui — LLM-produced JSON UI spec, SSE streaming, progressive rendering), and defines the recommended full-stack build order. Triggers: building an AI agent backend, a dynamic LLM-generated dashboard, SSE streaming of agent progress, or work spanning both backend and frontend."
---

# AI Agent Dev Guide (suite overview / router)

This is the **entry point for the AI Agent Dev suite**. When you want to build an AI agent application but aren't yet sure which skill to open, start here to decide the angle, then move on to the matching skill. This skill **only points the way and defines the full-stack build order**; all implementation details live in the domain skills.

The suite splits into two domains:

- **Backend** — agentic backend logic. Current framework: **Embabel** (`embabel-agent-backend`). Designed so additional frameworks can be added later as sibling `*-agent-backend` skills.
- **Frontend** — **`json-render-ui`**: a backend-agnostic, LLM-generates-UI generative rendering frontend. Usable on its own with any backend that meets its Backend Contract.

Evidence sources: the `learn-embabel` course (backend) plus the `embabel-json-render` POC (frontend). Version baseline (Embabel backend): Spring Boot 3.5.x + Embabel 0.4.0 + Spring AI 1.1.x (**Boot 4 is not yet supported — wait for Embabel 2.0**; see the backend skill for details).

## Routing table

| Your need | Which skill |
| ---- | ---------- |
| `@Agent`/`@Action`/`@AchievesGoal`, GOAP planning, Blackboard, type-driven routing (Embabel) | **embabel-agent-backend** |
| Domain `@Tool`, MCP, Agentic RAG, `@Condition`/SpEL gating (Embabel) | **embabel-agent-backend** |
| `@State` loops / human-in-the-loop (`WaitFor`), streaming output, cost tracking and budget guardrails (Embabel) | **embabel-agent-backend** |
| Autonomy `chooseAndRunAgent` / `AgentInvocation`, multi-agent orchestration and fusion, intent parameterization (Embabel) | **embabel-agent-backend** |
| Spring Boot / Spring AI wiring, version compatibility, testing, observability and failure handling (Embabel) | **embabel-agent-backend** |
| A dynamic dashboard for "natural language → backend generates a UI spec → frontend progressive rendering" | **json-render-ui** |
| json-render flat element-tree spec contract, component catalog ↔ frontend registry reconciliation | **json-render-ui** |
| SSE streaming (`fetch` + `ReadableStream`), `lenientParse`/`sanitize` fault-tolerant progressive rendering | **json-render-ui** |
| Backend step-progress / cost observability panel, click-to-drill-down, catalog maintenance page | **json-render-ui** |

## Backend frameworks

The backend domain is framework-pluggable. Today there is one:

| Framework | Skill | Status |
| ---- | ---- | ---- |
| Embabel (Spring AI / JVM, GOAP) | **embabel-agent-backend** | Available |
| _(others, e.g. LangChain, plain Spring AI, Node)_ | _(future sibling `*-agent-backend` skills)_ | Planned |

Adding a framework is additive: drop a sibling backend skill, add a row here, and add it to the plugin manifest. **`json-render-ui` needs no change** — it only requires the backend to satisfy its 4-point Backend Contract.

## Should you use an agent framework like Embabel? (decide before you start)

- **A good fit**: the flow has multiple steps, typed business objects, conditional branching, review gates, or audit requirements.
- **Not needed**: a single model call or a fixed pipeline → plain Spring AI (or your platform's LLM SDK) is enough; don't bolt on GOAP.
- For the detailed decision flow, see step 1 (Fit Assessment) in `embabel-agent-backend`.

## Full-stack build order (backend first, frontend second)

Backend only → use your backend framework skill (currently `embabel-agent-backend`). Frontend only → use `json-render-ui` (it works with any conformant backend). For a complete full-stack dynamic application, proceed in order:

1. **Design the backend flow** (`embabel-agent-backend`): use GOAP to design the action table, define Java record boundaries, implement `@Agent`/`@Action`, wire up Spring Boot/Spring AI, attach tools, and write tests.
2. **Nail down the frontend/backend spec contract** (`json-render-ui`): backend `{root, elements}` spec (▼ Embabel: a `DashboardSpec(root, elements)` record) ↔ frontend `Spec` type — lock the flat element-tree shape first.
3. **Wire SSE streaming + progressive rendering** (`json-render-ui`): backend opens SSE (▼ Embabel: `SseEmitter`); frontend `fetch`+`ReadableStream`, `lenientParse → sanitize → progressive setSpec`.
4. **Observability and interaction** (`json-render-ui`): step-progress + cost panel (▼ Embabel: GOAP plan/step), click-to-drill-down, catalog maintenance page.

> Cross-skill iron rules at a glance (details in the domain skills): **data arrays are always assembled deterministically into the spec by backend code, never produced by the LLM** (which drops items → empty table, breaks the JSON, tampers with numbers); the LLM only touches narrative text. **(Embabel) GOAP plan/action lifecycle events only go through the global listener**, not a per-call listener.

## Domain split

| Topic | Which to read |
| ---- | ---------- |
| Agent / GOAP / tools / backend orchestration and observability (Embabel) | **embabel-agent-backend** |
| Frontend/backend UI contract / streaming rendering / observability panel / drill-down / catalog maintenance | **json-render-ui** |

The skills are cross-linked; this guide is their shared upper-level entry point. When the requirement is vague or spans frontend and backend, dispatching from here is the fastest route.
````

- [ ] **Step 3: Verify the router has no stale tokens**

Run:
```bash
grep -rn "embabel-spring-ai-dev\|embabel-dev-guide\|embabel-generative-ui" skills/ai-agent-dev-guide/
```
Expected: **no output** (exit 1). (`embabel-agent-backend`, `embabel-json-render`, `learn-embabel`, `Embabel` prose are fine.)

- [ ] **Step 4: Commit (checkpoint)**

```bash
git add skills/ai-agent-dev-guide
git commit -m "refactor: rewrite router as generic ai-agent-dev-guide"
```

---

## Task 4: Update the plugin manifests

**Files:**
- Replace: `.claude-plugin/plugin.json`
- Replace: `.claude-plugin/marketplace.json`

- [ ] **Step 1: Overwrite `.claude-plugin/plugin.json`**

```json
{
  "name": "ai-agent-dev-guide",
  "description": "AI agent application development skill suite: a routing entry-point skill, pluggable backend framework skills (currently Embabel — GOAP / @Agent / JVM), and a backend-agnostic generative-UI frontend (LLM-generated dynamic dashboards, SSE streaming, progressive rendering).",
  "version": "0.1.0",
  "author": {
    "name": "kevintsai",
    "email": "kevintsai1202@gmail.com"
  }
}
```

- [ ] **Step 2: Overwrite `.claude-plugin/marketplace.json`**

```json
{
  "name": "ai-agent-dev-guide",
  "version": "0.1.0",
  "description": "Marketplace for the AI Agent Dev skill suite.",
  "owner": {
    "name": "kevintsai",
    "email": "kevintsai1202@gmail.com"
  },
  "plugins": [
    {
      "name": "ai-agent-dev-guide",
      "source": "./",
      "description": "AI agent development skills: routing entry-point (ai-agent-dev-guide), backend agent framework (embabel-agent-backend — Embabel / GOAP / JVM), and backend-agnostic generative UI frontend (json-render-ui).",
      "strict": false,
      "skills": [
        "./skills/ai-agent-dev-guide",
        "./skills/embabel-agent-backend",
        "./skills/json-render-ui"
      ]
    }
  ]
}
```

- [ ] **Step 3: Verify both JSON files parse and skill paths exist**

Run:
```bash
node -e "JSON.parse(require('fs').readFileSync('.claude-plugin/plugin.json')); JSON.parse(require('fs').readFileSync('.claude-plugin/marketplace.json')); console.log('json ok')"
for d in ai-agent-dev-guide embabel-agent-backend json-render-ui; do test -f "skills/$d/SKILL.md" && echo "ok skills/$d" || echo "MISSING skills/$d"; done
```
Expected: `json ok`, then three `ok skills/...` lines.

- [ ] **Step 4: Commit (checkpoint)**

```bash
git add .claude-plugin/plugin.json .claude-plugin/marketplace.json
git commit -m "refactor: rename plugin + marketplace to ai-agent-dev-guide"
```

---

## Task 5: Update the READMEs (4 languages)

**Files:**
- Modify: `README.md`, `README.zh-TW.md`, `README.zh-CN.md`, `README.ja.md`

- [ ] **Step 1: Mechanical token rename across all four READMEs**

Run:
```bash
sed -i \
  -e 's/embabel-spring-ai-dev/embabel-agent-backend/g' \
  -e 's/embabel-generative-ui/json-render-ui/g' \
  -e 's/embabel-dev-guide/ai-agent-dev-guide/g' \
  README.md README.zh-TW.md README.zh-CN.md README.ja.md
```
This fixes every slug: skill-file links, `<your-github-account>/…` repo shorthand, `/plugin install …@…`, the directory-tree root and skill subdirs. `embabel-json-render` and `learn-embabel` are untouched.

- [ ] **Step 2: Reposition the English `README.md` H1 + intro**

In `README.md`, apply these exact replacements:

Replace the H1:
- Old: `# Embabel Dev Guide — A Claude Code Skill Suite`
- New: `# AI Agent Dev Guide — A Claude Code Skill Suite`

Replace the intro paragraph:
- Old: `A set of Claude Code skills for turning business processes into testable **Embabel + Spring AI** agent applications — covering the full stack from the backend agent (GOAP / `@Agent` / JVM) to a frontend that generates UI dynamically (LLM produces a spec → SSE streaming → progressively rendered dashboard).`
- New: `A set of Claude Code skills for building **AI agent applications** — covering the full stack from the agentic backend (pluggable per framework; currently **Embabel + Spring AI**: GOAP / `@Agent` / JVM) to a backend-agnostic frontend that renders UI generated by an LLM (backend produces a spec → SSE streaming → progressively rendered dashboard).`

Replace the closing line:
- Old: `All three cross-reference each other; when the requirement is ambiguous or spans both tiers, dispatch from `ai-agent-dev-guide`.`
- New: `All three cross-reference each other; when the requirement is ambiguous or spans both tiers, dispatch from `ai-agent-dev-guide`. The frontend (`json-render-ui`) is also usable on its own with any conformant backend.`

(The closing line already says `ai-agent-dev-guide` after Step 1's sed.)

- [ ] **Step 3: Reposition the English `README.md` "Skills at a glance" table rows**

Replace the three table rows so the layer labels and descriptions are framework-generic:
- Router row description — Old fragment: `"should I use Embabel" decision` → New fragment: `"should I use an agent framework" decision`.
- Backend row — Old `Layer` cell `Backend` → New `Backend (Embabel)`.
- Frontend row — Old `GOAP progress/cost panels` → New `backend progress/cost panels`.

Concretely, replace the table body (the three `| [...] |` rows under the header) with:
```markdown
| [`ai-agent-dev-guide`](skills/ai-agent-dev-guide/SKILL.md) | **Entry / Router** | Start here when unsure which skill to use: routing table, "should I use an agent framework" decision, backend-framework list, full-stack build order |
| [`embabel-agent-backend`](skills/embabel-agent-backend/SKILL.md) | Backend (Embabel) | `@Agent`/`@Action`/`@AchievesGoal`, GOAP planning, Blackboard, domain `@Tool`, MCP/RAG, `@Condition` gates, `@State` loops, cost guardrails, multi-agent orchestration, testing & observability |
| [`json-render-ui`](skills/json-render-ui/SKILL.md) | Frontend (backend-agnostic) | json-render flat element-tree spec contract, component catalog ↔ registry reconciliation, SSE streaming (`fetch`+`ReadableStream`), lenient progressive rendering, backend progress/cost panels, click-to-drill-down, catalog maintenance page |
```

- [ ] **Step 4: Reposition the three translated READMEs to match Step 2–3**

For `README.zh-TW.md`, `README.zh-CN.md`, `README.ja.md`: read each file's current H1, intro paragraph, "Skills at a glance" table, and closing line, then apply the **same repositioning as Steps 2–3 expressed in that file's language** (zh-TW Traditional Chinese, zh-CN Simplified Chinese, ja Japanese). Use the English new-text from Steps 2–3 as the reference meaning:
  - H1 "Embabel Dev Guide" → "AI Agent Dev Guide" (keep the product name in English; translate any trailing descriptor).
  - Intro: shift from "Embabel + Spring AI agent applications" to "AI agent applications (pluggable backend; currently Embabel + Spring AI) + backend-agnostic LLM-generated UI frontend".
  - Backend table layer label → "Backend (Embabel)" equivalent; frontend label → "Frontend (backend-agnostic)" equivalent; "GOAP progress/cost" → "backend progress/cost".
  - Closing line: add "the frontend is also usable standalone with any conformant backend".

- [ ] **Step 5: Verify no stale slugs remain in any README**

Run:
```bash
grep -rn "embabel-spring-ai-dev\|embabel-dev-guide\|embabel-generative-ui" README.md README.zh-TW.md README.zh-CN.md README.ja.md
```
Expected: **no output** (exit 1).

- [ ] **Step 6: Verify markdownlint is still clean**

Run:
```bash
npx -y markdownlint-cli2 "**/*.md" "#node_modules" 2>&1 | grep "Summary"
```
Expected: `Summary: 0 error(s)`.

- [ ] **Step 7: Commit (checkpoint)**

```bash
git add README.md README.zh-TW.md README.zh-CN.md README.ja.md
git commit -m "docs: reposition READMEs for the AI Agent Dev suite"
```

---

## Task 6: Final whole-repo verification

**Files:** none (verification only)

- [ ] **Step 1: Zero stale skill slugs anywhere (excluding the design/plan docs that intentionally cite the old names)**

Run:
```bash
grep -rn "embabel-spring-ai-dev\|embabel-generative-ui" . \
  --include=*.md --include=*.json --include=*.yaml \
  | grep -v "docs/superpowers/"
grep -rn "embabel-dev-guide" . \
  --include=*.md --include=*.json --include=*.yaml \
  | grep -v "docs/superpowers/"
```
Expected: **no output** from both (exit 1). The `docs/superpowers/` specs/plans intentionally record the rename and are excluded.

- [ ] **Step 2: All three new skill directories exist with a SKILL.md and old ones are gone**

Run:
```bash
ls skills/
for d in ai-agent-dev-guide embabel-agent-backend json-render-ui; do test -f "skills/$d/SKILL.md" && echo "ok $d"; done
for d in embabel-dev-guide embabel-spring-ai-dev embabel-generative-ui; do test -e "skills/$d" && echo "STALE $d STILL EXISTS"; done
```
Expected: `skills/` lists exactly the three new dirs; three `ok …` lines; no `STALE …` lines.

- [ ] **Step 3: Frontmatter `name:` matches directory for each skill**

Run:
```bash
for d in ai-agent-dev-guide embabel-agent-backend json-render-ui; do
  printf "%-22s " "$d"; grep -m1 "^name:" "skills/$d/SKILL.md"
done
```
Expected: each line's `name:` equals its directory (`ai-agent-dev-guide`, `embabel-agent-backend`, `json-render-ui`).

- [ ] **Step 4: markdownlint clean across the repo**

Run:
```bash
npx -y markdownlint-cli2 "**/*.md" "#node_modules" 2>&1 | grep "Summary"
```
Expected: `Summary: 0 error(s)`.

- [ ] **Step 5: Plugin manifests valid and point at existing skills**

Run:
```bash
node -e "const m=JSON.parse(require('fs').readFileSync('.claude-plugin/marketplace.json'));const fs=require('fs');m.plugins[0].skills.forEach(p=>console.log((fs.existsSync(p+'/SKILL.md')?'ok ':'MISSING ')+p))"
```
Expected: three `ok ./skills/…` lines, no `MISSING`.

- [ ] **Step 6: Final commit (checkpoint)**

```bash
git add -A
git commit -m "refactor: complete AI Agent Dev suite restructure"
```

---

## Self-Review notes

- **Spec coverage:** naming map (Task 1–4), router rewrite incl. Backend-frameworks section (Task 3), backend rename-only (Task 1), frontend decouple incl. standalone note + conditional heading (Task 2), manifests (Task 4), 4 READMEs (Task 5), future-extensibility documented in router (Task 3 Step 2), verification (Task 6). All spec sections mapped.
- **Out-of-scope honored:** no backend content rewrite, no second framework added, no repo-on-GitHub rename (only the marketplace `name`, which uses relative `source: "./"`).
- **Token safety:** every rename uses distinct tokens; `embabel-json-render` / `learn-embabel` / `Embabel` (prose) deliberately preserved and asserted in Task 2 Step 8.
