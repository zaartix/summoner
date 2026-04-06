# Summoner

A bootstrap wizard for [Claude Code](https://docs.anthropic.com/en/docs/claude-code) that generates a specialized agent workflow for your project — tailored to your stack, structure, and standards.

## What it does

Summoner runs once on a new project. Through a guided conversation it analyzes your codebase and creates a set of prompt files (Claude Code slash commands) that form a complete agent workflow for tackling features:

```
/research → /design → /plan → /implement → /review-*
```

Each stage produces a document that becomes the input for the next. Code is written only at the final stage — everything before that is context preparation.

## Why

A generic "build me X" prompt produces unpredictable results because the agent makes all decisions at once: what to research, what architecture to use, how to split the work. Quality suffers.

Summoner creates **project-specific** prompts that guide agents through a structured process where each decision is made at the right time and reviewed by a human before moving forward.

**Quality formula:** (correctness + completeness) / (context size + noise)

## The workflow it generates

| Stage | Agent does | Output | Human gate |
|-------|-----------|--------|------------|
| **Research** | Scans the codebase for everything relevant to the task | Facts document with file/line references | — |
| **Design** | Creates architectural solution with diagrams (Mermaid), ADRs, test strategy | Design document | Review & approve |
| **Plan** | Breaks implementation into phases, each a committable unit with completion criteria | Phased plan | Review & approve |
| **Implement** | Writes code strictly following the plan, runs quality gates after each phase | Code + passing checks | — |
| **Review** | Three independent reviewers check quality, security, and design conformance | Issue lists | — |

Key principles (from [context engineering](https://www.youtube.com/watch?v=7oRBHxMvWxQ):
- Research reports only facts — no opinions, no suggestions
- Design and plan are human-reviewed gates before any code is written
- Implementer follows the plan strictly — if it can't be done as written, it stops and reports
- Reviewers never fix code — they only point out issues
- Each prompt is specialized for your project's stack, patterns, and tooling

## How Summoner works

Summoner goes through 6 phases to generate the prompts:

1. **Phase 0 — Understand the project.** Subagents scan your docs and code independently. You describe the project in your own words. Summoner cross-references findings and resolves discrepancies.

2. **Phases 1–4 — Generate stage prompts.** For each stage (research, design, plan, implement), Summoner asks targeted questions about your project's specifics, validates answers against the codebase, and generates a specialized prompt.

3. **Phase 5 — Generate review prompts.** Creates three review prompts (quality, security, design conformance) tailored to your project's standards and requirements.

All generated files are saved to `.claude/commands/` in your project.

## Generated files

```
.claude/commands/
├── research.md          # Codebase investigation prompt
├── design.md            # Architecture & diagrams prompt
├── plan.md              # Implementation planning prompt
├── implement.md         # Code writing prompt with quality gates
├── review-quality.md    # Code quality review prompt
├── review-security.md   # Security review prompt
└── review-design.md     # Design conformance review prompt
```

## Usage

### Bootstrap your project

```
/bootstrap
```

Answer the wizard's questions. It will scan your codebase, ask about your architecture, standards, and tooling, then generate all prompt files.

### Work on a feature

```
/research <task description>
# → creates research document

/design <task description>
# → creates design document, review it before proceeding

/plan <task description>
# → creates phased plan, review it before proceeding

/implement
# → implements phase by phase, quality gates after each

/review-quality
/review-security
/review-design
# → independent reviews after each implementation phase
```

## Repository structure

- **[MANUAL.md](MANUAL.md)** — Project documentation. Explains the idea, how the wizard works phase by phase, how it differs from the agent system it generates, and what the output looks like.
- **[SKILL.md](SKILL.md)** — The wizard prompt itself. This is the actual instruction that drives the bootstrap process — it tells the wizard what to do, what to ask, and how to generate each prompt.
- **[TODO.md](TODO.md)** — Backlog of potential improvements deferred in favor of a simple first version (context splitting, delta updates, task type specialization, subagent teams, etc.).

## Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI

## License

TBD
