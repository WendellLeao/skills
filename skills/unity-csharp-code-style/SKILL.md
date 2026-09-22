---
name: unity-csharp-code-style
description: C# code style and formatting conventions for Unity projects, split into industry-standard convention and the user's own personal preferences (explicitly labeled as such). Convention covers member ordering within a class (nested types, events, serialized fields, fields, constructors, properties, methods), method ordering (by access, Unity callbacks first), always-braced control flow, expression-body usage, method/event naming (On*/Handle* events, *Async for Task-returning methods, *Routine for coroutines), SerializeField attribute placement/naming (own line, underscore prefix kept, no blank lines between fields except when grouped by [Header]), nested-type vs. one-type-per-file decision (DTOs/structs), vertical whitespace/conceptual-affinity grouping within method bodies, extracting well-named helper methods out of long methods that mix distinct concerns, and target-typed `new()`. Personal preferences cover const field placement, single-space operators, and extracting call arguments into named locals. (Minimal-surface interfaces at system boundaries and SerializeField over GetComponent* live in the unity-engine-guidelines skill instead, they're design decisions, not formatting.) Use whenever writing or reviewing C# code in a Unity project.
---

# Unity C# Code Style

Formatting and naming conventions for C# code in Unity projects, so code reads consistently across scripts and doesn't need re-explaining on every task.

This doc is split in two:

- **Convention** — rules grounded in a recognized source (an official .NET guideline, a StyleCop rule, a widely-adopted Unity community practice, a named principle like SOLID). Follow these the same way you'd follow any team's style guide.
- **Personal preferences** — my own calls where the industry doesn't have a single agreed-upon answer. They're consistent and intentional, but don't present them to someone else as "the professional standard," they're house style, not law.

## Convention

### `[SerializeField]` placement and naming

1. The `[SerializeField]` attribute goes on its own line, above the field. Don't inline it with the field declaration.
2. The field keeps the `_camelCase` underscore prefix, exactly like any other private instance field. `[SerializeField]` doesn't change that: the field is still `private` in C#, it's just exposed to the Inspector.

```csharp
[SerializeField]
private float _moveSpeed;
```

Not `[SerializeField] private float _moveSpeed;` and not `[SerializeField] private float moveSpeed;`.

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

### Nested types vs. one type per file (DTOs, structs, small data types)

