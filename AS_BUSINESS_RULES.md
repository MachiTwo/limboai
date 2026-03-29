# BUSINESS RULES: ABILITY SYSTEM - GOVERNANCE CONTRACT

This document establishes the architectural boundaries and mandatory business rules. Any implementation violating these limits must be refactored immediately.

---

## 1. PHILOSOPHY AND RIGOROUS ENGINEERING

The project rejects **"Vibe-Coding"** (programming by intuition or luck). Every line of business logic is treated as an industrial engineering commitment.

### 1.1 Pair Programming and Governance

- **Radical Code Detachment:** If code fails, it's a communication or architecture flaw. Corrections are made via dialogue and documentation adjustment, never manual patches.
- **SSOT (Single Source of Truth):** This file is the Iron Law. Before any complex change, the rule must be documented here.
- **Language:** Code and technical documentation in **English**. Dialogue and pair programming pitch in **Portuguese**.

### 1.2 TDD Protocol (Red-Green-Refactor)

No business logic exists without a test justifying it.

1. **RED:** Write the failing test, defining the contract.
2. **GREEN:** Implement the minimal code to pass.
3. **REFACTOR:** Optimize while maintaining the passing status.

---

## 2. THE IDENTITY MATRIX: TAGS (THE "STATE")

Tags are not classes; they are **Superpowered Hierarchical Identifiers** based on `StringName`. They represent the absolute truth about an Actor's present state.

### 2.1 Tags Golden Rules (What I AM)

- **Role:** Represent continuous states, traits, and blocks (e.g., `State.Dead`, `Status.Stunned`).
- **Nature:** Persistent. Require formal addition (`add_tag`) and formal removal (`remove_tag`). They consume CPU time calculating RefCounts in `ASTagSpec`.
- **Answers the Question:** _"In this exact microsecond, is this actor under condition X?"_
- **Absolute Prohibition (Anti-Pattern):** NEVER use tags to represent instant occurrences (e.g., DO NOT use `State.JustGotHit`). If the condition lasts only 1 frame or serves to alert systems, it MUST be an Event, never a Tag.

### 2.3 The 3 Canonical Tag Types

The `Tag Type` defines the **semantic role** a tag has. It determines how the Singleton and Editor treat that identifier.

| Type          | Conventional Prefix  | Role                                                                                                 | Example                                     |
| ------------- | -------------------- | ---------------------------------------------------------------------------------------------------- | ------------------------------------------- |
| `NAME`        | `Char.` / `Team.`    | Static identity and categorization of the actor                                                      | `Char.Warrior`, `Team.Blue`                 |
| `CONDITIONAL` | `State.` / `Status.` | Persistent gameplay state that can be added/removed from the actor                                   | `State.Stunned`, `Status.Poisoned`          |
| `EVENT`       | `Event.`             | Occurrence identifiers. Registered for autocomplete, but **the payload never reaches the Singleton** | `Event.Weapon.Hit`, `Event.Damage.Critical` |

### 2.4 Tag Type Structures and Implementation

The unified tag system is implemented through three core structures in `as_tag_types.h`, each providing type-safe tag handling with built-in validation and convenience methods.

#### 2.4.1 ASNameTag - Persistent Identity Tags

**Purpose**: Long-lasting state identifiers that persist until explicitly removed
**Duration**: Until manually removed via `remove_tag()`
**Usage**: Character classes, persistent states, team affiliations

**Structure Definition:**

```cpp
struct ASNameTag : public ASTagBase {
    // Constructor
    ASNameTag(const StringName &p_name);

    // Factory Methods
    static ASNameTag create(const StringName &p_name);

    // Predefined Common Tags
    static ASNameTag STUNNED();        // "State.Stunned"
    static ASNameTag DEAD();           // "State.Dead"
    static ASNameTag INVISIBLE();      // "State.Invisible"
    static ASNameTag WARRIOR();        // "Class.Warrior"
    static ASNameTag MAGE();           // "Class.Mage"
    static ASNameTag ARCHER();         // "Class.Archer"
    static ASNameTag TEAM_BLUE();      // "Team.Blue"
    static ASNameTag TEAM_RED();       // "Team.Red"
};
```

**Key Features:**

- **Type Safety**: Compile-time guarantee of tag type
- **Validation**: Automatic validation against AbilitySystem registry
- **Convenience**: Predefined common tags for frequent use
- **Inheritance**: Extends `ASTagBase` with common functionality

#### 2.4.2 ASConditionalTag - Requirement/Block Tags

