# ELF EXPRESS

## AI Collaboration Rules (Mid / High Level)

### 0. Context

- Project type: medium / large monorepo (frontend Nuxt 4 / Vue 3 / Tailwind, backend NestJS /
  TypeORM, Docker deployment).
- Goal: only perform structural changes (ops, UI, feature extension) when **understanding ≥ 90%**,
  to avoid "changing things we dont fully understand".

---

### 1. Global Principles (Mid / High Level)

1. **Understanding first, modification second**
    - Default behavior: **read, map, document first**, do not change code at the beginning.
    - Before any medium / large change (Docker, multi-module refactor, cross-cutting changes), there
      must be a clear:
        - Architecture description (overview)
        - Impact analysis
        - Rollback strategy (how to revert if it fails)

2. **Docs before code**
    - Every new task starts with a TODO / planning document (e.g. `docs/todo/todoYYYY-MM-DD-XX.md`).
    - After each small implementation step, **update TODO / docs first, then move on**.
    - Docs should answer:
        - Why do we change this?
        - What exactly changed?
        - How to verify that it works?

3. **Small steps, fast iterations**
    - Large tasks must be split into small steps, each step should be independently verifiable.
    - Avoid "one big PR / one big change that touches everything".

4. **Separate design phase from implementation phase**
    - Design phase: **only propose designs, do not touch files**, provide multiple options (A/B/C)
      with pros/cons.
    - Implementation phase only starts **after** the user explicitly says: "Use plan X and implement
      it."

5. **Respect existing architecture and team conventions**
    - Prefer reusing existing:
        - File naming patterns
        - Docs structure (`*_OVERVIEW.md`, `ui-flow-*.md`, etc.)
        - TODO templates
    - If changing conventions, first document:
        - Why the current way is not enough.
        - Benefits of the new approach.

6. **Tightly control high-risk areas**
    - Be extra careful with:
        - Deployment-related files (Dockerfile, docker-compose, CI/CD scripts).
        - Global configs (`nuxt.config.*`, `main.ts`, DB connection settings).
    - For these areas:
        - Always write a design note first.
        - Change only a small part at a time and document verification steps.

---

### 2. Three Collaboration Modes with AI

At any moment, the AI must know which mode it is in. **If the user does not specify, default to Mode
1: Explore / Understand Only.**

#### Mode 1: Explore / Understand Only

**When to use**

- First time looking at a module or feature.
- The user wants to understand "how it works" and **does not want any code changes yet**.

**AI behavior**

- Allowed actions:
    - Read code / configs / docs.
    - Extract architecture, data flows, call graphs.
    - Output markdown explanations (overviews, flows, mappings).
- **Forbidden**:
    - No file modifications.
    - No new / deleted code or config.
- Recommended output forms:
    - `PROJECT_OVERVIEW`, `API_OVERVIEW`, `FRONTEND_ARCH`, `ui-flow-*.md`, etc.
    - Tables mapping Route → Controller → Service → Entity.
    - A checklist of "understood / not yet understood" points to reveal blind spots.

---

#### Mode 2: Design / Propose Changes

**When to use**

- The user roughly understands a part of the system and wants to **change Docker, refactor
  architecture, adjust UI/UX**, but has not decided the exact solution yet.
- The user wants AI to propose "how to change".

**AI behavior**

- Allowed actions:
    - Propose 1–3 options (A/B/C). For each option, clearly describe:
        - What will be changed?
        - Pros / cons.
        - Risks.
        - Prerequisites.
    - Do not modify files yet, only produce implementation-ready blueprints.
- **Forbidden**:
    - No direct application of a plan to code.
    - No silent changes to Dockerfile / compose / configs.
- Recommended internal guideline for AI:
    - "This round outputs design only, no writes to the repo."
    - "Wait for the user to choose a plan before entering Mode 3."

---

#### Mode 3: Implement / Refactor

**When to use**

- The user explicitly says:
    - "You can start implementing now."
    - "Please follow plan A, and only do step 1 for now."
- The change scope is clearly defined (e.g. only a local dev Docker compose, or only one module).

**AI behavior**

- Before modifying:
    - Ensure there is a matching TODO file (e.g. `todoYYYY-MM-DD-XX.md`) describing goals and steps.
    - Ensure there is a design document (from Mode 2) to follow.
- During modification:
    - **Implement in small steps**, each step touching only a few files.
    - After each step, record:
        - What changed.
        - How to verify (commands, expected outputs).
- After modification:
    - Update the corresponding TODO / docs with:
        - What this step accomplished.
        - Command outputs / screenshots if relevant.
        - Any new issues that deserve new TODOs.

**Risk control**

- For Docker / deployment / DB migration changes:
    - Only change one path at a time (e.g. add `docker-compose.local.yml` without modifying the
      original `docker-compose.yml` yet).
    - Always keep a known-good path to roll back to the current stable behavior.
