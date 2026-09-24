# Diagnosing missing concept instances and skipped file indexing

This note explains why `FindUsagesFacade.findInstances` can miss nodes present in a model, which settings and conditions
prevent file indexing, and how to diagnose the cause with search comparisons and indexing logs. Verified against **MPS
2025.1.4** and its patched IntelliJ Platform **MPS-251.28774.587** sources. No runtime failure was reproduced; the
mechanisms are verified, but their involvement in a particular failure requires observation.

A successful query against `mps.NodeUsage` can suppress model-local fallback when expected keys are missing. Diagnose
both the search pipeline and the production of those keys: an existing `VirtualFile`, a completed index update, or an
"indexed" diagnostic entry does not establish that the expected concept key is present.

## Search pipeline

`org.jetbrains.mps.openapi.module.FindUsagesFacade` exposes the consumer overload:

```java
void findInstances(SearchScope scope, Set<? extends SAbstractConcept> concepts,
                   boolean exact, Consumer<SNode> consumer, ProgressMonitor monitor)
```

Here `Consumer` is `org.jetbrains.mps.openapi.util.Consumer`. `FindUsagesManager` delegates to `InstancesSearchType`:

1. Copy the requested concepts; unless `exact`, add descendants from `ConceptDescendantsCache`.
2. Enumerate the scope's models. Changed `EditableSModel` instances bypass all search participants.
3. Invoke registered `FindUsagesParticipant` instances in order on the remaining models. A model reported to the
   processed-model consumer is removed from subsequent participants and fallback, even if no nodes were returned.
4. Run `InstanceLookup` for unclaimed models and changed models.
5. `InstanceLookup` calls `FastNodeFinderManager.get(model).getNodes(concept, false)` for each expanded concept.

The fallback therefore bypasses file-index selection but still uses the model-local node cache. A truly independent
baseline must traverse `SModel.getRootNodes()` and recursively `SNode.getChildren()`, comparing concepts directly.

## File-index false negatives

`MPSModelsFastFindSupport.findCandidates` handles unchanged `DefaultSModelDescriptor` models with `FileDataSource` or
`FilePerRootDataSource`. `ProjectModelFilter` requires model read access and checks that the model reference resolves to
the same object in its project repository. This is project visibility, not ownership by a project module.

For eligible files it obtains IntelliJ `VirtualFile` objects, queries the `mps.NodeUsage` file index with
`UsageEntry.ConceptInstance(MetaIdHelper.getConcept(concept))`, and maps matching files back to candidate models. It
then reports **all eligible models with mapped files** as processed if no query failed, including models absent from the
candidate set. Only candidates undergo `InstanceLookup`.

Consequently, a successful query with a missing concept key can suppress a model entirely. Source comments explicitly
identify the danger of mapping files that are not covered by indexable roots. A `VirtualFile` existing does not
establish that its contents were indexed. A model with multiple streams may be claimed after only some streams map
successfully.

If a query throws `ProcessCanceledException` or `IndexNotReadyException`, the participant does not claim these models,
allowing fallback. An unsupported data source, wrong descriptor type, or no mapped files also leaves a model available
to later participants/fallback. These conditions alone do not explain a false negative.