**Purpose**: Ability/effect requirements and blockers that control permissions
**Duration**: Typically short-term, tied to specific conditions or effects
**Usage**: Ability prerequisites, damage immunity, permission checks

**Structure Definition:**

```cpp
struct ASConditionalTag : public ASTagBase {
    // Constructor
    ASConditionalTag(const StringName &p_name);

    // Factory Methods
    static ASConditionalTag create(const StringName &p_name);

    // Predefined Permission Tags
    static ASConditionalTag CAN_PARRY();      // "Can.Parried"
    static ASConditionalTag CAN_DODGE();       // "Can.Dodged"
    static ASConditionalTag CAN_INTERRUPT();   // "Can.Interrupted"

    // Predefined Immunity Tags
    static ASConditionalTag IMMUNE_FIRE();     // "Immune.Fire"
    static ASConditionalTag IMMUNE_POISON();   // "Immune.Poison"
    static ASConditionalTag IMMUNE_PHYSICAL(); // "Immune.Physical"

    // Predefined State Condition Tags
    static ASConditionalTag GROUNDED();        // "State.Grounded"
    static ASConditionalTag FLYING();          // "State.Flying"
    static ASConditionalTag STEALTHED();       // "State.Stealthed"
};
```

**Key Features:**

- **Permission Control**: Fine-grained ability activation control
- **Immunity System**: Centralized damage type immunity handling
- **State Conditions**: Environmental and positional state tracking
- **Runtime Validation**: Automatic checking against current actor state

#### 2.4.3 ASEventTagTag - Event Dispatcher Tags

**Purpose**: Event identifiers for the dispatch system with built-in helper methods
**Duration**: Instantaneous (events are transient, but history persists briefly)
**Usage**: Combat events, ability lifecycle, state transitions, weapon interactions

**Structure Definition:**

```cpp
struct ASEventTagTag : public ASTagBase {
    // Constructor
    ASEventTagTag(const StringName &p_name);

    // Factory Methods
    static ASEventTagTag create(const StringName &p_name);

    // Combat Event Tags
    static ASEventTagTag DAMAGE_DEALT();        // "Event.Damage.Dealt"
    static ASEventTagTag DAMAGE_TAKEN();        // "Event.Damage.Taken"
    static ASEventTagTag DAMAGE_BLOCKED();      // "Event.Damage.Blocked"
    static ASEventTagTag HEAL_RECEIVED();       // "Event.Heal.Received"

    // Ability Event Tags
    static ASEventTagTag ABILITY_ACTIVATED();   // "Event.Ability.Activated"
    static ASEventTagTag ABILITY_FAILED();      // "Event.Ability.Failed"
    static ASEventTagTag ABILITY_COOLDOWN_END(); // "Event.Ability.CooldownEnd"
    static ASEventTagTag ABILITY_INTERRUPTED(); // "Event.Ability.Interrupted"

    // State Event Tags
    static ASEventTagTag STUN_BEGIN();          // "Event.Stun.Begin"
    static ASEventTagTag STUN_END();            // "Event.Stun.End"
    static ASEventTagTag DEATH();               // "Event.Death"
    static ASEventTagTag RESPAWN();             // "Event.Respawn"

    // Weapon Event Tags
    static ASEventTagTag WEAPON_HIT();          // "Event.Weapon.Hit"
    static ASEventTagTag WEAPON_MISS();         // "Event.Weapon.Miss"
    static ASEventTagTag WEAPON_CRITICAL();     // "Event.Weapon.Critical"

    // Helper Methods
    void dispatch(Node *p_instigator, float p_magnitude = 0.0f,
                  const Dictionary &p_payload = Dictionary()) const;
    bool occurred_recently(Node *p_target, float p_lookback_sec = 1.0f) const;
};
```

**Key Features:**

- **Dispatch Integration**: Built-in event dispatching capability
- **Historical Queries**: Direct access to recent event occurrence
- **Comprehensive Coverage**: Predefined tags for all common game events
- **Type Safety**: Compile-time event type validation

### 2.5 ASTagUtils - Tag Type Utilities and Validation

The `ASTagUtils` namespace in `as_tag_types.cpp` provides comprehensive tag validation, type detection, and historical query capabilities.

#### 2.5.1 Type Validation and Detection

