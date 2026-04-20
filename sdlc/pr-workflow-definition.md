# PR Workflow Definition

**Version:** 1.4.2  
**Last Updated:** 2026-04-20  
**Status:** Active

**Recent Changes (v1.4.2):**
- **Critical Fix:** Changed sync to use `upstream main` instead of `origin main`
- Ensures syncing with canonical repo, not just user's fork
- Added upstream remote setup instructions
- Prevents issues where fork is behind upstream

**Changes (v1.4.1):**
- Added automatic sync with main before creating PRs (Stage 7 and Stage 10)
- Ensures all PRs are based on latest main
- Prevents merge conflicts and keeps history clean
- Required step before `gh pr create` commands

**Changes (v1.4.0):**
- **Breaking:** Changed from `PLAN.md` to `PLAN-${JIRA_TICKET}.md` (e.g., `PLAN-RHCLOUD-45010.md`)
- Allows multiple concurrent PRs without file conflicts
- Added guidance on referencing Cursor plans from workflow PLAN files
- PLAN files can reference detailed implementation plans (e.g., Cursor plans) instead of duplicating content

**Changes (v1.3.1):**
- Updated Stage 8.5.1: Allow creating implementation branch before test PR merge
- Added Option 2: Create implementation branch from test branch to start work sooner
- Added rebase instructions for implementation branch after test PR merge

**Changes (v1.3.0):**
- Added PR strategy decision: single PR vs separate PRs for tests and implementation
- Added fresh context code review AFTER test implementation (Stage 8.5)
- Added separate PR branch handling for test-only PRs
- Made fresh context review a prerequisite for all commits/pushes
- Allow user to change PR strategy mid-workflow

This document defines the complete PR workflow process. It serves as the single source of truth for how PRs are created, reviewed, and merged.

---

## Overview

The PR workflow is an opinionated, test-first process that ensures:
- Clean git state before starting
- Proper Jira ticket selection and tracking
- Comprehensive planning before implementation
- Test-first development (TDD)
- Fresh context code reviews
- Quality gates at each stage

---

## Workflow Stages

### Stage 1: Git Safety Check

**Goal:** Ensure clean slate for new work

**Actions:**
1. Check current git branch
2. If not on `main`:
   - Inform user
   - Switch to `main` automatically
3. Check for uncommitted changes
4. If uncommitted changes exist:
   - Warn user to stash changes
   - Give 3 attempts
   - After 3 attempts: abort with message "We prefer a clean slate to keep workflow manageable"
5. Pull latest from upstream (`git pull upstream main` then `git push origin main`)
6. Display remote details:
   - Remote name
   - Remote URL
   - Latest commit hash
   - Commits pulled (if any)

**Success Criteria:**
- On `main` branch
- Working tree clean
- Latest changes pulled from remote

**Error Handling:**
- Pull conflicts → Abort, ask user to resolve manually
- Network issues → Retry once, then abort
- Uncommitted changes after 3 attempts → Abort workflow

---

### Stage 1.5: New or Existing PR Decision

**Goal:** Determine if starting fresh or continuing existing work

**Actions:**
1. After git safety check completes, Claude asks:
   ```
   Is this a new PR or are you continuing work on an existing PR?
   1. New PR - Start from Jira search
   2. Existing PR - Resume current work
   
   Choice (1/2):
   ```

2. **If New PR (option 1):**
   - Continue to Stage 2 (Jira Search)
   - Follow full workflow from beginning

3. **If Existing PR (option 2):**
   - Detect current branch name
   - Extract Jira ticket from branch name (e.g., RHCLOUD-46010 from RHCLOUD-46010-bring-tuples-endpoints)
   - Find associated PR using `gh pr list`
   - Ask: "What do you need to do?"
     - Address review feedback (→ jump to Stage 12)
     - Continue implementation (→ ask current stage)
     - Update tests (→ jump to Stage 8-9)
     - Update plan (→ jump to Stage 6 or 10)
   - Load context: Read PLAN-${JIRA_TICKET}.md, PR description, recent commits

**Success Criteria:**
- User's intent is clear (new vs existing)
- For existing: context loaded (branch, PR, Jira, PLAN-${JIRA_TICKET}.md)

**Error Handling:**
- Branch name doesn't match pattern → Ask for Jira ticket manually
- No PR found → Offer to create PR
- No PLAN-${JIRA_TICKET}.md found → Offer to create one

---

### Stage 2: Jira Ticket Search & Selection

**Goal:** Find and select appropriate Jira ticket (for new PRs)

**Actions:**
1. Prompt user: `"What ticket are you working on? (Enter keywords to search)"`
2. User enters keywords (e.g., "relations repository")
3. Search Jira using JQL:
   ```
   project = ${DEFAULT_PROJECT} AND text ~ "${keywords}" 
   AND assignee = currentUser()
   ```
