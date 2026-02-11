# Model Profiles

Model profiles control which Claude model each GSD agent uses. This allows balancing quality vs token spend.

## Profile Definitions

| Agent | `quality` | `balanced` | `budget` |
|-------|-----------|------------|----------|
| gsd-planner | opus | opus | opus |
| gsd-roadmapper | opus | opus | sonnet |
| gsd-executor | opus | opus | sonnet |
| gsd-phase-researcher | opus | opus | sonnet |
| gsd-project-researcher | opus | opus | sonnet |
| gsd-research-synthesizer | opus | sonnet | sonnet |
| gsd-debugger | opus | opus | sonnet |
| gsd-codebase-mapper | opus | sonnet | haiku |
| gsd-verifier | opus | opus | sonnet |
| gsd-plan-checker | opus | sonnet | sonnet |
| gsd-integration-checker | opus | sonnet | sonnet |

## Profile Philosophy

**quality** - Maximum accuracy, Opus everywhere
- Opus for every agent without exception
- Use when: critical projects, complex architecture, accuracy over speed

**balanced** (default) - Opus-first with Sonnet fallback
- Opus for all decision-making, execution, research, and verification
- Sonnet only for synthesis, mapping, and plan checking (structured output tasks)
- Use when: normal development, high quality with reasonable cost

**budget** - Sonnet-first, minimal Haiku
- Sonnet for everything that writes code, plans, researches, or verifies
- Haiku ONLY for codebase-mapper (read-only exploration, no reasoning needed)
- Use when: conserving quota, high-volume work, less critical phases
- NOTE: Haiku is restricted to purely read-only, non-critical tasks only

## Resolution Logic

Orchestrators resolve model before spawning:

```
1. Read .planning/config.json
2. Get model_profile (default: "balanced")
3. Look up agent in table above
4. Pass model parameter to Task call
```

## Switching Profiles

Runtime: `/gsd:set-profile <profile>`

Per-project default: Set in `.planning/config.json`:
```json
{
  "model_profile": "balanced"
}
```

## Design Rationale

**Why Opus for gsd-planner (all profiles)?**
Planning involves architecture decisions, goal decomposition, and task design. Quality here cascades through the entire project. Never compromise on planning.

**Why Opus for gsd-executor (quality + balanced)?**
Executors encounter deviations, bugs, and architectural decisions during implementation. Opus handles these better than Sonnet. The plan provides structure, but execution still requires strong reasoning.

**Why Opus for gsd-verifier (quality + balanced)?**
Verification requires goal-backward reasoning - checking if code *delivers* what the phase promised. Missing a gap here wastes an entire execution cycle. Accuracy over speed.

**Why Haiku ONLY for gsd-codebase-mapper in budget?**
Read-only exploration and pattern extraction. The only agent where reasoning depth genuinely doesn't matter - it's purely extracting structured output from file contents. Every other agent benefits from higher reasoning quality.