```cpp
namespace ASTagUtils {
    // Core Validation Functions
    bool validate_tag_type(const StringName &p_tag, ASTagType p_expected_type);
    ASTagType detect_tag_type(const StringName &p_tag);
    ASTagBase create_tag(const StringName &p_tag);

    // Pattern Recognition Functions
    bool is_state_tag(const StringName &p_tag);     // "State.*"
    bool is_class_tag(const StringName &p_tag);     // "Class.*"
    bool is_team_tag(const StringName &p_tag);      // "Team.*"
    bool is_event_tag(const StringName &p_tag);     // "Event.*"
    bool is_immune_tag(const StringName &p_tag);    // "Immune.*"
    bool is_can_tag(const StringName &p_tag);       // "Can.*"
}
```

**Type Detection Algorithm:**

1. **Event Tags**: Start with "Event." → `ASTagType::EVENT`
2. **Conditional Tags**: Start with "Can.", "Immune.", "State.Grounded", "State.Flying", "State.Stealthed" → `ASTagType::CONDITIONAL`
3. **Name Tags**: All other patterns (State._, Class._, Team.\*, etc.) → `ASTagType::NAME`

#### 2.5.2 Historical Tracking Structures

Each tag type maintains its own historical buffer with 128-entry circular buffers:

```cpp
// Historical Entry Structures
struct ASNameTagHistoricalEntry {
    StringName tag_name;
    ObjectID target_id;
    double timestamp = 0.0;
    uint64_t tick_id = 0;
    bool added = true; // true for add, false for remove

    // Helper Methods
    Node *get_target() const;
    void set_target(Node *p_node);
};

struct ASConditionalTagHistoricalEntry {
    // Same structure as ASNameTagHistoricalEntry
    // Used for CONDITIONAL tag changes
};

struct ASEventTagHistoricalEntry {
    ASEventTagData data;  // Complete event payload
    uint64_t tick_id = 0;
};
```

#### 2.5.3 Historical Query APIs

**ASNameTag Historical Query API:**

```cpp
// Basic Queries
ASTagUtils::name_was_tag_added("State.Stunned", target, 1.0f);
ASTagUtils::name_was_tag_removed("State.Stunned", target, 1.0f);
ASTagUtils::name_had_tag("State.Stunned", target, 1.0f);

// Data Retrieval
ASTagUtils::name_get_recent_additions(target, 1.0f);
ASTagUtils::name_get_recent_removals(target, 1.0f);
ASTagUtils::name_get_recent_changes(target, 1.0f);

// Counting Operations
ASTagUtils::name_count_additions("State.Stunned", target, 1.0f);
ASTagUtils::name_count_removals("State.Stunned", target, 1.0f);
```

**ASConditionalTag Historical Query API:**

```cpp
// Specialized for CONDITIONAL tag changes
ASTagUtils::cond_was_tag_added("Immune.Fire", target, 1.0f);
ASTagUtils::cond_was_tag_removed("Immune.Fire", target, 1.0f);
ASTagUtils::cond_had_tag("Immune.Fire", target, 1.0f);
```

**ASEventTag Historical Query API:**

```cpp
// Event-Specific Queries
ASTagUtils::event_did_occur("Event.Damage", target, 1.0f);
ASTagUtils::event_get_recent_events("Event.Damage", target, 1.0f);
ASTagUtils::event_get_all_recent_events(target, 1.0f);

// Event Data Access
ASTagUtils::event_get_last_data("Event.Damage", target);
ASTagUtils::event_get_last_magnitude("Event.Damage", target);
ASTagUtils::event_get_last_instigator("Event.Damage", target);

// Counting Operations
ASTagUtils::event_count_occurrences("Event.Damage", target, 1.0f);
```

**Unified Historical Query API:**

```cpp
// Cross-Type Queries
ASTagUtils::history_was_tag_present("State.Stunned", target, 1.0f);
ASTagUtils::history_get_tag_history("State.Stunned", target, 1.0f);
ASTagUtils::history_get_all_changes(target, 1.0f);

// Debug Utilities
ASTagUtils::history_dump(target, 5.0f);
ASTagUtils::history_get_total_size(target);
```

### 2.6 Event System Integration

**Events are now a specialized tag type** managed through the unified tag system. The old ASEvent Resource has been deprecated and replaced with ASTagUtils-based event handling.

#### 2.6.1 Event Tag Data Structure

Events use `ASEventTagData` struct in `ASUtils` for complete payload information:

