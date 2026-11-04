# ADR-0001: Working model with Claude

- **Status:** Accepted
- **Date:** 2026-11-03
- **Scope:** Whole project (how I work with Claude: context, sessions, memory, access levels, tool learning)

## Context

My previous project (DevOps thesis: Terraform/AWS, Ansible, Jenkins, Prometheus/Grafana) had good notes but a broken context architecture:

- The root `CLAUDE.md` imported the full work log, plan and guide (~67 KB, about 20–25k tokens) into every session. The current state was lost in history.
- Some imports never worked: a glob (`@etap-*-podsumowanie.md`) and a path with spaces. The repo files imported `@../CLAUDE.md`, which pointed to a file that did not exist. So some sessions got almost no shared context.
- The current state lived in 4 places (work log, stage summaries, "assumption change log", README). The versions drifted, and once a stage was marked "closed" with a missing deliverable.
- Handoff between threads was manual, in sections that went stale quickly.
- Files had mixed roles (the prompting guide was also the decision log). The docs folder was not in git.
- Safety rules existed only as prose in `CLAUDE.md`, which Claude treats as context, not as enforcement.

## Decisions

### A1. Where the work lives — repo is the single source of truth (W2)

- **Execution:** Claude Code locally (Windows) and cloud sessions started from the phone for repo-only tasks.
- **Thinking and learning:** chat. Optionally a thin claude.ai Project with the mentor role in its instructions only: no files, no project state.
- **Monitoring:** Code tab in the mobile app, GitHub PRs, `docs/STATE.md` read on GitHub.
- A decision exists only when it is in the repo. Information flows one way: chat → repo.

### A2.1. Repo layout — monorepo now (R3)

- One repo with `infra/`, `platform/`, `app/`, `simulators/`, `docs/`, `notes/`.
- Decision point in the roadmap: "split `app/` into its own repo?" when the app gets its own CI/CD. No split by default.

### A2.2. File roles and loading

Each fact has **one owner**. Other places only link to it.

```
CLAUDE.md            rules + map "where is what"            always loaded
docs/STATE.md        current state only (overwrite, not append)  always loaded (import)
docs/ROADMAP.md      milestones, "done" criteria, acceptance tests  on demand
docs/adr/README.md   index, one line per ADR                on demand
docs/adr/NNNN-*.md   immutable once accepted; changed only by a new ADR (superseded)
docs/lessons/        bugs with root cause                   on demand
docs/runbooks/       rebuild / restore / operate / session checklist  on demand
notes/               my cheat sheets (learning material)    never auto-loaded (not denied)
.claude/rules/       area rules with `paths:`               loaded when matching files are used
```

- **History (H1):** git only (commits, PR descriptions). No separate work log.
- **Loading (L1):** always loaded = `CLAUDE.md` (~100 lines) + `@docs/STATE.md` (~60 lines). **Budget: ~150 lines total.** If we go over it, we move content down, we do not raise the limit.
- Upgrade to a `SessionStart` hook (L2) happens in the "hooks" learning step (see A6).

### A2.3. Instruction hierarchy (K1)

- Working style (mentor role, Polish in conversation, English in docs, no boilerplate without agreement, challenge my ideas) and project facts → root `CLAUDE.md`.
- Area rules → `.claude/rules/*.md` with `paths:`.
- `~/.claude/CLAUDE.md` → machine-only notes or empty. `CLAUDE.local.md` → local machine notes only, nothing that correctness depends on.
- **No imports from outside the repo.**
- An instruction lives at the narrowest level that covers all its cases. A lower level never repeats a higher one.
- Hard safety rules are not prose: they go to `.claude/settings.json` (deny rules, hooks). `CLAUDE.md` only says that limits are enforced and must not be bypassed.

### A2.4. Keeping it current

- **Single writer (S2):** `STATE.md` is updated only on `main`, after a merge, by me or a local session. Cloud sessions never edit it. They add a **"State delta"** section to the PR description.
- A milestone is "done" **only after I confirm** that its acceptance test passed. Claude may propose closing, never mark it alone.
- `STATE.md` starts with `Last updated: <date>, <commit>`.
- Checks are introduced in stages:
  1. From day 1: end-of-session ritual, PR template with "State delta" and "New ADR? (number)".
  2. CI milestone: a CI check fails a code PR without a "State delta" (a label can skip it on purpose).
  3. Hooks milestone: `SessionStart` staleness check, `Stop` reminder.