4. Display results with type indicators:
   ```
   Found 5 matches for "relations repository":
   
   1. 📘 Story      RHCLOUD-45308 - Embed SpiceDB repository (New)
   2. 📘 Story      RHCLOUD-45010 - Refactor authorizer interface (In Progress)
   3. ⚠️  Epic       RHCLOUD-44628 - Merge Inventory and Relations (Cannot create PR)
   4. 🐛 Bug        RHCLOUD-45001 - Fix relations API timeout (New)
   5. ✅ Task       RHCLOUD-45100 - Update relations schema (To Do)
   
   Pick (1-5) or 'q' to search again:
   ```
5. User selects number or 'q'
6. If 'q': Return to step 1
7. Validate selection:
   - ✅ Allowed types: Story, Task, Bug, Sub-task
   - ❌ Blocked types: Epic, Initiative
8. If blocked type selected:
   ```
   ⚠️ RHCLOUD-44628 is an Epic. PRs should be created for Stories/Tasks/Bugs within this Epic.
   
   Search again? (Enter new keywords or 'q' to quit):
   ```
9. Fetch full ticket details (summary, description, acceptance criteria)

**Success Criteria:**
- Valid Story/Task/Bug selected
- Full ticket details retrieved

**Error Handling:**
- No search results → Let user search again
- Jira API error → Retry once, then fail gracefully
- Invalid selection → Ask again

**Type Indicators:**
- 📘 Story
- ✅ Task
- 🐛 Bug
- 📋 Sub-task
- ⚠️ Epic (blocked)
- ⚠️ Initiative (blocked)

---

### Stage 3: Branch Creation

**Goal:** Create properly named feature branch

**Actions:**
1. Display ticket summary:
   ```
   Ticket: RHCLOUD-45308
   Summary: Embed SpiceDB repository and tests in inventory-api
   ```
2. Ask user:
   ```
   Branch description:
   1. Use default: "embed-spicedb-repository-and-tests"
   2. Provide custom description
   
   Choice (1/2):
   ```
3. If custom:
   - Prompt: `"Enter branch description:"`
   - Slugify input (lowercase, spaces→dashes, remove special chars)
4. Create branch name: `${JIRA_TICKET}-${description}`
   - Example: `RHCLOUD-45308-embed-spicedb-repository`
5. Create and checkout branch:
   ```bash
   git checkout -b RHCLOUD-45308-embed-spicedb-repository
   ```
6. Confirm:
   ```
   ✓ Created and switched to branch: RHCLOUD-45308-embed-spicedb-repository
   ```

**Success Criteria:**
- Branch created with format: `JIRA-123-description`
- Checked out to new branch
- Branch name is valid (no special characters, reasonable length)

**Error Handling:**
- Branch already exists → Offer to checkout or create with suffix
- Invalid characters → Clean automatically
- Too long (>100 chars) → Truncate and confirm with user

---

### Stage 4: Understanding Check

**Goal:** Ensure proper context and staff engineer review for significant changes

**Actions:**
1. Ask user:
   ```
   Does this change involve API or database changes? (y/n)
   ```
2. If 'y':
   - Ask: `"Have you discussed this with staff engineers? (y/n)"`
   - If 'n':
     ```
     ⚠️ Recommended: Discuss API/DB changes with staff engineers first.
     
     This helps ensure:
     - Design alignment with architecture
     - No unintended breaking changes
     - Performance implications considered
     
     Continue anyway? (y/n)
     ```
   - If user confirms 'n' to continue: Abort workflow
   - If user confirms 'y': Proceed with note in plan

**Success Criteria:**
- User has confirmed understanding
- Staff engineer discussion confirmed (or acknowledged warning)

**Future Enhancements:**
- Jira labels/tags to detect API/DB changes automatically
- Integration with Slack to notify staff engineers
- CodeRabbit integration for automated checks

---

### Stage 5: TDD Planning Questions

**Goal:** Understand what needs to be tested before implementation

**Actions:**
1. Classify the task:
   ```
   Is this a:
   1. NEW FEATURE (adding new functionality)
   2. REFACTOR (changing internal structure, same behavior)
   3. BUG FIX (fixing incorrect behavior)
   4. ENHANCEMENT (extending existing functionality)
   
   Choice (1-4):
   ```

2. **For NEW FEATURES**, ask:
   - "What's the exact behavior we're adding?"
   - "What are all possible inputs (valid, invalid, edge cases)?"
   - "What are the success criteria?"
   - "What can go wrong (error scenarios)?"
   - "What's the simplest test that should fail first?"
   - "What tests should we write before any code?"
   - "What's the expected failure for each test?"

3. **For REFACTORS**, ask:
   - "What's the current behavior (exactly)?"
   - "What tests exist today?"
   - "What tests are MISSING before we refactor?"
   - "What behavior must stay the same?"
   - "What characterization tests do we need?"
   - "How will we verify the refactor is safe?"