- `event_tag`: Exact occurrence identifier (e.g., `Event.Interrupt`)
- `instigator`: The Node that caused it (offender)
- `target`: The affected Node (victim)
- `magnitude`: Base intensity of the event (Impact power)
- `custom_payload`: GDScript/Variant dictionary for transmitting metadata
- `timestamp`: Registered natively in precise milliseconds
- `tick_id`: Tick identifier for multiplayer synchronization

#### 2.6.2 Event Dispatch API

```gdscript
# Signature: dispatch_event(event_tag, instigator, magnitude, custom_payload)
var asc: ASComponent = target.get_node("ASComponent")
asc.dispatch_event(&"Event.Weapon.Hit", self, 35.0, {})
```

#### 2.6.3 Short-Term Memory (Events Historical)

Events die in 1 tick, but their _"memory"_ persists lightly:

- `ASComponent` keeps an `_event_history` (optimized C++ circular buffer up to 128 entries).
- **Practical use:** Reactive components (like _Parry_ or _Counter-Attack_) don't need to be in an eternal "parrying" state. They can check recent past: `has_event_occurred(&"Event.Damage.Block", 0.4)`. If it occurred in the last 0.4s, authorize the counter-attack.
- **Rule:** Never use this cache to persist history (quests, missions). Use exclusively for temporal reactivity frames.

#### 2.6.4 Individual Historical Tracking

All three tag types maintain individual historical buffers in `ASComponent`:

- **NAME History**: Tracks persistent state changes (`State.Stunned`, `Class.Warrior`)
- **CONDITIONAL History**: Tracks permission/immunity changes (`Can.Parried`, `Immune.Fire`)
- **EVENT History**: Tracks event occurrences with complete payload data

Each buffer maintains 128 entries with automatic overflow management. Historical APIs provide specialized queries:

```cpp
// NAME tag queries
ASTagUtils::name_was_tag_added("State.Stunned", target, 2.0f);
ASTagUtils::name_count_additions("State.Stunned", target, 10.0f);

// CONDITIONAL tag queries
ASTagUtils::cond_had_tag("Immune.Fire", enemy, 5.0f);

// EVENT tag queries
ASTagUtils::event_did_occur("Event.Damage", target, 1.0f);
ASTagUtils::event_get_last_magnitude("Event.Damage", target);

// Unified queries
ASTagUtils::history_was_tag_present("State.Stunned", target, 2.0f);
```

### 2.7 Tag Groups (Visual Organization)

**Tag Groups are not code entities.** They are an editorial convention that automatically emerges from the dot (`.`) hierarchy in tag identifiers.

- `ASTagsPanel` renders tags as a **tree**, using each dot-separated segment as a parent node.
- `State.Stunned`, `State.Dead` automatically group under the visual node `State`.
- There is no `TagGroup` C++ object — the "group" is just the shared prefix.
- **Mandatory Convention:** The root prefix MUST reflect its `Tag Type` (e.g., `Event.*` tags are always `ASTagType::EVENT`).

### 2.8 Logical Evaluation (Predicates)

The system supports 4 logical states in Blueprints (Ability, Effect, Cue) for evaluating requirements and target blocks:

- `Required All` (AND): Success if has ALL.
- `Required Any` (OR): Success if has AT LEAST ONE.
- `Blocked Any` (OR): Failure if has ANY ONE.
- `Blocked All` (AND): Failure if has ALL SIMULTANEOUSLY.

> [!NOTE]
> Predicates work exclusively with `CONDITIONAL` tags. `NAME` and `EVENT` tags do not enter requirement/block lists in blueprints. The Editor enforces this automatically via `ASInspectorPlugin`.

### 2.9 The Split Registry Pattern

Event identifiers (e.g., `Event.Weapon.Hit`) **are registered in the Singleton** like any other tag — for autocomplete and typo prevention. The difference is their type: registered as `Tag Type = EVENT`.

What **never** goes to the Singleton is the **data instance** — the `ASEventTagData` struct. This is the **Split Registry**:

- **Singleton (Registry):** Knows the _name_ `Event.Weapon.Hit`. Guarantees it exists, has the right type, appears in autocomplete.
- **ASComponent (Occurrence):** Knows the _happening_. Knows who hit, whom, with what force, and on what tick. The Singleton doesn't — and shouldn't — know this.

> [!IMPORTANT]
> **Golden Rule:** Never call `dispatch_event` with an `event_tag` not registered in the Singleton with `Tag Type = EVENT`. It's equivalent to using a hardcoded StringName without validation.

---

## 3. ASUtils Structure Registry (Centralized Data)

All internal structures are centralized in `ASUtils` with proper API, documentation, and serialization support. This replaces scattered internal structs with a unified, documented system.

