---
jira: N/A (Internal workflow improvement)
status: planning
type: refactor
---

# Implementation Plan: Refactor PR Workflow to Start from Jira by Default

## Objective

Refactor the PR workflow to make Jira the primary entry point, with a streamlined, opinionated flow that guides users through git safety, ticket selection, branch creation, planning, implementation, and review - all driven from a single workflow definition stored in version control.

## Current State

### Existing Files
- `~/.jira-github-slack-helpers.sh` - Helper functions
- `~/.pr-workflow-config` - User configuration  
- `~/.pr-workflow-guide.md` - Documentation
- `~/.pr-workflow-prompts.md` - Template prompts
- `~/CLAUDE.md` - Claude Code guidance

### Current Workflow Entry Points
1. `start-pr-workflow JIRA-123 'description'` - Manual description entry
2. `start-from-jira JIRA-123` - Fetch from Jira (separate option)
3. User tells Claude directly - inconsistent interpretation

### Current Issues
- Two entry points cause confusion
- No git safety checks
- No branch creation automation
- No approval gate mechanism
- Workflow steps scattered across multiple files
- No single source of truth for workflow definition

---

## Task Type

**REFACTOR**

This is a refactoring of the existing PR workflow system. External behavior (creating plans, implementing code, reviews) stays the same, but the internal structure and user experience changes significantly.

---

## 🧪 TDD Analysis

### What behavior must stay EXACTLY the same?

- ✅ Test-first planning approach
- ✅ Probing questions during planning
- ✅ Fresh context agent reviews
- ✅ Multi-stage workflow (plan → implement → review)
- ✅ Jira integration and commenting
- ✅ GitHub PR creation and management

### What behavior is changing?

- ❌ Entry point: Always starts from Jira (not manual description)
- ❌ Git safety: Added checks for clean state, main branch, latest pull
- ❌ Branch creation: Automated from Jira ticket
- ❌ Staff engineer checks: Added for API/DB changes
- ❌ Approval mechanism: `/lgtm` comment-based gates
- ❌ Workflow definition: Stored in git, version controlled

---

## Missing Tests (Add BEFORE Refactoring)

Since this is a refactor of shell scripts and workflow orchestration, "tests" means validation scripts and manual verification steps.

### 1. Characterization Tests (Document Current Behavior)

**Purpose:** Ensure refactor doesn't break existing functionality

Create: `ai-tools/sdlc/tests/validate-current-workflow.sh`

Tests:
- [ ] `test_jira_search_works` - Current jira-search returns results
- [ ] `test_config_loading` - Config loads from ~/.pr-workflow-config
- [ ] `test_jira_commands` - All jira commands (view, comment, list) work
- [ ] `test_github_commands` - All gh commands work
- [ ] `test_helper_functions_exist` - All current functions are callable

**Expected:** All pass (documents current working state)

### 2. Git Safety Validation (New Functionality)

Create: `ai-tools/sdlc/tests/test-git-safety.sh`

Tests:
- [ ] `test_detect_not_on_main` - Detects when not on main branch
- [ ] `test_detect_uncommitted_changes` - Detects dirty working tree
- [ ] `test_pull_latest` - Can pull from remote
- [ ] `test_abort_after_3_stash_attempts` - Workflow aborts after 3 tries

