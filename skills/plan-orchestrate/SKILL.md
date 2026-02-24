---
name: plan-orchestrate
description: Execute implementation plans with parallel TDD workers. Use when ready to run, execute, or start a plan. Triggers on "run the plan", "execute the plan", "implement the plan", "start the plan", "start implementation", "orchestrate", "begin autonomous execution". Requires a plan created by plan-create.
---

# Plan Orchestrate

Execute all tasks in a plan using parallel TDD workers. Fully autonomous after invocation.

## Usage

```
plan-orchestrate {plan-name}
```

Example: `plan-orchestrate user-auth`

## Prerequisites

1. Plan must exist at `.claude/plans/{plan-name}/`
2. Project must be configured (`.claude/testing.md` exists)
3. Plan status must be `ready` or `in_progress`

Check prerequisites:
```bash
ls .claude/plans/{plan-name}/_plan.md .claude/testing.md 2>/dev/null
```

If not found, output error and stop.

## Project Configuration

<testing>
@.claude/testing.md
</testing>

<code-standards>
@.claude/code-standards.md
</code-standards>

## Execution Algorithm

### Step 0: Check for Ralph Wiggum Plugin

Before starting execution, check if the ralph-wiggum plugin is installed:

```bash
# Check if ralph-wiggum commands are available
claude plugin list 2>/dev/null | grep -q "ralph-wiggum"
```

**If ralph-wiggum IS installed:**
- Automatically wrap execution with ralph-wiggum for session persistence
- Invoke: `/ralph-wiggum:loop "plan-orchestrate {plan-name}" --completion-promise "ALL_TASKS_COMPLETE" --max-iterations 100`
- This ensures the orchestrator continues even if context limits are reached

**If ralph-wiggum is NOT installed:**
- Display a warning:
  ```
  WARNING: ralph-wiggum plugin is not installed.

  For large plans, execution may stop if context limits are reached.
  The orchestrator will continue, but session persistence is not available.

  To install ralph-wiggum for automatic session recovery:
    claude plugin marketplace add anthropics/claude-code
    claude plugin install ralph-wiggum@anthropics-claude-code

  Continuing without ralph-wiggum...
  ```
- Continue with execution (the orchestrator will still work, but without session persistence)

### Step 1: Load Plan Context

Read plan files:
- `.claude/plans/{plan-name}/_plan.md` - Plan overview
- `.claude/plans/{plan-name}/*.md` - All task files (excluding _plan.md)

Note: Testing and code standards are auto-included above.

Parse each task file to extract:
- Task number (from filename)
- Status (pending | in_progress | completed | blocked)
- Dependencies (from `Depends on` field)
- Requirements (checkboxes)
- Retry count

Build a dependency graph as a data structure.

### Step 2: Update Plan Status

If plan status is `ready`, change to `in_progress`:
- Edit `.claude/plans/{plan-name}/_plan.md`
- Set `## Status` to `in_progress`

Capture the starting commit hash for later use in Step 4a:
```bash
git rev-parse HEAD
```
Store this as `{starting-commit}` — it's used to identify files changed during plan execution.

### Step 3: Find Ready Tasks

A task is **ready** when:
- Status is `pending`
- ALL dependencies have status `completed`

```
ready_tasks = []
for each task in tasks:
    if task.status == "pending":
        if all(dep.status == "completed" for dep in task.dependencies):
            ready_tasks.append(task)
```

### Step 4: Check Termination Conditions

**All Complete:**
```
if all(task.status == "completed" for task in tasks):
    Run Step 4a: Code Standards Enforcement (quality gate before final completion)
    STOP
```

**Blocked State:**
```
if len(ready_tasks) == 0:
    if any(task.status == "pending" for task in tasks):
        # Tasks exist but none are ready - dependency deadlock or all blocked
        blocked_tasks = [t for t in tasks if t.status == "blocked"]
        Output: TASKS_BLOCKED: {list blocked task numbers and reasons}
        STOP
```

### Step 4a: Code Standards Enforcement

**Why this exists:** TDD workers focus on tests and implementation. Asking them to also perfectly follow every coding standard overloads the context. This dedicated pass has a single focus: standards compliance.

When all tasks are complete, spawn a **single** Task tool subagent to enforce code standards on all files created or modified during plan execution.

**Worker Prompt Template:**

