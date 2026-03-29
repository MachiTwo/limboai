# BUSINESS RULES: LIMBOAI - GOVERNANCE CONTRACT

> [!TIP]
> **Read this in other languages / Leia isto em outros idiomas:**
> [**English**](LIMBOAI_BUSINESS_RULES.md) | [**Português**](LIMBOAI_BUSINESS_RULES.pt.md)

This comprehensive document establishes the architectural boundaries, engineering dogmas, and non-negotiable business rules for the **LimboAI** framework (Behavior Trees and Hierarchical State Machines). Due to the conceptual magnitude and extreme impact on game performance, this contract serves as the definitive technical documentation for engine developers (C++/GDScript). Any implementation violating these limits is considered architectural debt and must be refactored immediately.

---

## 1. PHILOSOPHY AND RIGOROUS ENGINEERING

The project rejects the culture of **"Vibe-Coding"** (programming guided by trial and error, intuition, or optimism). Every line of Artificial Intelligence in Behavior Trees (BT) and State Machines (HSM) is treated as a high-performance, deterministic, and thread-safe software engineering commitment.

### 1.1 Pair Programming and The Control Matrix

- **Radical Code Detachment:** The Tree or HSM is the **Single Source of Truth (SSOT)**. If an actor moves to the wrong position, the flaw lies exclusively in the tree's abstraction, the `LimboState`, or the `BlackboardPlan` schema. Correction is made by restructuring the visual node or its logical `_tick()`, never by manual patching.
- **Absolute Sovereignty (Forbidden Anti-Pattern):** Never apply velocity vectors (`velocity = ...`), play animations (`AnimationPlayer.play()`), or trigger crucial logic in loose scripts attached to the actor's `_physics_process` in parallel to the AI. The tree does not share control of the body. The tree _is_ the control of the body.
- **Base Imutability:** `.tres` files (BehaviorTree, BlackboardPlan) are read-only "Blueprints" at runtime (Native Godot Resources). What operates and mutates in the game is the RAM clone (`BTInstance`).

### 1.2 The TDD Protocol for AI

Artificial Intelligence must pass deterministic testing.

1. **RED:** Write the AI test scenario (e.g., "Actor should back away if health < 20%"). The expected sequence of the Blackboard should fail.
2. **GREEN:** Implement the minimal BT branch to fulfill the evasion condition.
3. **REFACTOR:** Optimize execution by replacing script loops with native C++ decorators or bindings.

---

## 2. THE COGNITIVE MATRIX: BLACKBOARD SYSTEM

The Behavior Tree, by canonical definition, is "Stateless". It forgets what it did as soon as it finishes. The **Blackboard** is its "RAM Memory", an optimized `StringName -> Variant` dictionary managed via RefCounted.

### 2.1 The Unified Blackboard API

Developers must strictly use the unified Blackboard API. Direct access to raw properties is contained by native C++ functions provided by the `Blackboard` class:

```cpp
class Blackboard : public RefCounted {
public:
    // Defines or overwrites a variable. O(1) operation.
    void set_var(const StringName &p_key, const Variant &p_value);

    // Retrieves a variable. default_value is MANDATORY for governance
    // to avoid crashes when the tree attempts to access null keys in C++.
    Variant get_var(const StringName &p_key, const Variant &p_default = Variant()) const;

    // Attests formal key existence in the dictionary (even if null).
    bool has_var(const StringName &p_key) const;

    // Actively destroys the entry. Use with extreme caution.
    void erase_var(const StringName &p_key);

    // Dynamically connects a base Actor property (See Section 3.2).
    void bind_property(const StringName &p_key, Object *p_object, const StringName &p_property);

    // References vertical memory inheritance (See Section 2.2).
    void set_parent_scope(const Ref<Blackboard> &p_board);
    Ref<Blackboard> get_parent_scope() const;
};
```

### 2.2 Parent Scopes and the "Hive-Mind" (Ancestral Scope)

Arguably the most powerful architectural paradigm under LimboAI tutelage.

