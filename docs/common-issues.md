# Common issues

- [Troubleshooting](#troubleshooting)
  - [Create using an existing branch as the PR branch](#create-using-an-existing-branch-as-the-pr-branch)
  - ["This branch is out-of-date with the base branch" warning](#this-branch-is-out-of-date-with-the-base-branch-warning)
- [Frequently requested features](#use-case-create-a-pull-request-to-update-x-on-release)
  - [Disable force updates to existing PR branches](#disable-force-updates-to-existing-pr-branches)
  - [Add a no-verify option to bypass git hooks](#add-a-no-verify-option-to-bypass-git-hooks)

## Troubleshooting

### Create using an existing branch as the PR branch

A common point of confusion is to try and use an existing branch containing changes to raise in a PR as the `branch` input. This will not work because the action is primarily designed to be used in workflows where the PR branch does not exist yet. The action creates and manages the PR branch itself.

If you have an existing branch that you just want to create a PR for, then I recommend using the official [GitHub CLI](https://cli.github.com/manual/gh_pr_create) in a workflow step.

Alternatively, if you are trying to keep a branch up to date with another branch, then you can follow [this example](https://github.com/peter-evans/create-pull-request/blob/main/docs/examples.md#keep-a-branch-up-to-date-with-another).

### "This branch is out-of-date with the base branch" warning

When creating a pull request to keep one branch synchronized with another (e.g., promoting changes from `dev` to `prod`), you may see a warning in GitHub that says "This branch is out-of-date with the base branch" even though the branches should be in sync.

#### Why this happens

This occurs when the workflow modifies the base branch locally but doesn't push those changes to the remote repository before creating the pull request branch. Here's what happens:

1. The workflow checks out the base branch (e.g., `prod`)
2. The workflow resets it to match another branch (e.g., `dev`)
3. The action creates a PR branch based on this locally modified base branch
4. However, the **remote** base branch is still at its old state
5. GitHub compares the PR branch against the **remote** base branch, not the local one
6. This causes GitHub to show the "out-of-date" warning

#### Solution

Push the updated base branch to the remote repository before creating the pull request. This ensures the remote base branch is synchronized with your local changes.

**Incorrect workflow (causes the warning):**
```yml
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

**Correct workflow (prevents the warning):**
```yml
- uses: actions/checkout@v4
  with:
    ref: prod
- name: Reset promotion branch
  run: |
    git fetch origin dev:dev
    git reset --hard dev
    git push origin prod  # Push the updated base branch to remote
- name: Create Pull Request
  uses: peter-evans/create-pull-request@v7
  with:
    branch: prod-promotion
```

By pushing the base branch before creating the pull request, the remote base branch will be up-to-date, and GitHub will correctly recognize that the PR branch is based on the latest version of the base branch.

#### Additional considerations

- **Permissions**: Ensure your workflow has write permissions to push to the base branch. You may need to configure workflow permissions in your repository settings or use a Personal Access Token (PAT) with appropriate permissions.
- **Protected branches**: If your base branch is protected, you may need to adjust branch protection rules or use a token with bypass permissions.
- **Force push**: If you need to force push the base branch (e.g., when resetting to a different commit), use `git push --force origin prod` instead.

See the [Keep a branch up-to-date with another](examples.md#keep-a-branch-up-to-date-with-another) example for a complete working workflow.

## Frequently requested features

### Disable force updates to existing PR branches

This behaviour is fundamental to how the action works and is a conscious design decision. The "rule" that I based this design on is that when a workflow executes the action to create or update a PR, the result of those two possible actions should never be different. The easiest way to maintain that consistency is to rebase the PR branch and force push it.

If you want to avoid this behaviour there are some things that might work depending on your use case:
- Check if the pull request branch exists in a separate step before the action runs and act accordingly.
- Use the [alternative strategy](https://github.com/peter-evans/create-pull-request#alternative-strategy---always-create-a-new-pull-request-branch) of always creating a new PR that won't be updated by the action.
- [Create your own commits](https://github.com/peter-evans/create-pull-request#create-your-own-commits) each time the action is created/updated.

### Add a no-verify option to bypass git hooks

Presently, there is no plan to add this feature to the action.
The reason is that I'm trying very hard to keep the interface for this action to a minimum to prevent it becoming bloated and complicated.

Git hooks must be installed after a repository is checked out in order for them to work.
So the straightforward solution is to just not install them during the workflow where this action is used.

- If hooks are automatically enabled by a framework, use an option provided by the framework to disable them. For example, for Husky users, they can be disabled with the `--ignore-scripts` flag, or by setting the `HUSKY` environment variable when the action runs.
  ```yml
  uses: peter-evans/create-pull-request@v7
  env:
    HUSKY: '0'
  ```
- If hooks are installed in a script, then add a condition checking if the `CI` environment variable exists.
   ```sh
   #!/bin/sh

   [ -n "$CI" ] && exit 0
   ```
- If preventing the hooks installing is problematic, just delete them in a workflow step before the action runs.
   ```yml
   - run: rm .git/hooks -rf
   ```