4. **For BUG FIXES**, ask:
   - "What's the incorrect behavior?"
   - "What should happen instead?"
   - "How do we reproduce it?"
   - "What test will catch this bug?"

5. **For ENHANCEMENTS**, ask:
   - Combination of NEW FEATURE + REFACTOR questions

**Success Criteria:**
- Task type identified
- Comprehensive answers to all questions
- Clear understanding of testing needs

---

### Stage 5.5: PR Strategy Decision

**Goal:** Decide whether to use single PR or separate PRs for tests and implementation

**Actions:**
1. Ask user:
   ```
   How would you like to structure your PRs?
   
   1. Single PR - Tests and implementation together (simpler, faster)
   2. Separate PRs - Test-only PR first, then implementation PR (safer for refactors)
   
   Recommendation:
   - Single PR: Good for small features, bug fixes
   - Separate PRs: Best for refactors, large features, risky changes
   
   Choice (1/2):
   ```

2. Store decision in workflow state
3. Display choice:
   ```
   ✓ Using [Single PR / Separate PRs] strategy
   
   Note: You can change this decision later by telling me
   "switch to single PR" or "switch to separate PRs"
   ```

**Success Criteria:**
- User has chosen PR strategy
- Strategy stored for later stages

**Changing Your Mind:**
User can say at any time:
- "Actually, let's use separate PRs instead"
- "Let's combine into a single PR"
- "Switch to [single/separate] PR strategy"

When strategy changes:
- If already have test-only PR: Offer to close or merge first
- If implementing: Adjust next steps accordingly
- Update PLAN-${JIRA_TICKET}.md status to reflect new strategy

---

### Stage 6: Test Plan Creation

**Goal:** Create comprehensive test plan before any implementation

**Actions:**
1. Create `PLAN-${JIRA_TICKET}.md` file (e.g., `PLAN-RHCLOUD-45308.md`) with structure:
   ```markdown
   ---
   jira: RHCLOUD-45308
   status: test-planning
   type: [new-feature|refactor|bug-fix|enhancement]
   ---
   
   # Implementation Plan: [Title from Jira]
   
   ## Jira Ticket
   RHCLOUD-45308 - https://redhat.atlassian.net/browse/RHCLOUD-45308
   
   ## Task Type
   [NEW FEATURE / REFACTOR / BUG FIX / ENHANCEMENT]
   
   ## Objective
   [What we're achieving and why - from Jira + user input]
   
   ## Current State
   [Current architecture/implementation]
   
   ---
   
   ## 🧪 Test Plan
   
   ### Tests That Should Fail First
   [For new features - what tests to write before code]
   
   ### Missing Tests
   [For refactors - what tests are missing today]
   
   ### Test Coverage Goals
   - Before: X%
   - After: Y%
   - Critical paths: 100%
   
   ### Test Cases
   
   #### Happy Path Tests
   1. Test: `Test_[Feature]_[Scenario]`
      - What it tests: [behavior]
      - Why it fails: [reason]
      - Expected failure: [error message]
   
   #### Edge Case Tests
   ...
   
   #### Error Handling Tests
   ...
   
   ---
   
   ## Implementation Plan
   [To be added after test plan approved]
   ```

2. Commit plan file:
   ```bash
   git add PLAN-RHCLOUD-45308.md
   git commit -m "RHCLOUD-45308: Add test plan
   
   Test-first plan for [feature/refactor].
   See PLAN-RHCLOUD-45308.md for comprehensive test strategy.
   
   Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
   ```

3. Push branch:
   ```bash
   git push -u origin RHCLOUD-45308-description
   ```

**Success Criteria:**
- PLAN-${JIRA_TICKET}.md created with comprehensive test plan
- Committed to branch
- Pushed to remote

**File Naming:**
- Use pattern: `PLAN-${JIRA_TICKET}.md`
- Example: `PLAN-RHCLOUD-45308.md`
- Allows multiple concurrent PRs without conflicts
- Easy to find: `ls PLAN-*.md`

**Integration with Tool-Specific Plans (Cursor, etc.):**

If you have detailed plans from development tools (e.g., Cursor plans in `.cursor/plans/`), you can reference them from the workflow PLAN file instead of duplicating content:

```markdown
## Implementation Plan

**Detailed implementation:** See Cursor plan at `.cursor/plans/feature_name_abc123.plan.md`

**Summary:**
- Phase 1: [High-level description]
- Phase 2: [High-level description]
[...]

**Key decisions:**
- [Decision 1 and rationale]
- [Decision 2 and rationale]
```

This approach:
- Avoids duplication
- Lets tools manage their own detailed plans
- Provides reviewers with context + link to details
- Both files coexist and serve different purposes

---

