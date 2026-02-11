<purpose>
Execute all plans in a phase (or multiple phases) using agent team-based parallel execution with maximum parallelism. Orchestrator creates an agent team, analyzes actual file dependencies, and runs everything independent simultaneously.
</purpose>

<core_principle>
The orchestrator's job is coordination, not execution. Each teammate loads the full execute-plan context itself. Orchestrator discovers plans, analyzes ACTUAL file dependencies (not pre-computed waves), creates an agent team with proper task dependencies, spawns teammates for all unblocked plans, and monitors until completion.

**MAXIMUM PARALLELISM:** If plans don't modify the same files, they MUST run simultaneously. Wave numbers from planning are ADVISORY ONLY. Actual parallelism is determined by file dependency analysis at execution time.
</core_principle>

<required_reading>
Read STATE.md before any operation to load project context.
Read config.json for planning behavior settings.
</required_reading>

<process>

<step name="resolve_model_profile" priority="first">
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
| general-purpose | — | — | — |

Store resolved models for use in Task calls below.
</step>

<step name="load_project_state">
Before any operation, read project state:

```bash
cat .planning/STATE.md 2>/dev/null
```

**If file exists:** Parse and internalize:
- Current position (phase, plan, status)
- Accumulated decisions (constraints on this execution)
- Blockers/concerns (things to watch for)

**If file missing but .planning/ exists:**
```
STATE.md missing but planning artifacts exist.
Options:
1. Reconstruct from existing artifacts
2. Continue without project state (may lose accumulated context)
```

**If .planning/ doesn't exist:** Error - project not initialized.

**Load planning config:**

```bash
# Check if planning docs should be committed (default: true)
COMMIT_PLANNING_DOCS=$(cat .planning/config.json 2>/dev/null | grep -o '"commit_docs"[[:space:]]*:[[:space:]]*[^,}]*' | grep -o 'true\|false' || echo "true")
# Auto-detect gitignored (overrides config)
git check-ignore -q .planning 2>/dev/null && COMMIT_PLANNING_DOCS=false
```

Store `COMMIT_PLANNING_DOCS` for use in git operations.

**Load git branching config:**

```bash
# Get branching strategy (default: none)
BRANCHING_STRATEGY=$(cat .planning/config.json 2>/dev/null | grep -o '"branching_strategy"[[:space:]]*:[[:space:]]*"[^"]*"' | sed 's/.*:.*"\([^"]*\)"/\1/' || echo "none")

# Get templates
PHASE_BRANCH_TEMPLATE=$(cat .planning/config.json 2>/dev/null | grep -o '"phase_branch_template"[[:space:]]*:[[:space:]]*"[^"]*"' | sed 's/.*:.*"\([^"]*\)"/\1/' || echo "gsd/phase-{phase}-{slug}")
MILESTONE_BRANCH_TEMPLATE=$(cat .planning/config.json 2>/dev/null | grep -o '"milestone_branch_template"[[:space:]]*:[[:space:]]*"[^"]*"' | sed 's/.*:.*"\([^"]*\)"/\1/' || echo "gsd/{milestone}-{slug}")
```

Store `BRANCHING_STRATEGY` and templates for use in branch creation step.
</step>

<step name="handle_branching">
Create or switch to appropriate branch based on branching strategy.

**Skip if strategy is "none":**

```bash
if [ "$BRANCHING_STRATEGY" = "none" ]; then
  # No branching, continue on current branch
  exit 0
fi
```

**For "phase" strategy — create phase branch:**

```bash
if [ "$BRANCHING_STRATEGY" = "phase" ]; then
  # Get phase name from directory (e.g., "03-authentication" → "authentication")
  PHASE_NAME=$(basename "$PHASE_DIR" | sed 's/^[0-9]*-//')

  # Create slug from phase name
  PHASE_SLUG=$(echo "$PHASE_NAME" | tr '[:upper:]' '[:lower:]' | sed 's/[^a-z0-9]/-/g' | sed 's/--*/-/g' | sed 's/^-//;s/-$//')

  # Apply template
  BRANCH_NAME=$(echo "$PHASE_BRANCH_TEMPLATE" | sed "s/{phase}/$PADDED_PHASE/g" | sed "s/{slug}/$PHASE_SLUG/g")

  # Create or switch to branch
  git checkout -b "$BRANCH_NAME" 2>/dev/null || git checkout "$BRANCH_NAME"

  echo "Branch: $BRANCH_NAME (phase branching)"
fi
```

**For "milestone" strategy — create/switch to milestone branch:**

