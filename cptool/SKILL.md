---
name: cptool
description: Automatically cherry-picks a specified commit (or the latest commit from the current branch) to multiple target branches, pushes them to the remote, and creates Pull Requests.
---

## When to use
- When a bug fix or feature needs to be backported to multiple release branches.
- When the user asks to "cherry-pick this to branch A, B, and C".
- When a change in the current branch needs to be applied to other environments or versions.

## Instructions

### 1. Preparation
- Ensure all remote branches are up to date: `git fetch --all`.
- Identify the correct default remote for pushing (usually `origin`) and the target remote for the PR (usually `upstream`).

### 2. Identify Source
- Identify the source commit hash. If not specified, use the latest commit on the current branch (`git log -n 1 --format=%H`).

### 3. Select Target Branches
- List available release branches from the upstream remote:
  `git branch -r | grep upstream/release/`
- **Action**: Present this list to the user and explicitly ask them to select or confirm which branches they want to cherry-pick the commit to.
- **Wait** for the user's response before proceeding.

### 4. Iterative Cherry-pick and PR
For each target branch selected by the user:
1. **Create Working Branch**: Create a new local branch based on the target branch.
   - Format: `git checkout -b cp-<commit_id_short>-to-<target_branch_name> <full_target_branch_path>`
2. **Cherry-pick**: Execute `git cherry-pick <source_commit_hash>`.
   - If conflicts occur, stop and ask the user for guidance.
3. **Push**: Push the new branch to the default remote (usually `origin`).
   - `git push origin <local_branch_name>`
4. **Create PR**: Use the `create_pull_request` tool to create a PR to the target repository.
   - **owner**: The owner of the *target* repository (e.g., the owner of `upstream`).
   - **repo**: The name of the target repository.
   - **base**: The target release branch (e.g., `release/v1.8.0`).
   - **head**: The name of the branch you just pushed. If using a fork, prefix with the username (e.g., `username:branch_name`).
   - **Title**: `<original_commit_title> (<target_version>)`

### 5. Finalize
- Provide a summary table or list of the created Pull Request URLs to the user.
- Switch back to the original branch if necessary.

## Example Triggers
- "Cherry-pick the last commit to branches release/v1.8.0 and release/v1.7.0"
- "Backport this fix to our chorus release branches"
- "Create PRs for this change to all active release versions"
