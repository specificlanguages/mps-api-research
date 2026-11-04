# Native file watcher in a headless MPS application

Can a long-running headless MPS application (started with `IdeaEnvironment` / `MPSHeadlessPlatformStarter`) use the
IntelliJ native file watcher (`fsnotifier`) to learn about external file changes, instead of walking folders with a
forced refresh before each operation? How is it turned on, which roots does it watch, and how does a caller make sure
that a change made just before an operation is visible to that operation?

Researched for MPS 2024.1.6 (platform `MPS-241.19416.417`), 2025.1.4 (`MPS-251.28774.587`), 2026.1.1
(`MPS-261.25134.711`) and the 2026.3 EAP build `MPS-263.7307.178` (platform `MPS-263.7307.752`). Platform sources were
read from JetBrains/intellij-community at `idea/241.19416.15`, `idea/251.28774.11`, `idea/261.25134.95` and `master`
(October 2026) for 263; the deciding methods were checked against the bytecode of each distribution. Runtime probes ran
on macOS 26 (arm64) only.

## Short answer

- The watcher is off because the headless application is a command-line application:
  `NativeFileWatcherImpl.isDisabled()` returns `true` for `Application.isCommandLine() || isUnitTestMode()`. No system
  property turns it on. **Verified** (all four versions).
- A plugin can register a subclass of `NativeFileWatcherImpl` that overrides `isDisabled()` as an extra
  `com.intellij.vfs.local.pluggableFileWatcher` extension. The watcher then starts and reports
  `Native file watcher is operational.` in a headless application. **Observed** (all four versions, macOS).
- The executable is found under `bin/<os>/` of the distribution, but the generic `com.jetbrains:mps` zip does not keep
  the executable bit, so the start fails with `Native file watcher is not executable`. Copy the binary somewhere, set
  the executable bit, and point `-Didea.filewatcher.executable.path` at the copy. **Observed** (macOS).
- MPS itself registers the watch roots: the project folder's children, every folder that an MPS file-system listener
  watches (modules, models, libraries, also outside the project), and the distribution. Files outside every watch root
  are never marked dirty. **Verified** (source) and **Observed**.
- A non-forced recursive refresh (`LocalFileSystem.refreshFiles(files, false, true, null)`,
  `VirtualFileManager.syncRefresh()`) then visits only dirty files. If the watcher is not operational, the same call
  marks its roots dirty recursively, which is the full walk. **Verified**.
- There is no API to flush pending watcher events. On macOS an event arrives up to about 300 ms after the change, so a
  refresh right after an edit can miss it. A fence (create a fresh directory under a watch root, wait until the watcher
  reports a path inside it, then refresh) closed the race in 200 probe iterations. **Observed** (macOS).

## Why the watcher is disabled

`com.intellij.openapi.vfs.impl.local.NativeFileWatcherImpl`
(`platform/platform-impl/src/com/intellij/openapi/vfs/impl/local/NativeFileWatcherImpl.java`; in `lib/app.jar` for
2024.1 and 2025.1, in `lib/intellij.platform.ide.impl.jar` for 2026.1 and 263):

```java
protected boolean isDisabled() {
  if (Boolean.getBoolean("idea.filewatcher.disabled")) return true;
  var app = ApplicationManager.getApplication();
  return app.isCommandLine() || app.isUnitTestMode();
}
```

The method is identical in the bytecode of all four distributions. `initialize` logs `Native file watcher is disabled`
when it returns `true` and does not look for the executable at all.

`IdeaEnvironment.createIdeaApplication()` (MPS `workbench/mps-platform/jetbrains.mps.ide.platform/source_gen/...`)
starts the platform through `MPSHeadlessPlatformStarter` (command name `mps-inspect`, `isHeadless() == true`), or
through `TestApplicationManager` in test mode. Either way `isCommandLine()` or `isUnitTestMode()` is `true`.

`idea.filewatcher.disabled` can only turn the watcher off. Nothing in the property set can turn it on.

## Turning it on