### 3.1 State Management Structures

- **ASStateCache:**
  - **Purpose**: High-performance circular buffer for rollback
  - **Features**: O(1) capture/restore, configurable size (default: 128), debug utilities
  - **Usage**: Multiplayer prediction and fast state restoration

- **ASStateCacheEntry:**
  - **Purpose**: Lightweight cache entry for single tick
  - **Data**: tick, attributes, tags, active_effects
  - **Methods**: to_dict(), from_dict(), validation

- **ASComponentState:**
  - **Purpose**: Complete component state for save/load
  - **Features**: Full historical buffers, cooldowns, diff computation
  - **Usage**: Save games, full serialization, network transfer

### 3.2 Effect System Structures

- **ASEffectState:**
  - **Purpose**: Active effect state representation
  - **Data**: tag, remaining_time, period_timer, stack_count, level
  - **Methods**: is_expired(), is_period_ready(), serialization

- **ASEffectModifier:**
  - **Purpose**: Single attribute modifier definition
  - **Data**: attribute, operation, magnitude
  - **Usage**: Effect resource definitions

- **ASEffectModifierData:**
  - **Purpose**: Runtime modifier with custom values
  - **Features**: Custom magnitude override support
  - **Usage**: ASEffectSpec runtime calculations

- **ASEffectRequirement:**
  - **Purpose**: Attribute requirement for activation
  - **Data**: attribute, amount
  - **Usage**: Effect activation conditions

### 3.3 Attribute System Structures

- **ASAttributeValue:**
  - **Purpose**: Base and current attribute values
  - **Features**: Difference calculation, value management
  - **Methods**: set_base(), set_current(), get_difference()

### 3.4 Cooldown System Structures

- **ASCooldownData:**
  - **Purpose**: Cooldown timing and associated tags
  - **Features**: Automatic update, group cooldown support
  - **Methods**: is_expired(), update(), serialization

### 3.5 Tag System Structures

- **ASEventTagData:**
  - **Purpose**: Complete event dispatch data
  - **Features**: Node references, payload, timing
  - **Methods**: get*instigator(), get_target(), set*\*()

- **ASEventTagHistoricalEntry:**
  - **Purpose**: Event occurrence history entry
  - **Data**: Complete ASEventTagData + tick
  - **Usage**: Event historical buffer

- **ASNameTagHistoricalEntry:**
  - **Purpose**: NAME tag change history entry
  - **Data**: tag_name, target_id, timestamp, tick_id, added flag
  - **Usage**: NAME tag historical buffer

- **ASConditionalTagHistoricalEntry:**
  - **Purpose**: CONDITIONAL tag change history entry
  - **Data**: tag_name, target_id, timestamp, tick_id, added flag
  - **Usage**: CONDITIONAL tag historical buffer

### 3.6 Universal Structure Features

All ASUtils structures implement:

- **Serialization**: `to_dict()` / `from_dict()` methods
- **Validation**: `is_valid()` and integrity checks
- **Helper Methods**: Type-specific convenience functions
- **Documentation**: Complete XML docs for Godot integration
- **Consistency**: Standardized API patterns across all structures

---

## 4. THE SINGLETON: ABILITY SYSTEM (PROJECT INTERFACE)

- **Role:** **Global Configuration API** and bridge to `ProjectSettings`.
- **Rules:**
  - Exclusively responsible for saving/loading the global tags list in `project.godot`.
  - Acts as a **Central Name Registry** ensuring resources don't conflict.
- **Limit:** Must not store state for any Actor.

---

## 5. TOOLS LAYER: EDITORS

Human interface to Resources.

### 5.1 ASEditorPlugin

- **Role:** **Bootloader**.
- **Rule:** Type registry, icons, sub-editors initialization. Forbidden from containing game logic.

### 5.2 ASTagsPanel

- **Role:** Visual interface for **Global Registry**.
- **Rule:** Exclusively manipulates the `AbilitySystem` Singleton tag dictionary.

### 5.3 ASInspectorPlugin (and Property Selectors)

- **Role:** Contextualization.
- **Rule:** Must provide smart selectors (tag dropdowns, attribute searches) for Inspector configuration.

### 5.4 The Compat Layer

Located in `src/compat/`.
The project is strictly architected under the **Dual Build Strategy**, supporting both Godot native Module and GDExtension compilation.