- **Context audit** at every big checkpoint: `/context` (budget), `/doctor prompt-audit`, ADR index review.

### A3. Memory

- **Auto memory off for this repo (P1):** `"autoMemoryEnabled": false` in the committed `.claude/settings.json`. Reason: it is machine-local, not visible in cloud sessions, and would become a hidden second state file. When Claude repeats a mistake, I add a rule to `CLAUDE.md` or `.claude/rules/`.
- **Account memory (M1):** only who I am, how I work, and a pointer: "OT lab project: source of truth is the repo". No project state or technical decisions. The thin chat Project uses memory connected to the account (one chat memory, not two).
- **Order of truth:** repo > current conversation > memory. When memory contradicts the repo, the repo wins and memory is fixed at once. Past chats are source material, not project state.

### A4. Session ritual

| Type | Shape | Hard rule |
|---|---|---|
| Phone, 5–15 min | Read `STATE.md`, review a PR, start a task from the Ready queue, theory in chat | No architecture decisions on the phone. Write them down as a question for the next session |
| 1 h | 5 min start, ~40 min one task with a "done" criterion, 10–15 min close | Stop at minute 45, even mid-task. Park the task, do not abandon it |
| 3 h | 2–3 tasks, each in its own Claude session (`/clear` between), commit after each | One task = one session |
| Long block | Checkpoint: rebuild from zero, context audit, ADR cleanup | Only here we plan the next roadmap stage |

- **Start (local):** one sentence "Session 1h, goal: X from STATE.md". Claude checks `Last updated` against `git log`. If they differ, align the state first.
- **End (local):** Claude proposes a `STATE.md` change, I accept it and commit it with the work. An interrupted task gets a `Resume here: <exact next step>` line at the top of `STATE.md`.
- **End (cloud):** a PR with "State delta". No `STATE.md` change.
- New lesson → `docs/lessons/`. New decision → ADR. Otherwise nothing extra.
- **Handoff:** a new thread starts from the repo only. Phone → cloud session → PR. Desktop → review on GitHub or teleport the session to the terminal (verify before first use).
- **Ready queue** section in `STATE.md`: 2–3 tasks that are well defined, repo-only and have an acceptance test. Only these are started from the phone.
- **After a break:** read `Resume here` and `git log --since`.
- **Rollout:** RT1 (checklist in `docs/runbooks/session.md`) for the first 2–3 sessions, then RT2 (`/start` and `/end` slash commands).

### A5. Access levels for Claude

**Principle:** the agent's boundary is enforced **outside the agent**: network, OS permissions, credential scope. Claude Code rules and hooks are the second line. `CLAUDE.md` is not a security control. **`bypassPermissions` is never used.**

**Where Claude runs:** Windows (E1) on levels 0–1. A dedicated **agent VM in the lab (E3)** from level 2. WSL2 (E2) is optional on level 1 if I want to practise the sandbox earlier.

| Level | Scope | Entry conditions | What I learn |
|---|---|---|---|
| 0 | Repo only (local + cloud) | Branch protection on `main` (PR only), no secrets in repo (secret scan), base deny rules | False reports ("tests pass"), closing too early, scope creep, `CLAUDE.md` is only a suggestion |
| 1 | Local execution (docker compose, tests, simulators on PC) | Level 0 + `ask` rules for docker; optional sandbox (WSL2) | How Bash bypasses Read/Edit rules; `docker.sock` risk |
| 2 | SSH to a sandbox VM in the lab, from the agent VM | VM rebuildable from IaC (one tested rebuild); snapshot before each session; own user and SSH key only for this VM; OPNsense: agent VM reaches only the sandbox; `PreToolUse` block hooks; narrow sudo | Live fixes instead of IaC (drift); restore from snapshot under pressure |
| 3 | Platform (Terraform via Proxmox API, Ansible on many VMs) | Proxmox API token limited to one resource pool, no node privileges; config and VM backups; one tested VM restore; `terraform apply/destroy` as `ask`, I read the plan | Token blast radius, plan → apply discipline, state fragility |
| 4 | Proxmox host and network (OPNsense) | Full lab restore runbook tested on the second Elite Mini; console access independent of the lab network | Locking myself out; real RTO of the lab |