- **Resolution Concept (Fallback):** The Blackboard is a directional Linked List. When a task asks for `get_var("target_pos")`, the engine searches the local Blackboard. If the key is missing and the board points to a `parent_scope`, the search climbs the hierarchy procedurally up to the Master Root.
- **"Squad/Hive-Mind" Implementation:** In RTS or massive combat games (e.g., 50 wolves attacking a player), DO NOT duplicate tracking the Player 50 times.
  1. Allocate **ONE** global master `Blackboard` in a "Director" Node (The `PackManager`).
  2. The `PackManager` updates the `target_pos` variable in its own Master Board.
  3. On spawn, the 50 wolves execute: `wolf_board.set_parent_scope(pack_manager_board)`.
  4. **Decoupling Magic:** No wolf knows who the leader is or who the target is through hardcoded means. Their trees run completely clean, pulling attached cognition, saving up to 80% RAM and unnecessary C++ access.

---

## 3. DATA MARSHALLING: BLACKBOARD PLAN AND BBPARAM

### 3.1 BlackboardPlan: The Inviolable Skeleton

The `BlackboardPlan` is not a simple dictionary; it is the **Strong Typing Signature** of the AI.

- **Golden Rule:** No task (`BTTask`) is authorized to "deduce" or "inject" ghost keys into the board at runtime. Every variable must be pre-registered and exported through the corresponding plan.
- **Typing Structures (BBParams):** The API is based on rigorous restriction. We use the following packages:
  - `BBNode`: Requires injection of `NodePath` or direct allocation.
  - `BBInt`, `BBFloat`, `BBString`, `BBVector2`, `BBVector3`.
  - `BBVariant`: The wildcard joker. AVOID it whenever possible to prevent corrupting strict typing in C++.

### 3.2 Property Bindings (Optimized Active Bridge)

**The Problem (Anti-Pattern):** A junior developer writes an Action called `BTCondition_HealthLow` which, every tick, executes `blackboard->set_var("hp", agent->get_health())`. This wastes 1 processing frame just for useless scope transport.

**The Hybrid Solution (Bindings):** The Blackboard does not hold the float. The Blackboard becomes a _C++ Pointer_ directly to the Actor's variable.

- **Mandatory Implementation:** If a variable reflects a mutable external property, register it at instantiation time (`_ready` of the Player):
  ```gdscript
  var board: Blackboard = bt_player.get_blackboard()
  board.bind_property(&"current_health", self, &"health")
  ```
- **Analytical Consequence:** Whenever the tree invokes `get_var(&"current_health")`, C++ descends via the Object bridge natively and returns the live number. Zero delay, zero useless sync-steps.

---

## 4. EXECUTION ENGINE: BEHAVIOR TREES AND LIFECYCLE VIRTUALS

LimboAI operates by evaluating reactive flow heuristics `Parent -> Child`. The atomic class, the core we extend, is `BTTask`.

### 4.1 The Numerical Status Matrix (The Primitive Language)

Any and all return values of evaluative methods in the Godot Behavior Tree are expressed in this canonical dictionary (enum in C++). These Statuses are the gears, and ignoring them breaks the engine:

1. **`FRESH` (Untouched):** Primordial state. A Task is `FRESH` when the tree is instantiated, OR when the branch is aborted/reset by a superior Composer transaction.
   - **Exact Prohibition:** If you write C++ and your `_tick()` loop returns `FRESH`, the engine will lock in a fatal silent loop. Pure leaf tasks NEVER return `FRESH` in the live pipeline.
2. **`RUNNING` (Continuous Transient Work):** Means "Invoke me again next tick". Anchored in an Action ("Attacking", "Running", "Casting Magic"). The tree pauses linear scanning at the same depth level and rests its computational focus on you.
3. **`SUCCESS` (Solution/True):** "Reached point A", "Finished the strike", "Had enough HP". Informs the Parent Composite (Sequences/Selectors) to proceed or bypass to the next structural candidate.
4. **`FAILURE` (Abstrusion/False/Impossible):** It is a conditional deviation, not an error. "Target fled", "Out of Range", "Interrupted". Signals the Parent Composite that the path broke, triggering contingency plans (Selector) or aborting the global mission (Sequence).

