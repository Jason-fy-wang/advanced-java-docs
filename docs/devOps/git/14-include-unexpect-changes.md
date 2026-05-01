---
tags:
  - git
  - checkout
  - switch
---
# Git: New Branch Contains Unexpected Changes

## Problem
When creating a new branch via a script, the new branch unexpectedly includes changes that were not intended to be there.

## Root Cause
The script was using the following commands to create branches:

```shell
# Using checkout
git checkout -b newBranch

# Using switch
git switch -c newBranch
```

By default, `git checkout -b` and `git switch -c` create a new branch based on the **current branch (`HEAD`)**. If the script is executed while a different feature branch is active, the new branch will inherit all changes from that branch instead of starting from `master`.

## Solution
To ensure the new branch is created from a specific base (like `master`), explicitly provide the starting point in the command:

```shell
# Specify master as the base
git checkout -b newBranch master

# Or using switch
git switch -c newBranch master
```

## Supported Base Points
When creating a branch, the `<start-point>` can be any valid Git "committish" object:

### 1. Branches (Local or Remote)
The most common base point.
```shell
git checkout -b feature-branch master
git checkout -b hotfix-branch origin/main
```

### 2. Tags
Useful for branching off a specific release version.
```shell
git checkout -b fix-v1.0 v1.0.2
```

### 3. Commit IDs (SHA)
Branch off a specific point in history using its unique hash.
```shell
git checkout -b archive-branch a1b2c3d4
```

### 4. Relative References
Use expressions like `HEAD~` to go back a specific number of commits.
```shell
# Branch from two commits ago
git checkout -b retry-branch HEAD~2
```

### Recommended Workflow
It is often safer to branch from the remote tracking branch to ensure you are starting from the latest upstream state:

```shell
git fetch origin
git checkout -b newBranch origin/master
```


