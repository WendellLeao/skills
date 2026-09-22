---
name: unity-csharp-code-style
description: C# code style and formatting conventions for Unity projects. Covers member ordering within a class (nested types, events, serialized fields, fields, constructors, properties, methods), method ordering (Unity callbacks first, then TryGet*/Get*/Set* at the bottom by access), always-braced control flow, expression-body usage, On*/Handle* event naming, SerializeField attribute placement/naming (own line, underscore prefix kept, no blank lines between fields except when grouped by [Header]), nested-type vs. one-type-per-file decision (DTOs/structs), vertical whitespace/conceptual-affinity grouping within method bodies, extracting well-named helper methods out of long methods that mix distinct concerns, and personal formatting preferences (const field placement, PascalCase static readonly fields, single-space operators, target-typed `new()`, minimal-surface interfaces at system boundaries, SerializeField over GetComponent*). Use whenever writing or reviewing C# code in a Unity project.
---

# Unity C# Code Style

Formatting and naming conventions for C# code in Unity projects, so code reads consistently across scripts and doesn't need re-explaining on every task.

## CRITICAL: `[SerializeField]` placement and naming

Always, without exception:

1. **The `[SerializeField]` attribute goes on its own line, above the field.** Never inline it with the field declaration.
2. **The field keeps the `_camelCase` underscore prefix**, exactly like any other private instance field. `[SerializeField]` does not change this.

```csharp
[SerializeField]
private float _moveSpeed;
```

Not `[SerializeField] private float _moveSpeed;` and not `[SerializeField] private float moveSpeed;`. The field is still `private` in C#, regardless of the Inspector being able to serialize it, so it follows the same naming rule as every other private field. This is a hard rule, not a case-by-case judgment call.

**Don't add blank lines between consecutive `[SerializeField]` fields.** Each field's own `[SerializeField]` + field pair stays tight, and the next field's pair follows immediately with no blank line in between:

```csharp
[SerializeField]
private UltimateCharacterLocomotion _locomotion;
[SerializeField]
private AGISPredictedLookSource _lookSource;
[SerializeField]
private NetworkTransform _networkTransform;
```

The only time serialized fields get visually separated is with `[Header("...")]`, when there are enough fields that grouping them helps whoever is wiring up the component in the Inspector:

```csharp
[Header("Components")]
[SerializeField]
private Rigidbody _rigidbody;
[SerializeField]
private Collider _collider;

[Header("Data")]
[SerializeField]
private MovementData _movementData;
```

Reach for `[Header]` once the serialized-field block is getting long/complex enough that grouping by role would make the Inspector easier to scan, not preemptively on every component.

## Nested types vs. one type per file (DTOs, structs, small data types)