### 4.2 The BTTask Lifecycle (API Mandatory Overrides)

If a developer (C++ Engineer or GDScripter) creates a New Node (`BTActionCustom`), the Limbo API exposes 4 canonical virtual methods. These compose the **Tool Cycle**:

```cpp
// --- EXTENDED VIRTUAL INTERFACE OF BTTask CLASSES ---

// 1. The Heap Spawn Moment
virtual void _setup() override;
/* Fired exactly 1 time after Resource->Instance copy (In _ready).
 * Use EXCLUSIVELY to fetch base references via SceneTree (e.g., RayCasts, AnimationPlayers)
 * which would cost inestimable resources to fetch every frame if placed in _tick. */

// 2. The Task Awakening
virtual void _enter() override;
/* Fired at the exact instant the Status variable shifts from FRESH and evolves.
 * This is where the event begins in time! Here you apply Animations, initialize a Timer
 * Date, or format Raycasts towards the target direction in a zero-frame. */

// 3. The Cycle-by-Cycle Mathematics
virtual int _tick(double p_delta) override;
/* The Engine. Invoked by Engine update rate (PhysicsProcess).
 * OBLIGATIONS: Operate continuous mathematics and invoke MANDATORY NUMERCIAL RETURN
 * (SUCCESS, FAILURE, RUNNING). Brutal processing lives here. */

// 4. The Purge (The Cleanup Mandate) - The Golden Rule
virtual void _exit() override;
/* Fired as soon as the task dispatches SUCCESS/FAILURE back to the tree,
 * OR, drastically, when a Parent Composite (e.g., DynamicSelector) ABORTS
 * YOU while you were still in a continuous RUNNING loop. */
```

### 4.3 The Golden Rule of Preemption (Why \_Exit saves the game)

Imagine: A `BTAction` task starts an internal Timer to release an explosion linked to the `_explode()` Callable that would occur in 3.0s (and reports `RUNNING`). At 2.0s, the enemy is Rooted and the entire upper tree (`BTSelector`) cancels combat to go to the defense branch. Your Action is aborted by forceps!
If the internal Timer is _NOT_ reset/destroyed in your virtual `_exit()` function, at second 3 it will call `_explode()` leaking memory, crashing the Godot C++ engine by calling zombie callbacks out of context.
**The LimboAI Mantra:** Everything structured in `_enter()` **MUST BE DECONSTRUCTED/RELEASED** in `_exit()`.

---

## 5. FLOW ROUTERS: COMPOSITE NODES

Composites control the mind's passage. They accept `N` children in the visual SceneHierarchy, iterating from Top to Bottom / Left to Right. The evaluation of "whether it locks or continues" dictates the semantic logic.

### 5.1 The "LOGICAL OR": BTSelector

- **C++ Implementation:** Scans its children through an iterator Loop. The moment the child of Index `I` fires `SUCCESS` or `RUNNING`, the Loop aborts. The Selector copies this return to itself and spits it upward into the tree.
- **Effect:** If the first ones failed, it "tries the next" (`FAILURE` -> Next i++). It cements a **Priority List**. Will only report `FAILURE` if the entire mesh fails miserably below it.
- **Strict Usage:** Determining the grand Master block ("Attack, or Flee, or Patrol").

### 5.2 The "LOGICAL AND": BTSequence

- **C++ Implementation:** Scans its children in a Loop. If child Index `I` says `SUCCESS`, the loop advances to Index `I+1`. The catch is: If A SINGLE CHILD aborts and spits `FAILURE`, the Loop bleeds immediately. The Sequence aborts and dispatches a massive `FAILURE` upwards, killing the plan.
- **Effect:** It only spits `SUCCESS` if **ALL** children managed to be 100% victorious one by one.
- **Strict Usage:** Ritual sequencing that doesn't operate halfway (1. "Turn Knob" -> 2. "Open Door" -> 3. "Enter and Walk").

