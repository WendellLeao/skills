# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) across all projects.

---

## Unity & Engine Development Guidelines

Behavioral guidelines to reduce common LLM coding mistakes, biasing toward caution and system integrity over speed: think before coding and surface assumptions/tradeoffs instead of hiding confusion, default to garbage-free and `[SerializeField]`-wired code with no speculative abstractions, make only surgical changes that match existing style, work toward verifiable goals with a stated plan for multi-step tasks, respect the MonoBehaviour lifecycle and default to Unity 6 stable APIs, never hand-edit serialized asset files (`.prefab`/`.meta`/`.unity`/`.asset`) except through Unity itself or a Unity MCP tool, always pair an event subscription with a matching unsubscription, and blend changes into existing code without leaving a trace (no unprompted comments, no drive-by refactors).

Invoke the `unity-engine-guidelines` skill for the full checklist and rationale whenever writing, editing, or reviewing Unity/C# code in any project.

---

## Preferred Architecture for New Projects

For brand-new projects with no established architecture yet, default to a SOLID/clean code baseline: domain-organized `_Project/Source/MyGame.<Module>/<Subdomain>/` folders and namespaces, sibling asset folders per kind (`Scenes/`, `Prefabs/`, `Materials/`, `UI/`, `Inputs/`, `Settings/`), scene hierarchies organized as empty GameObjects grouping objects by domain, a dumb/smart component split, a root orchestrator that explicitly propagates `Initialize`/`Tick`/`FixedTick`/`LateTick`/`Shutdown` steps down a manager hierarchy to guarantee deterministic init/update order instead of relying on Unity's Script Execution Order setting, interfaces at module boundaries, cross-cutting systems exposed as Service Locator services, and a logic/View split with event-driven state. This orchestration is reserved for components that are genuinely part of a larger system (especially meta-game systems like shop/screens/customization); standalone smart components keep self-initializing through normal Unity callbacks. It is a paved path, not a straitjacket, don't force it onto simple, pointed work.

Invoke the `unity-clean-architecture` skill for the full details and naming conventions whenever either is true:
- The user is starting a brand-new project with no established architecture yet.
- The working directory is one of the user's personal projects, which live under `~/Documents/_Projects/Unity/Personal/` (e.g. `~/Documents/_Projects/Unity/Personal/<project-name>/`). These are the user's sandbox for iterating on and refining this architecture, so treat this guide as relevant context there even if the project already has some code, unless that project has explicitly diverged onto its own established architecture (see the `unity-engine-guidelines` skill's "Blend Into Existing Code" rule and the existing-projects rule below).

For existing projects outside that personal sandbox, always follow the architecture already established in that codebase (see the `unity-engine-guidelines` skill's "Blend Into Existing Code" rule). Only introduce this pattern there when scaffolding something genuinely new with no existing precedent to match.

---

## Code Style Rules

C# formatting and naming conventions used across projects: file layout order (nested types, events, serialized fields, fields, constructors, properties, methods), method ordering (Unity callbacks first, then `TryGet*`/`Get*`/`Set*` at the bottom by access), always-braced control flow, `On*` event/handler naming (`Handle*` only on a same-class naming conflict), and personal formatting preferences (const field placement, `PascalCase` static readonly naming, single-space operators, minimal-surface interfaces at system boundaries, `[SerializeField]` over `GetComponent*`).

Invoke the `unity-csharp-code-style` skill for the full rules whenever writing or reviewing C# code in a Unity project.

---

## Writing Style (all generated prose, any project)

- **CRITICAL, repeatedly violated, treat as a hard blocker: never use the em dash or double-dash as sentence punctuation**, in absolutely any text generated for the user. This is not limited to chat replies. It covers code comments, variable/method names' doc comments, commit messages, PR descriptions, docstrings, summaries sent to other people, memory files, everything without exception. Example of what NOT to write: "the framework — screen systems, menus, and modals — kept it modular." Rewrite with a comma, colon, period, semicolon, or by restructuring the sentence into two sentences instead.
- Reason this matters: it is a strong AI writing tic, and code comments/text with em dashes read as obviously AI-generated, which actively hurts code that other people read.
- Before finishing any turn that generated prose or code comments, scan your own output for that punctuation used as a sentence break and rewrite it if found.
- This does **not** apply to hyphens in compound words or ranges (e.g. `client-side`, `peer-to-peer`, `80-100 FPS`), those are fine and expected. It also does not apply to `--` used as an actual CLI flag prefix (e.g. `git commit --amend`).

---

## Documentation Language

Any `.md` file or other document intended to be read by Claude (CLAUDE.md, TODO/GDD files, design docs, `Docs/` folders, etc.) must always be written in English, regardless of the language used in the conversation. This applies to both creating new documents and translating/editing existing ones. Chat responses to the user still follow the conversation's language, this rule is only about the persisted document content.

---

## Git Operations

Conventions for commit message formatting (conventional-commit tags, English, title-only, no body, no trailers) and for what "push" means as shorthand (stage only the files/hunks from the current conversation, commit, then push).

Everything written into a pull request (title, description, and any comments or replies posted on it) is always in English, regardless of the language used in the conversation, and never carries the Claude Code attribution footer or session link.

Invoke the `git-operations` skill for the full rules whenever creating a git commit or pushing changes in any project.

---

## Skills & CLAUDE.md Repository Sync

This file, the skills it references, and `RTK.md` are also tracked in the `WendellLeao/skills` GitHub repository, so the same setup can be installed on any device.

Whenever asked to modify this `CLAUDE.md` file or any skill under `~/.claude/skills/`, after making the local change, ask the user whether they also want it pushed to that repository. Never push automatically. Only push when the user explicitly confirms, for that specific change.

@RTK.md
