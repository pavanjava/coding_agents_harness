# The Harness

What is wired up in this repository, why it exists, and how to actually drive it.

> This document is about the **harness** — the skills, agents, commands, hooks, evals and plugin
> packaging that shape how an AI coding agent behaves here. For the upstream project pitch and
> install instructions, see [README.md](README.md). For the rules agents must follow when working
> on this repo, see [CLAUDE.md](CLAUDE.md) / [AGENTS.md](AGENTS.md).

---

## Why this exists

An AI coding agent left to its own devices is a competent but forgetful engineer. It will write
plausible code, skip the accessibility pass, forget that the grid it drew has to match the grid the
logic uses, declare victory without running anything, and produce a summary that sounds more
confident than the work deserves. Not because it is careless — because *nothing in the loop forces
otherwise*.

This harness is that forcing function. It encodes the parts of senior engineering that are usually
tacit — the checklist you run before merge, the second opinion you ask for, the question you ask
before writing code — as artifacts the agent has to read and follow. The value is not that the
instructions are novel. It is that they are **present at the moment of the decision**, every time,
instead of depending on the agent happening to remember.

Three properties make it work:

- **Explicit over implicit.** A skill is a file. It can be read, diffed, reviewed, and improved.
  "Be careful about accessibility" is a wish; `frontend-ui-engineering/SKILL.md` is a contract with
  a contrast threshold in it.
- **Adversarial over self-assessed.** An author is the worst reviewer of their own work. The
  personas in `agents/` exist so a *different* context, with a *different* brief, looks at the
  output and tries to find what is wrong with it.
- **Measured over asserted.** Skills that never trigger are decoration. `evals/` checks that each
  skill fires on the prompts users actually type, and that no two skills collide.

---

## The layers

| Layer | Where | What it is |
|---|---|---|
| **Skills** | `skills/<name>/SKILL.md` (24) | The workflows themselves. Each has Overview, When to Use, Process, Common Rationalizations, Red Flags, Verification. |
| **Commands** | `.claude/commands/*.md` (8) | Slash commands mapping the lifecycle to skills. Mirrored as `commands/*.toml` for other hosts. |
| **Agents** | `agents/*.md` (4) | Reviewer personas: `code-reviewer`, `test-engineer`, `security-auditor`, `web-performance-auditor`. |
| **Hooks** | `hooks/` | Lifecycle automation. `hooks.json` registers SessionStart; the cache and ignore hooks are opt-in. |
| **Evals** | `evals/`, `scripts/run-evals.js` | Three-tier check that the skills trigger, stay distinct, and change behaviour. 24 cases. |
| **References** | `references/*.md` (7) | Shared checklists — accessibility, security, performance, testing, observability, orchestration, definition-of-done. |
| **Scripts** | `scripts/` | Validators (skills, commands, versions, reference links, artifact paths) plus the eval runner. All CI-safe. |
| **Plugin** | `.claude-plugin/`, `plugin.json` | Marketplace packaging, `agent-skills` v0.6.7. |
| **Docs** | `docs/` (14) | Per-host setup guides and the skill-anatomy spec. |

### Skills, by phase

```
DEFINE     interview-me · idea-refine · spec-driven-development
PLAN       planning-and-task-breakdown
BUILD      incremental-implementation · test-driven-development · context-engineering
           source-driven-development · doubt-driven-development
           frontend-ui-engineering · api-and-interface-design
VERIFY     browser-testing-with-devtools · debugging-and-error-recovery
REVIEW     code-review-and-quality · code-simplification
           security-and-hardening · performance-optimization
SHIP       git-workflow-and-versioning · ci-cd-and-automation · deprecation-and-migration
           documentation-and-adrs · observability-and-instrumentation · shipping-and-launch
META       using-agent-skills
```

### Commands

| Phase | Command | Principle |
|---|---|---|
| Define | `/spec` | Spec before code |
| Plan | `/plan` | Small, atomic tasks |
| Build | `/build` (or `/build auto`) | One slice at a time |
| Verify | `/test` | Tests are proof |
| Review | `/review` | Improve code health |
| Review | `/code-simplify` | Clarity over cleverness |
| Review | `/webperf` | Measure before you optimize |
| Ship | `/ship` | Faster is safer |

### Hooks

`hooks/hooks.json` registers one hook: **SessionStart** runs `hooks/session-start.sh`, which injects
the `using-agent-skills` meta-skill into every new session so the agent starts with the skill
discovery flowchart already in context. It degrades gracefully — if `jq` is missing it says so and
skills remain available individually.

Two further hooks ship here but are **not** registered by default; see their docs before enabling:

- `sdd-cache-{pre,post}.sh` ([SDD-CACHE.md](hooks/SDD-CACHE.md)) — a PreToolUse cache for `WebFetch`,
  keyed by URL with freshness delegated to HTTP validators. `304 Not Modified` is the only signal to
  serve from cache; entries without `ETag` or `Last-Modified` are never cached, because nothing could
  revalidate them. No TTL, deliberately.
