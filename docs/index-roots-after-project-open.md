# When a freshly opened project's model files become index roots

This note answers whether a full index rescan requested immediately after opening an MPS project in a headless
IDEA-based environment covers the project's model files, and why the answer differs between MPS versions. Verified
against MPS **2024.1.6**, **2025.1.4**, **2025.2.4** and **2026.1.1** sources, and the patched IntelliJ Platform sources
of 2024.1.6 (MPS-241.19416.417), 2025.1.4 (MPS-251.28774.587) and 2026.1.1 (MPS-261.25134.711). No runtime probe was
run.

## Answer

| MPS version            | Roots provided by                                        | When the roots are computed                                                              | Rescan right after open covers the project? |
| ---------------------- | -------------------------------------------------------- | ---------------------------------------------------------------------------------------- | ------------------------------------------- |
| 2024.1, 2025.1         | `MPSIndexableSetContributor` → `IndexableRootCalculator` | Synchronously, in the caller's thread, whenever the platform asks and the cache is empty | Yes (verified mechanism)                    |
| 2025.2, 2025.3, 2026.1 | same classes                                             | Asynchronously on a pooled thread after each module change; callers get the last result  | No guarantee: may run before roots exist    |

The behavior changed in JetBrains/MPS commit `0cc1773e6f17` ("MPS-38385 deadlock due to IndexableRootCalculator",
2025-07-01). `git tag --contains` lists only 2025.2.x, 2025.3.x and 2026.1.x tags; no 2025.1.x tag contains it. The
development branch after 2026.1.1 keeps the asynchronous design.

In all four versions `MPSModelsFastFindSupport` reports an unindexed but otherwise eligible model as searched with no
matches when the index query does not throw, so a missed root makes index-backed searches silently empty.

## Who contributes the roots

The extension and the calculator are the same classes in all researched versions:

- `jetbrains.mps.ide.findusages.caches.MPSIndexableSetContributor extends com.intellij.util.indexing.IndexableSetContributor`
  returns `IndexableRootCalculator.getInstance(project).getIndexableRoots()` from
  `getAdditionalProjectRootsToIndex(Project)`. `getAdditionalRootsToIndex()` is empty. Since 2025.2 the contributor also
  filters out invalid files.
- `IndexableRootCalculator` is a package-private project service. `calcRoots()` walks every module of the project
  repository (including libraries) under an MPS read action and collects the `SOURCES` source roots of each
  `FileBasedModelRoot`, stepping into archives. It listens to repository and module events; `moduleAdded`,
  `moduleChanged` and `moduleRemoved` call `invalidateCache()`. Transient and temporary modules are ignored.

No workspace-model entity is involved in any researched version.

## 2024.1 and 2025.1: synchronous, on demand

```text
getIndexableRoots()                          // called by the platform's scanner
  if cache is null or holds an invalid file
    roots = calcRoots()                      // MPS read action, current repository modules
    cache.compareAndSet(null, roots)
  return roots

invalidateCache()                            // module added/changed/removed, inside the MPS write action
  old = cache.getAndSet(null)
  if old != null
    IndexableSetContributorModificationTracker.incModificationCount()
    invokeLaterOnWriteThread: write action { makeRootsChange(no-op, fileTypes=false, fireEvents=true) }
```

Consequences:

- **Verified:** any scan whose provider list is created after the project's modules are in the repository computes roots
  that include them. A full scan creates its providers when it runs (`UnindexedFilesScanner.scan` →
  `getIndexableFilesIterators` → `collectProviders`), and the `IndexableSetContributorFilesIterator` constructor
  evaluates `getAdditionalProjectRootsToIndex` immediately.
- **Verified:** a `RootsChangeRescanningInfo.TOTAL_RESCAN` requested after `openProject` returns therefore covers the
  project, because module loading happens while the project is being opened (`StandaloneMPSProject.loadState` →
  `update()` → `loadModules` in an MPS write action).
- **Verified:** if roots were computed before the modules arrived, the module events post a generic roots change
  (`makeRootsChange(Runnable, false, true)`, which the platform treats as `TOTAL_RESCAN`) with
  `invokeLaterOnWriteThread`. Only the posting happens during module loading; the rescan is queued when the event queue
  runs it.
