# Model save lifecycle (does MPS auto-save?)

**Question this answers:** can a headless MPS platform persist an in-memory model change to disk through idle auto-save,
save-on-close, save-on-command, or a background flush?

**Answer:** MPS's automatic model-save bridge is gated off in headless mode. A headless host must explicitly invoke a
save API, such as `SModel.save(...)`; neither completing a command nor disposing a project implicitly saves its models.
Host code can still introduce its own save calls or listeners.

Verified against MPS **2025.1.2** (`com.jetbrains:mps:2025.1.2`). Source references below are to the
[JetBrains/MPS](https://github.com/JetBrains/MPS) repository, paths relative to its root.

## The only automatic save bridge: `IdeMPSFileSaver`

MPS does have an automatic save, but it is a workbench feature, not a kernel one.

- `jetbrains.mps.ide.save.IdeMPSFileSaver` (`mps-platform.jar`; source
  `workbench/mps-platform/source/jetbrains/mps/ide/save/IdeMPSFileSaver.java`) implements
  `com.intellij.openapi.fileEditor.FileDocumentManagerListener`. Its `beforeAllDocumentsSaving()` runs
  `SaveRepositoryCommand` for **every open project's whole repository** — every dirty model, not just the ones a caller
  touched. Its own class comment: _"it saves everything whenever the platform saves everything."_
- It fires off the IntelliJ platform's "save all documents" event (`FileDocumentManager` / `SaveAndSyncHandler` — frame
  deactivation, idle auto-save, project/app close). That is the mechanism that would otherwise flush edits behind an API
  caller's back.

### Why it does not fire headlessly

It is registered with `activeInHeadlessMode="false"`:

```xml
# META-INF/MPSCore.xml (mps-workbench.jar)
<listener class="jetbrains.mps.ide.save.IdeMPSFileSaver"
          topic="com.intellij.openapi.fileEditor.FileDocumentManagerListener"
          activeInHeadlessMode="false"/>
```

The platform honors that flag when registering message-bus listeners:
`com.intellij.serviceContainer.ComponentManagerImpl` reads `Application.isHeadlessEnvironment()` and skips any
`ListenerDescriptor` whose `activeInHeadlessMode` is false (`app.jar`; `ListenerDescriptor` in `util-8.jar`). Bytecode
branch (`javap -c ComponentManagerImpl`): when the headless flag is set and `activeInHeadlessMode == false`, the
descriptor is dropped instead of added to the topic's listener list.

An environment started through `jetbrains.mps.tool.environment.IdeaEnvironment` uses `MPSHeadlessPlatformStarter` and
sets `java.awt.headless=true` (`workbench/mps-platform/.../IdeaEnvironment.java`). In that configuration,
`Application.isHeadlessEnvironment()` is true and `IdeMPSFileSaver` is never registered, so IntelliJ idle, deactivation,
and close saves do not reach the MPS repository.

## Commands and project disposal do not imply a save

- Every other caller of `SaveRepositoryCommand` / `EditableSModelBase.save` is an interactive workbench action —
  refactoring dialogs, module/model properties, `MakeActionImpl`, migration wizards, or VCS conflict tracking. These are
  explicit callers rather than a general command-completion hook.
- The `autosave` string in `EditableSModelBase.resolveConflict0()` is a _warning message_ shown on external-file
  conflict (it even notes MPS does "saveAll on each fs reload" — that is the `IdeMPSFileSaver` path above), not a
  self-scheduled save.
- Project dispose does not save: `ProjectBase.dispose()` has no `save()` call. A headless host that disposes a project
  without first invoking a save drops unsaved in-memory changes rather than flushing them.

## Consequence for rollback with `reloadFromSource()`

In a headless host with no additional save listeners or calls, `reloadFromSource()` can revert a batch that failed
**before** an explicit save: the on-disk copy is still the pre-batch state, and reloading discards the in-memory
mutations. Its limits are the explicit-save ones:

- `saveWithResolveInfo` writes affected models one at a time and stops at the first failure, so a batch spanning several
  models can leave earlier models already persisted.
- `reloadFromSource()` reverts to on-disk state, so any in-memory changes a model already held at batch start are
  discarded along with the batch.

## Gotchas / re-verify triggers

- This guarantee is contingent on **headless operation without host-provided saving**. In a non-headless environment, or
  if the host registers its own `FileDocumentManagerListener` or calls `FileDocumentManager.saveAllDocuments()`,
  `IdeMPSFileSaver` can save the **entire repository** — every dirty model across every open project, not just the
  current batch's models.
- The `activeInHeadlessMode` gate is an IntelliJ platform contract, not an MPS one; re-verify against
  `ComponentManagerImpl` if the platform build under MPS changes materially.