- **Role:** Shield the framework core from Module and GDExtension internal API divergences.
- **Rule:** All central logic (`src/core`, `src/resources`, `src/scene`) MUST be agnostic. Any structural difference needed to interact with the Engine (especially Editor GUI features) must be isolated in this folder, solving compatibility via `#ifdef ABILITY_SYSTEM_GDEXTENSION`.
- **Exclusivity:** Core files must never directly use GDExtension-exclusive classes that break Module C++ compilation; they must invoke the Wrapper in `compat/`.

---

## 6. THE BLUEPRINTS: RESOURCES (THE "WHAT")

Located in `src/resources/`. **Data Definitions**.

- **Resources (Blueprints):** Static (`.tres`) objects defining the "DNA". **Rule:** Must not be modified at runtime (except `ASStateSnapshot`).
- **Specs (Runtime Instances):** Lightweight instances holding dynamic state (cooldowns, stacks).
- **Snapshot Exception:** `ASStateSnapshot` breaks immutability for native persistence (Save/Load) and net caching.

### Usage & Performance Restrictions

> [!WARNING]
> **`ASStateSnapshot` is a heavy resource.** Capturing full state consumes significant CPU/RAM if misused.

1. **Player Exclusive:** Snapshots restricted to Playable Characters (where determinism/rollback are critical).
2. **NPCs/Enemies:** Non-playables **MUST NOT** use `ASStateSnapshot`.
3. **Golden Rule:** If it's playable, use `ASStateSnapshot`. If not, discard it.
4. **Reference Independence:** Stores primitive values and tag names, not pointers, ensuring safe serialization.

### 6.1 ASAbility & ASEffect (Actions & Modifiers)

- **ASAbility - Role:** Define action logic (Costs, Cooldown, Triggers).
- **ASAbility - Rule:** Only Resource capable of managing activation requirements and attribute costs. Supports **Ability Phases** for complex executions.
- **ASEffect - Role:** State modifier (Buffs, Debuffs, Damage).
- **ASEffect - Rule:** Defines stacking policies and attribute change magnitudes.

### 6.2 ASAttribute & ASAttributeSet (Attribute System)

- **ASAttribute - Role:** Represents metadata (min/max limits) of a single stat.
- **ASAttributeSet - Role:** Groups stats and defines an initial character state. Holds modification logic.
- **ASAttributeSet - Rule (Attribute Drivers):** Allows deriving base values from another (e.g., 2 \* Strength = 1 Attack). Recalculates automatically.
- **ASAttributeSet - Rule (Priority):** Modifiers (Flat, Multiplier) apply _after_ Driver calculation.

### 6.3 ASContainer & ASPackage (Archetypes & Payloads)

- **ASContainer - Role:** Complete archetype (Actor Identity Dictionary). Factory template for `ASComponent`.
- **ASPackage - Role:** Transport wrapper (Data Envelope). Used exclusively for `ASDelivery`.

### 6.4 ASCue (Visual Feedbacks)

- **Role:** Pure audiovisual feedback (Animation, Sound, Particles).
- **Rule:** Forbidden from altering gameplay data. Reactive dispatch only.

### 6.5 ASAbilityPhase (Complex Lifecycle)

Most powerful feature for designers, acting as embedded "State Machines" (Hierarchical Abilities).

- **Role:** Fragment rigid ability execution into granular stages (`Windup`, `Execution`, `Recovery`).
- **Nature:** Unlike "pistol" standard abilities, phased abilities act as "rituals" with timed steps.
- **Critical Rules:**
- **Temporary & Specific:** Each Phase can apply/remove transit `ASEffects` lasting only while the phase is active.
- **Duration or Event:** Advances via (a) expired duration; (b) _Transition Trigger Event_ occurrence (e.g., waiting for an Animation Node `.Hit` Event).
- **Autonomous Alerts:** Phase transitions always fire an automatic AS Event for UI/systems fluency.

---

## 7. THE EXECUTORS: SPECS (THE "HOW")

Located in `src/core/`. Where execution state and logic reside.

- **Role:** Represent the **Active Instance**. Owner of the **"Now"**.
- **Golden Rule: STATE SOVEREIGNTY.**
- **What lives here (not in Component):**
  - `duration_remaining`: Individual timers.
  - `stack_count`: Accumulation amount.
  - `calculate_...`: Logic dependent on variable attributes.

### 7.1 ASAbilitySpec & ASEffectSpec (Execution)

- **ASAbilitySpec:** Active/equipped ability handling cooldowns.
- **ASEffectSpec:** Active modifier holding sovereignty over `duration` and `stacks`.

