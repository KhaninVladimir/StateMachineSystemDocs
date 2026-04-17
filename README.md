# JP State Machine

`JP State Machine` is an Unreal Engine plugin for building gameplay state machines around `GameplayTag`, reusable state classes, reusable transition conditions, and a dedicated graph editor.

The plugin is split into two modules:
- `JPStateMachine` - runtime
- `JPStateMachineEditor` - graph editor, compile validation, asset creation, live debug, and editor tooling

## Overview

The system is built around four main concepts:
- `UJPBaseState` - runtime behavior for a single state
- `UJPStateCondition` - rule that decides whether a transition may happen
- `UJPStateMachineGraph` - asset that stores nodes, transitions, compiled data, and editor preview data
- `UJPStateSystemComponent` - actor component that runs the machine at runtime

The typical workflow looks like this:
1. Create one or more `State` assets/classes.
2. Create one or more `Condition` assets/classes.
3. Create a `State Machine Editor` asset.
4. Assemble the graph in the editor.
5. Compile the graph.
6. Assign the graph to `UJPStateSystemComponent` on an actor.
7. Run `PIE` or `Simulate` and debug the active state directly in the graph editor.

## Content Browser

The plugin adds a custom category:

`Add New -> State Machine`

Available entries:
- `State Machine Editor`
- `State`
- `Condition`

These are the main authoring assets used by the plugin.

## Core Runtime Types

### UJPBaseState

`UJPBaseState` is the base class for gameplay logic of a single state.

Main responsibilities:
- stores the `StateTag`
- optionally ticks every frame
- reacts to enter, exit, and tick lifecycle events
- exposes banned gameplay flags for external checks

Key properties:
- `StateTag : FGameplayTag`
- `BannedFlags : FGameplayTagContainer`
- `bTickEnabled : bool`

Key Blueprint events:
- `OnEnter`
- `OnExit`
- `OnTick`

Useful runtime getters:
- `GetStateTag()`
- `GetTimeInState()`
- `GetStateComponent()`
- `GetOwnerActor()`
- `GetBannedFlags()`
- `IsFlagBanned(...)`

### UJPStateCondition

`UJPStateCondition` is the base class for transition checks.

Main entry point:
- `Evaluate(AActor* OwnerActor, UJPStateSystemComponent* StateComponent, UJPBaseState* CurrentState, UJPBaseState* NextState)`

Blueprint override:
- `K2_Evaluate(...)`

Conditions are reusable classes. The graph decides which condition class is used on each transition.

### UJPStateSystemComponent

`UJPStateSystemComponent` is the runtime executor of the graph.

Main responsibilities:
- builds runtime copies of states from the graph
- enters the initial state
- ticks the active state
- evaluates transitions
- switches states
- supports sub-state graphs
- replicates the authoritative state from server to clients
- exposes debug data for the graph editor

Useful API:
- `StartStateMachine()`
- `StopStateMachine()`
- `SwitchStateByTag(...)`
- `IsStateMachineRunning()`
- `GetCurrentState()`
- `GetCurrentStateTag()`
- `GetCurrentStateTime()`
- `GetCurrentLeafState()`
- `GetCurrentLeafStateTag()`
- `GetCurrentLeafStateTime()`
- `FindStateByTag(...)`
- `GetBannedFlags()`
- `IsFlagBanned(...)`

Delegate:
- `OnStateChanged`

## Graph Asset

`UJPStateMachineGraph` is the asset edited in the custom graph editor.

It stores:
- `InitialStateTag`
- graph nodes
- transition edges
- compiled transitions
- editor graph data
- live preview/debug state

At compile time the graph is validated and converted into runtime transition data.

## Graph Nodes

The graph currently supports two node types.

### State Node

State nodes are represented by `UJPStateGraphState`.

They contain:
- `StateClass`
- `StateTemplate`

`StateTemplate` is an instanced template object created from the chosen `StateClass`. This allows per-node overrides of state parameters directly inside the graph editor.

Example:
- `UJPCrouchState` may contain `CrouchSpeed`
- default value can live in the class
- a specific graph node can override `CrouchSpeed` through its `StateTemplate`

### SubTree Node

Sub-tree nodes are represented by `UJPStateGraphSubTree`.

They contain:
- `SubTreeTag`
- `SubStateGraph`

Use a `SubTree` when you want one node on the outer graph to delegate logic to another state machine.

Typical use case:
- top level graph contains `Movement` as a `SubTree`
- movement subgraph contains `Idle`, `Walk`, `Sprint`, `Crouch`
- top level graph also contains `Death`
- when death should happen, the outer graph can transition from `Movement` to `Death` without wiring every internal movement state directly to `Death`