**Expected:** Initially FAIL (functionality doesn't exist yet)

### 3. Jira Ticket Type Filtering (New Functionality)

Create: `ai-tools/sdlc/tests/test-jira-filtering.sh`

Tests:
- [ ] `test_identify_story` - Correctly identifies Story type
- [ ] `test_identify_epic` - Correctly identifies Epic type
- [ ] `test_block_epic_selection` - Prevents PR creation from Epic
- [ ] `test_allow_story_selection` - Allows PR creation from Story/Task/Bug

**Expected:** Initially FAIL (functionality doesn't exist yet)

### 4. Branch Naming Validation

Create: `ai-tools/sdlc/tests/test-branch-naming.sh`

Tests:
- [ ] `test_branch_format` - Creates `RHCLOUD-1234-description` format
- [ ] `test_custom_description` - Handles user custom description
- [ ] `test_default_description` - Uses Jira summary as default
- [ ] `test_slugification` - Converts "Spaces Here" → "spaces-here"

**Expected:** Initially FAIL (functionality doesn't exist yet)

### 5. Approval Gate Mechanism

Create: `ai-tools/sdlc/tests/test-approval-gates.sh`

Tests:
- [ ] `test_detect_lgtm_comment` - Detects `/lgtm` in PR comments
- [ ] `test_wait_for_approval` - Workflow pauses until approval
- [ ] `test_continue_after_lgtm` - Workflow proceeds after `/lgtm`

**Expected:** Initially FAIL (functionality doesn't exist yet)

---

## Proposed Changes (Test-First Order)

### Phase 0: Setup and Validation

**Goal:** Create infrastructure for new workflow

**Files to create:**
```
~/git/ai-tools/                                   (new git repo)
├── .gitignore
├── README.md
├── sdlc/
│   ├── pr-workflow-definition.md                 (master workflow definition)
│   ├── tests/
│   │   ├── validate-current-workflow.sh          (characterization tests)
│   │   ├── test-git-safety.sh
│   │   ├── test-jira-filtering.sh
│   │   ├── test-branch-naming.sh
│   │   └── test-approval-gates.sh
│   └── examples/
│       └── example-plan.md                       (example PLAN.md structure)
```

**Actions:**
1. Initialize git repo at `~/git/ai-tools`
2. Create folder structure
3. Write characterization tests
4. Run tests - verify current state works

**Success criteria:**
- Git repo initialized
- Characterization tests all PASS
- New feature tests all FAIL (as expected)

---

### Phase 1: Create Workflow Definition

**Goal:** Single source of truth for workflow steps

**Files to create:**
- `ai-tools/sdlc/pr-workflow-definition.md`

**Content:**
```markdown
# PR Workflow Definition

## Workflow Stages

1. Git Safety Check
2. Jira Ticket Search & Selection
3. Branch Creation
4. Understanding Check (Staff Engineer for API/DB)
5. TDD Planning Questions
6. Test Plan Creation
7. Test Plan Approval (/lgtm)
8. Test Implementation
9. Test Approval (/lgtm)
10. Implementation Plan Creation
11. Implementation Approval (/lgtm)
12. Code Implementation
13. Fresh Context Code Review
14. Review Feedback Resolution
15. Final Approval & Merge

## Stage Details
[Full details for each stage with prompts, validations, error handling]
```

**Tests that should pass:**
- Workflow definition is parseable
- All stages documented
- Each stage has clear entry/exit criteria

---

### Phase 2: Git Safety Checks

**Goal:** Ensure clean slate before starting workflow

**Files to modify:**
- `~/.jira-github-slack-helpers.sh` - Add `check-git-safety` function

**New function:**
```bash
check-git-safety() {
    # 1. Check current branch
    # 2. Warn if not on main, switch to main
    # 3. Check for uncommitted changes
    # 4. Give 3 attempts to stash
    # 5. Pull latest from remote
    # 6. Display remote details
}
```

**Tests that should now pass:**
- `test_detect_not_on_main`
- `test_detect_uncommitted_changes`
- `test_pull_latest`
- `test_abort_after_3_stash_attempts`

---

### Phase 3: Enhanced Jira Search & Filtering

**Goal:** Search by keywords, filter by type, prevent Epic/Initiative PRs

**Files to modify:**
- `~/.jira-github-slack-helpers.sh` - Update `jira-search` function
- Add `jira-search-interactive` function

**New functionality:**
```bash
jira-search-interactive() {
    # 1. Ask for keywords
    # 2. Search Jira (full list, no limit)
    # 3. Display with type tags (Story 📘, Epic ⚠️, Bug 🐛, etc.)
    # 4. Flag epics/initiatives as non-selectable
    # 5. Let user pick number or 'q' to search again
    # 6. Validate selection is Story/Task/Bug
}
```

**Tests that should now pass:**
- `test_identify_story`
- `test_identify_epic`
- `test_block_epic_selection`
- `test_allow_story_selection`

---

### Phase 4: Automated Branch Creation

**Goal:** Create branch from Jira ticket with proper naming

**Files to modify:**
- `~/.jira-github-slack-helpers.sh` - Add `create-workflow-branch` function

**New functionality:**
```bash
create-workflow-branch() {
    local jira_ticket="$1"
    local jira_summary="$2"
    
    # 1. Ask: custom description or use default?
    # 2. Slugify description
    # 3. Create branch: JIRA-123-description
    # 4. Checkout new branch
}
```

**Tests that should now pass:**
- `test_branch_format`
- `test_custom_description`
- `test_default_description`
- `test_slugification`

---

### Phase 5: Staff Engineer Check

**Goal:** Prompt for staff engineer discussion on API/DB changes

**Files to modify:**
- `~/.jira-github-slack-helpers.sh` - Add `check-staff-engineer-review` function

**New functionality:**
```bash
check-staff-engineer-review() {
    # 1. Ask: "Does this involve API or DB changes? (y/n)"
    # 2. If yes: "Have you discussed with staff engineers? (y/n)"
    # 3. If no: Soft warning, ask to continue anyway
    # 4. Return true/false to continue workflow
}
```

**Validation:**
- Manual testing (interactive prompts)
- User can continue after warning
- Clear messaging

---

### Phase 6: Approval Gate Mechanism

**Goal:** Wait for `/lgtm` comments before proceeding

**Files to modify:**
- `~/.jira-github-slack-helpers.sh` - Add `wait-for-lgtm` function

**New functionality:**
```bash
wait-for-lgtm() {
    local pr_number="$1"
    
    # 1. Poll PR for comments
    # 2. Check for "/lgtm" in comment body
    # 3. When found, return and continue workflow
    # 4. Option for user to tell Claude "it's approved"
}
```

**Tests that should now pass:**
- `test_detect_lgtm_comment`
- `test_wait_for_approval`
- `test_continue_after_lgtm`

---

### Phase 7: Unified Workflow Orchestration

**Goal:** Single command that runs entire workflow

**Files to create:**
- `~/.jira-github-slack-helpers.sh` - Add `start-workflow` function (replaces old commands)

**New master function:**
```bash
start-workflow() {
    echo "Starting PR Workflow..."
    echo ""
    
    # Stage 1: Git Safety
    check-git-safety || return 1
    
    # Stage 2: Jira Search
    local jira_ticket=$(jira-search-interactive)
    
    # Stage 3: Branch Creation
    create-workflow-branch "$jira_ticket"
    
    # Stage 4: Understanding Check
    check-staff-engineer-review
    
    # Stage 5-15: Continue with planning, implementation, review...
    # (Prompts to Claude for each stage)
}
```

**Success criteria:**
- One command starts entire workflow
- Each stage validates before proceeding
- Clear error messages at each step
- User can abort at any point

---

### Phase 8: Update CLAUDE.md

**Goal:** Update Claude Code guidance for new workflow

**Files to modify:**
- `~/CLAUDE.md`

**Changes:**
- Remove references to old `start-pr-workflow` signature
- Document new `start-workflow` command
- Reference workflow definition at `~/git/ai-tools/sdlc/pr-workflow-definition.md`
- Update expected user questions

---

### Phase 9: Update Documentation

**Goal:** Update all user-facing documentation

**Files to modify:**
- `~/.pr-workflow-guide.md` - Update with new workflow
- `~/.pr-workflow-prompts.md` - Update prompts for new flow
- Create new: `~/git/ai-tools/sdlc/README.md` - Main documentation

**New documentation structure:**
```
ai-tools/sdlc/README.md
├── Overview
├── Quick Start
├── Workflow Stages (reference to pr-workflow-definition.md)
├── Configuration
├── Troubleshooting
└── Contributing (how to update workflow)
```

---

### Phase 10: Cleanup & Deprecation

**Goal:** Remove old/duplicate code

**Files to modify:**
- `~/.jira-github-slack-helpers.sh`

**Actions:**
- Mark `start-pr-workflow` as deprecated (or remove)
- Mark `start-from-jira` as deprecated (or remove)  
- Keep `jira-search` for standalone use
- Update help text to show new `start-workflow` command

**Files to archive:**
- Keep old prompts for reference
- Update `what-can-i-ask` with new workflow

---

## Test Coverage Goals

### Before Implementation
- Current coverage: Manual testing only
- No automated validation
- Workflow scattered across multiple files

### After Implementation
- Validation scripts for all new features
- Characterization tests for existing features
- All critical paths validated
- Single workflow definition as source of truth

### Test Execution Order

1. ✅ Run characterization tests (ensure current state works)
2. ❌ Run new feature tests (should fail)
3. ✅ Implement Phase 1 (workflow definition - tests pass)
4. ✅ Implement Phase 2 (git safety - tests pass)
5. ✅ Implement Phase 3 (jira filtering - tests pass)
6. ✅ Implement Phase 4 (branch creation - tests pass)
7. ✅ Implement Phase 5 (staff engineer - manual validation)
8. ✅ Implement Phase 6 (approval gates - tests pass)
9. ✅ Implement Phase 7 (orchestration - end-to-end validation)
10. ✅ Implement Phase 8-10 (documentation, cleanup)

---

## Migration/Rollout Plan

### Phase 1: Create Parallel Implementation
- New `start-workflow` command alongside old commands
- Both work simultaneously
- Users can opt into new workflow

### Phase 2: Documentation & Communication
- Update `what-can-i-ask` to recommend new workflow
- Add deprecation notices to old commands
- Update CLAUDE.md

### Phase 3: Transition Period
- Monitor usage
- Collect feedback
- Fix issues

### Phase 4: Cleanup
- Remove old commands
- Archive old documentation
- Finalize new workflow as default

---

## Risks and Mitigations

### Risk: Breaking existing workflows mid-task
**Mitigation:**
- Keep old commands working during transition
- Only deprecate after new workflow is stable
- Clear migration guide

### Risk: Git operations fail (conflicts, network issues)
**Mitigation:**
- Comprehensive error handling in git safety checks
- Clear error messages
- Abort workflow gracefully on failures

### Risk: Jira API rate limiting or failures
**Mitigation:**
- Cache search results temporarily
- Graceful degradation if Jira unavailable
- Allow manual ticket entry as fallback

### Risk: Approval gate polling creates tight loop
**Mitigation:**
- Add sleep/delay between poll attempts
- Option for user to manually trigger continuation
- Clear status messages during wait

### Risk: Workflow definition diverges from implementation
**Mitigation:**
- Workflow definition is version controlled
- Regular validation that code matches definition
- Tests reference workflow definition

---

## Dependencies

- Jira CLI (already installed)
- GitHub CLI (already installed)
- Git (already installed)
- Existing `~/.pr-workflow-config` (already exists)
- Shell helper functions (already exists)

---

## Estimated Complexity

**LARGE** (3-5 days of work)

**Reasoning:**
- Multiple new features (git safety, filtering, approval gates)
- Significant refactoring of existing code
- New validation/test scripts
- Comprehensive documentation updates
- Migration/rollout planning
- Version-controlled workflow definition

**Breakdown:**
- Phase 0-1: Setup & Definition (4 hours)
- Phase 2: Git Safety (3 hours)
- Phase 3: Jira Filtering (3 hours)
- Phase 4: Branch Creation (2 hours)
- Phase 5: Staff Check (1 hour)
- Phase 6: Approval Gates (4 hours)
- Phase 7: Orchestration (4 hours)
- Phase 8-10: Documentation & Cleanup (3 hours)

**Total: ~24 hours** (spread over multiple sessions)

---

## Review Checklist

- [ ] All characterization tests pass (current behavior preserved)
- [ ] All new feature tests pass
- [ ] Workflow definition is complete and version controlled
- [ ] Git safety checks prevent dirty state issues
- [ ] Jira filtering blocks Epic/Initiative PRs
- [ ] Branch naming follows convention
- [ ] Approval gates work reliably
- [ ] Documentation is comprehensive
- [ ] Old commands deprecated gracefully
- [ ] CLAUDE.md updated for new workflow
- [ ] End-to-end validation passes

---

## Next Steps

1. **Review this plan** - Get approval before implementation
2. **Create ai-tools repo** - Initialize git repository
3. **Write characterization tests** - Document current behavior
4. **Implement phase by phase** - Following test-first approach
5. **Validate each phase** - Run tests after each implementation
6. **Update documentation** - Keep docs in sync with code
7. **Test end-to-end** - Full workflow validation
8. **Migrate users** - Transition to new workflow

---

**Ready to proceed?** Comment `/lgtm` to approve this plan and begin implementation.