`PluggableFileWatcher` is the extension point `com.intellij.vfs.local.pluggableFileWatcher` (declared non-dynamic in
`platform/ide-core-impl/resources/intellij.platform.ide.core.impl.xml`). `FileWatcher` initializes every registered
watcher and `FileWatcher.isOperational()` is `true` when any of them is. So a second registration next to the platform's
own (disabled) one is enough:

```java
public class HeadlessNativeFileWatcher extends NativeFileWatcherImpl {
  @Override
  protected boolean isDisabled() { return false; }
}
```

```xml
<extensions defaultExtensionNs="com.intellij">
  <vfs.local.pluggableFileWatcher implementation="example.HeadlessNativeFileWatcher"/>
</extensions>
```

Only `isDisabled()` is overridden. `PluggableFileWatcher.initialize` changed its signature in 261
(`initialize(ManagingFS, FileWatcherNotificationSink)` became `initialize(FileWatcherNotificationSink)`), so a subclass
that overrides `initialize` is not binary compatible across the researched versions. One class compiled against 2026.1
that overrides only `isDisabled()` loaded and ran in 2024.1.6, 2025.1.4, 2026.1.1 and 263.7307.178. **Observed.**

### The executable

`getExecutable()` uses `-Didea.filewatcher.executable.path` if set, otherwise `PathManager.findBinFile("fsnotifier")`
(`fsnotifier.exe` on Windows). `PathManager.getBinDirectories` includes `bin/` and `bin/<win|mac|linux>/`, so the
distribution layout matches. Shipped binaries:

| Distribution | `bin/mac/fsnotifier`             | `bin/linux/fsnotifier` | `bin/linux-aarch64/` | `bin/win/fsnotifier.exe` |
| ------------ | -------------------------------- | ---------------------- | -------------------- | ------------------------ |
| 2024.1.6     | universal x86_64 + arm64, ad hoc | x86-64, dynamic        | none                 | x64                      |
| 2025.1.4     | same                             | x86-64, dynamic        | none                 | x64                      |
| 2026.1.1     | same                             | x86-64, static         | none                 | x64                      |
| 263.7307.178 | same                             | x86-64, static         | aarch64, static      | x64                      |

In the generic distribution zip, as extracted by Gradle, the files have mode `-rw-r--r--`. `initialize` then logs
`Native file watcher is not executable: .../bin/mac/fsnotifier` and the watcher stays off. **Observed** (2024.1.6). A
copy with the executable bit, passed through `-Didea.filewatcher.executable.path=<copy>`, starts. The macOS binary is ad
hoc, linker signed, and the extracted file had no `com.apple.quarantine` attribute, so it ran without any Gatekeeper
step. A file downloaded through a browser would carry the quarantine attribute (not tested).

On Windows `Files.isExecutable` is true for any readable file, so the extracted `fsnotifier.exe` is expected to start as
is. **Inferred.** Linux on aarch64 has a binary only in 263; `getExecutable()` returns the x86-64 binary path on earlier
versions and the start fails. **Inferred.**

## Which roots are watched

The platform side (`LocalFileSystem.addRootToWatch`, `WatchRootsManager` until 261, `WatchRootsServiceImpl` in 263)
watches only what someone requested. In MPS the requests come from:

- `WatchingFileSystemListenersContainer` (`workbench/mps-workbench/source/jetbrains/mps/ide/vfs/`), the application's
  `FileSystemListenersContainer`: every MPS `FileSystemListener` that is added (modules, model roots, models, libraries)
  calls `WatchedRoots.addWatchRequest(path)`, a recursive watch request unless an existing request covers the path.
- `ProjectRootListenerComponent.boostProjectRead` (`workbench/mps-platform/source/jetbrains/mps/ide/vfs/`): one listener
  per child of the project folder except `.git`.
- `FSNotificationsImprover`: the MPS home and the plugins path.
- `WatchedRootsUpdater` (`workbench/mps-workbench/jetbrains.mps.ide/source_gen/jetbrains/mps/watching/`): project and
  global library paths.

Observed recursive roots in a headless application with a project library outside the project folder (same set on all
four versions): the MPS home, `<config>/plugins`, `<system>/extResources`, the project's `.mps`, its other top-level
children, and the library folder. These requests exist whether or not the watcher runs; with the watcher off they are
only bookkeeping.