```bash
if [ "$BRANCHING_STRATEGY" = "milestone" ]; then
  # Get current milestone info from ROADMAP.md
  MILESTONE_VERSION=$(grep -oE 'v[0-9]+\.[0-9]+' .planning/ROADMAP.md | head -1 || echo "v1.0")
  MILESTONE_NAME=$(grep -A1 "## .*$MILESTONE_VERSION" .planning/ROADMAP.md | tail -1 | sed 's/.*- //' | cut -d'(' -f1 | tr -d ' ' || echo "milestone")

  # Create slug
  MILESTONE_SLUG=$(echo "$MILESTONE_NAME" | tr '[:upper:]' '[:lower:]' | sed 's/[^a-z0-9]/-/g' | sed 's/--*/-/g' | sed 's/^-//;s/-$//')

  # Apply template
  BRANCH_NAME=$(echo "$MILESTONE_BRANCH_TEMPLATE" | sed "s/{milestone}/$MILESTONE_VERSION/g" | sed "s/{slug}/$MILESTONE_SLUG/g")

  # Create or switch to branch (same branch for all phases in milestone)
  git checkout -b "$BRANCH_NAME" 2>/dev/null || git checkout "$BRANCH_NAME"

  echo "Branch: $BRANCH_NAME (milestone branching)"
fi
```

**Report branch status:**

```
Branching: {strategy} → {branch_name}
```

**Note:** All subsequent plan commits go to this branch. User handles merging based on their workflow.
</step>

<step name="validate_phase">
Confirm phase exists and has plans:

```bash
# Match both zero-padded (05-*) and unpadded (5-*) folders
PADDED_PHASE=$(printf "%02d" ${PHASE_ARG} 2>/dev/null || echo "${PHASE_ARG}")
PHASE_DIR=$(ls -d .planning/phases/${PADDED_PHASE}-* .planning/phases/${PHASE_ARG}-* 2>/dev/null | head -1)
if [ -z "$PHASE_DIR" ]; then
  echo "ERROR: No phase directory matching '${PHASE_ARG}'"
  exit 1
fi

PLAN_COUNT=$(ls -1 "$PHASE_DIR"/*-PLAN.md 2>/dev/null | wc -l | tr -d ' ')
if [ "$PLAN_COUNT" -eq 0 ]; then
  echo "ERROR: No plans found in $PHASE_DIR"
  exit 1
fi
```

Report: "Found {N} plans in {phase_dir}"
</step>

<step name="discover_plans">
List all plans and extract metadata:

```bash
# Get all plans
ls -1 "$PHASE_DIR"/*-PLAN.md 2>/dev/null | sort

# Get completed plans (have SUMMARY.md)
ls -1 "$PHASE_DIR"/*-SUMMARY.md 2>/dev/null | sort
```

For each plan, read frontmatter to extract:
- `wave: N` - Execution wave (pre-computed)
- `autonomous: true/false` - Whether plan has checkpoints
- `gap_closure: true/false` - Whether plan closes gaps from verification/UAT

Build plan inventory:
- Plan path
- Plan ID (e.g., "03-01")
- Wave number
- Autonomous flag
- Gap closure flag
- Completion status (SUMMARY exists = complete)

**Filtering:**
- Skip completed plans (have SUMMARY.md)
- If `--gaps-only` flag: also skip plans where `gap_closure` is not `true`

If all plans filtered out, report "No matching incomplete plans" and exit.
</step>

<step name="analyze_dependencies">
Analyze actual file dependencies between plans to determine maximum parallelism.

**Wave numbers are ADVISORY ONLY.** Override with actual file analysis.

```bash
# For each plan, extract files_modified, depends_on, and autonomous flag
for plan in $PHASE_DIR/*-PLAN.md; do
  files_modified=$(grep "^files_modified:" "$plan" | cut -d: -f2-)
  depends_on=$(grep "^depends_on:" "$plan" | cut -d: -f2-)
  autonomous=$(grep "^autonomous:" "$plan" | cut -d: -f2 | tr -d ' ')
  echo "$plan|$files_modified|$depends_on|$autonomous"
done
```

**Build dependency graph:**

1. For each pair of plans, check if their `files_modified` lists overlap
2. If overlap exists → sequential dependency (earlier plan runs first)
3. If `depends_on` references another plan → explicit dependency
4. Everything else → INDEPENDENT → runs simultaneously

**Determine execution groups (replacing waves):**
```
independent = [plan-01, plan-02, plan-03]  # No file overlap, no depends_on
blocked = {
  plan-04: [plan-01],     # plan-04 depends on plan-01
  plan-05: [plan-03],     # plan-05 depends on plan-03
}
```

