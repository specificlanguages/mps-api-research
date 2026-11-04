# Reusing an IntelliJ system directory across headless MPS runs

This note answers whether separate headless MPS processes started one after another can reuse one `idea.system.path`
(VFS, indexes, caches), what protects that directory against two processes at the same time, and what MPS itself does to
the directory at startup. Verified against the binaries of MPS **2024.1.6**, **2025.1.4**, **2026.1.1** and the **2026.3
EAP** build **MPS-263.6259.166**, and against the IntelliJ Platform sources of the 263 development branch and the MPS
development-branch sources. Runtime timings come from a profile on MPS 2024.1.6.

## Answer

- **Test mode takes no directory lock.** `IdeaEnvironment` in test mode starts the platform through
  `TestApplicationManager.getInstance()`, which ends in `com.intellij.testFramework.common.loadApp`. That path never
  creates `DirectoryLock`. Two test-mode processes on one system directory share its memory-mapped storages without any
  cross-process protection. (Verified: 263 sources; the `TestApplicationKt` class of all four distributions contains no
  `DirectoryLock` or `lockSystemDirs` reference.)
- **Normal mode locks, but splits the lock across both directories.** `IdeaEnvironment` outside test mode starts the
  platform through `MPSHeadlessPlatformStarter` → `PlatformStarter.startApplicationAsync()` →
  `com.intellij.platform.ide.bootstrap.startApplication`, which calls `lockSystemDirs`. `DirectoryLock` puts `.lock` in
  `idea.config.path` and `.port` (a Unix-domain socket on supported JVMs) in `idea.system.path`. With different config
  directories and one shared system directory, the second process cannot listen on the existing port file, connects to
  the first process instead, passes its command line to it and exits. (Verified: 263 sources of `DirectoryLock.java` and
  `startup.kt`; the `DirectoryLock` class with `.port` and `.lock` is present in all four distributions, in
  `lib/app.jar` up to 2026.1 and in `lib/intellij.platform.ide.bootstrap.jar` in 2026.3.)
- **MPS 2026.3 deletes the test-mode system directory at startup.** In test mode,
  `IdeaEnvironment.createIdeaApplication` deletes `PathManager.getSystemPath()` before
  `TestApplicationManager.getInstance()` unless the system property `mps.test.keep.system.dir` is `true`. (Verified: the
  string is in the `IdeaEnvironment` class of MPS-263.6259.166 and absent from 2024.1.6, 2025.1.4 and 2026.1.1; the
  development-branch source shows the code.)
- **Reuse pays off.** MPS waits for indexes before it opens a project in test mode, and it indexes the source roots of
  all repository modules, about 130 000 files of a distribution with its bundled plugins. With a reused system
  directory, no file was indexed again and only the startup scan remained. (Observed, MPS 2024.1.6: scan and index 15–29
  s cold, 5.6–10 s warm.)

- **A reused VFS is stale until refreshed, and headless startup does not refresh it in time.**
  `com.intellij.openapi.project.InitialVfsRefreshService` skips the initial refresh when `isUnitTestMode()` is true, and
  otherwise only schedules an asynchronous refresh of the IntelliJ content roots. In test mode, `loadAppInUnitTestMode`
  calls `PersistentFSImpl.cleanPersistedContents()`, which flags every file to reload its content and length, but
  directory children and change events are not refreshed. In normal mode cached file content is used as is. (Verified:
  263 sources. Observed on MPS 2024.1.6: a normal-mode headless process that reused a system directory opened a project
  whose model file had changed on disk since the previous process, and generated the old content.) A synchronous
  `VfsUtil.markDirtyAndRefresh(false, true, true, roots)` followed by
  `jetbrains.mps.ide.platform.watching.ReloadManager.getInstance().flush()` on the EDT brings the VFS and the loaded
  models up to date.

A process that wants to reuse a system directory therefore needs its own exclusive lock around the whole lifetime of the
MPS process, for example a `FileChannel.tryLock()` on a file next to the directory, and a synchronous refresh of the
folders it reads before it uses them.

## What lives in the system directory

Observed after headless runs on MPS 2024.1.6: `caches/` (VFS records, about 160 MB), `index/` (file-based indexes, 70–80
MB), `projects/<name>.<hash>/` (one small directory per opened project path), `LocalHistory/`, `extResources/`,
`mps-swap/` (`TransientSwapOwnerComponent`), `tmp/`, and, in normal mode, `.port`, `.home`, and `log/` when
`idea.log.path` is not set. MPS also resolves `mps-generator/` (`GeneratorCacheComponent`) and `merge-backup/`
(`MergeBackupUtil`) under the system path. (Verified: MPS 2024.1 sources under `workbench/mps-workbench/source` and
`plugins/mps-vcs`.)

## Inferred and unresolved

- **Inferred:** a different MPS or platform build may reuse an existing directory without a full rebuild when the VFS
  and index implementation versions match, since those versions, not the build number, gate a rebuild. Keying a reused
  directory by build number avoids depending on that.
- **Inferred:** the platform's corruption markers rebuild storages after an abrupt exit. Not probed.
- **Unresolved:** whether `IntermediateModelsCache` (the user of `mps-generator/`) is active in any headless build path;
  no caller outside its own class was found in the 2024.1 sources.

## Evidence

- MPS development branch:
  `workbench/mps-platform/jetbrains.mps.ide.platform/source_gen/jetbrains/mps/tool/environment/IdeaEnvironment.java`,
  `createIdeaApplication` (test-mode deletion, `mps.test.keep.system.dir`).
- MPS 2024.1 and development branch: `workbench/mps-platform/source/jetbrains/mps/ide/util/PlatformStarter.kt`
  (`startApplication` call),
  `workbench/mps-platform/source/jetbrains/mps/tool/environment/MPSHeadlessPlatformStarter.java`.
- IntelliJ Platform 263: `platform/platform-impl/bootstrap/src/com/intellij/platform/ide/bootstrap/DirectoryLock.java`
  (`lockOrActivate`: `tryListen`, then `tryConnect`), `.../bootstrap/startup.kt` (`lockSystemDirs`),
  `platform/testFramework/common/src/common/testApplication.kt` (`loadApp`, `loadAppInUnitTestMode`),
  `platform/platform-api/src/com/intellij/ide/SpecialConfigFiles.java` (`.port`, `.lock`),
  `platform/platform-impl/src/com/intellij/openapi/project/InitialVfsRefreshService.kt`,
  `platform/platform-impl/src/com/intellij/openapi/vfs/newvfs/persistent/PersistentFSImpl.java`
  (`cleanPersistedContents`).