### Stage 7: Create PR & Test Plan Approval

**Goal:** Get test plan reviewed before implementation

**Actions:**
1. **Sync with main before creating PR:**
   ```bash
   git checkout main
   git pull upstream main    # Pull from upstream (canonical repo), not origin (your fork)
   git push origin main       # Update your fork
   git checkout RHCLOUD-45308-description
   git rebase main
   # Resolve conflicts if any
   git push --force-with-lease
   ```
   
   **Why:** Ensures PR is based on latest upstream main, prevents merge conflicts, keeps history clean
   
   **Note:** If `upstream` remote doesn't exist, add it once:
   ```bash
   git remote add upstream git@github.com:project-kessel/inventory-api.git
   ```

2. Extract JIRA ticket from branch name:
   - Branch format: `RHCLOUD-45308-description`
   - Extract: `RHCLOUD-45308`
   - If extraction fails, ask user for JIRA ticket

3. Create PR with JIRA ticket in title:
   ```bash
   # Title format: "JIRA-TICKET: brief-description"
   gh pr create \
     --title "RHCLOUD-45308: embed-spicedb-repository" \
     --body "## Test Plan
   
   Jira: https://redhat.atlassian.net/browse/RHCLOUD-45308
   
   ### Stage: Test Planning
   
   This PR contains the test plan for implementing [feature].
   
   See \`PLAN-RHCLOUD-45308.md\` for comprehensive test strategy.
   
   ### Next Steps
   1. ✅ Test plan created
   2. ⏳ Awaiting test plan review
   3. ⏳ Implement tests
   4. ⏳ Implementation plan
   5. ⏳ Implement code
   6. ⏳ Code review
   7. ⏳ Merge
   
   **Comment \`/lgtm\` to approve test plan and proceed to test implementation.**
   "
   ```

   **IMPORTANT:** PR title MUST include JIRA ticket number. GitHub workflow checks require this format.

4. Add comment to Jira:
   ```
   PR created: [PR URL]
   
   Status: Test plan ready for review.
   Comment /lgtm on PR to approve and proceed to test implementation.
   ```

5. Wait for approval:
   - Poll PR comments for `/lgtm`
   - Or user tells Claude "test plan approved"

**Success Criteria:**
- PR created
- Jira updated with PR link
- `/lgtm` comment received

---

### Stage 8: Implement Tests

**Goal:** Implement tests from approved test plan

**Actions:**
1. Confirm approval received
2. Implement tests following TDD:
   - Write failing tests first
   - One test at a time
   - Verify each test fails for the right reason
3. **BEFORE COMMITTING:** Run fresh context code review (see Stage 8.5 below)
4. After review approval, commit tests:
   ```bash
   git add [test files]
   git commit -m "RHCLOUD-45308: Implement tests for [feature]
   
   - Add [test case 1]
   - Add [test case 2]
   - All tests currently fail (as expected - no implementation yet)
   
   Reviewed-by: Fresh Context Agent
   Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
   ```
5. Run linter:
   ```bash
   make lint
   ```
6. Push:
   ```bash
   git push
   ```
7. Update PR description based on strategy:
   
   **If Single PR:**
   ```markdown
   ### Stage: Tests Implemented
   
   1. ✅ Test plan created & approved
   2. ✅ Tests implemented & reviewed
   3. ⏳ Awaiting `/lgtm` to proceed to implementation plan
   ```
   
   **If Separate PRs:**
   ```markdown
   ### Stage: Test-Only PR
   
   1. ✅ Test plan created & approved
   2. ✅ Tests implemented & reviewed
   3. ⏳ Awaiting `/lgtm` to merge test-only PR
   
   **After merge, will create new PR for implementation.**
   ```

**Success Criteria:**
- All planned tests implemented
- Tests reviewed by fresh context agent
- All issues addressed
- Lint passing
- Tests fail for expected reasons (RED phase)
- Committed and pushed

**IMPORTANT:** Never commit or push without fresh context review first.

---

### Stage 8.5: Fresh Context Test Review

**Goal:** Get objective review of tests before committing

**Actions:**
1. **BEFORE any commit/push**, launch code review agent:
   ```
   "Launch a code review agent for test code review:
   
   **Important: Use fresh context - no knowledge of test implementation.**
   
   Agent task:
   - Read PLAN-${JIRA_TICKET}.md to understand test requirements
   - Review test changes: git diff --cached or git diff main
   - Check for:
     * Test coverage completeness
     * Test clarity and maintainability
     * Edge cases covered
     * Error scenarios tested
     * Test naming conventions
     * Unnecessary mocks vs fakes
     * Performance concerns (test speed)
   - Create TEST_REVIEW.md with findings
   - Report findings
   
   Use worktree isolation if possible."
   ```