Default (general C# convention, matches StyleCop `SA1402` / Microsoft guidance): **one type per file.** A folder with one file per class/struct is more professional and maintainable than a single file accumulating many unrelated types, it's easier to navigate, diff, and `git blame`.

The decision of whether to nest a small type (DTO, struct, prediction data, etc.) inside its owning class or give it its own file depends on coupling and reuse, not on type count:

- **Nest inside the class** when the type is a private implementation detail used by exactly one class and nowhere else, e.g. FishNet's `ReplicateData`/`ReconcileData` structs on a `NetworkBehaviour` (see `AGISPredictedCharacterMover.cs`). These only exist to satisfy that one class's prediction contract, are never reused elsewhere, and nesting them keeps the contract visible right next to the methods (`[Replicate]`/`[Reconcile]`) that consume it. This is also the idiomatic FishNet pattern used in its own examples, so it's the right call there specifically.
- **Give it its own file** when the type is shared across multiple classes/systems, e.g. backend request/response DTOs consumed by more than one handler, or any type meant to be part of a module's public surface. Put these in a dedicated folder (e.g. `DTOs/`, `Models/`) with one file per type.
- **Match the existing file's convention when editing it.** If asked to add a new struct/DTO to an existing file that already nests its DTOs internally (e.g. a `BackendHandler.cs` that already keeps its request/response types nested or grouped in that same file), follow that file's established pattern instead of unilaterally splitting it into new files. Respecting what's already there beats applying the "ideal" rule mid-file; if the pattern genuinely needs to change, that's a separate conversation with the user, not a silent drive-by during an unrelated task.

## Vertical whitespace inside method bodies (conceptual affinity)

Group statements by "concept," the same way paragraphs group sentences in prose: lines that belong to the same idea stay adjacent with no blank line between them; a blank line marks a real shift to a different concept. This is a recognized practice (Robert C. Martin's *Clean Code* calls it "conceptual affinity" / vertical formatting; Steve McConnell's *Code Complete* recommends grouping related statements), not just personal taste, even though most linters/formatters don't enforce it and plenty of programmers skip it by habit.

```csharp
float referenceTime = Time.time;
ReconcileData data = default;

_locomotion.CapturePredictionState(ref data.Locomotion);

if (_jump != null)
{
    _jump.CaptureJumpPredictionState(ref data.Jump, referenceTime);
    _jump.CapturePredictionState(ref data.JumpAbility, referenceTime);
}
```

- Local variable declarations/caching at the top of a method: grouped together, no blank line between them.
- Each subsequent independent operation (a method call, an `if` block handling one specific concern): its own group, separated from the previous one by a single blank line.
- Statements that are all part of the same block of logic (e.g. everything inside one `if` handling jump-specific state) stay tight together internally, only separated from what comes before/after.

Don't overdo it: a blank line should mark an actual change of concept, not appear every 1-2 lines by default, over-fragmenting hurts readability as much as never separating anything does.

## Method length: extract when concerns are separable

When a method grows long because it mixes several genuinely distinct concerns, extract each concern into a well-named private helper method instead of leaving it as one long block held together with comments or blank-line groups. The top-level method should end up reading as a short, ordered list of steps, e.g.:

```csharp
[Reconcile]
private void PerformReconcile(ReconcileData data, Channel channel = Channel.Unreliable)
{
    EvaluateReconcileCorrection(data.Locomotion.Position);

    CharacterLocomotion.PredictionReplaying = true;
    float tickDelta = (float)TimeManager.TickDelta;
    float referenceTime = GetTickTime(data.GetTick());
    AGISUccTimeManager.BeginTickOverride(referenceTime, tickDelta);

    _locomotion.ApplyPredictionState(data.Locomotion);
    Physics.SyncTransforms();

    ApplyJumpPrediction(data, referenceTime);
    ApplyAbilityPredictionInputState(_sprint, data.SprintAbility, referenceTime);
    ApplyAbilityPredictionInputState(_crouch, data.CrouchAbility, referenceTime);

    _previousButtons = data.PreviousButtons;
    AGISUccTimeManager.EndTickOverride();
}
```

- A method whose blank-line-separated groups (see "Vertical whitespace" above) are each their own concern, nameable independently of the others (e.g. "resolve which data to use," "process this tick's input," "simulate movement"), is a good extraction candidate.
- A repeated shape across an `if`/`if`/`if` chain (e.g. the same align-then-apply logic for jump/sprint/crouch abilities) is a strong signal to extract one shared helper that takes the varying parts as parameters, instead of duplicating the shape for each branch.
- Don't extract just because a method is long when the length comes from one mechanical, single-concern operation, e.g. copying a dozen scalar fields between a struct and a component for a snapshot/restore pair. Splitting that further adds indirection without adding clarity. Length alone isn't the signal, mixed concerns are.
- Place a new private helper right after the method that is its main caller, not at the bottom of the class, unless it's a `Get*`/`Set*`/`TryGet*` method, which still goes to the bottom per the method-ordering rule below regardless of who calls it.

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
- **Target-typed `new()`**: when the target type is already known from the left-hand side (a field or variable declaration, a parameter, a return type), omit the redundant type name after `new`. `private readonly AGISTickInputState _tickState = new();` not `new AGISTickInputState();`. Only spell out the type after `new` when it genuinely isn't inferable from context (e.g. assigning a concrete type to a field/variable declared as a less-derived type or an interface, where restating the concrete type helps the reader).
- Return values from helper/converter methods always stored in a local variable before being passed as argument.
- Add a blank line before `return` when there is logic above it in the same block.
- **Abstraction exposure**: whenever a component/object is handed to another system (a service registered in a Service Locator, an item object passed to a manager, etc.), expose an interface with only the minimal surface the consumer needs (read-only data, read-only events), not the concrete type. Keep mutating methods/setters on the concrete class only, so the consumer can observe/read but never mutate state it doesn't own. Skip this when there's no real external boundary (a type only ever consumed internally by a single system); a speculative interface there is indirection without benefit.
- **Explicit serialized references over `GetComponent*`**: wire a component's dependencies (sibling/child components, other scripts on the same prefab) via `[SerializeField]` fields set in the Inspector, instead of resolving them at runtime with `GetComponent`/`GetComponentInChildren` in `Awake`. Gameplay objects are encapsulated as self-contained prefabs, so there's no scenario where a dependency needs to be "discovered" dynamically; it's fixed at prefab-edit time. Serialized fields also make a component's dependencies visible just by looking at it in the Inspector, and an unwired reference shows up as an obvious "None" instead of failing silently at runtime. Since the reference is a required prefab-wiring dependency, skip null checks on it too (a missing wire is a prefab configuration bug, not a runtime case to guard against).
