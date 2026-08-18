# Skills

A collection of [Claude Code Skills](https://docs.claude.com/en/docs/claude-code/skills) that codify how I want an AI coding agent to work in specific domains, so the decisions don't have to be re-explained every time and any teammate can adopt the same conventions by installing the same skill.

## Installation

Each folder under `skills/` is one skill. To install a skill, copy its folder into your own `~/.claude/skills/` directory:

```
git clone https://github.com/WendellLeao/skills.git
cp -r skills/skills/unity-clean-architecture ~/.claude/skills/
```

Claude Code discovers skills from the current project or your global `~/.claude/skills/` directory and triggers them automatically based on the `description` in each skill's frontmatter, no manual invocation needed.

## Skills

### [`unity-clean-architecture`](skills/unity-clean-architecture/SKILL.md)

Default folder, namespace, scene, and lifecycle architecture for brand-new Unity projects with no established conventions yet: `_Project/Source/<Module>/<Subdomain>` layout, scene hierarchies organized by domain, a dumb/smart component split, explicit lifecycle orchestration instead of Script Execution Order, and cross-cutting systems exposed through a Service Locator. It's a paved path, not a mandate: existing projects keep following their own established architecture.

It pairs well with the [Unity Starter Kit](https://github.com/WendellLeao/unity-starter-kit), which installs the reusable service packages (Service Locator, event bus, pooling, save, audio) this architecture assumes, but the skill itself has no dependency on it.

### [`unity-engine-guidelines`](skills/unity-engine-guidelines/SKILL.md)

Behavioral guidelines for writing, editing, and reviewing Unity C# code to reduce common LLM coding mistakes: surface assumptions and tradeoffs before coding, default to garbage-free and `[SerializeField]`-wired code, make only surgical changes, work toward verifiable goals, respect the MonoBehaviour lifecycle and Unity 6 stable APIs, never hand-edit serialized asset files outside the Editor, pair every event subscription with an unsubscription, and blend changes into existing code without leaving a trace.

### [`unity-csharp-code-style`](skills/unity-csharp-code-style/SKILL.md)

C# formatting and naming conventions for Unity projects: member ordering within a class, method ordering (Unity callbacks first, then `TryGet*`/`Get*`/`Set*` at the bottom), brace and expression-body rules, `On*`/`Handle*` event naming, and formatting preferences like const field placement, static readonly naming, and preferring `[SerializeField]` over `GetComponent*`.

### [`git-operations`](skills/git-operations/SKILL.md)

Git commit and push conventions used across projects, not specific to Unity: conventional-commit-style message tags (`feat:`, `fix:`, `refactor:`, etc.), title-only commit messages with no body or trailers, and a "push" shorthand that scopes staging to only the files/hunks touched in the current conversation before committing and pushing.

## License

MIT, see [LICENSE](LICENSE).
