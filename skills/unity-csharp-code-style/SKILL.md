---
name: unity-csharp-code-style
description: C# code style and formatting conventions for Unity projects. Covers member ordering within a class (nested types, events, serialized fields, fields, constructors, properties, methods), method ordering (Unity callbacks first, then TryGet*/Get*/Set* at the bottom by access), always-braced control flow, expression-body usage, On*/Handle* event naming, and personal formatting preferences (const field placement, PascalCase static readonly fields, single-space operators, minimal-surface interfaces at system boundaries, SerializeField over GetComponent*). Use whenever writing or reviewing C# code in a Unity project.
---

# Unity C# Code Style

Formatting and naming conventions for C# code in Unity projects, so code reads consistently across scripts and doesn't need re-explaining on every task.

## File layout: order of members within a class

| Section | Ordering within section |
|---|---|
| Nested types | By access (public → private) |
| Events | By access (public → private) |
| Serialized fields | Keep together |
| Fields | `const` → `readonly` → instance (but see Personal preferences below) |
| Constructors | By access |
| Properties / Indexers | By access |
| Methods | See method ordering below |

## Method ordering within a class
1. **public**, Unity callbacks first, then the rest
2. **protected**, Unity callbacks first, then the rest
3. **private**, Unity callbacks first, then the rest
4. **TryGet* methods** (bottom, by access: `public` → `private`)
5. **Get* methods** (bottom, after TryGet*, by access: `public` → `private`)
6. **Set* methods** (bottom, after all Gets, by access: `public` → `private`)

Expression-body properties (`=> field`) are **not** treated as Get methods; they stay adjacent to the fields they expose.

## Expression bodies
Allowed. Use freely for single-line properties and methods where it improves readability.

## Braces
Always use braces `{ }` for `if`, `else`, `for`, `foreach`, `while`, etc., even for single-statement bodies. No one-line `if (x) return;`.

## Event and handler naming
- **Events**: `On*` prefix, e.g. `OnValueChanged`, `OnElementSelected`
- **Event handlers**: `On*` prefix, e.g. `private void OnValueChanged(int value)`
- **`Handle*` prefix**: only when a naming conflict exists within the same class, i.e. when the class both raises its own event (`OnX` as raiser) and consumes an external event of the same name (use `HandleX` for the consumer)

## Personal preferences
- **`const` field placement**: `const` fields stay with the other fields, not at the top of the class. Order within fields: `[SerializeField]` → `const` → `readonly` → instance. Never hoist `const` above serialized fields.
- **`private static readonly` field naming**: use `PascalCase`, no underscore prefix (e.g. `Services`, not `_services`), matching Rider/ReSharper's default "Static readonly fields (private)" naming rule. All other private fields (instance fields) keep the `_camelCase` underscore prefix as usual.
- No aligned spacing around operators: always single space, `_foo = x;` never `_foo   = x;`
- Return values from helper/converter methods always stored in a local variable before being passed as argument.
- Add a blank line before `return` when there is logic above it in the same block.
- **Abstraction exposure**: whenever a component/object is handed to another system (a service registered in a Service Locator, an item object passed to a manager, etc.), expose an interface with only the minimal surface the consumer needs (read-only data, read-only events), not the concrete type. Keep mutating methods/setters on the concrete class only, so the consumer can observe/read but never mutate state it doesn't own. Skip this when there's no real external boundary (a type only ever consumed internally by a single system); a speculative interface there is indirection without benefit.
- **Explicit serialized references over `GetComponent*`**: wire a component's dependencies (sibling/child components, other scripts on the same prefab) via `[SerializeField]` fields set in the Inspector, instead of resolving them at runtime with `GetComponent`/`GetComponentInChildren` in `Awake`. Gameplay objects are encapsulated as self-contained prefabs, so there's no scenario where a dependency needs to be "discovered" dynamically; it's fixed at prefab-edit time. Serialized fields also make a component's dependencies visible just by looking at it in the Inspector, and an unwired reference shows up as an obvious "None" instead of failing silently at runtime. Since the reference is a required prefab-wiring dependency, skip null checks on it too (a missing wire is a prefab configuration bug, not a runtime case to guard against).
