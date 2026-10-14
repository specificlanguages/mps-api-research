# Cached concept hierarchy queries

Verified against MPS 2024.1.6, 2025.1.4, and 2026.1.1: which API returns direct or transitive concept descendants,
including interface implementors, without enumerating every loaded language on each query?

## API and result contract

Use `jetbrains.mps.smodel.ConceptDescendantsCache`, supplied by `lib/mps-core.jar` in the MPS 2026.1.1 distribution. Its
source file is identical in the three release tags inspected.

```java
static ConceptDescendantsCache getInstance()
Set<SAbstractConcept> getDescendants(SAbstractConcept concept)
Set<SAbstractConcept> getDirectDescendants(SAbstractConcept concept)
```

**Verified:** `getDescendants` returns a fresh mutable `LinkedHashSet` containing the supplied concept and all reachable
(descendant) concepts. Even an unknown concept has a singleton result. Traversal uses a visited set, so repeated paths
and cycles do not produce duplicate results. Its Javadoc guarantees stable iteration order, not alphabetical ordering.
Filter out the supplied concept when requesting strict descendants, and sort explicitly when alphabetical output is
required.

**Verified:** `getDirectDescendants` returns the cached immediate children, or an empty set. Nonempty cached sets are
unmodifiable and replaced when edges change; do not mutate the returned set or retain it as a live view of future
runtime changes. The query does not add the supplied concept.

The cached edges use `ConceptDescriptor.getParentsIds()`. Its contract includes the immediate superclass and directly
extended or implemented interfaces. The cache therefore supports descendants of both class concepts and interface
concepts. It includes abstract concepts and interfaces. To restrict results to concrete concepts, filter using
`!concept.isAbstract()`; `SAbstractConcept.isAbstract()` explicitly covers interfaces as well as abstract concepts.

```text
getDescendants / getDirectDescendants
  process newly loaded language runtimes
    structure descriptors
      immediate parent IDs -> cached child edges
  transitive traversal / direct edge lookup
```

## Lifecycle and scope

**Verified:** `MPSCore.init()` creates and initializes the cache after the concept and language registries. `init()`
sets the singleton and registers a language registry listener; `dispose()` removes the listener and clears the
singleton. Obtain it only while the MPS core is initialized. `getInstance()` has no nullability annotation and returns
the singleton field directly; it does not create a cache on demand.

**Verified:** language load events queue runtimes for lazy processing by the next query. The structure aspect's
`getDescriptors()` supplies the concepts; an absent structure aspect contributes no concepts. Language unload events
remove pending runtimes or remove already processed concepts' parent edges. This is runtime metadata, not an inspection
of unbuilt language source models, and it has no repository/model search-scope parameter.

Each query processes pending runtimes before answering. Callers should reuse the platform singleton and let its registry
listener maintain the cache rather than build a second hierarchy index.

## Access and threading

**Verified:** the query methods do not assert a model-access or EDT requirement. Pending-runtime processing synchronizes
on its runtime queue; the child-edge map is concurrent and its values are immutable sets. These implementation details
do not establish that arbitrary calls racing language unload are safe.

**Inferred usage guidance:** call under repository read access while the owning platform is alive, so language runtime
changes cannot interleave with the query. MPS's own `BaseFastNodeFinder` and `InstancesSearchType` use the singleton to
expand concept-instance searches. No EDT dispatch is present in the cache's query path.

**Unresolved:** a general contract for lock-free callers concurrent with runtime unloading is not established here. The
absence of an access assertion is not evidence of such a guarantee.

## Evidence

The following paths are relative to the JetBrains/MPS repository. Cache coordinates apply to all three inspected tags;
other coordinates refer to the 2026.1.1 tag.

- `core/kernel/source/jetbrains/mps/smodel/ConceptDescendantsCache.java`: registry listener (57–75), parent-edge
  construction/removal (103–118), immutable cached sets (123–145), lifecycle (154–172), queries and traversal (181–223).
- `core/kernel/source/jetbrains/mps/smodel/runtime/ConceptDescriptor.java`: immediate parent contract (64–68).
- `core/openapi/source/org/jetbrains/mps/openapi/language/SAbstractConcept.java`: `isAbstract()` (66–69).
- `core/kernel/source/jetbrains/mps/core/platform/MPSCore.java`: cache initialization (157).
- `core/kernel/source/jetbrains/mps/smodel/BaseFastNodeFinder.java`: inclusive instance search expansion (123).
- `core/findUsages-runtime/source_gen/jetbrains/mps/findUsages/InstancesSearchType.java`: query expansion (39).
- `core/kernel/smodelRuntime/source_gen/jetbrains/mps/lang/smodel/generator/smodelAdapter/SConceptOperations.java`:
  `getAllSubConcepts(SAbstractConcept, Set<SLanguage>)` (91–104) calls the same cache, then filters by available
  languages.