### 7.2 ASCueSpec & ASTagSpec (Feedback & Identity)

- **ASCueSpec:** Manages scene feedback lifespan. Ensures Queue Free on end.
- **ASTagSpec:** Tag Reference Counter (Refcount). Guarantees tag leaves only when all origin Specs expire.

### 7.3 ASAbilitySpec (Phase Management)

- **Role:** Manages current phase index and timeline progression.
- **Rule:** Validates advancing via time ticks or receiving a specific `ASEventTag`.

---

## 8. THE ORCHESTRATOR: COMPONENT (THE HUB)

The `ASComponent` (ASC).

- **Role:** **Collection Manager** and **Signal Router**.
- **Rules:**
  - Does not manage individual timers (Spec does).
  - Holds `active_specs` and `unlocked_specs`.
  - Is the **Attribute Owner** (via `AttributeSet`).
  - Is the only entity capable of adding/removing tags on the Actor.
- **Determinism Guarantee:** Must rollback and predict state via `ASStateSnapshot`.
- **State Cache:** Maintains internal buffer of snapshots for net sync.
- **Individual Historical Buffers:**
  - **NAME History**: `_name_history` buffer for persistent state changes
  - **CONDITIONAL History**: `_cond_history` buffer for permission/immunity changes
  - **EVENT History**: `_event_history` buffer for event occurrences
  - Each buffer maintains 128 entries with automatic overflow management
  - Historical APIs provide specialized queries for each tag type
  - Unified history API for cross-type queries and debugging

---

## 9. DELIVERY & REACTIVITY SYSTEMS

### 9.1 ASDelivery (Payload Injections)

- **Role:** Decouples emitter from target in spatial interactions (AoEs, Projectiles).
- **Rule:** Transports `ASPackage` and injects upon colliding with ASC.

### 9.2 Ability Triggers (Reactive Automation)

- **Role:** Automatic ability activation based on State (Tags) or transient events (AS Events).
- **Rule:** Exclusively based on `ON_TAG_ADDED`, `ON_TAG_REMOVED`, or `ON_EVENT_RECEIVED`.

---

## 10. REPLICATION & PERSISTENCE (DETERMINISM)

Authoritative multiplayer with Prediction/Rollback support.

- **Truth Source (Physics Only):** Only `tick` is a valid temporal identifier. `ASComponent` operates **exclusively** via `physics_process`. Hard block on `_process`.

### 10.1 ASStateSnapshot (Heavy Resource)

- Heap-allocated **Godot Resource** (`.tres`). Supports native serialization for long-term Save/Load.
- Player-exclusive usage.

### 10.2 ASStateCache (Light Structure)

- **Pure C++ Struct** allocated on inline stack/vector in `ASUtils`.
- Fast, zero-overhead sync for non-playables/NPCs.
- Circular buffer with configurable size (default: 128 entries).
- O(1) capture and restore operations for high-performance rollback.
- Automatic buffer management with overwrite protection.
- Debug utilities and serialization support.

### 10.3 ASComponentState (Complete State)

- **Comprehensive structure** in `ASUtils` for full component serialization.
- Includes all historical buffers, cooldowns, and complete state data.
- Used for save/load operations and full state transfer.
- Supports diff computation for bandwidth optimization.
- Binary serialization options for network efficiency.

### 10.4 Net Activation Flow

1. **Request:** Client requests via `request_activate_ability(tag)`.
2. **Predict:** Client executes local prediction, generates `cache_buffer` entry.
3. **Confirm/Correct:** Server validates. If diverging, Server sends authoritative state, Client **Rollbacks** to corresponding tick.
4. **Determinism:** Gameplay logic MUST heavily rely purely on ASC and Spec properties.

---

## 11. THE REACTIVITY & FLOW PROTOCOL (TRICHROMATIC MATRIX)

Ensures the 3 Pillars operate in orchestrated unison without spaghetti coupling.

### 11.1 The Natural Order

1. **INPUT/ACTION:** Interaction, timeout, or physical impact emits an **AS Event** (`Event.Damage`).
2. **LISTEN/PROCESS:** Entity listens via Triggers (`ON_EVENT`).
3. **MUTATION:** Reactive ability meets requirements, invokes and applies modifier (`ASEffect`).
4. **STATE (Cycle End):** Effect mutates attributes or permanently adds an **AS Tag** (`State.Stunned`).

