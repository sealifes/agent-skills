---
name: cptool
description: Automatically cherry-picks a specified commit (or the latest commit from the current branch) to multiple target branches, pushes them to the remote, and creates Pull Requests. Also includes features to discover and clean up merged remote branches using GitHub.
---

## When to use
- When a bug fix or feature needs to be backported to multiple release branches.
- When the user asks to "cherry-pick this to branch A, B, and C".
- When a change in the current branch needs to be applied to other environments or versions.
- When the user asks to "clean up merged branches" or "delete old branches".

## Instructions

### Mode 1: Cherry-Pick & Backport

#### 1. Preparation
- Ensure all remote branches are up to date: `git fetch --all`.
- Identify the correct default remote for pushing (usually `origin`) and the target remote for the PR (usually `upstream`).

#### 2. Identify Source
- Identify the source commit hash. If not specified, use the latest commit on the current branch (`git log -n 1 --format=%H`).

#### 3. Select Target Branches
- List available release branches from the upstream remote:
  `git branch -r | grep upstream/release/`
- Include main branch:
  `upstream/master` or `upstream/main`
- **Action**: Present this list to the user and explicitly ask them to select or confirm which branches they want to cherry-pick the commit to.
- **Wait** for the user's response before proceeding.

#### 4. Iterative Cherry-pick and PR
For each target branch selected by the user:
1. **Create Working Branch**: Create a new local branch based on the target branch.
   - Format: `git checkout -b cp-<commit_titile_prefix>-to-<target_branch_name> <full_target_branch_path>`
     (e.g. `git checkout -b cp-G1-3235-to-chorus-v1.2.0 upstream/release/chorus-v1.2.0`)
2. **Cherry-pick**: Execute `git cherry-pick <source_commit_hash>`.
   - If conflicts occur, stop and ask the user for guidance.
3. **Push**: Push the new branch to the target remote (the owner of `upstream`).
   - `git push upstream <local_branch_name>`
4. **Create PR**: Use the `create_pull_request` tool to create a PR to the target repository.
   - **owner**: The owner of the *target* repository (e.g., the owner of `upstream`).
   - **repo**: The name of the target repository.
   - **base**: The target release branch (e.g., `release/v1.8.0`).
   - **draft**: true
   - **Title**: `<original_commit_title> (CP to <target_version_fullname>)`

#### 5. Finalize
- Provide a summary table or list of the created Pull Request URLs to the user.
- Switch back to the original branch if necessary.

---

### Mode 2: Cleanup Merged Branches

#### 1. Identify Parameters
- **Remote**: Identify which remote to clean (e.g., `upstream` or `origin`). Ask the user if not specified.
- **Author**: Identify the GitHub username to filter by.
  - If not specified, use `get_me` to find the current authenticated user.
- **Repo**: Identify the target repository (owner/name).

#### 2. Discover Merged Branches (via GitHub MCP)
- Use `search_pull_requests` to find PRs that have been merged.
  - Query: `repo:<owner/repo> is:merged author:<username> state:closed`
- Extract the **head branch names** from the search results.

#### 3. Filter & Verify
- Check which of these branches still exist on the target remote.
  - Use `git ls-remote --heads <remote> <branch_name>` for precise verification.
- **Action**: **List all existing remote branches** that correspond to merged PRs clearly for the user.
- **Permission Check**: **Explicitly ask the user for permission** to delete these branches.
- **Wait** for the user's response before proceeding to execution.

#### 4. Execution
- **Only** for branches explicitly confirmed by the user:
  - Execute: `git push <remote> --delete <branch_name>`
- Provide a final summary of deleted branches.

## Example Triggers
- "Cherry-pick the last commit to branches release/v1.8.0 and release/v1.7.0"
- "Backport this fix to our chorus release branches"
- "Create PRs for this change to all active release versions"
- "Clean up my merged branches on upstream"
- "Delete all old branches I've merged"