- `simplify-ignore.sh` ([SIMPLIFY-IGNORE.md](hooks/SIMPLIFY-IGNORE.md)) — exclusion list for
  `/code-simplify`.

### Evals

The part that keeps the rest honest.

| Tier | Checks | Runs | Cost |
|---|---|---|---|
| 1. Structural | Frontmatter, naming, required sections, command parity | CI | Free |
| 2. Trigger & routing | Positive prompts rank their skill top-k; negatives don't; no description collisions | CI | Free |
| 3. Behavioral | An agent following the skill satisfies its `expectations[]` | On demand | Tokens |

```bash
node scripts/run-evals.js                              # Tier 2, deterministic
node scripts/run-evals.js --min-rank1 80               # enforce the routing floor
node scripts/run-evals.js --behavioral <skill>         # Tier 3, spends tokens
node scripts/run-evals.js --behavioral <skill> --dry-run
```

Tier 2 is a lexical approximation (stemmed TF-IDF over descriptions), not semantics. It catches the
two failure modes that dominate real trigger bugs: a description missing the vocabulary users
actually say, and an over-broad description that outranks the right skill. **A Tier-2 failure
usually means fix the description, not the eval.**

---

## How the pieces fit

```
SessionStart hook ──▶ injects using-agent-skills ──▶ agent knows what exists
        │
        ▼
  user intent ──▶ slash command  ─┐
                                  ├─▶ skill(s) loaded ──▶ work happens
              ──▶ auto-activation ┘                            │
                                                               ▼
                                              persona subagents review it
                                                               │
                                                               ▼
                                                   evals verify the skills
                                                   still trigger correctly
```

Skills activate two ways: explicitly via a slash command, or automatically from intent — designing
an API pulls in `api-and-interface-design`, touching UI pulls in `frontend-ui-engineering`. The
intent→skill map is spelled out in [AGENTS.md](AGENTS.md).

---

## Using it well

The failure mode is not ignoring the harness. It is **name-dropping** it — producing a tidy table of
skills that were never opened, or citing `/review` in a plan without ever invoking it. A skill you
did not read did not help you.

The discipline is short:

1. **Inventory before planning.** Read the skills that bear on the task, not just their filenames.
2. **Invoke rather than reimplement.** If a command or skill covers the workflow, run it instead of
   hand-rolling an equivalent.
3. **Route review to the personas.** Independent context, adversarial brief. Note that these
   personas may not be registered as spawnable agent types in every host — the portable bridge is to
   start a general-purpose subagent whose first instruction is to read `agents/<name>.md` and adopt
   it.
4. **Declare skips.** Not every asset applies. Skipping `security-auditor` on a local, no-network,
   no-untrusted-input change is correct — saying so is what separates a judgment call from an
   omission.
5. **Report what was actually done.** Which skills were read, which were skimmed, which agents ran,
   what they found, what remains unverified.

---

## This environment

Local specifics for this checkout, current as of 2026-08-20:

- **Own git repository** at `coding_agent_harness/`, branch `main`, **no commits yet**. Untracked
  files here have no history to recover from — check before overwriting anything.
- **`src` is gitignored** ([.gitignore](.gitignore) line 12). Anything placed there is invisible to
  version control.
- **Node** drives `scripts/` (validators, eval runner). **`jq`** is required by the SessionStart
  hook. A Python `venv/` and `pyproject.toml` are present.
- **Multi-host packaging.** Beyond Claude Code: `.codex-plugin/`, `.gemini/commands/`,
  `.opencode/skills/`, `.agents/plugins/`. Setup guides for Cursor, Copilot, Windsurf, Codex, Gemini
  CLI, OpenCode and Antigravity live in `docs/`.
- **CI** is `.github/workflows/test-plugin-install.yml`; skill gaps have an issue template.

### Tooling gotchas worth knowing

Constraints of the surrounding agent environment, not of this repo — they cost real time when
rediscovered:

- The **browser extension refuses `file://` URLs**. Serve a directory over `http://localhost` to
  inspect local pages.
- The **automation tab reports `document.hidden === true`**, so `requestAnimationFrame` never fires
  in it. Anything driven by rAF will appear frozen. Verify animation-dependent behaviour through an
  explicit stepping seam rather than wall-clock waiting. (rAF *does* run for a frame during
  screenshot capture, which can make state appear to change between an eval and the image.)
- **`resize_window` moves an OS window that may not contain the automated tab**, so viewport-
  dependent checks — responsive breakpoints in particular — may not be exercised at runtime. Verify
  those at source level and say so.

---

## Extending it

[CONTRIBUTING.md](CONTRIBUTING.md) is the single source of truth for adding or reworking a skill —
run its pre-flight checks first. [docs/skill-anatomy.md](docs/skill-anatomy.md) defines the required
format. Two standing rules from [CLAUDE.md](CLAUDE.md): never add skills that are vague advice
instead of actionable process, and never duplicate content between skills — reference the other
skill instead.
