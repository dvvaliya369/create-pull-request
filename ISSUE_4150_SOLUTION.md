# Solution for Issue #4150: "This branch is out-of-date with the base branch" Warning

## Problem Description

When using the create-pull-request action to automatically create PRs from a `dev` branch to a `prod` branch, GitHub displays a warning: **"This branch is out-of-date with the base branch"**. This warning appears even though the PR should contain all the latest changes from `dev`.

## Root Cause

The issue occurs because the workflow is:
1. Checking out the **target branch** (`prod`)
2. Resetting it to match the **source branch** (`dev`)
3. Creating a PR branch from this state

This approach causes the PR branch to be based on the local state of `prod` (which was reset to `dev`), but GitHub compares it against the **remote** `prod` branch, which hasn't changed. This makes it appear as if the PR branch is behind the base branch.

## Solution

The correct approach is to **check out the source branch** and specify the target branch as the `base` input. Here's the corrected workflow:

### ✅ Correct Workflow

```yaml
name: Auto PR from dev to prod

on:
  push:
    branches:
      - dev

jobs:
  check-and-create-pr:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          ref: dev  # Check out the SOURCE branch
          
      - name: Create Pull Request
        uses: peter-evans/create-pull-request@v7
        with:
          branch: prod-promotion
          base: prod  # Specify the TARGET branch as base
          delete-branch: true
          title: "Auto PR: Merge changes from dev to prod"
          body: "This pull request has been automatically created to merge changes from the dev branch into the prod branch."
```

### Key Changes

1. **`ref: dev`** - Check out the source branch (`dev`) instead of the target branch (`prod`)
2. **`base: prod`** - Explicitly specify the target branch (`prod`) as the base for the PR
3. **Remove the reset step** - No longer needed since we're checking out the correct branch

## Why This Works

When you check out `dev` and specify `base: prod`, the action:
1. Creates a PR branch (`prod-promotion`) based on the current `dev` branch
2. Compares this branch against the `prod` branch
3. Creates a PR to merge `prod-promotion` → `prod`
4. The PR branch is properly ahead of `prod` (not behind or out-of-date)

## Alternative Approach (If You Need Custom Logic)

If you need to perform custom operations before creating the PR, you can still use a reset approach, but you must ensure the base is correctly specified:

```yaml
name: Auto PR from dev to prod

on:
  push:
    branches:
      - dev

jobs:
  check-and-create-pr:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          ref: prod
          fetch-depth: 0  # Fetch all history
          
      - name: Reset promotion branch to dev
        run: |
          git fetch origin dev:dev
          git reset --hard dev
          
      - name: Create Pull Request
        uses: peter-evans/create-pull-request@v7
        with:
          branch: prod-promotion
          base: prod  # IMPORTANT: Must specify base as prod
          delete-branch: true
          title: "Auto PR: Merge changes from dev to prod"
          body: "This pull request has been automatically created to merge changes from the dev branch into the prod branch."
```

**Note:** Even with this approach, you **must** specify `base: prod` to ensure the PR is created correctly.

## Verification

After implementing the fix, you should see:
- ✅ No "out-of-date" warning on the PR
- ✅ The PR shows the correct diff between `dev` and `prod`
- ✅ All commits from `dev` are included in the PR
- ✅ The PR can be merged without issues

## Additional Recommendations

1. **Use branch protection rules** on `prod` to require reviews before merging
2. **Enable "Automatically delete head branches"** in repository settings to clean up merged PR branches
3. **Consider using the `draft` input** if you want PRs to be created as drafts initially:
   ```yaml
   draft: true
   ```

## Related Documentation

- [Keep a branch up-to-date with another](docs/examples.md#keep-a-branch-up-to-date-with-another)
- [Providing a consistent base](docs/concepts-guidelines.md#providing-a-consistent-base)
- [Events and checkout](docs/concepts-guidelines.md#events-and-checkout)