- **Level 4 mode:** read-only diagnostics by default. Changes only in planned chaos exercises.
- **Phone:** level 0 via cloud sessions. From level 2 only via Remote Control to the agent VM, and I approve prompts on the phone.
- **Transition rule:** no level N+1 until I have seen Claude break something at level N and restored it with the runbook. Each transition ends with an ADR with lessons learned.

### A6. Learning Claude tools — trigger + latest deadline (T3)

| Tool | Trigger | Latest |
|---|---|---|
| Basics: plan mode, `/context`, `/clear`, deny rules, first cloud session | — | Repo start |
| Slash commands (`/start`, `/end`, maybe `/adr`) | Same prompt typed for the 3rd time | After 2–3 sessions |
| Hooks (`PreToolUse` blocks, `SessionStart` state) | Claude bypasses a rule or the state drifts | **Before level 2** (entry condition) |
| Subagents (e.g. read-only reviewer that checks a PR against its acceptance test) | I review big changes myself | App build phase |
| MCP (e.g. read-only access to the telemetry database) | I copy data from the DB into the chat | Data/monitoring phase |
| Cloud scheduled tasks (e.g. weekly STATE/dependency check) | After CI exists | CI/CD phase |
| Projects in Claude Code (W3) | I want to start several tasks from the phone | "Delegate from phone" milestone |

Each new tool is its own task with a "done" criterion. Its current behaviour is checked in the official docs at the time of adoption.

## Facts verified in the official docs (2026-11-02/03)

Features change fast: re-check before relying on them.

- `@` imports resolve relative to the importing file. Paths with spaces need a backslash before each space. Imports are paths, not globs. Imports do not reduce context cost. Target under 200 lines per `CLAUDE.md`.
- `CLAUDE.md` files in subdirectories and `.claude/rules/` with `paths:` load on demand. `/context` shows what is loaded.
- Cloud sessions: clone the repo, push a branch. They load the repo `CLAUDE.md` and `.claude/settings.json` hooks and permissions **only in a single-repo session**, and not `~/.claude`. Modes: Accept edits, Plan, Auto.
- The GitHub proxy blocks branch deletion and tag pushes but **does not limit which branches can be updated**, so branch protection on GitHub is needed.
- Threads in Claude Code Projects (beta) do not apply repo permission rules or hooks.
- Auto memory is machine-local and not shared with cloud environments.
- Deny rules apply in every permission mode. Read/Edit deny rules cover built-in file tools, not Bash subprocesses. The Bash sandbox runs on macOS, Linux and WSL2, not on native Windows.

## Consequences for the roadmap (Part B)

- **Day 1:** private GitHub repo, branch protection on `main`, `.gitattributes` (LF), secret scanning, `.claude/settings.json` with base deny rules and `autoMemoryEnabled: false`, PR template.
- **Agent VM (E3):** after the network milestone (OPNsense, segmentation), before level 2.
- **Hooks:** before level 2.
- **Decision points:** split `app/`? · W3 experiment · WSL2 sandbox on level 1 (optional).

## Rejected options

- **W1** (claude.ai Project as hub): copied knowledge drifts from the repo.
- **W3 now:** beta, threads skip repo permissions and hooks.
- **R2** (several repos): state and decisions split again, cloud sessions lose hooks.
- **H2** (journal): duplicated stage summaries in the past.
- **L2 / L3 now:** hook too early / too much context.
- **K2** (style in user `CLAUDE.md` + cloud setup script): two copies drift.
- **K3** (subdirectory `CLAUDE.md` instead of rules): fine, but rules are easier to audit in one place.
- **S1** (every session writes STATE): merge conflicts with parallel cloud sessions.
- **P2** (auto memory on + audit): hidden state most of the time.
- **M2** (keep account memory as is): stale facts in chat.
- **E1 / E2 for level 2+:** weaker boundary than a dedicated agent VM.
- **T1 / T2:** risk of a learning gap / artificial exercises.