### 5.3 The Re-evaluative Modifiers (Dynamics)

Standard composer systems anchor verification. If the 3rd child gave `RUNNING`, on the next frame the Sequence calls it directly (without re-evaluating child 1 and 2). This anchor blocks tactical reflex reactions.

- **`BTDynamicSelector`:** This is aggressive. Even if the 3rd child ("Attack at Distance") is still emitting `RUNNING`, every time the Engine calls `_tick` on this Selector, it forcibly re-evaluates the first and second child! If the first child ("Hit closely" due to approaching enemy) returns `SUCCESS/RUNNING`, the Selector cowardly and actively aborts the 3rd child, firing the Virtual `_exit()` call killing the old execution and routing on the new base. Pure C++ API guarantees "O(1) Instant Reflex".
- **`BTDynamicSequence`:** Forces the entire mesh before the Running node to report SUCCESS every damn tick, guaranteeing that "The vital condition to continue doing X still exists". Extremely expensive as a processing mesh and CPU overhead. Use carefully.

### 5.4 The Concurrent Abyss: BTParallel

- **The Abstraction:** It simulates Multithreading without the secondary thread. `BTParallel` sweeps the entire list pushing a `_tick` to **ALL** children in the EXACT SAME FRAME simultaneously (Doesn't matter if some return Running).
- **Canonical Return Policies:** The C++ class has a base acceptance flag. `SUCCESS_IF_ALL` (The parallel only spits success when everyone finishes together) or `SUCCESS_IF_ONE` (Just one microtask shines and the parallel cruelly aborts all the rest of the Runners processing with it).
- **Strict Anti-Pattern Dogma:** NEVER orchestrate mutually exclusive mutative base logics in Parallels. Inserting `WalkToA` parallel to `WalkToB` tasks will cause the engine to rewrite the Actor's C++ `Velocity` vector Matrix 2 times in the instant, destroying the parametric physics delta.

---

## 6. FLOW MUTATORS: DECORATOR NODES

Decorators are **absolutely limited** to anchoring 1 and ONLY ONE direct child. They agglutinate an "intercepting ring" between Composite nodes and Leaves, and act subverting responses.

### 6.1 The Timer Group and Watchdogs (Logical Stopwatches)

The Custom Node API encapsulates a reactive mechanism that locks processing at all costs.

- **`BTCooldown`:** Parameter: `BBFloat CooldownTime`. After the attached sub-node returns `SUCCESS`, this parasite instills a pure O(1) time barrier. The barrier locks access. If the tree descends into it while counting, it lies by answering `FAILURE` directly, diverting the parent's flow without invoking the sub-node C++. Ideal for: "Cast Fireball" (Wait 6 secs to skip this tree and try to strike).
- **`BTTimeLimit`:** The anti-lockdown leash. Parameter: `BBFloat Time_Limit`. If the child locks the branch holding response in `RUNNING` iteratively higher than the TimeMax limit... The TimeLimit applies the sentence: Aborts the root forcing `FAILURE` and forcing the `_exit()` routine to transition the child and kill it. (Final protection against unreachable A\* PathFindings or corrupted NavigationServer that would trap the AI trying to "run" into a wall infinitely in the Godot Engine).
- **`BTDelay`:** Forced suspension. Intercepts the start, throws a useless `RUNNING` variable to the engine for N seconds (faking heavy loading) until finally delegating the base call for real.

### 6.2 The Pure Static Manipulation Group

- **`BTInvert` (NOT logical gate):** In C++, pure O(1) interception that swaps `SUCCESS` for `FAILURE` and `FAILURE` for `SUCCESS`. (Commonly used associated with native Condition logics. If `Condition(IsAlive)` is associated, with invert it becomes The `While(Dead)`).
- **`BTAlwaysSucceed` / `BTAlwaysFail`:** Shields the base tree. Use on a node which the Master Composer (Selector/Seq) should not suffer base interruptions from, faking that the dirty debug tactic "always worked" forcedly without breaking the original contiguous tree cascade in the Godot Editor.
- **`BTRepeat`:** Pure for/while loop. Forces the child to restrictively respond to repetitive re-entries with SUCCESS counted `Times_X` (Or iteratively for Repeat While Failure infinite loop).

---

## 7. TERMINAL ENDPOINTS: ACTIONS AND CONDITIONS

All abstract visual machinery of the BT peaks in Leaves, where world C++ interactions with Godot's Environment Scene operate factually. The Dogma stringently separates them into 2 classes.

### 7.1 BTCondition (The Principle of Instant Evaluation)

- **Non-Negotiable Nature:** The C++/GDScript Condition class **MUST NEVER adopt delayed routine flows**. Computations must be `O(1)` or extremely light O(N) array checks. They look at BBPlan, attest numeric facts and must report the final return IMMEDIATELY.
- **ABSOLUTE Critical Prohibition:** It is **EXPRESSLY FORBIDDEN AND PUNISHED** to code the unstable constancy of numeric status `RUNNING` in the native `_tick()` motor of a `BTCondition`. They respond hastily: True Fact (`SUCCESS`) or False Fact (`FAILURE`).
- **Legal Base Cases:** Check booleans `bool can_jump`, check base collision `is_on_floor()`, static instant linear raycast (Without iterative frame sweeping). `await` Coroutine delays from GDScript in a Condition corrupt the parallel deterministic performance of the Limbo tree.

### 7.2 BTAction (The Occupational Operational Paradigm)

The absolute Modifier of the Actor. Moves Force vectors, dictates base collision paths for NavMesh Server, alters AnimTree BlendSpaces base status.

- **Integral Permitted Act:** Only an Action has authorized jurisdictional domain to declare temporal States in the engine holding the Tree through a `return RUNNING`.
- If the mechanic spends Continuous Gameplay Time... It is a BTAction. If it spends Zero Time (Just an IF)... It is a Pure Condition adapted visually. (Fake Actions are useless Actions that resolve into 1_tick = SUCCESS without loading status. In these cases write in condition format with passive checker nomenclature).

---

## 8. HIERARCHICAL STATE MACHINES (HSM): THE CONTINUOUS LOOP

When confronting massive complex enemies and Bosses that shift Forms and hardcoded temporal invulnerabilities, parametric pure Behavior Trees lose strength in a sea of Ifs. For these Enginered Continuous States of Abrupt Transition, we use `LimboHSM`.

### 8.1 Physical and Static Architecture

In brutal contrast to Matrix RAM Clone Trees in \_ready (`BTInstance`), the **HSM** deforms Godot based on `Hierarchical Live Scene Nodes`. Every HSM component, from Root to final Branch, is composed of Nodes Statically visualizable in Native SceneTree (`LimboState`).

- **Base Nodes:** `LimboHSM` is the parent of Scene Nodes. If it hosts `State1` and `State2` below it... It takes Maximum Control managing the Active Brain in a Unicolor process (Meaning only 1 state executing as Master).
- **Super-States Nesting (Hierarchical Power):** If we base the HSM on a sub-parent derived node `MovableState` which has below it the Scene-Node daughter leaves `WalkState` and `SprintState`. All mechanical injection of the Parent superimposes the child without extra code ("If Moving Base = True and Player inputs then..."). A State perfectly covers its underworld.

### 8.2 The HSM Golden Cycle (The LimboState Virtual Engine)

Every script or C++ inherited from `LimboState` is guided by a rigorous O(1) virtual engine:

1. **`_setup()`:** Node \_ready C++ initialization. Extract null variables ("GetNode", "FindChildRaycast_Path"). Execution of fixed "Isolated Inits".
2. **`_enter()`:** The Exact temporal Cell where "You won the keys to the empire". The Limbo Swap occurred and Focused on You! Start timers and base anims.
3. **`_update(delta)`:** Update-Fix. Procedural Physics and frame_rate locked linear interpolations roll here. Unlike BTs that report status, states remain locked in this Loop eternally until extremely external forces act or Bodyguards act via dispatch.
4. **`_exit()`:** Master Dogma of RAM C++ Cleanup (`Queue Free`). Everything chained in temporary `_enter` disappears instantly and without mercy triggered in the engine.

---

## 9. EVENT-DRIVEN DISPATCH: TRANSITIONS AND GUARDS

Critical architectural mechanism. Spaghetti-Polling C++ is expressly FORBIDDEN. (Doing `if (dead) change_state("Dead_Scene")` in a 60 FPS loop inside EVERY live state consuming immense CPU procedural processing for all Actors of an MMO map at the same time).

### 9.1 Event-Driven Network Dispatcher

Transitions are not hunted; They react. They live in formal parameterized static Setup:

```cpp
// 1. Establishing and creating impassable fixed bridges outside logic in C++ root setup
my_hsm->add_transition("State_Combat_Boss", "State_Cinematic_Stun", "boss_poise_broken_event");
```

**The Dispatcher Miracle:** When damage breaks the Boss's posture, the Brain in a remote random location hastily emits a Limbo shock event onto the AI network:

```cpp
my_hsm->dispatch("boss_poise_broken_event");
```

The Limbo core itself abstracts all `_exit` from Boss_Combat and natively activates procedural Limbo States engine keys for closed Scene Stun. ZERO wasted passive C++ loops sweeping entire arrays of estatic O(N) enemy flags lost in active `_update` gears throughout the Game.

### 9.2 Conditional Guards Keys (Limbo C++ Turnstiles)

Engine Transitions Support Intercepting Dispatches Based purely upon parameterized `Callable` checks.

```cpp
// Adding transaction based purely on a key if a Node Callable Func returns TRUE
my_hsm->add_transition("A", "B", "go_to_b_evt", Callable(this, "has_hp_to_use_skill"));
```

- **The Golden Rule of Guards:** Guards (The Callable Callback) are "ReadOnly" Validators of based facts. If a Func Guard applies damage to a native enemy... Or drops base items from player array in inventories just when trying to validate the bool... Vibe-Coding overflowed and you created the Worst Technical Consequences Side-Effects Base Godot Engine of Difficult Debug. A Guard READS and SPITS BOOL without altering ANYTHING of data.

---

## 10. HYBRID PARADIGM: BTSTATE INTEGRATION

The architectural glory: Envelope Behavior Trees in Limbo States Machines Native Leaf Scenes!
The solidified limitation of the HSM, allied with the Reactive visual flow of the Behavior Tree.

**How and Why to Operate (The Master Hybrid Loop):**
Native Node `BTState` runs sheltered in a base HSM Scene hierarchy.

1. Upon taking Sovereignty activating its `_enter()`, the Node State awakens in Godot Base the "Resource visually attached parametric Tree" attached to it. Every `_tick()` fluidity enters the transient wheel.
2. Dark Magic of Transition: Upon closing the arc of the Root cycle of the attached tree, releasing `SUCCESS` Outbound... The Node State Captures the message! Envelops it in the Limbo predatory string event and actively Forces the very dispatch of formal emission within the parent HSM calling base string events `EVENT_SUCCESS` or `EVENT_FAILURE` prefixed in the native LimboAI macro matrix class.

- We use this in the entire isolated gear with visual parametric perfection without extra code in boss "Phase 3 Combat Form" isolated from "Phase 1 Statters". And all this is governed by pure native dispatch logics already implemented in the root of the Limbo Engine Core Node to save RAM from empty crossed variables in a giant O(N) Base script.

---

## 11. MEMORY MANAGEMENT AND DETERMINISM

To solidify robust LimboAI for Networking (Where "Jitters" in Visual Base response screw Rollbacks and interpolated multi-player predictions):

- **Physics Update Engine C++ Core (The Lock Framerate):** In configurations of `BTPlayer` and Global Root Node `LimboHSM` the gear demands restrictive emission configuration. Set to `_PHYSICS_PROCESS_` process mode! Locks the isolated AI to emit and make Behavior Tree processing base CallBacks tied to Godot Fixed simulation pure C++ cadence (Standard 60Hz param). If you put normal framerate tied visual process (Variant Idle Sync 120+FPS) AI would try to think faster computing more branches in advanced Desktops and desynchronize the entire Netcode pipeline interpolated predictions parametric C++ of mechanics tied across the whole Godot Environment engine!
- **Zero RAM Initiation on-the-fly (`Queue_free` forbidden Mid-Loop):** When BTInstance (The Matrix memory) clones arrays of `Ref` from Immutable `.tres` tree into HEAPS memory... Godot suffers a giant Allocation OS impact on O(N) RAM processors. This "Duplicate Matrix Clone" occurs ONCE_AT_START. Deleting `Node_Enemies` in loop and instantiating Godot enemies from Absolute Zero hundreds of times will "Punish C++ FPS/CPU Godot Base Jittering GCs rates with absolute Freeze-Rate-Drops". If there are dozens of enemy zombies dying "Deactivate your `BTPlayer` from hierarchy and Change Visibility Scene", recycling C++ Pool in RAM scene to optimize brute logics in isolated engine param parametric API backend without dropping C++ Engine-Godot memory System Architecture Optimization Process O(1)!

---

## 12. C++ CUSTOM TASK EXTENSION API

Parametric extensions for projects utilizing Godot "Node/BTTask" class fall into extremely rigorous C++ Registration and Initialization methodological convention rules:

### 12.1 Binding and Blackboard Type Extraction

```cpp
// Canonical Base Numeric C++ Example of Board Interception:
// If "range" param of BTAction depends on BBFloat exportable variable:
void BTActionWalkTo::_bind_methods() {
	ClassDB::bind_method(D_METHOD("set_move_speed", "p_speed"), &BTActionWalkTo::set_move_speed);
	ClassDB::bind_method(D_METHOD("get_move_speed"), &BTActionWalkTo::get_move_speed);
	// ... (add ADD_PROPERTY definitions for C++ Editor Inspector)
}

// Parametric Clean Call Usage in Tick_Routine
int BTActionWalkTo::_tick(double p_delta) {
	 // 1. You MUST pass base Blackboard pointer for clean extraction in Board FallBack!
	 double speed_real_calc = move_speed->get_value(get_blackboard(), 5.0f); // default native fallback
}
```

---

## 13. NAMING STANDARDS AND TAXONOMY

### 13.1 Sacred Base Class Prefixes

The LimboAI Engine Framework will push CustomNodes via graphic panel tied to standard restrictive nomenclature.

- Active/Occupational Class: MANDAORILY starts with `BTAction<ClassName>`.
- Bool/Math Checker Validation: MANDAORILY starts with `BTCondition<ClassName>`.

### 13.2 Case Standardization

- **Snake_case** is mandatory in All Custom C++ Blackboard native properties nomenclature used for rendering in Inspector (e.g. `speed_boost`). Never camelCase in LimboAI Godot extensions native base.

---

## 14. DEBUGGING AND PROFILING PATTERNS

### 14.1 LimboDebugger Usage

LimboAI includes a visual debugger to inspect tree states during runtime.

- **Rule:** Do not clutter `_tick()` methods with `print()` statements for debugging. Rely exclusively on the LimboDebugger interface which natively traces Visual BT execution flow without penalizing CPU cycles with strings concat allocations.

---

## 15. TESTING PARADIGMS FOR AI

### 15.1 Component Isolation

When developing Custom BTTasks in C++, unit testing must:

- Mock the Blackboard explicitly without initializing full game Worlds.
- Test `_enter()`, `_tick(0.016)`, and `_exit()` manually in testing loops to verify expected internal state cleansing.