```markdown
You are a code standards enforcer. Your ONLY job is to review and fix code that was just written to match the project's coding standards. Do NOT add features, change behavior, or refactor logic.

## Code Standards
{pass the <code-standards> content from above}

## What to Review

Run this command to find all files created or modified during this plan:
```bash
git diff --name-only --diff-filter=ACM {starting-commit}..HEAD -- packages/
```

For each file, read it and check for violations of the code standards. Common issues to look for:

1. **`readonly class` promotion** — If ALL constructor-promoted properties are individually `readonly`, change to `readonly class` and remove `readonly` from each property
2. **No parentheses around `new` in chains** — `(new Foo())->method()` must be `new Foo()->method()`
3. **Avoid `mixed` types** — Use the narrowest possible type instead of `mixed`
4. **Multiline method signatures** — Every parameter on its own line with trailing comma
5. **Constructor property promotion** — Always use it, no manual assignment
6. **Strict types declaration** — Every file needs `declare(strict_types=1)`
7. **Import classes** — No inline fully qualified names with `\`
8. **No unnecessary curly braces** in string interpolation
9. **No traits** — Use composition
10. **`@throws` PHPDoc tags** — Every method that throws or calls throwing methods
11. **Named parameters** for exceptions (message, context, suggestion)
12. **No dead code** — Unused imports, variables, parameters
13. **Anonymous class braces** — Opening brace on next line

## Process

1. Get the list of modified files
2. Read each file
3. Fix any violations (use Edit tool, not rewriting entire files)
4. After all fixes, run the lint tools:
   ```bash
   {lint command from testing.md or code-standards.md}
   ```
5. Run the full test suite to verify fixes didn't break anything:
   ```bash
   {parallel test command from testing.md}
   ```
6. If tests pass, output: `STANDARDS_ENFORCED`
7. If tests fail after your changes, revert the breaking fix and try again. Output: `STANDARDS_ENFORCED` when stable.
```

**On STANDARDS_ENFORCED:**
1. Create a git commit if any files were changed:
   ```bash
   git add -A && git commit -m "style({plan-name}): enforce coding standards"
   ```
2. Update `_plan.md` status to `completed`
3. Output: `ALL_TASKS_COMPLETE`

**On failure or timeout:**
1. Still mark plan as `completed` (all tasks passed, standards are non-blocking)
2. Output a warning with `ALL_TASKS_COMPLETE`:
   ```
   ALL_TASKS_COMPLETE

   WARNING: Code standards enforcement encountered issues. Run linting manually:
     ./vendor/bin/phpcs
     ./vendor/bin/php-cs-fixer fix
   ```

### Step 5: Spawn Parallel Workers

For EACH ready task, spawn a Task tool subagent **in parallel** (single message with multiple Task tool calls):

```
Use the Task tool with subagent_type="general-purpose" for EACH ready task.
All Task tool calls should be made in a SINGLE message to enable parallel execution.
```

**Worker Prompt Template:**

Pass the auto-included testing and code-standards content to each worker:

```markdown
You are a TDD programmer implementing a single task. Work autonomously until complete.

## Project Configuration
{pass the <testing> content from above}

## Code Standards
{pass the <code-standards> content from above}

## Your Task
{contents of the task file}

## TDD Process (STRICT)

Use the test commands from the project's testing.md. Look for "TDD Workflow Commands" section for optimized commands (RED/GREEN/REFACTOR phases). If not present, use the standard test command.

For EACH unchecked requirement in order:

1. **RED**: Write a failing test
   - Test name MUST match the requirement exactly
   - Run tests with filter/bail flags if available (fast failure confirmation)
   - Verify test FAILS
   - **CRITICAL**: If test passes immediately, you over-implemented in a previous step. Note this and move on.

2. **GREEN**: Write the BARE MINIMUM code to pass
   - Only write enough code to make THIS test pass - nothing more
   - Do NOT handle edge cases that aren't tested yet
   - Do NOT implement other requirements yet
   - Run tests using the parallel test command from `.claude/testing.md` (always use parallel)
   - Verify ALL package tests pass

3. **REFACTOR** (Tidy First): Clean up while tests stay green
   - Separate STRUCTURAL changes (renaming, extracting methods, moving code) from BEHAVIORAL changes
   - Make structural changes first if both are needed
   - One refactoring change at a time
   - Run package-scoped tests after EACH change
   - Prioritize: eliminate duplication, improve clarity, make dependencies explicit

4. **MARK COMPLETE**: Update the task file
   - Change `- [ ]` to `- [x]` for this requirement
   - Add implementation notes if relevant

5. **REPEAT**: Move to next unchecked requirement

## Critical TDD Rules

**Avoid Over-Implementation:**
- NEVER write code that handles multiple cases at once
- Each test must fail before you write the code that makes it pass
- If a test passes immediately, you wrote too much implementation
- The simplest solution that could possibly work is the correct one

**One Requirement at a Time:**
- ONE requirement at a time - never skip ahead
- NEVER write implementation code before a failing test exists
- If a test already passes, note it and move to next requirement

**Code Quality (apply during REFACTOR):**
- Eliminate duplication ruthlessly (DRY)
- Keep methods small and focused (single responsibility)
- Express intent clearly through naming
- Make dependencies explicit
- Minimize state and side effects

**General:**
- Follow the code standards strictly
- Use existing patterns from the codebase
- Write code for production - tests adapt to code, not the other way around

## Output Format

During execution, show your progress:
```
Requirement 1: `{test name}`
  RED: Writing failing test...
  Running tests... FAILED (expected)
  GREEN: Implementing...
  Running tests... PASSED
  Marked [x]