Report dependency structure with context:
```
## Execution Plan

**Phase {X}: {Name}** — {total_plans} plans

| Plan | Dependencies | Status | What it builds |
|------|-------------|--------|----------------|
| 01-01 | None | Ready | {from plan objectives} |
| 01-02 | None | Ready | {from plan objectives} |
| 01-03 | None | Ready | {from plan objectives} |
| 01-04 | 01-01 | Blocked | {from plan objectives} |
| 01-05 | 01-03 | Blocked | {from plan objectives} |

**Parallelism: {N} of {total} plans start immediately**
```

The "What it builds" column comes from skimming plan names/objectives. Keep it brief (3-8 words).
</step>

<step name="execute_team">
Create an agent team and execute all plans with maximum parallelism.

**1. Create agent team:**

```
TeamCreate(team_name="gsd-phase-{phase}", description="Execute phase {phase}: {phase_name}")
```

**2. Create tasks for all plans:**

For each incomplete plan, create a task in the team's task list:

```
TaskCreate(
  subject="Execute {plan_id}: {plan_name}",
  description="Execute plan at {plan_path}. Create SUMMARY.md. Commit each task atomically.",
  activeForm="Executing {plan_id}"
)
```

Then set up dependency relationships from the analyze_dependencies step:

```
TaskUpdate(taskId="{blocked_plan_task_id}", addBlockedBy=["{dependency_task_id}"])
```

**3. Read file contents for teammates:**

Before spawning, read all plan contents. The `@` syntax does not work across Task() boundaries.

```bash
STATE_CONTENT=$(cat .planning/STATE.md)
CONFIG_CONTENT=$(cat .planning/config.json 2>/dev/null)
# Read each plan file individually
```

**4. Describe what's being built (BEFORE spawning):**

Read each plan's `<objective>` section. Summarize the full execution plan:

```
---

## Execution Starting

**{Plan ID}: {Plan Name}** — Ready
{2-3 sentences: what this builds, key technical approach}

**{Plan ID}: {Plan Name}** — Ready
{same format}

**{Plan ID}: {Plan Name}** — Blocked by {dependency}
{same format}

Spawning {count} teammates for {ready_count} ready plans...

---
```

**Examples:**
- Bad: "Executing terrain generation plan"
- Good: "Procedural terrain generator using Perlin noise — creates height maps, biome zones, and collision meshes. Required before vehicle physics can interact with ground."

**5. Spawn ALL unblocked plans as teammates simultaneously:**

For each plan with no unresolved dependencies, spawn a teammate in a SINGLE message with multiple Task calls:

```
Task(
  prompt="<objective>
Execute plan {plan_number} of phase {phase_number}-{phase_name}.

Commit each task atomically. Create SUMMARY.md. Update STATE.md.
</objective>

<execution_context>
@~/.claude/get-shit-done/workflows/execute-plan.md
@~/.claude/get-shit-done/templates/summary.md
@~/.claude/get-shit-done/references/checkpoints.md
@~/.claude/get-shit-done/references/tdd.md
</execution_context>

<context>
Plan:
{plan_content}

Project state:
{state_content}

Config (if exists):
{config_content}
</context>

<success_criteria>
- [ ] All tasks executed
- [ ] Each task committed individually
- [ ] SUMMARY.md created in plan directory
- [ ] STATE.md updated with position and decisions
</success_criteria>",
  subagent_type="gsd-executor",
  model="{executor_model}",
  team_name="gsd-phase-{phase}",
  name="executor-{plan_id}"
)
```

**CRITICAL:** Spawn ALL ready plans in a single message. If 5 plans are ready, send 5 Task calls in one message.

**6. Monitor completion and spawn newly unblocked plans:**

When a teammate completes:
- Mark its task as completed (TaskUpdate)
- Verify SUMMARY.md exists at expected path
- Read SUMMARY.md to extract what was built
- Check TaskList for newly unblocked tasks
- If any tasks are now unblocked, spawn new teammates for them immediately

**Report each completion:**
```
---

## {Plan ID} Complete

{What was built — from SUMMARY.md deliverables}
{Notable deviations or discoveries, if any}

{N} of {total} plans complete. {remaining_ready} more running.

---
```

**7. Handle failures:**

If any teammate fails:
- Report which plan failed and why
- Check if failed plan blocks other plans
- Ask user: "Continue with independent plans?" or "Stop execution?"
- If continue: keep running independent plans, skip blocked ones
- If stop: shut down all teammates, exit with partial completion report

**8. Handle checkpoint plans:**

Plans with `autonomous: false` have checkpoints. When a teammate hits a checkpoint:
- Teammate sends checkpoint message to team lead
- Lead presents checkpoint to user
- After user responds, spawn new teammate to continue from checkpoint
- See `<checkpoint_handling>` for details

