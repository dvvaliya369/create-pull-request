# Fix for GitHub Issue #4150: "This branch is out-of-date with the base branch"

## Quick Fix for Your Workflow

Your current workflow has an issue that causes the "out-of-date" warning. Here's the corrected version:

### ❌ Your Current Workflow (Problematic)

```yaml
name: Auto PR from dev to prod

on:
  push:
    branches:
      - dev

jobs:
  check-and-create-pr:
    runs-on: blacksmith-2vcpu-ubuntu-2204
    steps:
      - uses: actions/checkout@v4
        with:
          ref: prod  # ❌ Checking out target branch
      - name: Reset promotion branch
        run: |
          git fetch origin dev:dev
          git reset --hard dev
      - name: Create Pull Request
        uses: peter-evans/create-pull-request@v6
        with:
          branch: prod-promotion
          delete-branch: true
          title: "Auto PR: Merge changes from dev to prod"
          body: "This pull request has been automatically created to merge changes from the dev branch into the prod branch."
          # ❌ Missing 'base' input
```

### ✅ Corrected Workflow (Recommended)

```yaml
name: Auto PR from dev to prod

on:
  push:
    branches:
      - dev

jobs:
  check-and-create-pr:
    runs-on: blacksmith-2vcpu-ubuntu-2204
    steps:
      - uses: actions/checkout@v4
        with:
          ref: dev  # ✅ Check out SOURCE branch
          
      - name: Create Pull Request
        uses: peter-evans/create-pull-request@v7  # ✅ Updated to v7
        with:
          branch: prod-promotion
          base: prod  # ✅ Specify TARGET branch as base
          delete-branch: true
          title: "Auto PR: Merge changes from dev to prod"
          body: "This pull request has been automatically created to merge changes from the dev branch into the prod branch."
```

## What Changed?

1. **`ref: dev`** - Now checking out the source branch (dev) instead of target (prod)
2. **`base: prod`** - Added explicit base parameter to specify the target branch
3. **Removed reset step** - No longer needed with the correct approach
4. **Updated to v7** - Using the latest version of the action

## Why This Fixes the Issue

The original workflow was:
- Checking out `prod` (target)
- Resetting it to `dev` (source)
- Creating a PR branch from this state
- GitHub compared the PR branch to the remote `prod` branch
- Since remote `prod` hadn't changed, it appeared "out-of-date"

The corrected workflow:
- Checks out `dev` (source) directly
- Creates a PR branch based on `dev`
- Specifies `prod` as the base for the PR
- GitHub correctly sees the PR branch as ahead of `prod`
- No "out-of-date" warning appears

## Alternative Approach (If You Need the Reset)

If you have specific reasons to use the reset approach, you can keep it but **must** add the `base` parameter:

```yaml
name: Auto PR from dev to prod

on:
  push:
    branches:
      - dev

jobs:
  check-and-create-pr:
    runs-on: blacksmith-2vcpu-ubuntu-2204
    steps:
      - uses: actions/checkout@v4
        with:
          ref: prod
          fetch-depth: 0  # Fetch all history
          
      - name: Reset promotion branch
        run: |
          git fetch origin dev:dev
          git reset --hard dev
          
      - name: Create Pull Request
        uses: peter-evans/create-pull-request@v7
        with:
          branch: prod-promotion
          base: prod  # ✅ CRITICAL: Must specify base
          delete-branch: true
          title: "Auto PR: Merge changes from dev to prod"
          body: "This pull request has been automatically created to merge changes from the dev branch into the prod branch."
```

## Expected Results After Fix

✅ No "out-of-date with the base branch" warning  
✅ PR shows correct diff between dev and prod  
✅ All commits from dev are included  
✅ PR can be merged without issues  
✅ No audit log concerns  

## Additional Recommendations

1. **Branch Protection**: Set up branch protection rules on `prod` to require reviews
2. **Auto-delete branches**: Enable "Automatically delete head branches" in repository settings
3. **Draft PRs**: Consider adding `draft: true` if you want PRs created as drafts initially

## Need More Information?

- See `ISSUE_4150_SOLUTION.md` for detailed explanation
- Check `docs/examples.md` for more examples
- Review `docs/common-issues.md` for troubleshooting tips
