# Editor cell dependency tracking and concept-instance queries

Verified against MPS 2025.1.4; the files cited below are identical in MPS 2023.2.3 and on the 2026.1 branch (2026.1.1
plus 240 commits, `50652adb`). How does the MPS editor record which model reads a cell depends on, how does it match
model changes against those records, and does a cell computed from `model.nodes(Concept)` (the fast node finder) get
refreshed when a matching node is added?

## Short answer

- The editor records dependencies per **node cell** ("big cell"), not per individual cell. While a node's editor is
  built, a fresh `NodeReadAccessInEditorListener` collects every node read, property read and reference-target pointer
  reported through `NodeReadAccessCasterInEditor`. The node itself is always added. Dependencies of nested node cells
  are merged into every enclosing node cell.
- After each command, the editor rebuilds incrementally. Each node cell whose recorded dependencies contain a modified
  node is recreated; every other node cell is reused. There is no fallback to a full rebuild for a change that matches
  no dependency: the old cells are simply kept.
- `model.nodes(C)` records **no** dependency. Building the concept index runs with read events blocked, and the only
  event the finder fires itself, the "model nodes read" event, goes to a different caster. Every `modelNodesReadAccess`
  implementation in MPS is a no-op.
- So a cell computed from `model.nodes(C)` can stay stale when a new `C` node is added. Whether it refreshes depends on
  coincidental dependencies: the parent of the added node must be in the dependencies of the cell's node cell. Root
  additions never match. Removals match when the cell read the removed node, for example its name.
- Reference cells stay fresh through recorded target pointers, not through any model-wide rule. Scopes are evaluated
  only when a completion menu is built, so they are not part of the rendered state.

## Two read-event casters

`jetbrains.mps.smodel` (in `lib/mps-core.jar`) has two independent thread-local casters:

| Caster                         | Listener type                            | Events                                                                     | Used by                                           |
| ------------------------------ | ---------------------------------------- | -------------------------------------------------------------------------- | ------------------------------------------------- |
| `NodeReadAccessCasterInEditor` | `NodeReadAccessInEditorListener` (stack) | node read, property read (dirty/clean/existence), reference-target pointer | editor cell building                              |
| `NodeReadEventsCaster`         | `INodesReadListener` (stack)             | unclassified node read, property read, referent read, **model nodes read** | typesystem, `DependencyRecorder`, language errors |

**Verified:** `AttachedNodeOwner.fireNodeRead` and `firePropertyRead` notify both casters
(`core/kernel/source/jetbrains/mps/smodel/AttachedNodeOwner.java:193-196`, `:216-217`). `StaticReference` reports the
target pointer (model reference plus node id) to the editor caster before it tries to resolve the target
(`core/smodel/source/jetbrains/mps/smodel/StaticReference.java:117-121`). `SModel.getRootNodes()` fires
`NodeReadEventsCaster.fireModelNodesReadAccess` only (`core/kernel/source/jetbrains/mps/smodel/SModel.java:161-162`,
`:315-321`). The editor caster has no model-wide event.

**Verified:** `NodeReadAccessCasterInEditor.runReadTransparentAction` sets the blocked flag on **both** casters for the
duration of the action and restores the previous state afterwards (`NodeReadAccessCasterInEditor.java:81-105`). Both
casters drop all events while blocked, including `fireModelNodesReadAccess` (`NodeReadEventsCaster.java:136-139`). It
does not affect `ModelEventDispatch` access listeners, which `AttachedNodeOwner` notifies before checking the casters
(`AttachedNodeOwner.java:185-188`).

**Verified:** `NodeReadAccessInEditorListener` stores node reads in `myNodesToDependOn`, reference targets in
`myReferentTargetsToDependOn`, and dirty and existence property reads as `(SNodeReference, name)` pairs. Dirty and
existence property reads also add the node
(`core/smodel/source/jetbrains/mps/smodel/NodeReadAccessInEditorListener.java:28-33`, `:72-107`). Its
`modelNodesReadAccess` is `/* ignored */` (`:89-91`). The other implementations, `AbstractNodesReadListener`
(`core/kernel/source/jetbrains/mps/smodel/AbstractNodesReadListener.java:38`) and
`NodeReadEventsCaster.NullNodesReadListener`, are also empty. No class in the MPS sources overrides them.

## Recording dependencies while building cells

`jetbrains.mps.nodeEditor.EditorManager` (`editor/editor-runtime/source/jetbrains/mps/nodeEditor/EditorManager.java`)
creates each node cell in `createEditorCell_internal`:

```text
createEditorCell_internal(refContext)                 :374
  push new NodeReadAccessInEditorListener             :383-385
  cellFactory.createEditorCell(node)                  :386   all cell providers of this node's editor run here;
                                                             child node cells recurse via updateChildNodeCell
  listener.nodeUnclassifiedReadAccess(node)           :414   the node always depends on itself
  removeCellBuildNodeAccessListener()                 :415   merges this listener's nodes and ref targets
                                                             into the enclosing node's listener
  addNodeDependenciesToEditor(cell, listener)         :423 -> UpdateSession.registerDependencies(...)  :496-504
```

**Verified:** the pop merges the child listener's nodes and reference targets into the parent listener
(`NodeReadAccessCasterInEditor.java:143-150`). When an old child cell is reused, `EditorManager` adds its stored
dependencies to the parent listener to produce the same effect (`EditorManager.java:225-243`). As a result, the root
node cell depends on the union of all reads in the editor. `syncEditorCell` applies the same steps to synchronizable
cells (`:317-372`).

**Verified:** `UpdaterImpl` keeps the results in weak maps keyed by node cell: `myRelatedNodes`, `myRelatedRefTargets`,
and the dirty, clean and existence property maps
(`editor/editor-runtime/source/jetbrains/mps/nodeEditor/updater/UpdaterImpl.java:64-69`). A read during any cell
provider, including a custom provider that queries the model, is attributed to the node cell being built.

## Matching model changes

**Verified:** after every update, `UpdaterModelListenersController.attachListeners` subscribes to the models that
contain a recorded node or a resolvable recorded reference target, and unsubscribes from the others
(`updater/UpdaterModelListenersController.java:44-83`; called from `UpdaterImpl.java:137`). The edited node's model is
always included because the root node depends on itself.

**Verified:** `UpdaterModelListener` extends `jetbrains.mps.smodel.ModelsEventsCollector`. The collector buffers node
add, remove, property and reference events during a command and delivers them when the command finishes
(`core/kernel/source/jetbrains/mps/smodel/ModelsEventsCollector.java:61-68`, `:122-140`). `NodeChangeBridge` turns a
node addition without a parent into `SModelRootEvent` and one with a parent into `SModelChildEvent`
(`core/smodel/source/jetbrains/mps/smodel/event/NodeChangeBridge.java:87-108`).

`UpdaterModelListener.eventsHappened` (`updater/UpdaterModelListener.java:36-50`):

1. If the batch is a single property change that no cell read "dirtily" (and is not an add or remove of a property whose
   existence was checked), it only synchronizes the property cells (`UpdaterImpl.requiresUpdate`, `:342-344`).
2. Otherwise it calls `UpdaterImpl.update(events)`, which runs a new update session with the events
   (`UpdaterImpl.java:94-117`).

**Verified:** `SModelModificationsCollector` converts the events into `(SNode, SNodeReference)` modifications
(`editor/editor-runtime/source/jetbrains/mps/nodeEditor/SModelModificationsCollector.java:61-100`):

| Event               | Modified nodes                                     |
| ------------------- | -------------------------------------------------- |
| root added/removed  | the root and all its descendants                   |
| child added/removed | the parent, plus the child and all its descendants |
| reference changed   | the source node                                    |
| property changed    | the node                                           |
| import/language     | none                                               |

**Verified:** `EditorManager.isNodeChanged` treats a node cell as changed when its node set contains the modified node
or its reference-target set contains the modification's pointer (`EditorManager.java:299-311`, `UpdaterImpl.isRelated`
at `UpdaterImpl.java:317-326`). An unchanged cell is returned as is (`:224-246`). A changed cell is rebuilt; its
children are looked up again and reused when unrelated (`:248-259`).

**Verified:** a full rebuild happens only when the update runs with `events == null`, because modifications are then
`null` and no old cell is consulted (`EditorManager.java:220`; `UpdateSessionImpl.java:86`). That path is
`Updater.update()` (`UpdaterImpl.java:84-92`), reached from `EditorComponent.rebuildEditorContent()`
(`editor/editor-runtime/source/jetbrains/mps/nodeEditor/EditorComponent.java:2066-2080`). Callers include:

- model replaced or unloaded (`updater/UpdaterRepositoryContentAdapter.java:58-74`);
- editor settings changes (`EditorComponent.java:268-276`);
- language reload (`workbench/mps-editor/source/jetbrains/mps/ide/editor/EditorComponentTracker.java:116-133`); and
- explicit actions such as hint changes.

An ordinary model event never triggers one. **Verified:** `EditorComponent` and `NodeEditorComponent` register no other
model or node-change listeners.

## Concept-instance queries record nothing