2. Agent creates `TEST_REVIEW.md`:
   ```markdown
   ## Test Code Review
   
   ### Test Coverage
   [Are all test cases from PLAN-${JIRA_TICKET}.md implemented?]
   
   ### Test Quality
   [Clarity, maintainability, naming]
   
   ### Edge Cases & Errors
   [Are edge cases and error scenarios covered?]
   
   ### Testing Patterns
   [Using fakes over mocks? Appropriate test isolation?]
   
   ### Specific Issues
   1. File: path/to/test.go:123
      - Issue: [description]
      - Suggestion: [fix]
   
   ### Recommendation
   [Approve / Approve with minor fixes / Needs revision]
   ```

3. Address any issues raised
4. Only proceed to commit after approval

**Success Criteria:**
- Fresh context agent completes test review
- TEST_REVIEW.md created
- All issues addressed
- Agent approves tests

**CRITICAL RULE:** No code commits (test or implementation) without fresh context review first.

---

### Stage 8.5.1: Separate PR Branch Handling (If Applicable)

**Goal:** For separate PR strategy, create test-only PR and prepare for implementation PR

**Actions:**

**Only applies if user chose "Separate PRs" in Stage 5.5**

1. Ask user:
   ```
   Test PR ready. How would you like to proceed?
   
   1. Wait for test PR approval/merge, then create implementation branch (safer)
   2. Create implementation branch now from test branch (start work sooner)
   
   Recommendation:
   - Option 1: Best if test PR might need significant changes
   - Option 2: Best if tests are solid and you want to start implementation
   
   Choice (1/2):
   ```

**Option 1: Merge Test PR First (Original Flow)**

2a. Wait for test PR approval (`/lgtm`)
3a. Merge test-only PR:
   ```bash
   gh pr merge --squash
   ```
4a. Checkout main and pull from upstream:
   ```bash
   git checkout main
   git pull upstream main
   git push origin main
   ```
5a. Create new branch for implementation:
   ```bash
   # New branch name format: JIRA-123-implementation-description
   # Example: RHCLOUD-45308-implement-spicedb-repository
   git checkout -b RHCLOUD-45308-implement-[description]
   ```
6a. Update Jira:
   ```
   ✅ Test-only PR merged: [PR URL]
   
   Starting implementation PR: [new branch name]
   ```

**Option 2: Create Implementation Branch Now**

2b. Create new branch from current test branch:
   ```bash
   # New branch name format: JIRA-123-implementation-description
   # Example: RHCLOUD-45308-implement-spicedb-repository
   git checkout -b RHCLOUD-45308-implement-[description]
   ```
3b. Note in Jira:
   ```
   Test PR pending: [test PR URL]
   
   Starting implementation on separate branch: [impl branch name]
   
   Implementation branch created from test branch.
   Will rebase onto main after test PR merges.
   ```
4b. **Important:** After test PR merges, rebase implementation branch:
   ```bash
   git checkout main
   git pull upstream main
   git push origin main
   git checkout RHCLOUD-45308-implement-[description]
   git rebase main
   # Resolve conflicts if any
   git push --force-with-lease
   ```

**Success Criteria:**

**Option 1:**
- Test-only PR merged to main
- New implementation branch created from main
- On new branch ready for implementation
- Jira updated

**Option 2:**
- New implementation branch created from test branch
- On new branch ready for implementation
- Jira updated with note about pending test PR
- User reminded to rebase after test PR merge

**If Single PR:**
- Skip this stage entirely
- Continue to Stage 9 on same branch

---

### Stage 9: Implementation Plan Creation

**Goal:** Plan the actual implementation

**Actions:**
1. Confirm test approval (`/lgtm` received)
2. Update `PLAN-${JIRA_TICKET}.md` with implementation section:
   ```markdown
   ## Implementation Plan
   
   ### Phase 1: [Name]
   - File: path/to/file.go
     - Change: [specific change]
     - Rationale: [why needed]
     - Tests that should pass: Test_[Name]
   
   ### Phase 2: [Name]
   ...
   
   ### Files to Modify
   - path/to/file1.go
   - path/to/file2.go
   
   ### Files to Create
   - path/to/newfile.go
   
   ### Migration/Rollout
   [If applicable]
   ```

3. Commit:
   ```bash
   git add PLAN-RHCLOUD-45308.md
   git commit -m "RHCLOUD-45308: Add implementation plan
   
   Implementation strategy for [feature].
   See Implementation Plan section in PLAN-RHCLOUD-45308.md.
   
   Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
   ```

4. Push and update PR:
   ```markdown
   ### Stage: Implementation Planning
   
   1. ✅ Test plan created
   2. ✅ Test plan approved
   3. ✅ Tests implemented
   4. ✅ Tests approved
   5. ✅ Implementation plan created
   6. ⏳ Awaiting plan review
   ...
   
   **Comment `/lgtm` to approve implementation plan and proceed to coding.**
   ```

