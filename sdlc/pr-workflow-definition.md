# PR Workflow Definition

**Version:** 1.1.0  
**Last Updated:** 2026-04-16  
**Status:** Active

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
5. Pull latest from remote (`git pull origin main`)
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
   - Load context: Read PLAN.md, PR description, recent commits

**Success Criteria:**
- User's intent is clear (new vs existing)
- For existing: context loaded (branch, PR, Jira, PLAN.md)

**Error Handling:**
- Branch name doesn't match pattern → Ask for Jira ticket manually
- No PR found → Offer to create PR
- No PLAN.md found → Offer to create one

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

### Stage 6: Test Plan Creation

**Goal:** Create comprehensive test plan before any implementation

**Actions:**
1. Create `PLAN.md` file with structure:
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

2. Commit PLAN.md:
   ```bash
   git add PLAN.md
   git commit -m "RHCLOUD-45308: Add test plan
   
   Test-first plan for [feature/refactor].
   See PLAN.md for comprehensive test strategy.
   
   Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
   ```

3. Push branch:
   ```bash
   git push -u origin RHCLOUD-45308-description
   ```

**Success Criteria:**
- PLAN.md created with comprehensive test plan
- Committed to branch
- Pushed to remote

---

### Stage 7: Create PR & Test Plan Approval

**Goal:** Get test plan reviewed before implementation

**Actions:**
1. Create PR:
   ```bash
   gh pr create \
     --title "RHCLOUD-45308: embed-spicedb-repository" \
     --body "## Test Plan
   
   Jira: https://redhat.atlassian.net/browse/RHCLOUD-45308
   
   ### Stage: Test Planning
   
   This PR contains the test plan for implementing [feature].
   
   See `PLAN.md` for comprehensive test strategy.
   
   ### Next Steps
   1. ✅ Test plan created
   2. ⏳ Awaiting test plan review
   3. ⏳ Implement tests
   4. ⏳ Implementation plan
   5. ⏳ Implement code
   6. ⏳ Code review
   7. ⏳ Merge
   
   **Comment `/lgtm` to approve test plan and proceed to test implementation.**
   "
   ```

2. Add comment to Jira:
   ```
   PR created: [PR URL]
   
   Status: Test plan ready for review.
   Comment /lgtm on PR to approve and proceed to test implementation.
   ```

3. Wait for approval:
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
3. Commit tests:
   ```bash
   git add [test files]
   git commit -m "RHCLOUD-45308: Implement tests for [feature]
   
   - Add [test case 1]
   - Add [test case 2]
   - All tests currently fail (as expected - no implementation yet)
   
   Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
   ```
4. Push:
   ```bash
   git push
   ```
5. Update PR description:
   ```markdown
   ### Stage: Tests Implemented
   
   1. ✅ Test plan created
   2. ✅ Test plan approved
   3. ✅ Tests implemented
   4. ⏳ Awaiting test review
   5. ⏳ Implementation plan
   ...
   
   **Comment `/lgtm` to approve tests and proceed to implementation planning.**
   ```

**Success Criteria:**
- All planned tests implemented
- Tests fail for expected reasons
- Committed and pushed
- Awaiting `/lgtm`

---

### Stage 9: Implementation Plan Creation

**Goal:** Plan the actual implementation

**Actions:**
1. Confirm test approval (`/lgtm` received)
2. Update `PLAN.md` with implementation section:
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
   git add PLAN.md
   git commit -m "RHCLOUD-45308: Add implementation plan
   
   Implementation strategy for [feature].
   See Implementation Plan section in PLAN.md.
   
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
- Implementation plan added to PLAN.md
- Committed and pushed
- Awaiting `/lgtm`

---

### Stage 10: Implement Code

**Goal:** Implement the feature/fix following the plan

**Actions:**
1. Confirm implementation plan approval (`/lgtm`)
2. Implement following TDD Red-Green-Refactor:
   - **RED:** Tests already fail (done in Stage 8)
   - **GREEN:** Write minimal code to make tests pass
   - **REFACTOR:** Clean up code while keeping tests passing
3. Implement phase by phase from plan
4. Commit logically:
   ```bash
   git commit -m "RHCLOUD-45308: Implement [phase name]
   
   - [Change 1]
   - [Change 2]
   
   Tests passing: Test_[Name1], Test_[Name2]
   
   Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
   ```
5. After all phases:
   - Run full test suite
   - Verify all tests pass
   - Check test coverage
6. Push:
   ```bash
   git push
   ```
7. Update PR:
   ```markdown
   ### Stage: Implementation Complete
   
   1. ✅ Test plan created & approved
   2. ✅ Tests implemented & approved
   3. ✅ Implementation plan created & approved
   4. ✅ Code implemented
   5. ⏳ Awaiting code review
   ...
   
   **Ready for code review.**
   ```

**Success Criteria:**
- All phases implemented
- All tests passing
- Code follows plan
- Committed and pushed

---

### Stage 11: Fresh Context Code Review

**Goal:** Get objective code review from agent with no implementation knowledge

**Actions:**
1. Launch code review agent with fresh context:
   ```
   "Launch a code review agent with these requirements:
   
   **Important: Use fresh context - no knowledge of implementation.**
   
   Agent task:
   - Read PLAN.md to understand intent
   - Review changes: git diff main
   - Check for:
     * Plan adherence
     * Code quality issues
     * Security/performance concerns
     * Test adequacy
     * Bugs or edge cases
   - Create CODE_REVIEW.md with findings
   - Post review as PR comment
   
   Use worktree isolation if possible."
   ```

2. Agent creates `CODE_REVIEW.md`:
   ```markdown
   ## Code Review
   
   ### Plan Adherence
   [Does implementation match plan?]
   
   ### Code Quality
   [Issues with structure, naming, patterns]
   
   ### Security & Performance
   [Concerns?]
   
   ### Testing
   [Adequate coverage?]
   
   ### Specific Issues
   1. File: path/to/file.go:123
      - Issue: [description]
      - Suggestion: [fix]
   
   ### Recommendation
   [Approve / Approve with minor fixes / Needs revision]
   ```

3. Agent posts review as PR comment

**Success Criteria:**
- Fresh context agent completes review
- CODE_REVIEW.md created
- Review posted to PR

---

### Stage 12: Address Review Feedback

**Goal:** Fix issues identified in code review

**Actions:**
1. Read CODE_REVIEW.md
2. Address each issue:
   - Fix the problem
   - Verify fix with tests
   - Commit:
     ```bash
     git commit -m "RHCLOUD-45308: Address review - [what was fixed]
     
     Fixed: [issue from CODE_REVIEW.md]
     
     Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
     ```
3. Update CODE_REVIEW.md (check off addressed items)
4. Push all changes:
   ```bash
   git push
   ```
5. Update PR:
   ```markdown
   ### Stage: Review Feedback Addressed
   
   1. ✅ Code review complete
   2. ✅ Feedback addressed
   3. ⏳ Awaiting final approval
   
   **Ready for final review and merge.**
   ```

**Success Criteria:**
- All review issues addressed
- Tests still passing
- CODE_REVIEW.md updated
- Changes pushed

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
6. Checkout main and pull:
   ```bash
   git checkout main
   git pull origin main
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

## Approval Mechanism

All approval gates use `/lgtm` comments on the PR:

- **Test Plan Approval:** Comment `/lgtm` after reviewing `PLAN.md` test section
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
- 1.1.0 (2026-04-16): Added Stage 1.5 - Support for continuing work on existing PRs
- 1.0.0 (2026-04-16): Initial workflow definition
