# AI Engineer Roadmap Scaffold Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create a GitHub-ready, roadmap-aligned AI Engineer learning repository with repeatable module structure and a capstone workspace.

**Architecture:** The repository separates the two learning paths by sequence: an eight-week System Design path leads into an advanced AI and cloud path. Both contribute artifacts to one Enterprise Document RAG Assistant capstone. Markdown templates enforce consistent learning outputs without prescribing implementation technology.

**Tech Stack:** Markdown, Git, GitHub conventions, Python/Jupyter-oriented ignore rules.

**Spec:** `docs/superpowers/specs/2026-08-29-ai-engineer-roadmap-design.md`

## Global Constraints

- Keep the two source roadmap documents unchanged and move them to `roadmap/`.
- Make System Design the primary eight-week path; advanced AI/cloud is the follow-on path.
- Every module must expose knowledge, artifact, measurement, and decision outputs.
- Never commit secrets, generated Python artifacts, virtual environments, or Jupyter checkpoints.
- Do not create or push a GitHub remote without an explicit user destination or authorization.

---

### Task 1: Create the curriculum and capstone skeleton

**Files:**
- Create: `learning-paths/00-setup/.gitkeep`
- Create: `learning-paths/01-system-design-rag/week-00-environment/` through `week-08-deployment-portfolio/`
- Create: `learning-paths/02-advanced-ai-cloud/01-model-foundations/` through `08-iac-cicd-reliability-finops/`
- Create: `capstone/enterprise-document-rag-assistant/{app,docs,evaluation,experiments,infrastructure}/.gitkeep`
- Move: source roadmaps into `roadmap/`

**Interfaces:**
- Consumes: ordered learning-path names from the design specification.
- Produces: a navigable directory hierarchy used by root documentation and templates.

- [ ] **Step 1: Create every named directory and add `.gitkeep` to empty directories.**
- [ ] **Step 2: Move both source roadmap files into `roadmap/` without changing their content.**
- [ ] **Step 3: Verify `week-08-deployment-portfolio`, `08-iac-cicd-reliability-finops`, and the capstone `infrastructure` folder exist with `test -d`.**
- [ ] **Step 4: Leave the new skeleton unstaged; it will be included in the single initial commit after full validation.**

### Task 2: Add repeatable learning templates

**Files:**
- Create: `templates/learning-session.md`
- Create: `templates/weekly-module.md`
- Create: `templates/architecture-decision-record.md`
- Create: `templates/experiment.md`
- Create: `templates/project-readme.md`

**Interfaces:**
- Consumes: the four-output module contract in the design specification.
- Produces: reusable Markdown files for study sessions, modules, decisions, experiments, and capstone documentation.

- [ ] **Step 1: Write `weekly-module.md` with headings for Learning goals, Knowledge note, Lab and artifact, Measurements, Decision record, Reflection, and Resources.**
- [ ] **Step 2: Write the session template with topic, available time, learning mode, questions, practice, and understanding check.**
- [ ] **Step 3: Write ADR, experiment, and project templates with purpose, evidence, results, trade-offs, and next action.**
- [ ] **Step 4: Verify `weekly-module.md` contains the headings `Knowledge note`, `Measurements`, `Decision record`, and `Reflection` with `rg -q`.**
- [ ] **Step 5: Leave the new templates unstaged; they will be included in the single initial commit after full validation.**

### Task 3: Add GitHub-facing documentation and safe defaults

**Files:**
- Create: `README.md`
- Create: `CONTRIBUTING.md`
- Create: `resources/system-design-primer.md`
- Create: `resources/ai-engineering.md`
- Create: `resources/cloud-azure.md`
- Create: `.gitignore`
- Create: `.github/.gitkeep`
- Create: `LICENSE`

**Interfaces:**
- Consumes: learning paths, templates, and capstone folders.
- Produces: GitHub navigation, contribution guidance, resource usage notes, and safe ignore rules.

- [ ] **Step 1: Write the README with roadmap links, learning sequence, progress checklist, module contract, capstone milestones, and template links.**
- [ ] **Step 2: Write CONTRIBUTING and resource guides explaining the learning loop: understand, design, implement, measure, explain.**
- [ ] **Step 3: Add `.gitignore` entries for `.env`, `.env.*`, `__pycache__/`, `*.py[cod]`, `.venv/`, `venv/`, `.ipynb_checkpoints/`, `.DS_Store`, and common IDE metadata. Use MIT license text.**
- [ ] **Step 4: Verify README references both roadmaps and the weekly-module template, then check representative ignored paths with `git check-ignore -v`.**
- [ ] **Step 5: Leave the new documentation unstaged; it will be included in the single initial commit after full validation.**

### Task 4: Initialize and validate local Git history

**Files:**
- Create: `.git/` with Git initialization.
- Modify: all scaffold files by staging them as the initial history.

**Interfaces:**
- Consumes: the complete scaffold from Tasks 1–3.
- Produces: a validated local Git repository ready to connect to GitHub.

- [ ] **Step 1: Run `git init` and set the default branch to `main`.**
- [ ] **Step 2: Verify required roadmap and template files with `test -f`, check ignore behavior for `.env`, Python bytecode, and `.venv`, then inspect `git status --short`.**
- [ ] **Step 3: Stage intended files and commit with `git commit -m "docs: initialize AI engineer learning roadmap"`.**
- [ ] **Step 4: Run `git status --short --branch`, `git log -1 --oneline`, and `git remote -v`; confirm the branch is clean, a commit exists, and no remote is configured before the user chooses one.**