> [!CAUTION]
> **Fatal Errors punished by deep refactoring:**
>
> - Waiting for an ability to start based on "tag loss" (acception coupling; emit an Event instead).
> - If an Ability fails requirements, NEVER apply temporary hack tags. Emit `Event.Ability.Failed` passing the reason (Payload Dict).

### 11.2 Hybrid Era Triggers

- `TRIGGER_ON_EVENT`: Gold standard for combat responsiveness (Instant Reaction).
- `TRIGGER_ON_TAG_ADDED/REMOVED`: Standard for environmental automation (e.g., Slow aura on entering water).

### 11.3 The ASDelivery Pact

- **Obligation:** Every ASDelivery successfully finishing its route MUST emit `target_asc->dispatch_event(package_tag)`. Fills the Historical buffer, enabling clean reactive blocks.

---

---

## 12. AI / BEHAVIOR TREE INTEGRATION (ASBridge)

To interact with the LimboAI framework for Behavior Trees and State Machines, the Ability System provides an explicit bridge component rather than embedding AI concepts into core gameplay code.

- **Role:** Exposes Ability System queries (e.g., `has_tag`, requirement evaluation) directly to LimboAI Blackboard variables and triggers actions via custom `BTNode` or states.
- **Rule:** The `ASBridge` evaluates states strictly using the `ASComponent` APIs. It NEVER bypasses standard rules, costs, or cooldowns.
- **Strict Decoupling:** Behavior Tree tasks NEVER use `try_activate_ability` directly on their own context natively. They dispatch commands through the bridge (or trigger Events), enforcing the **Reactivity Protocol**. This absolute isolation ensures the core Ability System logic is 100% agnostic to any BT/HSM node.

---

## 13. TECHNICAL RIGOR & TEST QUALITY

### 13.1 The 300% Standard (Iron Law)

Every feature proved by at least **3 variations** in the same test:

1. **Happy Path:** Ideal base scenario.
2. **Negative:** Invalid input or expected fail.
3. **Edge Case:** Complex combinations (multi-tags, bounds).

### 13.2 Test Suites

- **Core:** Atomic, no side-effects.
- **Advanced:** Periodic DoT/HoT, complex RPG flows.
- **Multiplayer:** Executed via `runner.py` with injected latency.

---

## 14. API NAMING PATTERNS

### 14.1 Method Prefixes

- **🎮 Gameplay Layer (Game Logic Usage)**
  - `try_activate_...`: **Safe Execution.** Validates requirements then invokes.
  - `can_...`: **Pre-authorization.** Answers if feasible.
  - `is_...`, `has_...`: Query statuses mapping.
  - `get_...`, `cancel_...`, `request_...` (Net intention).
- **🏗️ Internal/Infrastructure Layer**
  - `apply_...`: Forced application ignoring capabilities (for Init or ASDelivery).
  - `add_...` / `remove_...`: Low-level collections mutator. (Do not use for gameplay logic).
  - `unlock_...` / `lock_...`, `register_...`, `rename_...`, `set_...`, `clear_...`, `capture_...`.

> [!IMPORTANT]
> **There is no direct `activate` in the public API.** `try_...` is the only safe gameplay path.

### 14.2 View Suffixes

- `..._by_tag` vs `..._by_resource`
- `..._to_self` vs `..._to_target`
- `..._debug`, `..._preview`, `..._all`.

### 14.3 Types & Parameters

- `StringName` (tags), `float` (magnitude/levels), `Dictionary` (Custom Payloads).
- Order: Identifier (`tag`) > Magnitude (`level`) > Context (`target_node`).

### 14.4 Signals and C++ Properties

- **Signals:** Passive voice past-tense (`ability_activated`, NOT `on_ability_activate`).
- **Internal C++:** Privates prefixed with `_`. Reactive Setters required over direct modifications.

---

## 15. DESIGN PATTERNS

- **Spec Pattern:** Separation between Data Blueprint (Resource) and Runtime Instance (Spec).
- **Flyweight:** ASC holds pointers to Resources; heavy static data is never duplicated in RAM.
- **Command:** Each ability is a self-contained Command.
- **Data-Driven:** Configured via `.tres`, zero hardcoded values.

---

## 16. TEST PATTERNS

- **Deterministic Ticking:** Tested strictly via `tick(delta)`.
- **Isolation/Mocking:** Temporary resources created in memory.
- **State Injection:** Injecting an expiring Spec instead of waiting for full timer loops.
- **Signal Auditing:** Every activation test must audit if the canonical Signal accurately fired.