A file that is not under any watch root is never marked dirty by the watcher. In a probe, an external write to a file
outside every root was missed 40 times out of 40 by the non-forced refresh. **Observed.** Roots that the watcher reports
as unwatchable (network mounts, UNC paths on Windows) become "manual" roots, which every refresh marks dirty
recursively. **Verified** (`LocalFileSystemImpl.markSuspiciousFilesDirty`).

### Platform limits

- **macOS:** `fsnotifier` opens one FSEvents stream for `/` with a latency of 0.3 s and
  `kFSEventStreamCreateFlagNoDefer` (`native/fsNotifier/mac/fsnotifier.c`). Events have directory granularity; the Java
  side filters them to the watch roots. CPU use was negligible (0.2 s of CPU in 13 minutes on a busy machine).
- **Linux:** one inotify watch per directory, added when roots are set (`native/fsNotifier/linux/`). The main loop
  sleeps 50 ms per iteration. When `inotify_add_watch` fails with `ENOSPC` (`fs.inotify.max_user_watches`, which is a
  per-user limit shared by all processes of the user), `fsnotifier` sends a `MESSAGE` once and carries on: the
  directories past the limit are not watched and not reported as unwatchable, while the watcher stays operational.
  **Verified** (source). The message reaches `FileWatcher.notifyOnFailure`, which logs a warning and publishes a
  notification of group `File Watcher Messages` on `Notifications.TOPIC` of the application message bus. **Verified**
  (source; not exercised).
- **Windows:** one thread per drive, `ReadDirectoryChangesW` recursive on the drive root
  (`native/WinFsNotifier/fileWatcher3.c`). Event order is kept within a drive, not across drives. **Inferred.**
- **Restarts:** if the process dies, `NativeFileWatcherImpl` restarts it up to 10 times and sends the roots again; it
  does not mark anything dirty on restart, so changes made while the process was down can be missed. **Verified**
  (source, `startupProcess(true)`). After 10 attempts `isOperational()` becomes `false`.
- **Setting roots** is asynchronous: `FileWatcher.setWatchRoots` runs on a single-thread executor, and `fsnotifier`
  registers the roots after that. Changes made before a new root is registered are not reported.
  `FileWatcher.isSettingRoots()` is `true` until both steps finish. **Verified** (source).

## How changes reach the VFS

```text
fsnotifier stdout line ("DIRTY <path>", "RECDIRTY", "CREATE", "RESET", ...)
  NativeFileWatcherImpl.MyProcessHandler          (reader thread, in order)
    FileWatcher.MyFileWatcherNotificationSink     maps the path to watch roots, adds it to DirtyPaths
LocalFileSystemImpl.storeRefreshStatusToFiles    every 1 s (scheduled only when the watcher is operational),
                                                 and synchronously from markSuspiciousFilesDirty
  marks the cached VirtualFiles dirty            nothing is refreshed yet
RefreshSessionImpl.scan                          for every refresh that is recursive or asynchronous
  LocalFileSystemImpl.markSuspiciousFilesDirty(queue)
    watcher operational:  drain DirtyPaths, mark manual roots dirty recursively
    watcher not operational: mark every queued file dirty recursively   (= full walk)
  RefreshWorker visits only dirty files and directories
```

Nothing refreshes the VFS on its own in a headless application: in the IDE, `SaveAndSyncHandler` refreshes on frame
activation. The caller must still start a refresh, but it can be a cheap one:

- `LocalFileSystem.getInstance().refreshFiles(files, false, true, null)` or
  `VirtualFileManager.getInstance().syncRefresh()` (all local roots) do not mark anything dirty themselves. With an
  operational watcher they visit only what the watcher reported; without it they fall back to the full walk of the given
  files (for `syncRefresh`, of every local root).
- `VfsUtil.markDirtyAndRefresh(...)` and MPS `IFile.refresh(DefaultCachingContext)` (which calls it) always mark the
  files dirty recursively, so they always walk everything, watcher or not.
- A synchronous non-recursive refresh is forced (`forceRefresh = !isRecursive && !isAsync` in
  `RefreshSessionImpl.scan`); use `recursive = true` to benefit from the watcher.