Default (general C# convention, matches StyleCop `SA1402` / Microsoft guidance): **one type per file.** A folder with one file per class/struct is more professional and maintainable than a single file accumulating many unrelated types, it's easier to navigate, diff, and `git blame`.

The decision of whether to nest a small type (DTO, struct, prediction data, etc.) inside its owning class or give it its own file depends on coupling and reuse, not on type count:

- **Nest inside the class** when the type is a private implementation detail used by exactly one class and nowhere else, e.g. FishNet's `ReplicateData`/`ReconcileData` structs on a `NetworkBehaviour` (see `AGISPredictedCharacterMover.cs`). These only exist to satisfy that one class's prediction contract, are never reused elsewhere, and nesting them keeps the contract visible right next to the methods (`[Replicate]`/`[Reconcile]`) that consume it. This is also the idiomatic FishNet pattern used in its own examples, so it's the right call there specifically.
- **Give it its own file** when the type is shared across multiple classes/systems, e.g. backend request/response DTOs consumed by more than one handler, or any type meant to be part of a module's public surface. Put these in a dedicated folder (e.g. `DTOs/`, `Models/`) with one file per type.
- **Match the existing file's convention when editing it.** If asked to add a new struct/DTO to an existing file that already nests its DTOs internally (e.g. a `BackendHandler.cs` that already keeps its request/response types nested or grouped in that same file), follow that file's established pattern instead of unilaterally splitting it into new files. Respecting what's already there beats applying the "ideal" rule mid-file; if the pattern genuinely needs to change, that's a separate conversation with the user, not a silent drive-by during an unrelated task.

### Vertical whitespace inside method bodies (conceptual affinity)

Group statements by "concept," the same way paragraphs group sentences in prose: lines that belong to the same idea stay adjacent with no blank line between them; a blank line marks a real shift to a different concept. This is a recognized practice (Robert C. Martin's *Clean Code* calls it "conceptual affinity" / vertical formatting; Steve McConnell's *Code Complete* recommends grouping related statements), not just personal taste, even though most linters/formatters don't enforce it and plenty of programmers skip it by habit. Applying it well requires judgment, so treat it as a principle to aim for rather than a mechanical rule.

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

### Method length: extract when concerns are separable

When a method grows long because it mixes several genuinely distinct concerns, extract each concern into a well-named private helper method instead of leaving it as one long block held together with comments or blank-line groups (Single Responsibility Principle applied at the method level). The top-level method should end up reading as a short, ordered list of steps, e.g.:

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
- Place a new private helper right after the method that is its main caller, not at the bottom of the class.

### File layout: order of members within a class

| Section | Ordering within section |
|---|---|
| Nested types | By access (public → private) |
| Events | By access (public → private) |
| Serialized fields | Keep together |
| Fields | `const` → `readonly` → instance (see "const field placement" under Personal preferences below for the actual position relative to serialized fields) |
| Constructors | By access |
| Properties / Indexers | By access |
| Methods | See method ordering below |

### Method ordering within a class

1. **public**, Unity callbacks first, then the rest
2. **protected**, Unity callbacks first, then the rest
3. **private**, Unity callbacks first, then the rest

Unity callbacks first within each access level (`Awake`, `OnEnable`, `Update`, etc.) is a common Unity convention, it makes the lifecycle easy to scan at the top of each section. Beyond that, place a method near the caller that actually uses it (see the extraction rule above); there's no separate bucket for `Get*`/`Set*`/`TryGet*` methods, they follow normal placement like anything else.

Expression-body properties (`=> field`) are **not** treated as methods; they stay adjacent to the fields they expose.

### Expression bodies

Allowed. Use freely for single-line properties and methods where it improves readability.

### Braces

Always use braces `{ }` for `if`, `else`, `for`, `foreach`, `while`, etc., even for single-statement bodies. No one-line `if (x) return;`. This matches most enterprise C# style guides (Google's C# style guide among them) and avoids dangling-else / accidental-scope bugs when a body later grows past one line.

### Method and event naming

- **Events**: `On*` prefix, e.g. `OnValueChanged`, `OnElementSelected`.
- **Event handlers**: `On*` prefix, e.g. `private void OnValueChanged(int value)`.
- **`Handle*` prefix**: only when a naming conflict exists within the same class, i.e. when the class both raises its own event (`OnX` as raiser) and consumes an external event of the same name (use `HandleX` for the consumer). This whole block matches the official .NET Framework Design Guidelines for event naming.

```csharp
public event Action<int> OnHealthChanged;

private void OnEnable()
{
    _health.OnHealthChanged += HandleHealthChanged;
}

private void OnDisable()
{
    _health.OnHealthChanged -= HandleHealthChanged;
}

private void HandleHealthChanged(int newHealth)
{
    OnHealthChanged?.Invoke(newHealth);
}
```

Without the conflict, skip `Handle*` and just use `On*` for the subscriber too, e.g. `private void OnHealthChanged(int newHealth)`.
- **Async methods** (`Task`/`Task<T>` return type): `*Async` suffix, e.g. `LoadSceneAsync`, `FetchInventoryAsync`. This is the standard Task-based Asynchronous Pattern (TAP) naming rule from Microsoft's own guidelines, it signals to every call site that the method should be awaited.
- **Coroutines** (`IEnumerator` methods driven by `StartCoroutine`): `*Routine` suffix, e.g. `FadeRoutine`, `SpawnRoutine`. Not an official Unity rule, but a widely-adopted community convention, it makes the call site self-explanatory (`StartCoroutine(FadeRoutine(duration))`) and tells the reader at a glance that this isn't a normal synchronous method.

```csharp
private async Task LoadSceneAsync(string sceneName) { /* ... */ }

private IEnumerator FadeRoutine(float duration) { /* ... */ }
```

### Target-typed `new()`

When the target type is already known from the left-hand side (a field or variable declaration, a parameter, a return type), omit the redundant type name after `new`. This is the modern C#/.NET-recommended idiom (the `dotnet_style_prefer_new` analyzer default), not just a personal taste.

```csharp
private readonly AGISTickInputState _tickState = new();
```

Not `new AGISTickInputState();`. Only spell out the type after `new` when it genuinely isn't inferable from context (e.g. assigning a concrete type to a field/variable declared as a less-derived type or an interface, where restating the concrete type helps the reader).

> Two related rules, minimal-interface exposure at system boundaries and explicit `[SerializeField]` wiring over `GetComponent*`, live in the `unity-engine-guidelines` skill instead of here: they're design/architecture decisions (what a system exposes, how a dependency gets resolved), not formatting.

## Personal preferences

These are my own calls, not industry convention, flag them as such if you're explaining this style to someone else.

### `const` field placement

`const` fields stay with the other fields, not at the top of the class the way most style guides default to. Order within fields: `[SerializeField]` → `const` → `readonly` → instance. Never hoist `const` above serialized fields.

```csharp
[SerializeField]
private float _moveSpeed;
[SerializeField]
private float _jumpHeight;

private const float Gravity = -9.81f;
private readonly WaitForSeconds _respawnDelay = new(2f);

private bool _isGrounded;
```

Why: this keeps everything that configures the component (serialized values, constants, readonly values) visually grouped together above the "live" runtime state, instead of splitting `const` off to the very top of the file where it's disconnected from the serialized fields it's often related to.

### Extract call arguments into a named local variable

Return values from helper/converter methods are always stored in a local variable before being passed as an argument, instead of nesting the call inline.

```csharp
// Not this:
ApplyDamage(CalculateDamage(baseDamage, armor, criticalMultiplier, resistance));

// This:
float finalDamage = CalculateDamage(baseDamage, armor, criticalMultiplier, resistance);
ApplyDamage(finalDamage);
```

Why: the intermediate value gets a name (self-documenting at the call site) and becomes inspectable/breakpointable in the debugger, which a nested call expression isn't. It also scales better: as `CalculateDamage` picks up more arguments over time, an inline call keeps growing into one dense, hard-to-parse line, while the extracted-local version stays just as readable no matter how many arguments accumulate.

### Operator spacing

No aligned spacing around operators: always single space.

```csharp
_speed = 5f;
_jumpHeight = 2f;
```

Not:

```csharp
_speed      = 5f;
_jumpHeight = 2f;
```