## Transition Rules

Transitions are stored as `UJPStateGraphTransitionEdge`.

Each transition contains:
- source node
- target node
- `ConditionClass`
- `bInvertResult`
- optional rule graph data used by the editor

Important behavior:
- `bInvertResult` belongs to the graph transition, not to the condition class
- the same `ConditionClass` may be reused more than once
- `Condition + Invert=false` and `Condition + Invert=true` are treated as different rules

The editor shows inverted rules with a small `!` badge on the transition bubble.

## Compile Validation

Use the `Compile` button in the graph editor to validate the asset and rebuild compiled data.

Validation checks include:
- missing `StateClass` on a state node
- invalid or duplicate state tags
- missing `ConditionClass` on a transition
- broken source or target links
- invalid sub-tree setup
- duplicate transition/rule usage in invalid local cases

Duplicate rule validation is local, not global. An error is reported only when:
- two identical rules leave the same source node
- two fully identical transitions connect the same source and target nodes

Compile feedback is shown in three places:
- Output Log
- node highlight state
- inline error text directly under the invalid state or rule

## State Parameters in the Graph

State nodes support per-node parameter overrides through `StateTemplate`.

This is useful when:
- several nodes use the same `StateClass`
- but each node needs different values

Example:
- `UJPCrouchState` has `CrouchSpeed`
- `CrouchLow` node can use one value
- `CrouchCombat` node can use another value

The runtime duplicates the node template, so the gameplay instance is based on the graph node configuration, not only on the class defaults.

## Runtime Execution Flow

When the machine starts:
1. The component builds runtime copies from the graph.
2. State instances are initialized with the owning component.
3. The initial node is entered.
4. Every tick:
   - the current state is ticked if ticking is enabled
   - top-level transitions are evaluated
   - if the current node is a `SubTree`, the nested graph is also evaluated
   - the first valid transition wins

Transition order matters.

## SubTree Behavior

`SubTree` is intended for hierarchical graphs.

Behavior summary:
- the outer graph owns the high-level flow
- while a `SubTree` node is active, a nested state machine runs inside it
- the outer graph still has a chance to exit the subtree
- if no outer transition is taken, internal transitions of the subtree continue running

This keeps the high-level graph readable and avoids repetitive global transitions from every low-level state.

## Networking and Replication

`UJPStateSystemComponent` supports authoritative replication of the active state.

Behavior:
- the server is the source of truth
- the active state tag is replicated to clients
- standalone, listen server, and dedicated server flows are supported
- clients do not re-apply state changes if the replicated state is already current

This is designed to keep runtime behavior stable without redundant client-side re-entry logic.

## Banned Flags

Each state can expose a set of banned gameplay flags through `BannedFlags`.

This allows external systems to query whether an action is currently blocked by the active state.

Example:

```cpp
if (!StateSystemComponent->IsFlagBanned(JumpBanTag))
{
    Jump();
}
```

This is useful for checks such as:
- jump disabled during a heavy attack
- sprint disabled while reloading
- aim disabled during a cinematic state

## Live Debugging

The editor supports live preview during `PIE` and `Simulate`.

Features:
- toolbar selector with matching `UJPStateSystemComponent` instances from running worlds
- support for standalone, listen server, dedicated server, and client PIE worlds
- active node highlighted in dark green
- time spent in the active state shown near the node
- recent transition highlighted on the graph
- current state and timer shown in the top toolbar

The debug selector is intended to let you inspect the same graph on different actors and on different network worlds from the same editor UI.

## Editor Shortcuts and Quality of Life

Implemented editor behavior includes:
- double click state node opens its `StateClass`
- double click transition opens its `ConditionClass`
- `Ctrl+Z / Ctrl+Shift+Z` for create, delete, and move operations
- copy/paste and duplicate for selected graph nodes and connections
- copy/paste and duplicate for rule graph nodes
- hidden main graph tab with a clean graph area
- graph entry still available from the `Window` menu

## Recommended Workflow

1. Create your `State` classes first.
2. Keep each state focused on one clear gameplay responsibility.
3. Use `BannedFlags` for cheap action gating.
4. Use `SubTree` for large feature groups such as movement, combat modes, or vehicle behavior.
5. Keep `Condition` classes small and reusable.
6. Compile often while editing.
7. Verify runtime behavior with live debug in `PIE`.

## Plugin Metadata

- Friendly name: `State Machine System`
- Category: `Gameplay`
- Modules: `JPStateMachine`, `JPStateMachineEditor`