**Success Criteria:**
- Implementation plan added to PLAN-${JIRA_TICKET}.md
- Committed and pushed
- Awaiting `/lgtm`

---

### Stage 10: Implement Code

**Goal:** Implement the feature/fix following the plan

**Actions:**
1. Confirm implementation plan approval (`/lgtm`)

2. **Sync with main before creating PR (if creating new PR):**
   ```bash
   git checkout main
   git pull upstream main    # Pull from upstream (canonical repo), not origin (your fork)
   git push origin main       # Update your fork
   git checkout RHCLOUD-45308-implement-[description]
   git rebase main
   # Resolve conflicts if any
   git push --force-with-lease
   ```
   
   **Note:** 
   - For Single PR strategy: Skip this if PR already exists, just continue on same branch
   - For Separate PRs: Always sync before creating implementation PR
   - If `upstream` remote doesn't exist, add it once:
     ```bash
     git remote add upstream git@github.com:project-kessel/inventory-api.git
     ```

3. **If Separate PRs:** Create implementation PR:
   ```bash
   gh pr create \
     --title "RHCLOUD-45308: Implement [description]" \
     --body "## Implementation PR
   
   Jira: https://redhat.atlassian.net/browse/RHCLOUD-45308
   
   Depends on test PR: #[test-pr-number] (merged)
   
   See \`PLAN-RHCLOUD-45308.md\` for implementation details."
   ```

4. Implement following TDD Red-Green-Refactor:
   - **RED:** Tests already fail (done in Stage 8)
   - **GREEN:** Write minimal code to make tests pass
   - **REFACTOR:** Clean up code while keeping tests passing

5. Implement phase by phase from plan

6. **BEFORE each commit:** Run fresh context code review (similar to Stage 8.5)

7. After review approval, commit logically:
   ```bash
   git commit -m "RHCLOUD-45308: Implement [phase name]
   
   - [Change 1]
   - [Change 2]
   
   Tests passing: Test_[Name1], Test_[Name2]
   
   Reviewed-by: Fresh Context Agent
   Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
   ```

8. After all phases:
   - Run full test suite: `make test`
   - Verify all tests pass
   - Run linter: `make lint`
   - Check test coverage

9. Push:
   ```bash
   git push
   ```

10. Update PR:
   ```markdown
   ### Stage: Implementation Complete
   
   1. ✅ Test plan created & approved
   2. ✅ Tests implemented & reviewed
   3. ✅ Implementation plan created & approved
   4. ✅ Code implemented & reviewed
   5. ⏳ Awaiting final code review
   ...
   
   **Ready for final code review.**
   ```

**Success Criteria:**
- All phases implemented
- Each phase reviewed by fresh context agent before commit
- All tests passing
- Linter passing
- Code follows plan
- Committed and pushed

**IMPORTANT:** Every commit must be reviewed by fresh context agent first. No exceptions.

---

### Stage 11: Fresh Context Final Implementation Review

**Goal:** Get comprehensive objective review of complete implementation

**Actions:**

**Note:** This is a final comprehensive review. Individual commits should have already been reviewed in Stage 10.

1. Launch code review agent with fresh context:
   ```
   "Launch a final code review agent with these requirements:
   
   **Important: Use fresh context - no knowledge of implementation.**
   
   Agent task:
   - Read PLAN-${JIRA_TICKET}.md to understand overall intent
   - Review ALL changes: git diff main
   - Check for:
     * Overall plan adherence
     * Code quality and patterns
     * Security/performance concerns
     * Test adequacy and coverage
     * Integration issues
     * Bugs or edge cases
     * Documentation completeness
   - Create FINAL_CODE_REVIEW.md with findings
   - Post review as PR comment
   
   Use worktree isolation if possible."
   ```

2. Agent creates `FINAL_CODE_REVIEW.md`:
   ```markdown
   ## Final Code Review
   
   ### Overall Assessment
   [High-level view of the implementation]
   
   ### Plan Adherence
   [Does implementation match plan?]
   
   ### Code Quality
   [Issues with structure, naming, patterns, consistency]
   
   ### Security & Performance
   [Concerns? Optimizations needed?]
   
   ### Testing
   [Adequate coverage? Edge cases covered?]
   
   ### Integration
   [Does it integrate well with existing code?]
   
   ### Documentation
   [Is code self-documenting? Comments where needed?]
   
   ### Specific Issues
   1. File: path/to/file.go:123
      - Issue: [description]
      - Suggestion: [fix]
   
   ### Recommendation
   [Approve / Approve with minor fixes / Needs major revision]
   ```

3. Agent posts review as PR comment

**Success Criteria:**
- Fresh context agent completes final review
- FINAL_CODE_REVIEW.md created
- Review posted to PR

---

### Stage 12: Address Review Feedback

**Goal:** Fix issues identified in code review