**Verified:** `SModelOperations.nodes(model, concept)` in the smodel language runtime
(`core/kernel/smodelRuntime/source_gen/jetbrains/mps/lang/smodel/generator/smodelAdapter/SModelOperations.java:87-93`)
delegates to `jetbrains.mps.smodel.SModelOperations.getNodes`. That method calls
`FastNodeFinderManager.get(model).getNodes(concept, true)`
(`core/kernel/source/jetbrains/mps/smodel/SModelOperations.java:123-125`). Regular models implement
`FastNodeFinder.Factory` through `SModelDescriptorStub` and create a `DefaultFastNodeFinder`
(`core/kernel/source/jetbrains/mps/extapi/model/SModelDescriptorStub.java:305-308`, `SModel.java:604-606`).

**Verified:** `BaseFastNodeFinder.getNodes` (`core/kernel/source/jetbrains/mps/smodel/BaseFastNodeFinder.java:103-118`):

- calls `myModel.getRootNodes().iterator()` without iterating, commented "notify 'model nodes read access'". This fires
  only `NodeReadEventsCaster.fireModelNodesReadAccess`, which every listener ignores;
- answers from the cached map when it is populated, with no read events at all;
- otherwise builds the map through `build(...)`, which `DefaultFastNodeFinder` wraps in `runReadTransparentAction`. That
  blocks both casters (`core/kernel/source/jetbrains/mps/smodel/DefaultFastNodeFinder.java:47-49`). Incremental index
  updates on node add or remove use the same wrapper (`BaseFastNodeFinder.java:64-85`).

**Verified:** therefore `model.nodes(C)` contributes nothing to the node cell's dependencies. The cell gains
dependencies only from what its code reads afterwards. For example, reading each returned node's `name` adds that node
and a dirty `(node, name)` pair.

**Inferred** (from the code paths above, not observed at runtime): suppose a node cell `N` contains a cell computed from
`model.nodes(C)`, and a new `C` node is added to the model.

| Where the new node is added                        | Cell recomputed?                                                                                                                     |
| -------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| as a new root of the same model                    | No. The modifications are the new subtree only, which no cell has read.                                                              |
| under a node `P` shown in the same editor          | Only if `N` is `P`'s node cell or an ancestor of it (dependencies propagate upward), or `N` read `P`. A sibling node cell is reused. |
| under a node `P` in another root of the same model | Only if `N` happened to read `P`, for example via `getParent()` on a returned node.                                                  |
| in another model                                   | Only if the editor listens to that model (it read some node there) and `P` is in `N`'s dependencies. Root additions never match.     |
| removal of a node the cell displayed by reading it | Yes. The removed node is in the modifications and in `N`'s dependencies.                                                             |

A cell in the edited root's own node cell is recomputed on any structural change to nodes displayed in that editor,
because the root node cell depends on every displayed node. This makes the gap easy to miss in a single-root test. A
full rebuild (reopening the editor, model reload, or a settings change) always corrects the cell.

There is no rule that rebuilds node cells because the changed node lies inside the edited root. Matching is by node
identity against the recorded set. The rule that makes a node cell depend on its own node (`EditorManager.java:397-414`)
covers structural changes to displayed nodes, not to arbitrary nodes in the model.

## Reference cells and completion

**Verified:** `SReferenceCellProvider.createCell` resolves the reference
(`editor/editor-runtime/source/jetbrains/mps/lang/editor/cellProviders/SReferenceCellProvider.java:77-91`).
`StaticReference.getTargetNode_internal` reports the target pointer even when the target is missing. The cell therefore
depends on the pointer and, through presentation reads, on the target node. The target model is subscribed when it
resolves. Renaming, deleting, or re-adding the target with the same id produces a modification whose node or pointer
matches, so the reference cell is rebuilt.

**Verified:** a reference scope is evaluated in `ReferenceScopeSubstituteMenuPart.doCreateItems`
(`editor/editor-runtime/source/jetbrains/mps/lang/editor/menus/substitute/ReferenceScopeSubstituteMenuPart.java:60-92`),
which builds completion items. **Inferred:** menu parts run when a completion or substitute menu is requested, so their
results are computed fresh and are not tracked as cell dependencies. Rendering a reference cell does not consult the
scope. Neither rule "a reference cell depends on the whole model" nor "rebuild fully on an unmatched event" exists in
the editor updater.

## Consequences for caches that reuse this pattern

A cached computation that records reads through `NodeReadEventsCaster` or `NodeReadAccessCasterInEditor` has the same
blind spot for `FastNodeFinder` queries. The finder suppresses reads while it builds or updates its index. Its only
model-level signal is `modelNodesReadAccess` on `NodeReadEventsCaster`, which a listener could handle, but no MPS
listener does. A cache that needs to observe concept-instance additions must handle `modelNodesReadAccess` as a
dependency on the whole model, or subscribe to node additions in the queried models itself.

## Unresolved

- No runtime probe confirmed the stale-cell scenarios above. They follow from the cited code but have not been observed.
- Third-party code may call `rebuildEditorContent()` on events of its own. That is outside the scope of this note.
