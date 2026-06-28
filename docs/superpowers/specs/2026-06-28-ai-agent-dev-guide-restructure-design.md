# Design: Reposition the Embabel skill suite into a generic "AI Agent Dev" suite

**Date:** 2026-06-28
**Status:** Approved (pending written-spec review)

## Goal

Reposition the project from an *Embabel-specific* skill suite into a generic
**AI Agent application development** suite with two domains underneath:

- **Backend** — agentic backend logic. Currently one framework: Embabel.
  Structured so additional frameworks (LangChain, plain Spring AI, Node, …)
  can be added later as sibling skills.
- **Frontend** — LLM-generates-UI generative rendering (`json-render`),
  fully backend-agnostic and usable on its own.

This iteration is **rename + reposition + decouple the frontend** only. No
backend content is rewritten; no new framework is added yet.

## Naming

| Role | Before | After |
| ---- | ------ | ----- |
| repo / plugin / marketplace | `embabel-dev-guide` | `ai-agent-dev-guide` |
| router / entry skill | `embabel-dev-guide` | `ai-agent-dev-guide` |
| backend skill (pure Embabel, content unchanged) | `embabel-spring-ai-dev` | `embabel-agent-backend` |
| frontend skill (already agnostic, decoupled) | `embabel-generative-ui` | `json-render-ui` |

Naming rationale: `embabel-agent-backend` pairs with `json-render-ui` on the
backend ↔ ui axis without narrowing the backend's scope (it covers GOAP, tools,
RAG, orchestration — not just UI-spec generation).

## Target structure

```text
ai-agent-dev-guide/                      # repo / plugin / marketplace name
├─ .claude-plugin/
│  ├─ plugin.json                        # name + generic description
│  └─ marketplace.json                   # name, plugin name, skills[] paths
├─ skills/
│  ├─ ai-agent-dev-guide/                # router: backend framework ↔ frontend UI
│  ├─ embabel-agent-backend/             # backend framework #1 (Embabel) — content unchanged
│  └─ json-render-ui/                    # generative-UI frontend — standalone-capable
├─ docs/
└─ README*.md                            # 4 languages
```

## Per-skill changes

### ai-agent-dev-guide (router) — rewrite to generic

- Frontmatter `name` → `ai-agent-dev-guide`; `description` rewritten to a generic
  router for AI-agent app development (backend frameworks + generative-UI frontend).
- Routing table reframed: "backend agentic logic (currently Embabel) →
  `embabel-agent-backend`; LLM-generates-UI frontend → `json-render-ui`".
- New **Backend frameworks** subsection: Embabel is framework #1; explicitly
  states more frameworks can be added as sibling `*-agent-backend` skills.
- Full-stack build order updated to the new skill names.

### embabel-agent-backend (backend) — rename only

- Directory rename `embabel-spring-ai-dev/` → `embabel-agent-backend/`.
- Frontmatter `name` → `embabel-agent-backend`.
- Update the 3 navigational skill-name references (to `json-render-ui` and
  `ai-agent-dev-guide`).
- **All technical content unchanged** — GOAP, `@Agent`, `@Action`, references,
  section numbering (§13/§14/§15) stay as-is.

### json-render-ui (frontend) — rename + decouple framing

- Directory rename `embabel-generative-ui/` → `json-render-ui/`.
- Frontmatter `name` → `json-render-ui`; `description` rewritten to lead with
  **backend-agnostic / standalone-usable**; Embabel demoted from "companion" to
  "one worked example".
- "Division of labor when the backend is Embabel" section kept, reframed as a
  conditional: "If your backend is Embabel → see `embabel-agent-backend`".
- §13/§14/§15 cross-references updated to the new backend name.
- Add an explicit note: this skill is installable and usable **without** the
  backend skill.
- The backend-agnostic body and all `▼ Embabel example` markers stay unchanged.

## Cross-file mechanics

- 33 skill-name occurrences across 4 files (`embabel-spring-ai-dev/SKILL.md` ×3,
  `embabel-dev-guide/SKILL.md` ×19, `embabel-generative-ui/SKILL.md` ×10,
  `embabel-generative-ui/references/observability-and-interaction.md` ×1).
  Only **navigational** references (skill names / cross-links) are rewritten;
  in-content identifiers (`@Agent`, etc.) are not touched.
- README (en / zh-TW / zh-CN / ja): reposition the top-level framing from
  "Embabel suite" to "AI Agent dev guide (backend frameworks + generative-UI
  frontend)"; update the skill list and the install `skills/` paths.
- `.markdownlint-cli2.jsonc` is path-agnostic — no change needed.

## Future-framework extensibility

Adding a framework later is additive and isolated:

1. Add a sibling skill `skills/<framework>-agent-backend/`.
2. Add one row to the router table in `ai-agent-dev-guide`.
3. Add one entry to `marketplace.json` `skills[]`.

`json-render-ui` needs **no change** — it only depends on the backend satisfying
its 4-point Backend Contract (produce a `{root, elements}` spec; emit
`status/plan/step/chunk/complete/error` SSE events; emit step/stage progress;
aggregate cost).

## Out of scope

- Rewriting/genericizing backend content (rejected: would gut Embabel value).
- Adding any second backend framework now.
- Splitting the backend into "generic backend" + "embabel" sub-skills.
- Renaming the GitHub repository (user's action; marketplace uses relative `./`).

## Notes / constraints

- This working directory is **not a git repository**, so the design doc cannot
  be auto-committed. The user can `git init` / commit when ready.
- Directory renames are done via file moves.
