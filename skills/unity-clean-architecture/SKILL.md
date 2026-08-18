---
name: unity-clean-architecture
description: Default folder, namespace, scene, and lifecycle architecture for brand-new Unity projects with no established conventions yet. Use when scaffolding a new Unity project or package, deciding where a new script or scene object belongs, or setting up cross-system lifecycle orchestration or a Service Locator. Do not use on an existing project that already has its own architecture; follow that instead.
---

# Unity Clean Architecture

A default architecture for brand-new Unity projects: how to lay out folders, namespaces, scene hierarchies, and lifecycle code so a team doesn't reinvent that structure on every project, and so you, as the agent scaffolding new work, have a concrete pattern to follow instead of guessing.

It pairs naturally with a starter-kit style package that installs reusable service packages (Service Locator, event bus, pooling, save, audio, and similar), but does not require one. Everything below stands on its own.

## Scope

This describes a **default for brand-new projects with no established architecture yet**. It is not a mandate for existing codebases: a project that already has its own conventions keeps following them. Apply this pattern only when scaffolding something genuinely new with no precedent to match.

## Guiding philosophy

- Follow SOLID and clean-code principles as the baseline for how classes are shaped and how they depend on each other.
- This architecture is a **paved path, not a straitjacket**. Its job is to prevent headaches on complex, multi-system work, not to force ceremony onto every trivial script. A developer implementing something simple should never have to fight the architecture or get lost figuring out where a one-off script belongs.
- Assembly definitions (`.asmdef`) and the module split (`MyGame.<Module>`) exist for the same reason: they pave the road toward decoupled, independently reasoned-about systems, not to gatekeep simple work.
- Default to the simplest thing that solves the problem in front of you. Reach for the heavier patterns below (manual orchestration, dumb/smart split) only when the system's actual complexity calls for them.
- Where a piece of infrastructure is genuinely project-agnostic (dependency resolution, eventing, pooling, audio playback), build it once as its own reusable package instead of rewriting it per project. A new project should start with its plumbing already solved, so a mid-level or junior developer only ever has to write game-specific code on top of it, never re-derive the plumbing itself.

## 1. Folder & namespace organization (code)