**Actions:**

1. **Detect PR and fetch comments:**
   ```bash
   # Get current branch and extract JIRA ticket
   BRANCH=$(git branch --show-current)
   JIRA=$(echo $BRANCH | grep -oE 'RHCLOUD-[0-9]+')
   
   # Find PR for this branch
   PR_NUM=$(gh pr list --head $BRANCH --json number --jq '.[0].number')
   
   # Fetch PR comments
   gh pr view $PR_NUM
   ```

2. **Interactive comment selection:**
   
   Claude presents comments with numbers:
   ```
   PR Comments for review:
   
   1. [user-alice] Line 45 in auth.go:
      "This should use context.WithTimeout instead of bare context"
   
   2. [user-bob] Line 123 in handler.go:
      "Add error handling for nil pointer"
   
   3. [user-alice] General comment:
      "Great work! Just minor suggestions above"
   
   Which comments would you like to address?
   - Type "all" to address all comments
   - Type comment numbers (e.g., "1,2" or "1 2")
   - Type "skip" to skip addressing feedback
   
   Your choice:
   ```

3. **Address selected comments:**
   
   For each selected comment:
   - Read the comment context
   - Fix the issue
   - Verify fix with tests
   - Commit with reference:
     ```bash
     git commit -m "Address PR review comment #1
     
     Fixed: Use context.WithTimeout in auth.go (comment by user-alice)
     
     Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
     ```

4. **Track progress:**
   - Mark addressed comments
   - Show remaining comments
   - Ask if user wants to address more

5. **Push changes:**
   ```bash
   git push
   ```

6. **Add PR comment summarizing changes:**
   ```bash
   gh pr comment $PR_NUM --body "### Review feedback addressed
   
   ✅ Addressed comments: #1, #2
   
   **Changes:**
   - Comment #1: Use context.WithTimeout in auth.go
   - Comment #2: Add nil pointer checks in handler.go
   
   Ready for re-review."
   ```

**Success Criteria:**
- Selected review issues addressed
- Tests still passing
- Changes committed with comment references
- Changes pushed
- PR updated with summary

**Interactive Flow:**
- User can address comments incrementally
- User can skip non-critical comments
- Clear tracking of which comments are addressed
- Each commit references specific comment

---

### Stage 13: Final Approval & Merge

**Goal:** Get human approval and merge to main

**Actions:**
1. Wait for GitHub approval (not just `/lgtm` comment)
2. Verify all checks pass:
   - CI/CD pipeline (if configured)
   - All tests passing
   - No merge conflicts
3. Final Jira update:
   ```
   PR approved and ready to merge: [PR URL]
   
   All stages completed:
   ✅ Test plan
   ✅ Tests implemented
   ✅ Implementation plan
   ✅ Code implemented
   ✅ Code reviewed
   ✅ Feedback addressed
   ✅ Final approval received
   ```
4. Merge PR:
   ```bash
   gh pr merge --squash
   ```
   Or via GitHub UI (preferred if complex)
5. Delete feature branch (after merge):
   ```bash
   git branch -d RHCLOUD-45308-description
   git push origin --delete RHCLOUD-45308-description
   ```
6. Checkout main and pull from upstream:
   ```bash
   git checkout main
   git pull upstream main
   git push origin main
   ```
7. Final Jira update:
   ```
   ✅ PR merged to main: [PR URL]
   
   Workflow complete!
   ```

**Success Criteria:**
- PR merged to main
- Feature branch deleted
- On main branch with latest changes
- Jira ticket updated

---

## Pre-Commit/Pre-Push Requirements

**CRITICAL:** All code changes (tests or implementation) MUST be reviewed by a fresh context agent BEFORE committing.

### Before Every Commit

1. **Stage changes:**
   ```bash
   git add [files]
   ```

2. **Run fresh context review:**
   - Launch agent with fresh context (no implementation knowledge)
   - Agent reviews: `git diff --cached` (staged changes)
   - Agent checks code quality, tests, security, patterns
   - Agent approves or requests changes

3. **Address issues if any**

4. **Only after approval, commit:**
   ```bash
   git commit -m "..."
   ```

5. **Run linter:**
   ```bash
   make lint
   ```

6. **Push:**
   ```bash
   git push
   ```

### Review Frequency

- **Test code:** Fresh review for each test commit (Stage 8.5)
- **Implementation code:** Fresh review for each implementation commit (Stage 10)
- **Final review:** Comprehensive review of all changes (Stage 11)

### Why Fresh Context?

Fresh context reviews ensure:
- **Objectivity:** Agent has no preconceptions about the implementation
- **Unbiased:** No knowledge of discussions or decisions made during development  
- **Fresh eyes:** Catches issues that the implementer might miss
- **Quality:** Enforces consistent standards across all code

### Exceptions

