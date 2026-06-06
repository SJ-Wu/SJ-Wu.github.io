---
title: "A Practical Guide to AI-Assisted Development: Agent.md + Skills"
date: 2026-03-31
draft: false
summary: "How to use Agent.md (CLAUDE.md / AGENTS.md) and Skills to make AI-assisted development more effective: core concepts, meta-prompts, skill-creator, and the differences between Claude Code and Codex."
tags: ["AI", "Claude Code", "Codex", "Skills"]
---

This guide collects what I learned adopting AI-assisted development. It explains
two features — **Agent.md** (CLAUDE.md / AGENTS.md) and **Skills** — and how to
use a **meta-prompt** (letting the AI generate the config files for you) to bring
them into a real workflow.

---

## 1. Core concepts

> Agent.md = the project's permanent memory (always loaded)
>
> Skills = expert skills pulled in on demand (loaded when needed)

|  | Agent.md | Skills |
|---|---|---|
| When it loads | Automatically, every conversation | Only when invoked |
| What it holds | Project architecture, conventions, commands | The SOP for a specific task |
| Update cadence | Evolves with the project | Evolves with the workflow |
| Token impact | Fixed cost | On-demand cost |

**Cross-platform**

- Agent.md (CLAUDE.md) works with Claude Code, Codex, and most tools.
- SKILL.md follows the [Agent Skills Standard](https://agentskills.io) and works
  across 30+ tools (Claude Code, Codex, Cursor, Gemini CLI, GitHub Copilot, and
  more). Write a SKILL.md once and use it everywhere.

---

## 2. Building an Agent.md / CLAUDE.md

### 2.1 Section skeleton

A CLAUDE.md for a development project should usually contain these sections (each
with a clear responsibility):

```
## Conversation — language preference and reply style
## Architecture Overview — layered architecture, directory structure (table)
## Technology Stack — language versions, frameworks, core dependencies
## Build and Development Commands — build/run/test commands (code block)
## Configuration — environment config, profiles, environment variables
## Development Notes — architectural conventions, naming, common patterns
## Key Business Domains — core business domains
## Testing Strategy — test framework, mocking strategy, known pitfalls (❌/✅)
```

### 2.2 Meta-prompt template

You don't have to hand-write CLAUDE.md. Paste the prompt below into an AI tool and
let it scan the codebase and generate the file (Claude Code and Codex can also
bootstrap a project with `/init`):

```
Scan the entire codebase, then produce markdown with the following structure:

1. Conversation — the developer's language preference and reply style
2. Architecture Overview — project type, directory structure, module purposes (table)
3. Technology Stack — language versions, frameworks, main dependencies
4. Build and Development Commands — common build, run, and test commands
5. Configuration — how environments are configured (profiles, env vars)
6. Development Notes — architectural conventions, naming, common patterns
7. Key Business Domains — a brief description of the core business domains
8. Testing Strategy — test framework, mocking strategy, known pitfalls

Requirements:
- Present the directory structure as a table (module name | purpose)
- Put commands in code blocks with comments
- Record pitfalls as ❌/✅ comparisons
- Keep the total length to 200–400 lines
- Only write what's genuinely needed for AI collaboration; skip the obvious
```

### 2.3 Presentation tips

Using a Java microservices project as an example, a few things that make a
CLAUDE.md more useful:

**Architecture Overview** — list the services in a table so they're scannable:

```
| Service         | Purpose |
|-----------------|---------|
| account-service | Account management (user accounts, balances, history) |
| order-service   | Order management (order creation, status tracking) |
| ...             | ... |
```

**Testing Strategy** — record pitfalls as ❌/✅ comparisons so the AI follows
them every time:

```java
// ❌ Wrong: null is skipped by MyBatis Plus, so a NOT NULL column errors out
config.setRemark(null);

// ✅ Right: an empty string is included in the INSERT SQL
config.setRemark("");
```

**Build Commands** — code blocks with comments, ready for the AI to copy when it
needs to build:

```bash
# Use project Java version
source ~/.sdkman/bin/sdkman-init.sh && sdk use java <version>

# Build all modules
mvn clean install

# Run specific test class
mvn test -Dtest=OrderControllerTest
```

---

## 3. Building a Skill

### 3.1 SKILL.md structure

Each skill is a `.claude/skills/{name}/SKILL.md` (or
`.codex/skills/{name}/SKILL.md`) file. The structure is simple:

```yaml
---
name: skill-name          # required: the invocation name
description: A description # required: what it's for (also the auto-trigger match)
---

# Skill title

Put the instructions here, in Markdown.
Use $ARGUMENTS to receive arguments the user passes in.
```

Optional: put template files in a `references/` subdirectory and reference them
from SKILL.md.

**Anatomy — the frontmatter of a code-review skill:**

```yaml
---
name: code-review
description: Perform a thorough code review of backend code, covering
  conformance to architectural conventions, naming, response wrapping,
  security, business logic, distributed transactions, performance
  (N+1 queries, caching), null safety, and test coverage. Use when a
  feature is done, before opening a PR, or when a quality review is needed.
  Outputs a 🔴🟡🟢 severity-graded report.
---
```

The more precise the description, the more accurate the auto-triggering.

**Minimal example — the grill-me skill** (a few lines of body):

```yaml
---
name: grill-me
description: Interview the user relentlessly about a plan or design until
  reaching shared understanding. Use when user wants to stress-test a plan.
---

Interview me relentlessly about every aspect of this plan until we reach
a shared understanding. Walk down each branch of the design tree.
If a question can be answered by exploring the codebase, explore instead.
```

**Advanced example — with a references subdirectory + MCP integration:**

```
.claude/skills/task-flow/
├── SKILL.md                              # main instructions (several steps)
└── references/
    ├── api-change-template.md            # HTML template for API change notes
    └── test-report-template.md           # HTML template for the self-test report
```

SKILL.md can reference `references/api-change-template.md` to keep output
consistent — the recommended way to manage templates inside a skill. Any MCP
tools a skill integrates can be swapped for ones you prefer (Notion, Obsidian,
or other MCP-capable tools).

### 3.2 Generating a skill with skill-creator (6 steps)

You don't have to hand-write SKILL.md. Both Claude Code and Codex ship with
`/skill-creator`:

1. Type `/skill-creator`.
2. Describe what you need in natural language (e.g. "I need a skill that, when I
   mention a task ID, reads the card from my task system, analyzes the
   requirements, and produces API docs and a test report").
3. skill-creator asks about the details (trigger conditions? output format?
   arguments?).
4. It generates `.claude/skills/{name}/SKILL.md`.
5. Test it with `/skill-name`.
6. It's cross-tool: the same SKILL.md is read directly by Codex, Cursor, etc.

---

## 4. Using Claude Code

### 4.1 Setting up CLAUDE.md

- Put it in the **project root**; Claude Code reads it automatically on startup.
- It supports **additional CLAUDE.md files in subdirectories** (nearest one wins)
  — e.g. `services/order-service/CLAUDE.md` for that service's specific rules.
- No extra setup needed; it just works once it's there.

### 4.2 Generating a skill with /skill-creator

A sample interaction:

```
You: /skill-creator
AI:  What kind of skill do you want to build? Describe what it's for.
You: I need a code review skill that thoroughly checks backend code for
     architectural conventions, security, business logic, and distributed
     transactions.
AI:  Got it — a few questions:
     1. How is the review target specified? A path or a PR?
     2. Preferred output format?
     3. Which review dimensions do you need?
You: [answers...]
AI:  [produces .claude/skills/code-review/SKILL.md]
```

### 4.3 Invoking a skill

**Explicit invocation** (slash commands):

```
/code-review services/order-service
/write-test OrderServiceImpl.createOrder
/task-flow TASK-1234
```

**Auto-triggering** (matched against the description text):

- Mentioning a task ID → triggers the task-flow skill.
- Saying "do a code review" → triggers the code-review skill.
- Saying "I need a new API" → triggers the new-api skill.

**Key point:** write the description so the trigger conditions and boundaries are
clear.

### 4.4 An example set of team skills

In practice you can build a skill for each kind of repetitive team task. Here's an
illustrative set:

| Skill | Purpose | Trigger |
|---|---|---|
| `code-review` | Thorough code review (multi-dimension, 🔴🟡🟢 report) | Before opening a PR |
| `write-test` | Write JUnit 5 unit tests (Given/When/Then) | When tests are needed |
| `task-flow` | Full task workflow (analyze → spec → API docs → self-test report) | When a task ID is mentioned |
| `new-api` | Scaffold a new API endpoint (Controller→Service→VO→DTO→ErrorCode) | When a new API is needed |
| `new-service` | Scaffold a full new microservice | When a new service is needed |
| `new-feign-client` | Add a Feign client (looks up the service registry, e.g. Eureka) | Cross-service calls |
| `review-security` | Focused security review (IDOR, sensitive-resource access, token validation) | Reviewing sensitive operations |
| `review-transaction` | Distributed-transaction review (transaction boundaries, undo logs) | Multi-service writes |
| `annual-review` | Broad security scan (OWASP Top 10, P0–P3 grading) | Security audits |
| `grill-me` | Deep interview on a plan/design (walks the decision tree) | Stress-testing a plan |

---

## 5. Using Codex

### 5.1 Comparison table

| Item | Claude Code | Codex |
|---|---|---|
| Project memory file | `CLAUDE.md` | `AGENTS.md` |
| Hierarchical override | Subdirectory CLAUDE.md | Subdirectory AGENTS.md + override files |
| Size limit | No explicit limit | 32 KiB by default |
| Skills format | `.claude/skills/*/SKILL.md` | Same (shared standard) `.codex/skills/*/SKILL.md` |
| Skill generator | `/skill-creator` built in | `$skill-creator` built in |
| Auto-trigger config | Description text match | Extra config in `agents/openai.yaml` |

### 5.2 Key points

- **Skills are fully portable** — no changes needed. Generate one with Claude
  Code's `/skill-creator`, and Codex reads the same SKILL.md directly.
- Codex has `agents/openai.yaml`, where you can set
  `allow_implicit_invocation: false` to turn off auto-triggering.
- AGENTS.md has a structure similar to CLAUDE.md, but the two are **not
  interchangeable** (different filenames; each tool reads its own).
- Recommended approach: maintain one CLAUDE.md as the source of truth, and copy
  it to AGENTS.md with small tweaks when needed.

---

## 6. Case study: using the grill-me skill to write this guide

This guide was itself produced with the `/grill-me` skill. Here's a look back at
the experience.

**The flow:**

1. Run `/grill-me` with a draft outline.
2. grill-me walks the decision tree branch by branch (target audience → scope →
   format → depth of content).
3. Each round asks 2–3 questions, each with a recommended answer.
4. Once every branch is resolved, it consolidates everything into a complete plan.

**Pros:**

- **Forces clear thinking** — every "I thought I had this figured out" part got a
  follow-up that surfaced a blind spot (e.g. AGENTS.md and CLAUDE.md not being
  interchangeable).
- **Leaves a decision trail** — the series of Q&As becomes a complete record of
  the design decisions.
- **Less rework** — format, emphasis, and how to present examples were settled
  before any writing began.
- **Self-directed research** — during the interview it checked agentskills.io to
  confirm skill portability.

**Cons / caveats:**

- **Time** — several rounds take about 15–20 minutes; simple tasks don't need
  this heavy a process.
- **Token usage** — a deep interview plus web search plus codebase exploration
  uses more tokens.
- **When to use** — best for tasks where the direction is unclear or the impact is
  large; for a clear bug fix or small feature, just do it.

**Conclusion:** grill-me fits situations where you need to think before you write
— technical docs, architecture design, planning a new feature.

---

## 7. Further reading

- [Claude Code docs](https://docs.anthropic.com/en/docs/claude-code)
- [Claude Code Skills](https://docs.anthropic.com/en/docs/claude-code/skills)
- [Codex docs](https://developers.openai.com/codex)
- [Codex AGENTS.md](https://developers.openai.com/codex/guides/agents-md)
- [Agent Skills open standard](https://agentskills.io)
- [The Complete Guide to Building Skills for Claude](https://resources.anthropic.com/hubfs/The-Complete-Guide-to-Building-Skill-for-Claude.pdf)