**9. Clean up team after all plans complete:**

```
# Shut down all remaining teammates
SendMessage(type="shutdown_request", recipient="executor-{plan_id}", content="All plans complete")
# ... for each teammate

# Delete team
TeamDelete()
```

</step>

<step name="checkpoint_handling">
Plans with `autonomous: false` require user interaction.

**Detection:** Check `autonomous` field in frontmatter.

**Execution flow for checkpoint plans:**

1. **Spawn agent for checkpoint plan:**
   ```
   Task(prompt="{subagent-task-prompt}", subagent_type="gsd-executor", model="{executor_model}")
   ```

2. **Agent runs until checkpoint:**
   - Executes auto tasks normally
   - Reaches checkpoint task (e.g., `type="checkpoint:human-verify"`) or auth gate
   - Agent returns with structured checkpoint (see checkpoint-return.md template)

3. **Agent return includes (structured format):**
   - Completed Tasks table with commit hashes and files
   - Current task name and blocker
   - Checkpoint type and details for user
   - What's awaited from user

4. **Orchestrator presents checkpoint to user:**

   Extract and display the "Checkpoint Details" and "Awaiting" sections from agent return:
   ```
   ## Checkpoint: [Type]

   **Plan:** 03-03 Dashboard Layout
   **Progress:** 2/3 tasks complete

   [Checkpoint Details section from agent return]

   [Awaiting section from agent return]
   ```

5. **User responds:**
   - "approved" / "done" → spawn continuation agent
   - Description of issues → spawn continuation agent with feedback
   - Decision selection → spawn continuation agent with choice

6. **Spawn continuation agent (NOT resume):**

   Use the continuation-prompt.md template:
   ```
   Task(
     prompt=filled_continuation_template,
     subagent_type="gsd-executor",
     model="{executor_model}"
   )
   ```

   Fill template with:
   - `{completed_tasks_table}`: From agent's checkpoint return
   - `{resume_task_number}`: Current task from checkpoint
   - `{resume_task_name}`: Current task name from checkpoint
   - `{user_response}`: What user provided
   - `{resume_instructions}`: Based on checkpoint type (see continuation-prompt.md)

7. **Continuation agent executes:**
   - Verifies previous commits exist
   - Continues from resume point
   - May hit another checkpoint (repeat from step 4)
   - Or completes plan

8. **Repeat until plan completes or user stops**

**Why fresh agent instead of resume:**
Resume relies on Claude Code's internal serialization which breaks with parallel tool calls.
Fresh agents with explicit state are more reliable and maintain full context.

**Checkpoint in parallel context:**
If a plan in a parallel wave has a checkpoint:
- Spawn as normal
- Agent pauses at checkpoint and returns with structured state
- Other parallel agents may complete while waiting
- Present checkpoint to user
- Spawn continuation agent with user response
- Wait for all agents to finish before next wave
</step>

<step name="aggregate_results">
After all waves complete, aggregate results:

```markdown
## Phase {X}: {Name} Execution Complete

**Waves executed:** {N}
**Plans completed:** {M} of {total}

### Wave Summary

| Wave | Plans | Status |
|------|-------|--------|
| 1 | plan-01, plan-02 | ✓ Complete |
| CP | plan-03 | ✓ Verified |
| 2 | plan-04 | ✓ Complete |
| 3 | plan-05 | ✓ Complete |

### Plan Details

1. **03-01**: [one-liner from SUMMARY.md]
2. **03-02**: [one-liner from SUMMARY.md]
...

### Issues Encountered
[Aggregate from all SUMMARYs, or "None"]
```
</step>

<step name="verify_phase_goal">
Verify phase achieved its GOAL, not just completed its TASKS.

**Spawn verifier:**

```
Task(
  prompt="Verify phase {phase_number} goal achievement.

Phase directory: {phase_dir}
Phase goal: {goal from ROADMAP.md}

Check must_haves against actual codebase. Create VERIFICATION.md.
Verify what actually exists in the code.",
  subagent_type="gsd-verifier",
  model="{verifier_model}"
)
```

**Read verification status:**

```bash
grep "^status:" "$PHASE_DIR"/*-VERIFICATION.md | cut -d: -f2 | tr -d ' '
```

**Route by status:**

| Status | Action |
|--------|--------|
| `passed` | Continue to update_roadmap |
| `human_needed` | Present items to user, get approval or feedback |
| `gaps_found` | Present gap summary, offer `/gsd:plan-phase {phase} --gaps` |

**If passed:**

