---
name: gsd:execute-phase
description: Execute all plans in a phase with maximum parallelization via agent teams
argument-hint: "<phase-number> [--gaps-only] [--all]"
allowed-tools:
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - Bash
  - Task
  - TaskCreate
  - TaskUpdate
  - TaskList
  - TaskGet
  - TeamCreate
  - TeamDelete
  - SendMessage
  - AskUserQuestion
---

<objective>
Execute all plans in a phase (or multiple phases with --all) using agent team-based parallel execution.

Orchestrator creates an agent team, analyzes actual file dependencies between plans, creates tasks with dependency relationships, and spawns teammates to execute all independent plans simultaneously. Plans that don't touch the same files run in parallel as team members.

Key principle: MAXIMUM PARALLELISM. If plans don't modify the same files, they run at the same time. Wave numbers from planning are ADVISORY ONLY -- actual parallelism is determined by file dependency analysis at execution time.

Context budget: ~15% orchestrator, 100% fresh per teammate.
</objective>

<execution_context>
@~/.claude/get-shit-done/references/ui-brand.md
@~/.claude/get-shit-done/workflows/execute-phase.md
</execution_context>

<context>
Phase: $ARGUMENTS

**Flags:**
- `--gaps-only` — Execute only gap closure plans (plans with `gap_closure: true` in frontmatter). Use after verify-work creates fix plans.
- `--all` — Execute ALL remaining phases that have plans. Analyzes cross-phase dependencies and runs everything independent simultaneously.

@.planning/ROADMAP.md
@.planning/STATE.md
</context>

<process>
0. **Resolve Model Profile**

   Read model profile for agent spawning:
   ```bash
   MODEL_PROFILE=$(cat .planning/config.json 2>/dev/null | grep -o '"model_profile"[[:space:]]*:[[:space:]]*"[^"]*"' | grep -o '"[^"]*"$' | tr -d '"' || echo "balanced")
   ```

   Default to "balanced" if not set.

   **Model lookup table:**

   | Agent | quality | balanced | budget |
   |-------|---------|----------|--------|
   | gsd-executor | opus | opus | sonnet |
   | gsd-verifier | opus | opus | sonnet |

   Store resolved models for use in Task calls below.

1. **Validate phase(s) exist**
   - If `--all`: find ALL phase directories with PLAN.md files
   - Otherwise: find phase directory matching argument
   - Count PLAN.md files across target phases
   - Error if no plans found

2. **Discover plans**
   - List all *-PLAN.md files across target phase directories
   - Check which have *-SUMMARY.md (already complete)
   - If `--gaps-only`: filter to only plans with `gap_closure: true`
   - Build list of incomplete plans

3. **Analyze dependencies**
   - Read `files_modified` and `depends_on` from each plan's frontmatter
   - Build dependency graph based on ACTUAL file overlap, not wave numbers
   - Plans that modify the same files MUST run sequentially
   - Plans with `depends_on` references MUST wait for dependencies
   - Everything else runs in parallel simultaneously
   - Wave numbers from planning are ADVISORY ONLY -- override with actual analysis
   - Report dependency structure to user

4. **Create agent team and execute**
   - Create agent team with TeamCreate
   - Create tasks (TaskCreate) for each plan with proper blockedBy relationships
   - Spawn gsd-executor teammates for all initially-unblocked plans
   - As plans complete, teammates pick up newly unblocked plans
   - Monitor progress, handle failures
   - Maximum parallelism: if 10 plans have no dependencies, run all 10 simultaneously

5. **Aggregate results**
   - Collect summaries from all plans
   - Report phase(s) completion status
   - Clean up agent team with TeamDelete

6. **Commit any orchestrator corrections**
   Check for uncommitted changes before verification:
   ```bash
   git status --porcelain
   ```

   **If changes exist:** Orchestrator made corrections between executor completions. Commit them:
   ```bash
   git add -u && git commit -m "fix({phase}): orchestrator corrections"
   ```

   **If clean:** Continue to verification.

7. **Verify phase goal**
   Check config: `WORKFLOW_VERIFIER=$(cat .planning/config.json 2>/dev/null | grep -o '"verifier"[[:space:]]*:[[:space:]]*[^,}]*' | grep -o 'true\|false' || echo "true")`

   **If `workflow.verifier` is `false`:** Skip to step 8 (treat as passed).

   **Otherwise:**
   - Spawn `gsd-verifier` subagent with phase directory and goal
   - Verifier checks must_haves against actual codebase (not SUMMARY claims)
   - Creates VERIFICATION.md with detailed report
   - Route by status:
     - `passed` → continue to step 8
     - `human_needed` → present items, get approval or feedback
     - `gaps_found` → present gaps, offer `/gsd:plan-phase {X} --gaps`