- `Assets/_Project/` holds all project-authored content, isolated from Unity's default `Assets/` root. The underscore sorts the folder to the top of the Project window regardless of what else gets imported later, so "our stuff" is always the fastest thing to find and never mixed with generated or third-party content.
- Inside `_Project/`, split by asset kind first, as sibling top-level folders: `Source/` (code), `Scenes/`, `Prefabs/`, `Materials/`, `UI/` (UI Toolkit `.uxml`/`.uss`/panel settings), `Inputs/` (Input System actions asset), `Settings/` (render pipeline assets, project-level ScriptableObjects).
- Inside `Source/`, organize by domain module, then subdomain: `Source/MyGame.<Module>/<Subdomain>/<Script>.cs`, one namespace per module. A module with no natural subdomain split (a small, cross-cutting utility module) keeps its scripts directly under the module folder instead of forcing a subdomain.
- Example module layout (`MyGame` is a placeholder for the actual project's root namespace):

```
Source/
  MyGame.Gameplay/
    Characters/
    Items/
    Score/
  MyGame.UI/
    Hud/
    TitleScreen/
  MyGame.Backend/
    Highscores/
  MyGame.Core/
    SingletonWaiter.cs
```

- Cross-module, cross-subdomain communication goes through interfaces that expose only the minimal surface the consumer needs, never the concrete type (see section 5 for the two shapes this takes in practice).
- Give a module its own `.asmdef` once the project is big enough that it matters. A compiler-enforced boundary actually keeps modules decoupled long-term; a namespace alone is only a convention. Skip this on a small or early project, where it would just add friction with no benefit yet.
- Reusable, project-agnostic tooling (Service Locator, event bus, object pooling, audio playback, etc.) does not live under `Source/` at all. It comes in as an imported package (Package Manager git URL or local package) from its own separate repository, updated independently of any single project. `Source/MyGame.<Module>/` stays game-specific.

## 2. Scene organization (empty GameObjects as domain folders)

Top-level GameObjects act as category folders, one per domain, grouping the actual functional objects that belong to it. These parent objects carry no components of their own; they exist purely to keep the Hierarchy window readable, grouped by responsibility instead of by transform proximity or spawn order.

Simplified example:

```
Gameplay
  ScoreManager
  PickupSpawner
UI
  GameplayHud
Rendering
  Main Camera
  Directional Light
```

A scene with dozens of loose root objects is a maintenance problem; a scene with a handful of labeled domain groups is not. Pick category names that match the code's module names where possible (`Gameplay` maps to `MyGame.Gameplay`, `UI` maps to `MyGame.UI`), so the mapping between "where this lives in the scene" and "where this lives in code" is obvious at a glance.

## 3. Dumb vs. smart components, and when orchestration applies

- **Dumb components**: no behavior of their own, they just hold display data pushed in from outside. Populated through `Initialize()`, same as everything else (see the naming convention below). Example: a UI list item, a simple data-holding prefab.
- **Smart components**: contain actual behavior. These split further into two cases:
  - **Standalone smart components**: self-contained, with no real ordering dependency on the rest of the scene. These self-initialize through Unity's own `Awake`/`OnEnable`/`OnDestroy`, exactly like a normal MonoBehaviour would. No manual orchestration is needed, and none should be added just for the sake of consistency. This covers most small, pointed pieces of gameplay.
  - **Orchestrated smart components**: part of a larger system where init order actually matters, most commonly meta-game systems (shop, screens/navigation, character customization, inventory) where several controllers depend on another's state being ready first. These are driven through the manual lifecycle interface described in section 4, not through Unity's own callbacks.
- Default to standalone. Only promote a smart component into the orchestrated lifecycle when it's genuinely the cross-system kind of ceremony the "Guiding philosophy" section above warns against.

## 4. Lifecycle orchestration

For orchestrated smart components: a root orchestrator (e.g. `GameplaySystem`, or a meta-game system like `ShopSystem`) owns the real Unity lifecycle (`Awake`/`Start`/`Update`/`FixedUpdate`/`LateUpdate`/`OnDestroy`) and explicitly propagates lifecycle steps down through its hierarchy of managers and children, in a deterministic, code-visible order, instead of relying on Unity's Script Execution Order project setting.

Example:

```
GameplaySystem
  - CharacterManager
    - CharacterSpawner
  - EnemyManager
    - EnemySpawner
```

`CharacterManager` is dispatched before `EnemyManager` by `GameplaySystem`, so characters spawn before enemies, which need the character's spawn position to know which path to head toward.

### Why over Script Execution Order

- Script Execution Order is a global, hidden project setting (Project Settings, not the script itself). It's easy to forget to configure for a new script, and hard to review or reason about from the code alone.
- Explicit orchestration makes cross-system init-order dependencies visible directly in the code that owns them.
- The same principle shows up in larger engines and frameworks: Unity DOTS, for instance, uses explicit `SystemGroup` ordering rather than implicit callback order.

### A lifecycle interface, not the literal Unity callback names

Do not reuse the names `Awake`/`Start`/`Update`/`FixedUpdate`/`LateUpdate`/`OnDestroy` for methods that are manually invoked by a parent orchestrator. Reusing them is ambiguous: a reader can't tell whether Unity itself is calling the method or whether the parent orchestrator is.

Instead, define a custom interface implemented by managers, spawners, and controllers. Only the root orchestrator hooks into real Unity `MonoBehaviour` callbacks; everything below it is driven through the custom interface. Example, with `GameplaySystem` driving `CharacterManager`, which drives `CharacterSpawner`:

```csharp
public sealed class GameplaySystem : MonoBehaviour
{
    [SerializeField] private CharacterManager _characterManager;
    [SerializeField] private EnemyManager _enemyManager;

    private void Awake()
    {
        _characterManager.Initialize();
        _enemyManager.Initialize();
    }
}

public sealed class CharacterManager : MonoBehaviour
{
    [SerializeField] private CharacterSpawner _characterSpawner;

    public void Initialize()
    {
        _characterSpawner.Initialize();
    }
}
```

### Naming convention

- **`Initialize()`**: brings a controller component into a ready state, whether that means starting an ongoing process (subscribing to events, starting timers or coroutines, something that keeps running until torn down) or just applying one-shot configuration. A single verb covers both cases. `Setup()` is never used; `Initialize()` is the only entry-point verb. Preferred over `Begin()`: `Initialize()` is the more conventional term in C#/.NET and Unity/DI frameworks (VContainer, Zenject), and pairs cleanly with `Shutdown()`, while `Begin()` reads more like a transient, async-style call (`BeginInvoke`, `BeginRead`) than "bring this object into a ready state."
- **`Shutdown()`**: the paired teardown for `Initialize()`. Unsubscribes events, cancels timers, releases references. Preferred over `Dispose()` unless the type is actually implementing `IDisposable`, since `Dispose()` carries a BCL contract (`using`/`using var`, deterministic disposal) that a custom convention shouldn't imply without actually honoring it.
- **`Tick(float deltaTime)`**: mirrors `Update`, invoked by the orchestrator once per frame with the delta time it's working with.
- **`FixedTick(float fixedDeltaTime)`**: mirrors `FixedUpdate`, for physics-rate logic.
- **`LateTick(float deltaTime)`**: mirrors `LateUpdate`, for logic that must run after every `Tick()` for the frame has completed (e.g. camera follow).

Passing `deltaTime`/`fixedDeltaTime` as an explicit parameter, rather than each child reading `Time.deltaTime`/`Time.fixedDeltaTime` itself, keeps a single source of truth for the frame's timing coming from the orchestrator's own callback, which makes the child methods easier to reason about and to unit test without a live `MonoBehaviour`/`Update` loop.

## 5. Cross-module references: services vs. direct wiring

Two shapes cover how one module talks to another:

- **Services**, for anything naturally singleton-ish (input, audio, save data, analytics, camera): a system the whole scene talks to. See section 6.
- **Direct, narrow cross-module contracts**, wired via `[SerializeField]`, when one module needs to invoke or react to another module's component without depending on its concrete type. Expose only the one or two members the consumer actually needs, never the full concrete API.

Crossing a module boundary is necessary but not sufficient to justify an interface on its own. It earns its keep only where there's real variability to decouple: either the field could genuinely hold different, unrelated concrete types at different times, or the interface-typed reference already comes for free (a value handed back from elsewhere that's already interface-typed, so there's no extra wiring cost to keeping it that way). A `[SerializeField]` that always points at one specific concrete type, fixed at prefab-edit time with no other implementer ever expected, is fine left concrete. Introducing an interface there is ceremony, not decoupling.

## 6. Cross-cutting services via Service Locator

Global, singleton-ish systems (input, audio, camera, save data, analytics, event bus) are exposed as **services**: resolved through an interface via the Service Locator, never referenced by concrete type, and never wired via `[SerializeField]` the way a sibling or child dependency would be. `[SerializeField]` wiring is for dependencies fixed at prefab-edit time; a service can be resolved from anywhere in the scene, so it goes through the locator instead.

### Build once, reuse everywhere

The Service Locator itself, and any service generic enough to be project-agnostic (an event bus, object pooling, audio playback, save/load), belongs in its own standalone repository, not in a given project's `Source/` folder. Import it as a package into each new project, then build only project-specific services (e.g. a scoring service for one particular game's rules) on top of it inside `Source/MyGame.<Module>/`. It's plug-and-play: game-specific services are the only thing that need to be written from scratch.