Phase goal verified. Proceed to update_roadmap.

**If human_needed:**

```markdown
## ✓ Phase {X}: {Name} — Human Verification Required

All automated checks passed. {N} items need human testing:

### Human Verification Checklist

{Extract from VERIFICATION.md human_verification section}

---

**After testing:**
- "approved" → continue to update_roadmap
- Report issues → will route to gap closure planning
```

If user approves → continue to update_roadmap.
If user reports issues → treat as gaps_found.

**If gaps_found:**

Present gaps and offer next command:

```markdown
## ⚠ Phase {X}: {Name} — Gaps Found

**Score:** {N}/{M} must-haves verified
**Report:** {phase_dir}/{phase}-VERIFICATION.md

### What's Missing

{Extract gap summaries from VERIFICATION.md gaps section}

---

## ▶ Next Up

**Plan gap closure** — create additional plans to complete the phase

`/gsd:plan-phase {X} --gaps`

<sub>`/clear` first → fresh context window</sub>

---

**Also available:**
- `cat {phase_dir}/{phase}-VERIFICATION.md` — see full report
- `/gsd:verify-work {X}` — manual testing before planning
```

User runs `/gsd:plan-phase {X} --gaps` which:
1. Reads VERIFICATION.md gaps
2. Creates additional plans (04, 05, etc.) with `gap_closure: true` to close gaps
3. User then runs `/gsd:execute-phase {X} --gaps-only`
4. Execute-phase runs only gap closure plans (04-05)
5. Verifier runs again after new plans complete

User stays in control at each decision point.
</step>

<step name="update_roadmap">
Update ROADMAP.md to reflect phase completion:

```bash
# Mark phase complete
# Update completion date
# Update status
```

**Check planning config:**

If `COMMIT_PLANNING_DOCS=false` (set in load_project_state):
- Skip all git operations for .planning/ files
- Planning docs exist locally but are gitignored
- Log: "Skipping planning docs commit (commit_docs: false)"
- Proceed to offer_next step

If `COMMIT_PLANNING_DOCS=true` (default):
- Continue with git operations below

Commit phase completion (roadmap, state, verification):
```bash
git add .planning/ROADMAP.md .planning/STATE.md .planning/phases/{phase_dir}/*-VERIFICATION.md
git add .planning/REQUIREMENTS.md  # if updated
git commit -m "docs(phase-{X}): complete phase execution"
```
</step>

<step name="offer_next">
Present next steps based on milestone status:

**If more phases remain:**
```
## Next Up

**Phase {X+1}: {Name}** — {Goal}

`/gsd:plan-phase {X+1}`

<sub>`/clear` first for fresh context</sub>
```

**If milestone complete:**
```
MILESTONE COMPLETE!

All {N} phases executed.

`/gsd:complete-milestone`
```
</step>

</process>

<context_efficiency>
Orchestrator: ~10-15% context (dependency analysis, team management, results).
Teammates: Fresh 200k each (full workflow + execution, independent context windows).
Agent team coordination via shared task list.
Maximum parallelism: all independent plans run simultaneously as team members.
</context_efficiency>

<failure_handling>
**Teammate fails mid-plan:**
- SUMMARY.md won't exist
- Orchestrator detects missing SUMMARY
- Reports failure, asks user how to proceed
- Other independent teammates keep running (they're not affected)

**Dependency chain breaks:**
- Plan A fails
- Plans depending on A become permanently blocked
- Independent plans keep running
- Orchestrator reports: "Plan A failed. Plans B, C are blocked. Plans D, E continue independently."
- Ask user: skip blocked plans or abort?

**All teammates fail:**
- Something systemic (git issues, permissions, etc.)
- Stop execution, clean up team
- Report for manual investigation

**Checkpoint fails to resolve:**
- User can't approve or provides repeated issues
- Ask: "Skip this plan?" or "Abort phase execution?"
- Record partial progress in STATE.md

**Git conflicts between parallel teammates:**
- Two teammates modifying nearby (but not same) files may cause merge conflicts
- If detected: pause conflicting teammate, resolve conflict, continue
- Dependency analysis should prevent this, but handle gracefully if it occurs
</failure_handling>

<resumption>
**Resuming interrupted execution:**

If phase execution was interrupted (context limit, user exit, error):

1. Run `/gsd:execute-phase {phase}` again
2. discover_plans finds completed SUMMARYs
3. Skips completed plans
4. Re-analyzes dependencies among remaining plans
5. Creates new agent team with remaining work
6. Maximum parallelism still enforced

**STATE.md tracks:**
- Last completed plan
- Remaining incomplete plans
- Any pending checkpoints
</resumption>