8. **Update roadmap and state**
   - Update ROADMAP.md, STATE.md

9. **Update requirements**
   Mark phase requirements as Complete:
   - Read ROADMAP.md, find this phase's `Requirements:` line (e.g., "AUTH-01, AUTH-02")
   - Read REQUIREMENTS.md traceability table
   - For each REQ-ID in this phase: change Status from "Pending" to "Complete"
   - Write updated REQUIREMENTS.md
   - Skip if: REQUIREMENTS.md doesn't exist, or phase has no Requirements line

10. **Commit phase completion**
    Check `COMMIT_PLANNING_DOCS` from config.json (default: true).
    If false: Skip git operations for .planning/ files.
    If true: Bundle all phase metadata updates in one commit:
    - Stage: `git add .planning/ROADMAP.md .planning/STATE.md`
    - Stage REQUIREMENTS.md if updated: `git add .planning/REQUIREMENTS.md`
    - Commit: `docs({phase}): complete {phase-name} phase`

11. **Offer next steps**
    - Route to next action (see `<offer_next>`)
</process>

<offer_next>
Output this markdown directly (not as a code block). Route based on status:

| Status | Route |
|--------|-------|
| `gaps_found` | Route C (gap closure) |
| `human_needed` | Present checklist, then re-route based on approval |
| `passed` + more phases | Route A (next phase) |
| `passed` + last phase | Route B (milestone complete) |

---

**Route A: Phase verified, more phases remain**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 GSD ► PHASE {Z} COMPLETE ✓
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**Phase {Z}: {Name}**

{Y} plans executed
Goal verified ✓

───────────────────────────────────────────────────────────────

## ▶ Next Up

**Phase {Z+1}: {Name}** — {Goal from ROADMAP.md}

/gsd:discuss-phase {Z+1} — gather context and clarify approach

<sub>/clear first → fresh context window</sub>

───────────────────────────────────────────────────────────────

**Also available:**
- /gsd:plan-phase {Z+1} — skip discussion, plan directly
- /gsd:verify-work {Z} — manual acceptance testing before continuing

───────────────────────────────────────────────────────────────

---

**Route B: Phase verified, milestone complete**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 GSD ► MILESTONE COMPLETE 🎉
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**v1.0**

{N} phases completed
All phase goals verified ✓

───────────────────────────────────────────────────────────────

## ▶ Next Up

**Audit milestone** — verify requirements, cross-phase integration, E2E flows

/gsd:audit-milestone

<sub>/clear first → fresh context window</sub>

───────────────────────────────────────────────────────────────

**Also available:**
- /gsd:verify-work — manual acceptance testing
- /gsd:complete-milestone — skip audit, archive directly

───────────────────────────────────────────────────────────────

---

**Route C: Gaps found — need additional planning**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 GSD ► PHASE {Z} GAPS FOUND ⚠
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**Phase {Z}: {Name}**

Score: {N}/{M} must-haves verified
Report: .planning/phases/{phase_dir}/{phase}-VERIFICATION.md

### What's Missing

{Extract gap summaries from VERIFICATION.md}

───────────────────────────────────────────────────────────────

## ▶ Next Up

**Plan gap closure** — create additional plans to complete the phase

/gsd:plan-phase {Z} --gaps

<sub>/clear first → fresh context window</sub>

───────────────────────────────────────────────────────────────

**Also available:**
- cat .planning/phases/{phase_dir}/{phase}-VERIFICATION.md — see full report
- /gsd:verify-work {Z} — manual testing before planning

───────────────────────────────────────────────────────────────

---

After user runs /gsd:plan-phase {Z} --gaps:
1. Planner reads VERIFICATION.md gaps
2. Creates plans 04, 05, etc. to close gaps
3. User runs /gsd:execute-phase {Z} again
4. Execute-phase runs incomplete plans (04, 05...)
5. Verifier runs again → loop until passed
</offer_next>

<team_execution>
**Agent team-based parallel execution:**

1. **Create the team:**
   ```
   TeamCreate(team_name="gsd-phase-{phase}", description="Execute phase {phase} plans")
   ```

2. **Analyze file dependencies:**
   Read `files_modified` from each plan's frontmatter:
   ```bash
   for plan in $PHASE_DIR/*-PLAN.md; do
     files=$(grep "^files_modified:" "$plan" | cut -d: -f2-)
     depends=$(grep "^depends_on:" "$plan" | cut -d: -f2-)
     echo "$plan|$files|$depends"
   done
   ```

   Build dependency map:
   - If plan A modifies `src/auth.rs` and plan B modifies `src/auth.rs` → B depends on A (or vice versa, based on wave order)
   - If plan A has `depends_on: 01-01` → A depends on plan 01-01
   - Plans with NO file overlap and NO explicit depends_on → fully independent → run simultaneously