No exceptions. Even "trivial" changes get reviewed.

Rationale:
- "Trivial" changes often introduce bugs
- Builds good habits
- Prevents technical debt accumulation
- Maintains code quality standards

---

## Approval Mechanism

All approval gates use `/lgtm` comments on the PR:

- **Test Plan Approval:** Comment `/lgtm` after reviewing `PLAN-${JIRA_TICKET}.md` test section
- **Test Implementation Approval:** Comment `/lgtm` after reviewing test code
- **Implementation Plan Approval:** Comment `/lgtm` after reviewing implementation section
- **Final Merge:** Use GitHub's approval system (required reviewer approval)

**Alternative:** User can tell Claude directly:
- "The test plan is approved, continue"
- "Tests look good, implement the code"
- "Plan approved, proceed"

---

## Configuration

Workflow behavior controlled by:
- `~/.pr-workflow-config` - User preferences
  - `DEFAULT_PROJECT` - Jira project to search
  - `DEFAULT_SEARCH_SCOPE` - Search scope (your_tickets, team_tickets, all)
  - `DEFAULT_SLACK_CHANNEL` - Slack notifications (future)
  
---

## Error Recovery

### Common Issues & Solutions

**Issue: Git pull conflicts**
- Abort workflow
- Ask user to resolve conflicts manually
- Restart workflow after resolution

**Issue: Branch already exists**
- Offer to checkout existing branch
- Or create with numeric suffix (e.g., `-2`)

**Issue: Jira API timeout**
- Retry once
- If fails again, allow manual ticket ID entry

**Issue: Tests fail during implementation**
- Expected during RED phase
- Verify failure is for correct reason
- Proceed to GREEN phase

**Issue: `/lgtm` not detected**
- Check PR comment format
- Offer manual override option
- User can tell Claude "it's approved"

---

## Customization

To customize this workflow:

1. Edit this file: `~/git/ai-tools/sdlc/pr-workflow-definition.md`
2. Update corresponding functions in `~/.jira-github-slack-helpers.sh`
3. Update tests in `~/git/ai-tools/sdlc/tests/`
4. Commit changes:
   ```bash
   cd ~/git/ai-tools
   git add .
   git commit -m "Update workflow: [what changed]"
   git push
   ```

---

## Future Enhancements

- [ ] Slack integration for notifications
- [ ] CodeRabbit integration for automated reviews
- [ ] Jira label/tag detection for API/DB changes
- [ ] Automated deployment after merge
- [ ] Metrics and analytics on workflow stages
- [ ] Support for multiple main branches (develop, staging, main)
- [ ] Integration with project-specific CLAUDE.md files

---

**Version History:**
- 1.4.2 (2026-04-20):
  - **Critical Fix:** Changed all `git pull origin main` to `git pull upstream main`
  - Syncs with canonical repo (upstream), not user's fork (origin)
  - Added `git push origin main` after upstream pull to keep fork updated
  - Added upstream remote setup instructions
  - Affects: Stage 1, Stage 7, Stage 8.5.1, Stage 10
- 1.4.1 (2026-04-20):
  - Added automatic sync with main before creating PRs
  - Stage 7: Sync before creating test PR
  - Stage 10: Sync before creating implementation PR
  - Prevents merge conflicts and ensures PR based on latest main
- 1.4.0 (2026-04-20):
  - **Breaking:** Changed from `PLAN.md` to `PLAN-${JIRA_TICKET}.md` for multi-PR support
  - Updated all stages to use per-ticket plan files (e.g., `PLAN-RHCLOUD-45010.md`)
  - Added guidance on integrating with tool-specific plans (Cursor, etc.)
  - PLAN files can reference external detailed plans instead of duplicating
  - Allows multiple concurrent PRs without file conflicts
- 1.3.1 (2026-04-17):
  - Updated Stage 8.5.1 - Added option to create implementation branch before test PR merge
  - Added Option 2 to Stage 8.5.1 - Branch from test branch to start implementation sooner
  - Added rebase instructions for implementation branch after test PR merges
- 1.3.0 (2026-04-16): 
  - Added Stage 5.5 - PR strategy decision (single vs separate PRs)
  - Added Stage 8.5 - Fresh context test review before test commits
  - Added Stage 8.5.1 - Separate PR branch handling
  - Made fresh context review mandatory for all commits/pushes
  - Added comprehensive pre-commit/pre-push requirements section
  - Updated Stage 10 to require fresh context review for each implementation commit
  - Updated Stage 11 to clarify it's a final comprehensive review
  - Added ability to change PR strategy mid-workflow
- 1.2.0 (2026-04-16): Added JIRA ticket auto-extraction and interactive review feedback
- 1.1.0 (2026-04-16): Added Stage 1.5 - Support for continuing work on existing PRs
- 1.0.0 (2026-04-16): Initial workflow definition
