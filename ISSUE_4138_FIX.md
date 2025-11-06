# Fix for Issue #4138: Reviews may only be requested from collaborators

## Problem
When using the `reviewers` or `team-reviewers` inputs with users who are not collaborators of the repository, the GitHub API returns an error:
```
Reviews may only be requested from collaborators. One or more of the users or teams you specified is not a collaborator of the $org/$repo repository.
```

Previously, this error would cause the entire action to fail, preventing the pull request from being created or updated.

## Solution
The fix implements graceful error handling for this specific scenario:

1. **Added Error Constant**: Added `ERROR_PR_REVIEWER_NOT_COLLABORATOR` constant to identify this specific error
2. **Enhanced Error Handling**: Modified the error handling in `createOrUpdatePullRequest` method to:
   - Detect when the error is due to non-collaborator reviewers
   - Log warnings instead of failing the action
   - Provide helpful guidance to users
   - Allow the PR creation/update to continue successfully

## Changes Made

### File: `src/github-helper.ts`

#### 1. Added new error constant (line 12-13):
```typescript
const ERROR_PR_REVIEWER_NOT_COLLABORATOR =
  'Reviews may only be requested from collaborators'
```

#### 2. Updated error handling (lines 209-231):
```typescript
} catch (e) {
  const errorMessage = utils.getErrorMessage(e)
  if (errorMessage.includes(ERROR_PR_REVIEW_TOKEN_SCOPE)) {
    core.error(
      `Unable to request reviewers. If requesting team reviewers a 'repo' scoped PAT is required.`
    )
    throw e
  } else if (errorMessage.includes(ERROR_PR_REVIEWER_NOT_COLLABORATOR)) {
    core.warning(
      `Unable to request reviewers. One or more of the users or teams you specified is not a collaborator of this repository.`
    )
    core.warning(
      `Reviews may only be requested from collaborators. Please ensure the specified reviewers have been added as collaborators to the repository, or consider using team reviewers with users added to the team.`
    )
    core.warning(
      `See: https://docs.github.com/rest/pulls/review-requests#request-reviewers-for-a-pull-request`
    )
    // Don't throw - allow PR creation/update to continue
  } else {
    throw e
  }
}
```

## Behavior After Fix

### Before:
- Action fails completely when a non-collaborator is specified as a reviewer
- Pull request is not created/updated
- Workflow fails

### After:
- Action logs warnings about the non-collaborator reviewer issue
- Pull request is still created/updated successfully
- Workflow continues and succeeds
- Users receive helpful guidance on how to resolve the issue

## User Guidance Provided

The fix provides three warning messages to help users understand and resolve the issue:

1. **Error Description**: Explains that one or more specified reviewers is not a collaborator
2. **Resolution Steps**: Suggests adding reviewers as collaborators or using team reviewers
3. **Documentation Link**: Provides link to GitHub API documentation for reference

## Testing

All existing tests pass:
- ✅ Unit tests: 12 passed
- ✅ TypeScript compilation: Success
- ✅ Linting: No issues
- ✅ Code formatting: Compliant with Prettier

## Workarounds for Users

Users experiencing this issue have several options:

1. **Add users as collaborators**: Ensure all specified reviewers have been added as collaborators to the repository
2. **Use team reviewers**: Create a GitHub team, add users to the team, and use `team-reviewers` input instead
3. **Remove non-collaborators**: Remove users who are not collaborators from the `reviewers` input

## Related Issues

- Issue #4138: Reviews may only be requested from collaborators
- Issue #1638: Similar issue in peter-evans/create-pull-request repository