- **Inferred:** `IndexingTestUtil.waitUntilIndexesAreReady` called off the event dispatch thread checks only queued and
  running scans and dumb tasks. It does not dispatch the posted roots change, so it could return before that rescan is
  queued if an earlier scan computed roots before module loading. Dispatching pending events first, or requesting the
  rescan explicitly, closes this window.
- **Inferred, not observed:** the `compareAndSet` publication can leave a stale result if `invalidateCache` runs between
  the end of `calcRoots()` and the `compareAndSet`. The invalidation then sees an empty cache and posts nothing, and the
  stale set stays until the next module change. The window is a few instructions wide.

## 2025.2 and later: asynchronous recalculation

```diff
 getIndexableRoots()
-  if cache is null or holds an invalid file
-    roots = calcRoots()
-    cache.compareAndSet(null, roots)
-  return roots
+  return cache                               // initially the empty set

+IndexableRootCalculator(project)            // service constructor
+  register listeners
+  invalidateCache()

 invalidateCache()
-  old = cache.getAndSet(null)
-  if old != null
-    notify platform
+  ReadAction.nonBlocking(calcRoots).coalesceBy(listener).expireWith(this).submit(app executor)
+    .onSuccess(cache.set)
+    .then: if result is empty, invalidateCache() again
+           else notify platform               // modification tracker + invokeLaterOnWriteThread roots change
```

Consequences:

- **Verified:** until a background calculation finishes, the contributor returns whatever the cache held before: the
  empty set right after the service is created, or a set that predates the project's modules. The service is created
  lazily, so the first caller, typically the scanner, gets the empty set and only starts the calculation.
- **Verified:** the rescan that picks up new roots is posted only after the background calculation completes. Nothing
  that runs synchronously during `openProject` either computes the roots or queues that rescan.
- **Verified:** a `TOTAL_RESCAN` requested immediately after `openProject` builds its providers from the cache, so it
  can scan without the project's roots. `waitUntilIndexesAreReady` does not know about the pending MPS calculation and
  can return as soon as that scan finishes.
- **Inferred:** the calculation is a non-blocking read action that restarts when write actions intervene and is
  coalesced across module events, so its latency is not bounded by any open or flush call. Dispatching pending events
  does not help until the calculation has posted its notification.

A reliable sequence for headless callers in these versions:

```text
open project
wait until IndexableSetContributor.getProjectRootsToIndex(contributor, project) covers the project's model roots
request makeRootsChange(no-op, RootsChangeRescanningInfo.TOTAL_RESCAN) in a write action on the EDT
waitUntilIndexesAreReady(project)            // outside read and write actions
```

The roots check must use the platform's static `getProjectRootsToIndex`, which applies the same filtering the scanner
sees. For 2024.1 and 2025.1 the same check returns true on the first call, because the call itself computes the roots.

## Searches over unindexed models

`MPSModelsFastFindSupport.findCandidates` is identical in 2024.1.6, 2025.1.4 and 2025.2.4. In 2026.1.1 it adds only a
warning log in the exception handler. For each unchanged `DefaultSModelDescriptor` with a `FileDataSource` or
`FilePerRootDataSource`:

1. It maps each model file to a `VirtualFile` with `FileSystemBridge.asVirtualFile`. It does not check whether the file
   is under an indexable root; a source comment notes this exact hazard.
2. It queries `FileBasedIndex.getContainingFiles("mps.NodeUsage", entry, scope)` through
   `MPSModelsIndexer.getContainingFiles` for each searched element.
3. Only `ProcessCanceledException` or `IndexNotReadyException` make it skip claiming. Otherwise it reports **every**
   model with a mapped file to the processed-model consumer, and only models whose files matched become candidates.