A service can itself depend on another service resolved the same way. An event bus service is a common example: registered and resolved through the Service Locator exactly like any other service, but other services and gameplay code then use it as a shared communication channel instead of hand-rolling a one-off C# event for anything that needs to reach more than one or two listeners.

### Recipe for adding a new service

1. Define `I{X}Service` in the owning module's folder, exposing only the minimal, read-only surface consumers actually need. Never leak setters or mutators a consumer shouldn't have.
2. Implement `internal sealed class {X}Service : MonoBehaviour, I{X}Service` in the same folder.
3. Register itself in `Awake()`: `ServiceLocator.Register<I{X}Service>(this);`
4. Unregister itself in `OnDestroy()`: `ServiceLocator.Unregister<I{X}Service>();`
5. Place the component on its own prefab, instanced wherever the project's bootstrap sequence guarantees it registers before any consumer's `Awake()` runs.
6. Consumers hold a private field typed as the **interface**, resolved once via `ServiceLocator.Get<I{X}Service>()`, never the concrete class.

Each service registers and unregisters **itself**. A bootstrapper or orchestrator (section 4) may sequence *when* scenes or services load, but it never registers a service on another component's behalf.

### Wrap engine types behind a service

A gameplay component should never hold a direct reference to a generic Unity type it doesn't own the lifecycle of (`Camera.main`, `AudioListener`, `Time`, `Application`, etc.). Wrap it behind a service (`ICameraService`, `IAudioService`) instead, and resolve the engine reference lazily inside the service rather than caching it too early, e.g. before the scene's camera exists. This keeps gameplay code portable across scenes and easier to reason about without a live engine context.

## 7. Logic vs. view split, event-driven state

The component that owns a piece of gameplay state (a root script like `Health`, `CharacterMovement`) is not the same component that renders the effects of that state.

A child GameObject named **View** holds everything that only renders or reacts to state: `SpriteRenderer`, `Animator`, particle systems, and any script that only touches those (`{X}View`, damage-flash-style feedback). A `View` script only ever subscribes to the root's events and exposes a passive method like `ToggleFlipY`; it never calls back into gameplay logic.

Rule of thumb: a component that makes a decision or owns state belongs on the root, even if its only visible effect is triggering an animation. A component that only exists to be rendered, or only reacts passively by touching a renderer, belongs on `View`. Cross-references between the root and `View` are ordinary `[SerializeField]` wiring, the same as any other sibling dependency.

Pair this with how state changes are communicated: the component that owns state broadcasts moments another system might care about as `On*` C# events (`OnDamaged`, `OnDied`). It never lets outside code poll a getter from `Update()`. Every reactor, `View` or otherwise, subscribes and unsubscribes symmetrically (`OnEnable`/`OnDisable`, `+=`/`-=`). Multiple reactors can listen to the same event without knowing about each other, so a health bar, a damage flash, and a death sequence can all react to the same `OnDamaged` independently.