- A refresh only ever visits files whose children the VFS already loaded. Folders the application never listed are not
  walked by either kind of refresh.

MPS model reload works unchanged: the refresh produces VFS events and `ReloadManager.getInstance().flush()` applies
them. In a probe, an external edit of a model in a library folder outside the project was visible to the next MPS read
after a fence, `syncRefresh()` and `flush()`. **Observed.**

`com.intellij.mcpserver` (intellij-community `plugins/mcp-server/src/com/intellij/mcpserver/util/analysis.util.kt`) uses
the same idea for agents: `markSuspiciousFilesDirty(emptyList())`, then an asynchronous recursive refresh of the dirty
project base directories. It does not wait for pending watcher events.

## Seeing an edit made just before the operation

There is no flush API. `FileWatcher`, `NativeFileWatcherImpl` and `fsnotifier` accept only `ROOTS` and `EXIT` commands;
macOS `FSEventStreamFlushSync` is not reachable. **Verified.**

Measured latency between an external write and the first non-forced refresh that saw it (macOS, 2024.1.6): with the
stream idle, about 10 ms (the `NoDefer` flag delivers the first event at once); within 300 ms of another event anywhere
on the volume, 80–250 ms. **Observed.** A refresh started right after an edit therefore misses it.

A fence closes this race, because each `fsnotifier` reports its events in order and the Java side consumes them in
order:

```text
before refresh:
  create <fenceRoot>/<unique>/ and a file inside it     fenceRoot is under a recursive watch root
  wait until the watcher reports a path under <fenceRoot>/<unique>
  refresh (non-forced, recursive)
```

The fence must be a fresh directory with a file in it. Waiting for "the fence file is dirty" is not enough: a late event
for an earlier fence can mark it dirty, and on macOS events name directories, not files. A fresh directory can only be
named by events that happened after it was created. In 40 iterations of "write the fence file, wait until it is dirty"
one edit was missed. With fresh fence directories 0 of 200 iterations missed (40 each: a project folder on 2024.1.6, and
a library folder outside the project on 2024.1.6, 2025.1.4, 2026.1.1 and 263). **Observed** (macOS).

The only hook on raw watcher events found is `FileWatcher.myTestNotifier` (private, `volatile Consumer<String>`, set by
the `@TestOnly startup(Consumer)`), present with the same name and type in all four versions; the probes set it by
reflection. **Verified** (source) / **Observed**. A wrapping `FileWatcherNotificationSink` passed from an overridden
`initialize` would be an alternative, but that signature differs between 251 and 261.

Fence wait on macOS: median about 270 ms and at most 300 ms when anything changed on the volume in the previous 300 ms,
10–45 ms otherwise. On Linux the wait is bounded by the 50 ms loop sleep plus processing. **Inferred.** On Windows the
fence must be on the same drive as the files it guards. **Inferred.**

## Cost measurements

Warm OS caches, macOS arm64, MPS 2024.1.6, a project with 260 modules and 32,675 files (fully generated, including
`source_gen` and `classes_gen`). Per operation, after the first:

| Step                                                 | Forced walk (`markDirtyAndRefresh`) | Watcher, non-forced                 |
| ---------------------------------------------------- | ----------------------------------- | ----------------------------------- |
| refresh of the 260 module folders                    | 180–225 ms (median 210 ms)          | 1.1–1.3 ms                          |
| `VirtualFileManager.syncRefresh()` (all local roots) | —                                   | 9–20 ms                             |
| fence wait                                           | —                                   | 10–300 ms (median 270 ms when busy) |

The module-folder walk covers only what the VFS has loaded. A forced walk of folders whose children are all loaded costs
more: 1.0–1.3 s for 48,893 files and 1.8–1.9 s for 68,813 files (two other generated MPS project trees). **Observed.**

## Unresolved

- Linux and Windows were not exercised at runtime: the inotify limit message path, the effect of root re-registration on
  Linux (`update_roots` unregisters all roots and adds them again), and the cross-drive order on Windows.
- Whether a watcher restart after a crash can be detected without log scraping.
