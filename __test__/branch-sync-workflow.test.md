# Test Case: Branch Synchronization Workflow

## Issue
GitHub Issue #4150: "This branch is out-of-date with the base branch" warning

## Test Scenario
Verify that the workflow for keeping one branch synchronized with another works correctly without showing the "out-of-date" warning.

## Setup
1. Repository with two branches: `dev` (source) and `prod` (target)
2. Workflow that triggers on push to `dev` branch
3. Workflow creates a PR to merge changes from `dev` to `prod`

## Test Steps

### Test 1: Incorrect workflow (should show warning)
```yaml
- uses: actions/checkout@v4
  with:
    ref: prod
- name: Reset promotion branch
  run: |
    git fetch origin dev:dev
    git reset --hard dev
- name: Create Pull Request
  uses: peter-evans/create-pull-request@v7
  with:
    branch: prod-promotion
```

**Expected Result**: GitHub shows "This branch is out-of-date with the base branch" warning

**Reason**: The remote `prod` branch is not updated before creating the PR branch

### Test 2: Correct workflow (should NOT show warning)
```yaml
- uses: actions/checkout@v4
  with:
    ref: prod
- name: Reset promotion branch
  run: |
    git fetch origin dev:dev
    git reset --hard dev
    git push origin prod  # Push updated base branch
- name: Create Pull Request
  uses: peter-evans/create-pull-request@v7
  with:
    branch: prod-promotion
```

**Expected Result**: No "out-of-date" warning; PR shows correct diff

**Reason**: The remote `prod` branch is synchronized before creating the PR branch

## Verification Steps

1. **Check remote branch state**:
   ```bash
   git fetch origin
   git log origin/prod --oneline -5
   git log origin/dev --oneline -5
   ```
   After the correct workflow, `origin/prod` should match `origin/dev`

2. **Check PR branch state**:
   ```bash
   git log origin/prod-promotion --oneline -5
   ```
   The PR branch should be based on the updated `prod` branch

3. **Verify in GitHub UI**:
   - Open the created PR
   - Check that no "out-of-date" warning appears
   - Verify the PR shows the correct diff between `prod-promotion` and `prod`

## Manual Testing Instructions

To manually test this fix:

1. Create a test repository with `dev` and `prod` branches
2. Make a commit to `dev` branch
3. Run the incorrect workflow and observe the warning
4. Close the PR and delete the `prod-promotion` branch
5. Run the correct workflow and verify no warning appears

## Automated Testing

The integration tests in `create-or-update-branch.int.test.ts` cover the core functionality of branch creation and updates. This specific issue is a workflow pattern problem rather than an action bug, so the fix is primarily in documentation and examples.

## Related Files
- `docs/examples.md` - Updated example workflow
- `docs/common-issues.md` - New troubleshooting section