Root coverage, file-size limits, input filters and persistence-indexer failures can all leave expected keys absent. See
[settings that change coverage](#settings-that-change-coverage) and
[other reasons a file has no index entries](#other-reasons-a-file-has-no-index-entries).

## Settings that change coverage

Properties can be supplied as `key=value` in custom IDE properties or `-Dkey=value` in custom VM options. Restart after
changing startup properties. Registry settings are a separate mechanism: in this platform, a stored Registry value wins
over a system property of the same name, which wins over the bundled Registry default.

| Setting                                                            | Verified effect                                                                                                                                                                                                                                                                 |
| ------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `idea.max.intellisense.filesize`                                   | Content-indexing size threshold in **KiB** (1024 bytes). MPS ships **100000**, or 102400000 bytes, about 97.7 MiB. The platform fallback is 2500 KiB. Files strictly larger than the applicable limit normally skip content-dependent indexes.                                  |
| `idea.max.content.load.filesize`                                   | Contributes to the default content-loading limit. In this build the effective limit is the **maximum of 20 MiB, the configured intelligence limit, and this configured content limit**. Raising this property alone does not lift the ordinary intelligence/indexing threshold. |
| `idea.ignore.plain.text.indexing=true`                             | Suppresses the ordinary plain-text branch of `IdIndex` eligibility. It is **not** a switch disabling all indexes for plain-text files, nor the MPS node-usage index. An explicit ID indexer can still make a file type eligible.                                                |
| `idea.skip.indices.initialization=true`                            | Makes `IndexInfrastructure.hasIndices()` false; initialization, scanning and indexing paths short-circuit. A global switch, not a per-file filter.                                                                                                                              |
| `idea.suspend.indexes.initialization=true`                         | Starts the scanning executor in a suspension loop awaiting cancellation. This delays work globally rather than selectively filtering files.                                                                                                                                     |
| Registry `indexer.follows.symlinks=false`                          | Rejects symlinks during root traversal, including contributed roots. With following enabled, broken or recursive links are still rejected.                                                                                                                                      |
| Registry `scanning.trust.indexing.flag` (call-site default `true`) | Allows an early return for files whose indexing stamp says they are already indexed. This normally avoids redundant work. Incorrect stamps or missed changes are diagnostic hypotheses, not demonstrated causes.                                                                |

`FileSizeLimit` also allows extension-specific limits through `com.intellij.fileEditor.fileSizeChecker`. Consequently,
the effective limit for a particular extension can differ from the global limit in a diagnostic dump.

The relevant size decision is:

```text
if file exceeds its intelligence limit:
  if file type has no indexing size exemption:
    skip content indexing
  else if file exceeds its content-loading limit:
    skip content indexing
```

Index extensions can nominate exempt file types through
`FileBasedIndexExtension.getFileTypesWithSizeLimitNotApplicable()`. The platform collects these file types across
indexes. `MPSModelsIndexer` itself does not override this method. There is also a per-VirtualFile size-check bypass in
`SingleRootFileViewProvider`. Do not assume that every plugin configuration uses the same thresholds.

The property parser accepts integer KiB values, multiplies by 1024, and caps at `Integer.MAX_VALUE` bytes. Negative
values also resolve to that cap; invalid integer text falls back to the default. Avoid treating negative values as a
literal unlimited-file guarantee.

`-Xmx` affects available heap, and indexing/scanning thread settings affect throughput; neither is the file-size
eligibility threshold above. `idea.vfs.max-file-length-to-cache` governs VFS content caching, not index eligibility.

## Other reasons a file has no index entries

```text
MPS source roots → VFS traversal → file status/size → required indexes → persistence indexer → stored keys
      missing root    rejected link    already current    wrong type       read/parse failure    empty result
```

1. **No indexable root reaches the file.** `MPSIndexableSetContributor` obtains roots from `IndexableRootCalculator`,
   which derives paths from `FileBasedModelRoot` source roots of kind `SOURCES`, including repository-visible libraries.
   Failed path-to-VirtualFile mapping, unavailable archives, missing model roots or stale root notifications can leave
   files outside coverage. Their involvement in a particular failure needs observation.
2. **Traversal rejects the file.** Symlink rules apply as above. The common traversal also rejects files without a
   positive persistent `VirtualFileWithId` ID. An invalid file is rejected by the scanner.
3. **Exclusions depend on the root provider.** Ordinary project traversal rejects excluded or ignored files via
   `ProjectFileIndex.isExcluded()`. However, `IndexableSetContributorFilesIterator` passes
   `excludeNonProjectRoots=false`; MPS uses this contributor mechanism. An IDEA exclusion or ignored-file rule alone
   therefore does not establish that an MPS-contributed root is skipped.
4. **The file is current, or its data is supplied by an infrastructure extension.** No fresh content-indexing operation
   is required. Deduplicated overlapping roots likewise need not process the same file twice. These are normal skips,
   not evidence of missing data.
5. **A particular index rejects the file type/input.** `RequiredIndexesEvaluator` applies per-index file-type hints and
   input filters. A file appearing in a filename/file-type index does not prove membership in `mps.NodeUsage`.
   Project/workspace files have a special path that retains only the file-type index for regular files.
6. **MPS persistence cannot supply keys.** In 2025.1.4 `MPSModelsIndexer.getFileTypes()` lazily populates its map while
   empty from registered `IndexAwareModelFactory` implementations and recognized MPS file types. It also adds the
   `.model` and `.mpsr` auxiliary file types when the primary factory is present. Its input filter accepts the mapped
   types. A missing factory during mapping returns an empty result. A caught `IOException` logs a warning and returns
   whatever keys were accumulated, potentially an incomplete result. The platform can therefore finish an update
   successfully without storing all expected concept keys.
7. **Content cannot be loaded, or work is interrupted.** Files can disappear or become invalid between scanning and
   reading. I/O failures, mapping exceptions, cancellation and project disposal can prevent completion. These must be
   distinguished from successful indexing with zero keys.

Search scope, query-time project filters, dumb mode and model-local lookup can also affect search results. They do not
by themselves establish that a file was skipped during indexing. The
[concept-instance search pipeline](#search-pipeline) and
[file-index false-negative mechanism](#file-index-false-negatives) explain how missing index keys can suppress
model-local fallback in `FindUsagesFacade.findInstances`.

## Other ways instances disappear

- `InternalModelsFindUsagesParticipant.findInstances` claims descriptor-stereotype models without returning nodes. Its
  implementation assumes these internal models are empty.
- `ConceptDescendantsCache` builds descendant relationships from loaded language runtimes' structure descriptors. An
  unavailable or inconsistent derived-language runtime can affect inclusive queries even when the base concept resolves.
  Test exact lookup of a missing node's own concept and inspect its membership in the expanded query set.
- `FastNodeFinderManager` caches finders by model reference. Models may supply a finder through
  `FastNodeFinder.Factory`; otherwise it uses `BaseFastNodeFinder`, which does not track node changes itself.
  `DefaultFastNodeFinder` attaches a change tracker only to writable editable models. Lifecycle events dispose finders.
  A missing notification or unsuitable custom finder is a hypothesis; compare its exact results with raw traversal.
- Concept names are insufficient evidence of identity: the file index uses `SConceptId`, while node lookup uses concept
  objects as keys. Record qualified names, IDs, validity, and equality with the requested concept.
- Scope exclusion, post-search filters, result limits, stale model state, and model loading errors should be separated
  from index selection before interpreting an empty result.

## Diagnostic procedure

Record the runtime version, actual scope membership, model reference and implementation, data source and streams,
read-only/changed/loaded state, registered participants, requested concept identity, and exact/inclusive mode. Capture
the normal search before forcing a full tree traversal, since loading models and initializing caches can change later
observations.

Compare serialized node-reference sets, not just counts:

1. Full facade search, before application filters and limits.
2. `InstanceLookup` with exactly the same expanded concept set, bypassing participants.
3. Raw traversal matching membership in that expanded concept set.
4. Raw traversal using concept equality for exact mode or `node.getConcept().isSubConceptOf(requested)` for inclusive
   mode.

Interpretation:

| Difference                                                          | Layer implicated                             |
| ------------------------------------------------------------------- | -------------------------------------------- |
| Facade misses nodes found by model-local lookup                     | Participant selection/claiming               |
| Model-local lookup misses raw nodes belonging to the same query set | Fast node finder                             |
| Semantic traversal matches concepts absent from the expanded set    | Concept descendants/runtime metadata         |
| All layers agree, application output omits nodes                    | Scope, filters, limits, or output adaptation |

For participant attribution, replay the registered participants locally in order with recording node and processed-model
consumers, retaining the same changed-model bypass rule. Do not unregister participants or toggle global search
settings. This is a separate diagnostic execution, not a trace of the original invocation; report disagreements between
runs. A claim with zero nodes is not itself an error: only contradictory node evidence establishes a discrepancy.

For file-index attribution, inspect the implicated model's streams and query concept keys over those concrete files. An
empty key query alone does not distinguish an unindexed file from an indexed file without that key. Report unavailable
coverage information as unknown. Account for all streams of file-per-root models.

Use the [diagnostic logging recipe](#diagnostic-logging-recipe) to collect file/index update events and provider-root
reports. The [evidence interpretation](#reading-the-evidence) explains why neither a completed update nor an "indexed"
report entry proves that the expected concept key was stored.

Repository/model inspection requires MPS read access; `ProjectModelFilter` explicitly checks it. The production caller's
IntelliJ indexing/locking context must also be preserved. This investigation does not establish a general-purpose safe
background-thread contract for standalone index probes. Do not wait for smart mode while holding model access;
coordinate any such wait outside the read action. Queries and traversal can initialize caches and load models even
without edits.

Avoid repair during evidence collection: no cache invalidation, model marking as changed, saves, or forced reindexing.
Record cancellation and loading exceptions as incomplete diagnosis, not zero matches.

## Diagnostic logging recipe

First preserve the failing state: record the IDE/build number, model stream URLs and byte lengths, file types, search
scope, and the time of the failed query. Save existing logs before invalidating caches or requesting reindexing.

In **Help → Diagnostic Tools → Debug Log Settings**, enable:

```text
#com.intellij.util.indexing.FileBasedIndexImpl:trace
#com.intellij.util.indexing.IndexingReasonExplanationLogger:trace
#com.intellij.util.indexing.RequiredIndexesEvaluator
```

The first category is also recommended in
[JetBrains indexing diagnostics guidance](https://youtrack.jetbrains.com/projects/WI/articles/SUPPORT-A-597/Indexing-in-JetBrains-IDE-is-slow-or-stuck-on-a-particular-step).
The other two categories are verified in the matching platform sources:

- `IndexingReasonExplanationLogger` logs the first ten explanations per category at INFO and subsequent explanations at
  TRACE. They name requested indexes and reasons for scheduling, or scanner-side index updates/removals. Enabling only
  `FileBasedIndexImpl:trace` does not enable this separate logger.
- `RequiredIndexesEvaluator` logs file-type-to-index candidate lists at DEBUG. Check whether `mps.NodeUsage` appears for
  the actual file type. Candidate lists can still contain indexes whose slow input filters reject an individual file;
  this is not proof of a completed update.

For a focused reproduction, add these custom VM options and restart:

```text
-Dtrace.file.based.index.update=true
-Dintellij.indexes.diagnostics.should.dump.paths.of.indexed.files=true
-Dintellij.indexes.diagnostics.should.dump.provider.root.paths=true
```

The update property is independent of logger level. It enables INFO messages for individual persistent index updates and
deletions, including the index ID and file information. Look for `mps.NodeUsage` and the affected file. Remove verbose
settings after capturing the relevant activity; retaining them produces unnecessary log volume.

Reproduce the natural scan/update and failed query, then collect **Help → Collect Logs and Diagnostic Data**. Existing
current files may not be reindexed merely because logging was enabled. A separately recorded forced reindex can test
repair, but cannot reconstruct why the original state occurred.

## Reading the evidence

| Evidence                                                | Interpretation and limit                                                                                                                                                                           |
| ------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `File: … is too large for indexing`                     | Positive evidence of rejection at the content-loading stage. A file rejected earlier by the scanner's size check need not emit this message. Absence is not proof that size was acceptable.        |
| `index mps.NodeUsage update finished for …`             | The persistent update completed. It does not prove that a particular concept key was emitted or that the persistence parser produced a complete result.                                            |
| `index … deletion finished for …`                       | Existing index data was removed. Correlate with roots, file changes and subsequent updates.                                                                                                        |
| `MPSModelsIndexer` warning beginning `Indexing failed:` | The persistence indexer caught `IOException`; an incomplete key set can still be returned. The warning does not itself include the input file path. Correlate carefully with surrounding activity. |
| Content-loading exceptions                              | Missing-file exceptions are DEBUG under `FileBasedIndexImpl`; other I/O/invalid-file failures are throttled INFO, and unexpected failures are ERROR. Throttling limits completeness.               |
| No line for a file                                      | Inconclusive: it may be outside roots, rejected early, already current, or absent from this particular activity.                                                                                   |

The platform writes per-project JSON and HTML activity reports beneath the IDE log directory's `indexing-diagnostic/`.
Reports contain scanning/indexing statistics; the path options above additionally capture scanned/indexed files and
provider roots. Runtime information includes `maxSizeOfFileForIntelliSense` and `maxSizeOfFileForContentLoading` in
**bytes**, useful for verifying global effective limits rather than just reading a configuration file.

Interpret dump labels conservatively:

- `ScanningStatistics` records a scanned file's `isUpToDate` flag as `!shouldIndex`. This is not proof that every
  desired index has valid entries: oversized files also need not be scheduled for content indexing.
- `IndexingFileSetStatistics.addTooLargeForIndexingFile()` includes the file in its indexed-file list with
  `IndexesEvaluated.NOTHING_TO_WRITE` and increments the too-large counter. A path in that list is not by itself proof
  of successful content indexing.
- A missing provider root or file is a lead only after checking that the report covers the relevant full scan rather
  than a partial update. Provider overlap/deduplication also affects where a file is attributed.
- Default retention is bounded (300 reports and, absent an explicit report-count property, 10 MiB per project). Collect
  the reports promptly. Report-count limits do not mean that a report is a complete index inventory.

To investigate one missing concept, correlate root coverage, the actual stream's size and file type, scheduling reasons,
`mps.NodeUsage` updates, and MPS parser warnings. Then compare the concrete file's stored concept keys with its
serialized contents and model traversal. Logs explain processing events; they do not enumerate every semantic key or
certify a negative lookup. Follow the [missing-instance diagnostic procedure](#diagnostic-procedure) to compare facade
results, participant-free lookup and raw traversal, accounting for concept expansion and every model stream.

## Version scope

The cited search-pipeline, participant, root-calculation and model-local lookup sources are unchanged between MPS
2025.1.2 and 2025.1.4. `MPSModelsIndexer` differs: 2025.1.2 populates the file-type/factory map in its constructor;
2025.1.4 populates it lazily in `getFileTypes()` while the map is empty, and passes a factory lookup function to the
indexer. The missing-factory and caught-`IOException` behavior is unchanged. The platform settings and logging details
in this note were verified against the 2025.1.4 platform build; they are not asserted for other platform builds.

## Source evidence

MPS paths are relative to [JetBrains/MPS tag 2025.1.4](https://github.com/JetBrains/MPS/tree/2025.1.4):

- `build/version.properties`: patched platform build selection.
- `bin/idea.properties`: shipped intelligence limit.
- `workbench/mps-platform/source/jetbrains/mps/workbench/findusages/MPSModelsIndexer.java`: `getFileTypes`,
  `getInputFilter`, `dependsOnFileContent`, `ModelIndexer.map`.
- `workbench/mps-workbench/source/jetbrains/mps/ide/findusages/caches/MPSIndexableSetContributor.java` and
  `IndexableRootCalculator.java`: root contribution and source-root discovery.
- `core/openapi/source/org/jetbrains/mps/openapi/module/FindUsagesFacade.java`
- `core/openapi/source/org/jetbrains/mps/openapi/persistence/FindUsagesParticipant.java`
- `core/findUsages-runtime/source_gen/jetbrains/mps/findUsages/FindUsagesManager.java`
- `core/findUsages-runtime/source_gen/jetbrains/mps/findUsages/InstancesSearchType.java`
- `core/findUsages-runtime/source_gen/jetbrains/mps/findUsages/InstanceLookup.java`
- `core/kernel/source/jetbrains/mps/smodel/ConceptDescendantsCache.java`
- `core/kernel/source/jetbrains/mps/smodel/FastNodeFinderManager.java`
- `core/kernel/source/jetbrains/mps/smodel/BaseFastNodeFinder.java`
- `core/kernel/source/jetbrains/mps/smodel/DefaultFastNodeFinder.java`
- `workbench/mps-platform/source/jetbrains/mps/workbench/ProjectModelFilter.java`
- `workbench/mps-platform/source/jetbrains/mps/workbench/findusages/MPSModelsFastFindSupport.java`
- `workbench/mps-platform/jetbrains.mps.ide.platform/source_gen/jetbrains/mps/workbench/findusages/InternalModelsFindUsagesParticipant.java`

The search APIs and model-local finders are in the openapi/kernel/findUsages runtime components; file indexing and
search participants are workbench platform components. Anchors are the class and method names discussed above.

Platform paths below are archive-relative to that checkout's patched `lib/src/platform-sources.zip`, identified by
`lib/build.txt` as **MPS-251.28774.587**, rather than unpatched upstream sources. These are IntelliJ Platform components
in MPS's `lib/`; MPS-specific indexing lives in the workbench platform components.

- `com/intellij/openapi/util/io/FileUtilRt.java`: size constants, `parseKilobyteProperty`.
- `com/intellij/openapi/vfs/PersistentFSConstants.java`: intelligence limit and VFS cache threshold.
- `com/intellij/openapi/vfs/limits/FileSizeLimit.kt`: per-extension overrides and effective limits.
- `com/intellij/psi/SingleRootFileViewProvider.java`: intelligence/content checks and per-file bypass.
- `com/intellij/openapi/util/registry/RegistryValue.kt`: stored-value/system-property precedence.
- `com/intellij/psi/impl/cache/impl/id/IdIndex.java`: `isIndexable` plain-text handling.
- `com/intellij/openapi/roots/impl/ProjectFileIndexImpl.java`: `isExcluded`.
- `com/intellij/util/indexing/`: `FileBasedIndexEx.java`, `FileBasedIndexImpl.java`, `FileBasedIndexExtension.java`,
  `RegisteredIndexes.java`, `RequiredIndexesEvaluator.kt`, `UnindexedFilesFinder.java`, `IndexInfrastructure.java`,
  `UnindexedFilesScannerExecutorImpl.kt`, `IndexingReasonExplanationLogger.kt`, `SingleIndexValueApplier.java`,
  `SingleIndexValueRemover.java`.
- `com/intellij/util/indexing/roots/`: `IndexableFilesIterationMethods.kt`, `IndexableSetContributorFilesIterator.kt`,
  `IndexableFilesDeduplicateFilter.java`.
- `com/intellij/util/indexing/contentQueue/IndexUpdateRunner.kt`: size/read failures and their logging.
- `com/intellij/util/indexing/diagnostic/`: `IndexDiagnosticDumper.kt`, `utils.kt`, `ScanningStatistics.kt`,
  `IndexingFileSetStatistics.kt`, `dto/JsonRuntimeInfo.kt`: dump controls, location, retention and semantics.

**Unresolved for any particular installation:** actual startup/Registry values, plugin-supplied exemptions or limits,
root coverage at the time of failure, and whether a missing key resulted from skipped processing, stale state or an
incomplete persistence-indexer result. No runtime observation is claimed by this note.