3. **Create tasks with dependencies:**
   For each incomplete plan, create a task:
   ```
   TaskCreate(subject="Execute {plan_id}: {plan_name}", description="...")
   ```
   Then set up blockedBy relationships:
   ```
   TaskUpdate(taskId="{plan_b_task_id}", addBlockedBy=["{plan_a_task_id}"])
   ```

4. **Read file contents for teammates:**
   Before spawning, read plan contents. The `@` syntax does not work across Task() boundaries.
   ```bash
   STATE_CONTENT=$(cat .planning/STATE.md)
   CONFIG_CONTENT=$(cat .planning/config.json 2>/dev/null)
   ```

5. **Spawn ALL unblocked plans as teammates simultaneously:**
   For each plan with no unresolved dependencies, spawn a teammate:
   ```
   Task(
     prompt="Execute plan at {plan_path}\n\nPlan:\n{plan_content}\n\nProject state:\n{state_content}\n\nConfig:\n{config_content}",
     subagent_type="gsd-executor",
     model="{executor_model}",
     team_name="gsd-phase-{phase}",
     name="executor-{plan_id}"
   )
   ```

   Spawn ALL independent plans in a single message with multiple Task calls.

   **Example with 5 plans, 2 blocked:**
   Plans 01, 02, 03 are independent → spawn all 3 simultaneously
   Plan 04 depends on 01 → blocked, spawned after 01 completes
   Plan 05 depends on 03 → blocked, spawned after 03 completes

6. **Monitor and spawn newly unblocked plans:**
   When a teammate completes and reports back:
   - Mark its task as completed (TaskUpdate)
   - Check TaskList for newly unblocked tasks
   - Spawn teammates for any plans that are now unblocked
   - Continue until all plans complete

7. **Clean up:**
   After all plans complete:
   - Shut down all teammates (SendMessage type: "shutdown_request")
   - Delete team (TeamDelete)

**Maximum parallelism enforced.** If 10 plans have no file overlap, all 10 run simultaneously as team members.
</team_execution>

<checkpoint_handling>
Plans with `autonomous: false` have checkpoints. The execute-phase.md workflow handles the full checkpoint flow:
- Subagent pauses at checkpoint, returns structured state
- Orchestrator presents to user, collects response
- Spawns fresh continuation agent (not resume)

See `@~/.claude/get-shit-done/workflows/execute-phase.md` step `checkpoint_handling` for complete details.
</checkpoint_handling>

<deviation_rules>
During execution, handle discoveries automatically:

1. **Auto-fix bugs** - Fix immediately, document in Summary
2. **Auto-add critical** - Security/correctness gaps, add and document
3. **Auto-fix blockers** - Can't proceed without fix, do it and document
4. **Ask about architectural** - Major structural changes, stop and ask user

Only rule 4 requires user intervention.
</deviation_rules>

<commit_rules>
**Per-Task Commits:**

After each task completes:
1. Stage only files modified by that task
2. Commit with format: `{type}({phase}-{plan}): {task-name}`
3. Types: feat, fix, test, refactor, perf, chore
4. Record commit hash for SUMMARY.md

**Plan Metadata Commit:**

After all tasks in a plan complete:
1. Stage plan artifacts only: PLAN.md, SUMMARY.md
2. Commit with format: `docs({phase}-{plan}): complete [plan-name] plan`
3. NO code files (already committed per-task)

**Phase Completion Commit:**

After all plans in phase complete (step 7):
1. Stage: ROADMAP.md, STATE.md, REQUIREMENTS.md (if updated), VERIFICATION.md
2. Commit with format: `docs({phase}): complete {phase-name} phase`
3. Bundles all phase-level state updates in one commit

**NEVER use:**
- `git add .`
- `git add -A`
- `git add src/` or any broad directory

**Always stage files individually.**
</commit_rules>

<success_criteria>
- [ ] All incomplete plans in phase executed
- [ ] Each plan has SUMMARY.md
- [ ] Phase goal verified (must_haves checked against codebase)
- [ ] VERIFICATION.md created in phase directory
- [ ] STATE.md reflects phase completion
- [ ] ROADMAP.md updated
- [ ] REQUIREMENTS.md updated (phase requirements marked Complete)
- [ ] User informed of next steps
</success_criteria>