Requirement 2: `{test name}`
  ...
```

## When Complete

After ALL requirements are [x]:
1. Run full test suite using the parallel test command from `.claude/testing.md`
2. Verify all tests pass
3. Output exactly: `TASK_COMPLETE`

If you encounter an unrecoverable error:
1. Document the error in Implementation Notes
2. Output exactly: `TASK_FAILED: {brief reason}`
```

### Step 6: Collect Results

Wait for ALL parallel Task tool calls to complete. For each result:

**On TASK_COMPLETE:**
1. Verify all requirements are checked in task file
2. Set task status to `completed`
3. Create a git commit:
   ```bash
   git add -A && git commit -m "feat({plan-name}): {task title}"
   ```
4. Update the task table in `_plan.md`

**On TASK_FAILED:**
1. Increment retry count in task file
2. If retry count >= 3:
   - Set status to `blocked`
   - Set blocked reason from error message
   - Update task table in `_plan.md`
3. If retry count < 3:
   - Keep status as `pending` (will retry in next batch)
   - Log the failure for visibility

### Step 7: Report Progress

After processing the batch, output:

```
Batch complete:
  Completed: {list of completed task numbers}
  Failed (will retry): {list of failed tasks with retry < 3}
  Blocked: {list of newly blocked tasks}

Progress: {completed}/{total} tasks
Ready for next batch: {count of newly ready tasks}
```

### Step 8: Continue Loop

Return to Step 3 and find the next batch of ready tasks.

Continue until:
- `ALL_TASKS_COMPLETE` - All tasks finished successfully
- `TASKS_BLOCKED: [list]` - No progress possible

## Handling Edge Cases

### Task Already In Progress
If a task has status `in_progress` (from interrupted previous run):
- Treat it as `pending` and include in ready check
- The worker will pick up where it left off based on [x] marks

### Partial Completion
If some requirements are already [x] in a task:
- Worker will skip those and continue with unchecked ones
- This enables resumption after interruption

### Dependency on Blocked Task
If task A depends on blocked task B:
- Task A can never become ready
- It should be reported in final TASKS_BLOCKED output

### Test Failures in Completed Requirements
If a previously passing test starts failing:
- Worker should report TASK_FAILED
- Investigation needed - likely a breaking change

## Session Persistence (Automatic)

The orchestrator automatically uses ralph-wiggum for session persistence if installed (see Step 0).

- If ralph-wiggum is installed: Execution automatically wraps with `/ralph-wiggum:loop` for session recovery
- If not installed: A warning is shown, but execution continues without persistence

The orchestrator outputs `ALL_TASKS_COMPLETE` when done, which satisfies ralph-wiggum's completion promise.

If blocked, it outputs `TASKS_BLOCKED: [...]` which will NOT satisfy the promise, but provides visibility into what's blocking progress.

## Output Summary

Final output should be one of:

**Success:**
```
ALL_TASKS_COMPLETE

Plan: {plan-name}
Total tasks: {N}
All tests passing.
Code standards enforced.
Commits created: {N}
```

**Blocked:**
```
TASKS_BLOCKED: [003, 007]

Plan: {plan-name}
Completed: {X}/{N}
Blocked tasks:
  003: {blocked reason}
  007: {blocked reason}

Manual intervention required for blocked tasks.
```

**Partial Progress (for visibility during execution):**
```
Batch {X} complete.
Progress: {completed}/{total}
Continuing...
```

## Performance Expectations

With proper parallelization:
- 10 independent tasks: ~1-2 batches
- 50 tasks with shallow dependencies: ~3-5 batches
- 100 tasks: ~5-10 batches

Each batch runs tasks in parallel, dramatically reducing total time compared to sequential execution.