A file outside every indexable root has no `mps.NodeUsage` entries, and the query succeeds in smart mode, so its model
is claimed with no results and excluded from the slower fallback lookup (**verified** in MPS sources; that the platform
does not index such a file on demand is **inferred**). The same applies to `findUsages`, `findInstances` and
`findModelUsages`. See
[Diagnosing missing concept instances](indexing-file-skips-and-diagnostics.md#file-index-false-negatives) for the full
search pipeline.

## Source evidence

MPS paths are relative to the JetBrains/MPS tags [2024.1.6](https://github.com/JetBrains/MPS/tree/2024.1.6),
[2025.1.4](https://github.com/JetBrains/MPS/tree/2025.1.4), [2025.2.4](https://github.com/JetBrains/MPS/tree/2025.2.4)
and [2026.1.1](https://github.com/JetBrains/MPS/tree/2026.1.1). The two calculator files are byte-identical in 2024.1.6
and 2025.1.4, and in 2025.2.4 and 2026.1.1. `MPSModelsFastFindSupport.java` is byte-identical in 2024.1.6, 2025.1.4 and
2025.2.4.

`workbench/mps-workbench/source/jetbrains/mps/ide/findusages/caches/IndexableRootCalculator.java`:

| Anchor                       | 2024.1.6 / 2025.1.4 | 2025.2.4 / 2026.1.1                          |
| ---------------------------- | ------------------- | -------------------------------------------- |
| `myRootsCache` initial value | line 69 (`null`)    | line 71 (empty set)                          |
| constructor → `register()`   | lines 82–91         | lines 84–94 (also calls `invalidateCache()`) |
| `getIndexableRoots()`        | lines 116–124       | lines 108–112                                |
| `calcRoots()`                | lines 127–151       | lines 120–150 (cancellable read action)      |
| `invalidateCache()`          | lines 190–208       | lines 189–216                                |
| `ChangeListener.moduleAdded` | lines 219–225       | lines 227–233                                |

`workbench/mps-workbench/source/jetbrains/mps/ide/findusages/caches/MPSIndexableSetContributor.java`:
`getAdditionalProjectRootsToIndex`, lines 35–38 (2024.1.6, 2025.1.4) and 36–41 (2025.2.4, 2026.1.1).

`workbench/mps-platform/source/jetbrains/mps/workbench/findusages/MPSModelsFastFindSupport.java`: `findCandidates`
starts at line 182 in all four versions; `asVirtualFile` at line 226; the index query and exception handler at lines
252–256 (252–258 in 2026.1.1); the processed-model reporting at lines 263–266 (265–268 in 2026.1.1).

Module loading during project open:

- 2024.1.6 and 2025.1.4: `workbench/mps-workbench/source/jetbrains/mps/project/StandaloneMPSProject.java`, `loadState` →
  `update()` → `loadModules` under `getModelAccess().runWriteAction`.
- 2026.1.1: the same file registers `StandaloneMPSProject$Activity` as an essential, non-parallel `initProjectActivity`
  (`workbench/mps-workbench/source/META-INF/MPSComponents.xml`); it calls `projectOpened()`, and
  `core/project/source/jetbrains/mps/project/ProjectBase.java` `projectOpened()` calls `update()`.
- `workbench/mps-platform/jetbrains.mps.ide.platform/source_gen/jetbrains/mps/tool/environment/IdeaEnvironment.java`:
  `openProjectInIdeaEnvironment` runs `ProjectManagerEx.loadAndOpenProject` in `invokeAndWait`; in test mode
  `doOpenProject` then calls `IndexingTestUtil.waitUntilIndexesAreReady`.

IntelliJ Platform paths are archive-relative to each checkout's `lib/src/platform-sources.zip`:

- `com/intellij/util/indexing/UnindexedFilesScanner.kt` (`.java` in 241): full scans with no predefined iterators call
  `collectProviders` when the scan runs.
- `com/intellij/util/indexing/roots/IndexableSetContributorFilesIterator.kt`: the project-aware constructor calls
  `getAdditionalProjectRootsToIndex` eagerly.
- `com/intellij/util/indexing/IndexableSetContributor.java`: static `getProjectRootsToIndex`.
- `com/intellij/openapi/roots/impl/ProjectRootManagerImpl.kt`: `makeRootsChange(Runnable, Boolean, Boolean)` and the
  generic change mapping to `RootsChangeRescanningInfo.TOTAL_RESCAN`.
- `com/intellij/testFramework/IndexingTestUtil.kt`: `shouldWait()` considers queued and running scanner tasks, dumb
  tasks and dumb mode only.

## Unresolved

- No 2025.2 platform sources were inspected. Platform claims for 2025.2 rely on the 251 and 261 sources agreeing.
- Whether the initial on-open scan in 2024.1 and 2025.1 computes MPS roots before or after module loading was not
  established, so whether the posted-rescan window described there occurs in practice is unknown.
- No runtime probe measured how long the 2025.2+ background calculation takes after `openProject`.
