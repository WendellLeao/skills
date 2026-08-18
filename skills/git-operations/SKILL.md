---
name: git-operations
description: Git commit and push conventions for this user's projects. Covers conventional-commit-style message tags (feat:, fix:, refactor:, docs:, test:, chore:, style:, perf:, etc.), title-only commit messages with no body or trailers, and the "push" shorthand that scopes staging to only the files/hunks from the current conversation before committing and pushing. Use whenever creating a git commit or pushing changes in any project.
---

# Git Operations

Conventions for how commits and pushes are made across projects, so the same rules don't need to be re-explained on every commit.

## Commit message format

- Always prefix the commit title with a conventional-commit-style tag matching the nature of the change. Pick the tag that accurately reflects the change:
  - `feat:` a wholly new feature or capability
  - `fix:` a bug fix
  - `refactor:` restructuring existing code without changing behavior
  - `docs:` documentation-only changes
  - `test:` test-only changes
  - `chore:` maintenance work with no source-level behavior change (deps, config, tooling)
  - `style:` formatting-only changes with no logic change
  - `perf:` a performance improvement
- Commit messages are always written in English, regardless of the language used in the conversation.
- Commit with the title only: no description, no body, no trailers (including `Co-Authored-By`). Just:
  ```
  git commit -m "type: title"
  ```

## "Push" shorthand

When the user says "push" (or an equivalent phrasing, "I want to push", "push commit", "git push", "da um push", etc.) during or right after a round of changes, treat it as shorthand for this full sequence, not a literal `git push` alone:

1. Identify only the files/hunks actually related to the change just discussed/made in the current conversation.
2. Stage those specific files (if not already staged). Never blanket-stage with `git add -A` / `git add .`, and never stage unrelated changes sitting in the working tree.
3. Commit the staged changes (see commit message format above).
4. Push the commit.

If it's ambiguous which dirty files belong to the current work, ask before staging them rather than guessing.
