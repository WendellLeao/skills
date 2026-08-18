---
name: unity-engine-guidelines
description: Behavioral engineering guidelines for writing, editing, and reviewing Unity C# code, to reduce common LLM coding mistakes. Covers surfacing assumptions and tradeoffs before coding, defaulting to garbage-free and SerializeField-wired code, making only surgical changes, working toward verifiable goals, respecting the MonoBehaviour lifecycle and Unity 6 stable APIs, never hand-editing serialized asset files outside the Editor, pairing every event subscription with an unsubscription, and blending changes into existing code without leaving a trace. Use whenever writing, editing, or reviewing Unity/C# code in any project.
---

# Unity Engine Guidelines

Behavioral guidelines to reduce common LLM coding mistakes in Unity projects. Bias toward caution and system integrity over speed.

## 1. Think Before Coding
*Don't assume. Don't hide confusion. Surface tradeoffs.*
- State assumptions about execution order, dependencies, and engine lifecycles.
- If multiple implementations exist, present them before picking.
- If a simpler approach exists, propose it. Push back on complexity.
- If something is unclear (e.g., interaction with existing state machines or subsystems), stop and ask.

## 2. Simplicity & Performance
*Minimum code that solves the problem. Nothing speculative.*
- No speculative abstractions. If it's not used, it doesn't get written.
- Performance: Default to garbage-free code. In hot paths (`Update`, per-frame loops), `foreach` over a concrete `List<T>` or array does not allocate, since the enumerator is a struct. The real allocation risk is iterating through an interface-typed reference (`IEnumerable<T>`, `IList<T>`, etc.), which boxes the enumerator. Prefer keeping hot-path collections typed concretely (avoid boxing) over reflexively rewriting every `foreach` as `for`.
- References: Prefer `[SerializeField]` wiring over `GetComponent`/`GetComponentInChildren` for a component's dependencies (sibling/child components, other scripts). Reserve `GetComponent*` for cases where the dependency truly can't be wired in the Editor. See the `unity-csharp-code-style` skill for the full rationale.
- Serialization: Respect `[SerializeField]`. Keep state private. Do not suggest public fields for internal state.
- If you write 100 lines and it could be 40, rewrite it.

## 3. Surgical Changes
*Touch only what you must. Clean up only your own mess.*
- Match existing style and architectural patterns (e.g., State Machine/Service Locator patterns).
- Do not refactor adjacent code unless it directly causes the bug/feature to fail.
- Exception: If a change creates orphans or risks breaking SerializedProperty references/Prefab links, fix the dependency chain.
- Clean up only your own unused variables/functions.
- **Never touch blank lines that have no code to change.** Do not "normalize" trailing whitespace on a blank line (e.g. a line with leftover indentation spaces collapsed to a truly empty line) unless that exact line is part of the code block you are intentionally editing. This happens silently when rewriting a method/class block, so always check the diff afterward for stray blank-line-only changes and revert them.

## 4. Goal-Driven Execution
*Define success criteria. Loop until verified.*
- Transform tasks into verifiable goals: "Add interaction" → "Implement logic, verify Prefab reference is set, verify event trigger."
- For multi-step tasks, state a brief plan:
  1. [Step] → verify: [check]
  2. [Step] → verify: [check]
- If a goal cannot be verified within the Editor/Engine context, state why before implementing.

## 5. Engine Context
- *Default:* Assume MonoBehaviour lifecycle.
- *Scope:* Respect project-specific architectures. Do not introduce patterns that contradict existing modular gathering or system designs.
- *Constraint:* Default to Unity 6 stable APIs. Avoid experimental features unless explicitly asked.

## 6. Never Hand-Edit Unity Asset Files, Unless Going Through Unity Itself
- Never directly write or hand-edit the raw contents of `.prefab`, `.meta`, `.unity`, `.asset`, or any other Unity engine/editor-serialized file (e.g. via a text-editing tool). Only edit code files (`.cs`, `.md`, `.c`, `.js`, etc.) that way.
- `.meta` GUIDs are generated automatically by Unity when the user compiles/imports in the Editor. Do not invent or guess them.
- When a Unity MCP tool is available, use it (e.g. running Editor C# through it) to perform the actual prefab/scene/ScriptableObject wiring: attaching scripts, setting serialized fields, adding components, creating/editing prefabs, saving scenes. This goes through Unity's own serialization and is not "hand-editing"; it's allowed and preferred over asking the user to do manual Editor steps.
- Only fall back to telling the user which manual Editor steps remain when no such tool is available for that task.

## 7. Event Subscription Lifecycle
*Every subscribe needs an unsubscribe.*
- Always pair an event subscription (`+=`) with a matching unsubscription (`-=`), mirroring the lifecycle where the subscription happened (e.g. subscribe in `OnEnable`/`Awake`, unsubscribe in `OnDisable`/`OnDestroy`).
- If unsubscribing truly isn't needed (subscriber and event source share the exact same lifetime and are destroyed together) or genuinely isn't possible, stop and notify the user of the specific reason before deciding to leave it unsubscribed.

## 8. Blend Into Existing Code ("Ninja" Edits)
*Enter, change only what's needed, leave no trace.*
- When adding to or modifying an existing script, match its exact style, naming, formatting, and structure so the change reads as if it were always there, never "out of place" against the surrounding code.
- Keep the footprint minimal: mirror how nearby code already solves similar problems (reference caching, event-handling shape, logging style, etc.) instead of introducing a new pattern.
- Exception: if the surrounding code you're touching is itself buggy or poorly written, fix it rather than replicating the defect, but flag that fix to the user.
- **Do not add code comments by default**, even to explain a non-obvious "why" behind a fix. Only add comments when the script being touched is already comment-heavy, or is genuinely very complex. Match the file's existing comment density before adding one, the same way you match its style, naming, and formatting. A comment-free file that suddenly gets new comment blocks reads as out of place, breaking the "no trace" rule above, no matter how justified the comment would be in isolation.
