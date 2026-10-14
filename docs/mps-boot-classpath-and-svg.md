# Boot classpath and SVG plugin loading in MPS 2025.1–2026.1

Which JARs belong on the application boot classpath of an embedded or headless MPS process, and why can an SVG parser
error appear during startup? This note covers **MPS 2025.1.4** (build `251.28774.766`), **2025.3** (build
`253.28294.432`), and **2026.1.1** (build `261.25134.877`). The SVG failure was observed on 2025.1.4; the later versions
were inspected for packaging and startup-classpath differences.

## Use the declared boot classpath

**Verified:** Each inspected distribution has `product-info.json` with a `launch` entry containing
`bootClassPathJarNames`. The shipped `bin/mps.sh` enumerates those JARs in the same order, omitting only the trailing
`boot-macos.jar` from the Linux script. The list is version-specific. It is not equivalent to every JAR found under
`lib/`, and it does not include `lib/modules/*.jar` in 2025.1.4 or 2025.3.

```text
application boot classpath  ← product-info.json launch[].bootClassPathJarNames
module JARs outside boot    ← lib/modules/*.jar (2025.1.4 and 2025.3)
plugin JARs outside boot    ← plugins/*/lib/*.jar
```

| Distribution | Boot JARs | JARs outside the boot list                     |
| ------------ | --------- | ---------------------------------------------- |
| 2025.1.4     | 74        | `lib/modules/*.jar`; source and Javadoc JARs   |
| 2025.3       | 180       | `lib/modules/*.jar`; `lib/app-backend.jar`     |
| 2026.1.1     | 379       | `lib/intellij.spellchecker*.jar`; no `modules` |

The counts include `boot-macos.jar` in the generic archive's Linux/amd64 launch record, although that file is absent
from these generic archives. Some other named boot JARs are also absent. **Inferred:** a custom launcher should derive
classpath membership and order from the matching product-info launch record, then account explicitly for files missing
from its distribution rather than treating a directory sweep as the product's classpath contract. The inspected generic
archives expose a Linux/amd64 launch record. **Observed on macOS/aarch64:** an external launcher used that sole generic
record's existing boot JARs to start headless MPS 2025.1.4 and 2026.1.1 and run focused tests. The record therefore can
be useful outside its declared host in the generic archive, but this probe does not establish a Windows launch contract
or a general rule for mapping it to platform-specific installers.

## Why 2025.1.4 can log an SVG failure

**Verified:** In 2025.1.4, `com.intellij.platform.images.backend.svg.SvgParserDefinition` is in
`lib/modules/intellij.platform.images.backend.svg.jar`. Its module descriptor declares a dependency on plugin
`com.intellij.platform.images` and registers a parser definition for language `SVG`. The plugin's
`plugins/platform-images/lib/platform-images.jar` contains `org.intellij.images.fileTypes.impl.SvgLanguage` and declares
the SVG file type. Neither JAR is in `bootClassPathJarNames`.

**Observed on 2025.1.4:** A headless host that put every `lib/modules/*.jar` on the application classpath caused
`SvgParserDefinition` to be instantiated by `AppClassLoader` during background indexing. Without the images plugin JAR
on that classpath, construction logged `NoClassDefFoundError` for `SvgLanguage`. Adding the plugin JAR to the
application classpath changed the log to
`ImplementationConflictException: Language with ID 'SVG' is already registered`, naming `com.intellij.platform.images`
and `com.intellij` as conflicting plugins. The same selected MPS tests could reach execution in other runs, so the log
alone did not establish that startup always aborts.

**Inferred:** Flattening the module and plugin JARs onto the application classpath crosses the classloader boundary
declared by the SVG module's plugin dependency. The two observed error variants are consistent with that boundary being
lost. A runtime probe with the product-info boot classpath, leaving the module and plugin JARs to their normal loaders,
completed a selected MPS 2025.1.4 test without the SVG registration log. This supports the classloader explanation for
the observed conflict.

Test-mode logging is a separate failure amplifier. **Verified:** `EnvironmentBase` sets `RuntimeFlags` to test mode when
configured with `withTestModeOn()`, and `IdeaEnvironment` then creates `TestApplicationManager`; in ordinary mode it
creates `MPSHeadlessPlatformStarter`. **Observed on 2025.1.4:** under test mode, the SVG indexing log was wrapped in
`TestLoggerAssertionError` and could escape project opening before test discovery. In ordinary headless mode, the same
SVG conflict remained in the log while test discovery and execution completed. Changing application mode therefore
changes whether the log aborts startup; it does not repair the classpath conflict. With the product-info boot classpath,
the same test worker reaches execution in ordinary mode without the SVG log.

## Test-mode workers need a different classpath

**Verified for MPS 2025.1.4:** The shipped Ant `<launchtest>` task starts `LaunchTestWorker` in a forked JVM. Its
`createEnvironment()` calls `withTestModeOn()` before initializing `IdeaEnvironment`. `MpsLoadTask` builds that JVM's
classpath by recursively collecting JARs under the MPS `lib/` directory, including `lib/modules/`. This differs from the
product-info boot classpath. In test mode, `IdeaEnvironment` calls `TestApplicationManager.getInstance()`; ordinary mode
uses `MPSHeadlessPlatformStarter`.

**Verified in 2025.1.4:** `ScriptJUnit5Launcher` creates a project from test module files using
`ModuleFilesListProjectStrategy`, which collects module handles into an empty project.
`IdeaEnvironment.createEmptyProject` creates a temporary project file and calls `EnvironmentBase.openProject`. That
method calls `IdeaEnvironment.doOpenProject(File)` and then `flushAllEvents`. The IDEA opening path waits for indexes in
test mode. The temporary project therefore still exercises IDEA project opening and can encounter the SVG indexing
failure; see the stock-worker reproduction below. The SVG backend JAR being on the worker's classpath does not by itself
instantiate the parser: plugin activation and a parser lookup are also involved.

```text
<launchtest> -> LaunchTestWorker -> withTestModeOn() -> TestApplicationManager
             classpath: recursive lib/ JARs, including lib/modules/

headless boot -> product-info bootClassPathJarNames -> MPSHeadlessPlatformStarter
```

**Observed on MPS 2025.1.4 / JBR 21.0.8:** A headless `IdeaEnvironment` worker opening a project and selecting an
ordinary passing test produced these outcomes when its MPS classpath and test-mode flag were varied:

| MPS classpath variant                                                                  | IDEA test mode | Outcome                                                                         |
| -------------------------------------------------------------------------------------- | -------------- | ------------------------------------------------------------------------------- |
| Product-info boot JARs; no extra module JARs                                           | Off            | Test passed; no SVG registration log                                            |
| Same                                                                                   | On             | `InstanceNotRegisteredException: SettingsController` during application startup |
| Same, plus `intellij.platform.settings.local.jar`                                      | On             | Advanced to missing `KernelService`                                             |
| Same, plus settings and `intellij.platform.kernel.backend.jar`                         | On             | Advanced to a VCS action registration error for missing `VcsGroups`             |
| Same, plus every `lib/modules/*.jar` except `intellij.platform.images.backend.svg.jar` | On             | Test passed; no SVG registration log                                            |

**Verified:** `intellij.platform.settings.local.jar` and `intellij.platform.kernel.backend.jar` are under `lib/modules/`
in this distribution and absent from `bootClassPathJarNames`. The MPS module descriptors register
`SettingsControllerMediator` as the `SettingsController` application service and `BackendKernelService` as the
`KernelService` application service. The VCS group is declared by a platform VCS module descriptor.

**Inferred:** The test application path needs platform module registrations that the product-info boot classpath alone
does not supply. Adding a single missing JAR only exposes the next startup dependency. The passing `lib/modules` probe
establishes this for one 2025.1.4 test selection; it is not a recommended general classpath recipe and does not prove
that every test-mode fixture or MPS version works with the SVG JAR excluded.

## Test module loading across releases

**Verified in 2024.1 and 2025.1 sources:** `ScriptJUnit5Launcher` creates an MPS project with
`ModuleFilesListProjectStrategy` from the test module descriptor files. The strategy mines those descriptors with
`ModulesMiner` and fills an empty project with the resulting module handles. The worker does not open the test project's
disk project file for discovery. Test fixtures can still receive the original project directory separately.

**Verified in 2026.1 and local master revision `5d6bed88c917` (2026-09-22):** `LaunchTestWorker` instead registers test
module descriptors through `LibraryInitializer`, mines their module references, and passes `TestData` to
`ScriptJUnit5Launcher`. This path does not create a module-only project for discovery. `ModuleFilesListProjectStrategy`
remains in the source tree but is not called by this launcher.

```text
2024.1 / 2025.1  module files -> ModuleFilesListProjectStrategy -> temporary MPS project -> discovery
2026.1 / master  module files -> LibraryInitializer + ModulesMiner -> TestData -> discovery
```

**Observed in a focused 2025.1.4 probe:** A test-mode worker with the product-info boot JARs plus all
`lib/modules/*.jar`, including `intellij.platform.images.backend.svg.jar`, ran ordinary and language tests when it
created a project from a test module descriptor. The same worker configuration opening the disk project had failed while
indexing with the SVG registration error. This isolates project opening as a trigger in that fixture; the SVG JAR's
presence on the classpath alone did not cause the failure.

**Observed in a focused 2026.1.1 probe:** The same module-only project approach ran an ordinary test. This shows the
approach can still work with 2026.1.1 APIs, although that release's own test worker uses the module repository path
above. **Observed in a 2024.1.6 probe:** IDEA test-mode initialization stopped before project creation with
`You must run JVM with -Djava.system.class.loader=com.intellij.util.lang.PathClassLoader`. A module-only project does
not remove that version's JVM launcher requirement.

**Verified in 2025.1:** `TestParametersCache.initCachedValues` calls `environment.openProject(...)` and can close and
reopen the project for a test fixture. **Unresolved:** Avoiding disk-project opening during discovery does not establish
that every test will avoid the SVG indexing path. The focused module-only probes above do not establish equivalence to
Ant execution for arbitrary fixtures that open or reopen projects. Evidence:
`plugins/mps-testing/languages/lang.test/solutions/jetbrains.mps.lang.test.runtime/source_gen/jetbrains/mps/lang/test/runtime/TestParametersCache.java`,
project opening and reopening at lines 135–138 in the 2025.1 source snapshot.

## Ant plugin-selection probe

**Observed on 2025.1.4 / JBR 21.0.8-b895.146, macOS/aarch64:** The stock Ant `LaunchTestWorker` ran compiled ordinary,
native BaseLanguage, language, and generator fixtures with its default plugin configuration without SVG parser errors.
The language fixture opened its disk project. With `autoPluginDiscovery="true"` on the same Ant task, the worker logged
`SvgParserDefinition` failing to resolve `SvgLanguage`, including `TestLoggerAssertionError` during indexing. Its XML
test outcomes still matched the default-plugin run.

**Inferred:** Plugin selection affects whether this classloading path is exercised. A module-only discovery project
alone is not a general fix. An Ant-compatible launcher must account for the build's plugin configuration as well as its
classpath and project lifecycle. Test mode does not ensure every asynchronous platform error becomes a failed test.

The default-plugin Ant path also ran the fixture suite on 2024.1.6, 2026.1.1, and development build `262.9437.476`
without SVG parser errors. See [Ant worker compatibility](ant-test-worker-compatibility.md) for exact inputs, reporting
results, and limitations. These observations do not establish behavior for arbitrary plugin sets.

### The activating plugin is Images

**Observed on the same 2025.1.4 runtime:** Controlled plugin-selection probes identified **Images**, plugin ID
`com.intellij.platform.images`, distributed in `plugins/platform-images/lib/platform-images.jar`. Each worker asserted
`RuntimeFlags.isTestMode()` and reported `TestMode.USUAL`. The boot classpath and compiled test fixture were unchanged.

| Plugin configuration                 | Images state | SVG parser failure |
| ------------------------------------ | ------------ | ------------------ |
| Ant defaults                         | Absent       | No                 |
| Ant defaults plus only Images        | Enabled      | Yes                |
| Automatic discovery                  | Enabled      | Yes                |
| Automatic discovery, Images disabled | Disabled     | No                 |

The final configuration used `com.intellij.platform.images` in its isolated configuration directory's
`disabled_plugins.txt`. Its 13 test results matched the default-plugin run. In this mixed passing/failing fixture probe,
adding only Images aborted before XML reporting; automatic discovery logged the error while still producing test
results. The all-passing fixture below completed successfully even with only Images added. Thus an exit code or XML
report alone is not a sufficient signal for the presence of this asynchronous classloading error.

**Observed:** A smaller probe initialized the inherited test-mode environment, then requested the SVG parser before
calling the worker's test-discovery or execution code:

```java
FileTypeManager.getInstance().getFileTypeByExtension("svg");
Language svg = Language.findLanguageByID("SVG");
LanguageParserDefinitions.INSTANCE.forLanguage(svg);
```

With Images enabled, the file type was `SVG`, the language existed, and the parser request reproduced
`NoClassDefFoundError: org/intellij/images/fileTypes/impl/SvgLanguage`. With Images absent, the file type was unknown
and there was no registered SVG language, so no parser was requested. This probe did not open a test project.

**Verified and observed:** The SVG backend module in the core MPS plugin declares dependencies on Images and the XML and
language platform modules. The latter two were enabled in all four configurations. Images was loaded by a
`PluginClassLoader`; parser-instantiation errors identified `AppClassLoader` as the parser's classloader. The class
needed by the parser resides in the Images plugin JAR, outside that application classpath.

```text
Images enabled -> SVG language and dependent parser become available
indexing or explicit parser lookup -> instantiate SvgParserDefinition using AppClassLoader
AppClassLoader cannot resolve SvgLanguage from the Images plugin loader -> failure
```

**Conclusion:** Images activates the failing path in this configuration. The underlying defect is the classloader
boundary described above; the result does not establish that the Images plugin is generally defective in a normal MPS
launch. Indexing is one trigger, not a necessary cause: explicit parser lookup reproduces the failure without test
project opening. Disabling Images is a targeted avoidance for workers that do not need that plugin; it does not repair
SVG support. No change to test mode was needed.

Evidence: the distribution's Images `META-INF/plugin.xml`; `intellij.platform.images.backend.svg` in
`workbench/mps-workbench/source/META-INF/IdeaPlugin.xml` and `module-descriptors.xml` at revision
`f4d90532bcac5e0339b3161cec38abf49567cffb`; and the controlled runtime comparisons above.

## Effect on an otherwise passing Ant build

**Observed on 2025.1.4 / JBR 21.0.8-b895.146, macOS/aarch64:** An all-passing fixture contained one generator test and
one language test that opened a disk project. It was generated and compiled once with the distribution's `mps.make`,
then run unchanged through stock Ant `launchtests`, with `haltonfailure="true"` and the stock worker's test-mode
initialization. No custom worker, launcher adapter, or explicit parser lookup was used. Each run used fresh
configuration, system, and log directories.

| Plugin configuration                 | Runs | SVG error and TestLoggerAssertionError | Test results per run | Ant exit |
| ------------------------------------ | ---- | -------------------------------------- | -------------------- | -------- |
| Ant defaults                         | 1    | No                                     | 2 passed             | 0        |
| Ant defaults plus only Images        | 3    | Yes                                    | 2 passed             | 0        |
| Automatic discovery                  | 3    | Yes                                    | 2 passed             | 0        |
| Automatic discovery, Images disabled | 1    | No                                     | 2 passed             | 0        |

The six Images-enabled runs logged the SVG classloading error and actual `TestLoggerFactory$TestLoggerAssertionError`
exceptions, yet each ended with `BUILD SUCCESSFUL`. Logs included scanning errors reported as ignored, exceptions on
`DefaultDispatcher` workers, and exceptions on `AWT-EventQueue-0`. Test mode did turn error logging into exceptions;
those exceptions did not necessarily become test failures or fail the Ant invocation.

**Verified in 2025.1.4 sources:** `FailureDetector.executionFinished` records JUnit `FAILED` execution results and
`flushErrors` forwards those failures to the worker callback. It does not inspect a global collection of platform logger
errors. `WorkerBase.workFromMain` catches exceptions reaching its environment/work/cleanup control flow and exits
unsuccessfully; otherwise, after `failBuild` checks recorded worker errors, it exits with status 0.
`IdeaEnvironment.flushAllEvents` pumps the event queue with `invokeAndWait`; it does not itself collect every exception
previously thrown on another thread.

```text
SVG parser classloading error -> test logger throws TestLoggerAssertionError
  reaches a JUnit execution result or worker control flow -> can fail the Ant run
  stays on a background/EDT path without reaching either -> Ant can still succeed
```

**Conclusion:** A logged SVG error is not equivalent to a failed Ant task. The claim that test mode automatically fails
an otherwise passing run for every such error is contradicted by these probes. A propagated exception can still fail
startup, a test, or cleanup. Keeping test mode enabled does not guarantee that asynchronous errors are accounted for in
the test result.

**Observed:** A further 15 sequential runs with Images enabled passed. Running the same stock worker and all-passing
fixture in groups of three concurrent processes reproduced an abort: one of six runs failed before XML reporting; the
other five passed both tests. Every process had separate configuration, system, log, report, and Ant build files. No
instrumentation, custom worker, or forced parser lookup was used in this reproduction. Thus the SVG error can fail an
otherwise passing Ant build. These counts describe this experiment, not an estimated failure probability.

### The failing exception crosses the explicit event drain

**Observed in both the mixed-fixture abort and the stock all-passing abort:** The root cause was
`NoClassDefFoundError: org/intellij/images/fileTypes/impl/SvgLanguage`. Indexing wrapped and logged it through
`MergingQueueGuiExecutor`, producing `TestLoggerAssertionError: Failed to execute background index update task`. The
failure's suppressed rethrow stack passed through `FlushQueue.runNextEvent`,
`PlatformTestUtil.dispatchAllEventsInIdeEventQueue`, and `IdeaEnvironment.flushAllEvents`.

The caller stack was:

```text
LaunchTestWorker.work -> reflectively invoke ScriptJUnit5Launcher.launchTests
  EnvironmentBase.createProject(ModuleFilesListProjectStrategy)
    IdeaEnvironment.createEmptyProject
      EnvironmentBase.openProject
        IdeaEnvironment.flushAllEvents -> invokeAndWait
          PlatformTestUtil.dispatchAllEventsInIdeEventQueue
            queued indexing failure rethrown
```

`invokeAndWait` returned the failure to the worker as a `RuntimeException`; reflection wrapped it in
`InvocationTargetException`. `LaunchTestWorker.work` caught that exception, recorded `unexpected error`, and called
`failBuild`. With `haltonfailure="true"`, `WorkerBase.forceFailBuild` created `BuildFailureException` with exit value
`-13`, observed as worker process exit **243** on macOS. Ant itself exited **1** with `BUILD FAILED`. This happened
while creating the temporary discovery project, before collecting or executing the fixture's tests and before creating
its `FailureDetector`. A module-only discovery project therefore does not guarantee avoidance.

**Verified in the 2025.1.4 distribution bytecode:** `FlushQueue.runNextEvent` rethrows caught exceptions when
`Application.isUnitTestMode()` is true. `IdeEventQueue.processException` also rethrows in test mode.
`PlatformTestUtil.dispatchAllEventsInIdeEventQueue` dispatches events directly; it does not catch these assertion
errors. **Observed in successful runs:** The corresponding rethrow stack instead reached the ordinary
`EventDispatchThread.pumpOneEventForFilters` path, with an uncaught exception printed for `AWT-EventQueue-0`, while the
worker completed successfully.

**Inferred from the two complete stacks and the stock reproduction:** Scheduling determines whether the queued indexing
failure is handled inside MPS's explicit nested event drain, where it propagates to its waiting caller, or by the
ordinary AWT event loop, where it does not fail that caller. This is an exception-propagation race, not a difference in
the test assertions. Exact scheduling probabilities and all other possible propagation paths remain unverified.

### An extra event drain after the indexing wait

**Verified in 2025.1.4 sources:** `IdeaEnvironment.doOpenProject` already calls
`IndexingTestUtil.waitUntilIndexesAreReady` before returning to `EnvironmentBase.openProject`, which then calls
`flushAllEvents`. The observed failure occurred after this indexing-wait call. `IdeaEnvironment` is declared `final`, so
overriding `doOpenProject` in a subclass is not an available extension point.

**Inferred:** Adding another direct event-dispatch loop at this boundary can encounter the same throwing callback.
Letting the ordinary AWT loop process callbacks before continuing is a different approach, but a queue barrier alone has
not been shown to cover all indexing-completion callbacks or callbacks enqueued by other callbacks. Index readiness must
not be assumed to imply that every error-delivery callback has been consumed. A workaround based on that ordering
requires its own verification; no such workaround is established by these probes.

### Synchronous versus asynchronous logging

**Observed on 2025.1.4:** Five all-passing stock Ant runs (two sequential and three concurrent) with
`-Dintellij.platform.log.sync=true` still logged the SVG error and `TestLoggerAssertionError`, passed both tests, and
returned Ant exit 0. The stock failure above occurred without setting that property. Synchronous logging is therefore
neither necessary for this abort nor sufficient to make every such error fail the run.

**Verified in the matching distribution's `testFramework.jar`:** `TestLoggerFactory$TestLogger.error` calls the logged
error processor, emits diagnostics, and throws `TestLoggerAssertionError` directly on its calling thread when the
`RETHROW` action applies. The asynchronous log sink does not defer that throw to a logging thread. The logging property
can affect timing; these observations do not establish that it has no effect on reproduction frequency. They do
distinguish it from the event-dispatch mechanism demonstrated above.

These propagation probes cover 2025.1.4 only. They do not establish an equivalent defect in other MPS releases.

Source coordinates at revision `f4d90532bcac5e0339b3161cec38abf49567cffb`:

- `plugins/mps-testing/languages/junit5/source_gen/jetbrains/mps/lang/test/junit5/FailureDetector.java`:
  `executionFinished`, `flushErrors`.
- `core/tool/builder/source_gen/jetbrains/mps/tool/builder/WorkerBase.java`: `workFromMain`, `forceFailBuild`.
- `plugins/mps-testing/solutions/launcher/source_gen/jetbrains/mps/lang/test/launcher/LaunchTestWorker.java`: `work`,
  `failBuild`.
- `core/tool/environment/source_gen/jetbrains/mps/tool/environment/EnvironmentBase.java`: `openProject`.
- `workbench/mps-platform/jetbrains.mps.ide.platform/source_gen/jetbrains/mps/tool/environment/IdeaEnvironment.java`:
  `flushAllEvents`.

## Later packaging

**Verified:** In 2025.3 and 2026.1.1, `org.intellij.images.fileTypes.impl.SvgParserDefinition` and `SvgLanguage` are
both in `plugins/platform-images/lib/platform-images.jar`. The 2025.1.4 split between an IntelliJ module JAR and the
plugin JAR no longer applies. `product-info.json` still defines a selective boot classpath, and the exclusions in the
table remain reasons to avoid loading every `lib/*.jar` into the application loader.

**Unresolved:** Arbitrary broad-classpath/plugin combinations on 2025.3 and 2026.1.1 remain unverified. The passing
2026.1.1 Ant probe above establishes its tested configuration. The observed 2025.1.4 error must not be generalized to
other versions solely from their JAR lists.

## Evidence and reproduction

- [2024.1 `ScriptJUnit5Launcher`][script-junit-launcher-2024] and [2025.1 module-files strategy][module-files-strategy]:
  module-only project creation from test descriptor files. [2026.1 `LaunchTestWorker`][launch-test-worker-2026] and
  [2026.1 `ScriptJUnit5Launcher`][script-junit-launcher-2026]: library registration, `TestData`, and direct discovery.
  [Local master revision `5d6bed88c917` worker][launch-test-worker-master] has the same repository-based shape.
- Distribution `product-info.json` (`launch[].bootClassPathJarNames`) and `bin/mps.sh` for 2025.1.4, 2025.3, and
  2026.1.1. Compare the script's `CLASSPATH` assignments with the descriptor order, then compare both with JARs under
  `lib/`, `lib/modules/`, and `plugins/platform-images/lib/`.
- [2025.1.4 SVG module descriptor][svg-module]: parser registration and images-plugin dependency. The parser class is in
  `lib/modules/intellij.platform.images.backend.svg.jar`; the SVG language class and file-type registration are in
  `plugins/platform-images/lib/platform-images.jar` and its `META-INF/plugin.xml`.
- [2025.1.4 `EnvironmentBase`][environment-base] and [2025.1.4 `IdeaEnvironment`][idea-environment]: test-mode and
  ordinary headless application initialization.
- [2025.1.4 `ScriptJUnit5Launcher`][script-junit-launcher] and [module-files project strategy][module-files-strategy]:
  module-only project creation, compared with [IDEA disk-project opening][idea-environment].
- [2025.1.4 `LaunchTestTask`][launch-test-task], [MpsLoadTask][mps-load-task], and
  [MPSClasspathUtil][mps-classpath-util]: forked worker classpath construction. [LaunchTestWorker][launch-test-worker]
  enables test mode before IDEA startup. [Platform module descriptors][platform-modules] register the settings and
  kernel services and the VCS action group.
- Runtime probe: open an MPS project and run a compiled test selection in an isolated 2025.1.4 headless worker. Keep
  `lib/modules/*.jar` on the application classpath and vary whether `platform-images.jar` is appended. Inspect the
  worker log for the parser's reported classloader, the SVG cause, and whether discovery and execution occur. The
  classpath variants above were observed with the same MPS release and JBR 21.0.8. The test-mode variants in the table
  use the same worker, project, test selection, and product-info-derived base classpath, with only the listed module
  JARs added. Compare the report outcome and startup diagnostic, and inspect the worker log for SVG registration.

[svg-module]:
  https://github.com/JetBrains/MPS/blob/f4d90532bcac5e0339b3161cec38abf49567cffb/workbench/mps-workbench/source/META-INF/module-descriptors.xml#L995-L1005
[environment-base]:
  https://github.com/JetBrains/MPS/blob/f4d90532bcac5e0339b3161cec38abf49567cffb/core/tool/environment/source_gen/jetbrains/mps/tool/environment/EnvironmentBase.java#L45-L51
[idea-environment]:
  https://github.com/JetBrains/MPS/blob/f4d90532bcac5e0339b3161cec38abf49567cffb/workbench/mps-platform/jetbrains.mps.ide.platform/source_gen/jetbrains/mps/tool/environment/IdeaEnvironment.java#L190-L225
[launch-test-task]:
  https://github.com/JetBrains/MPS/blob/f4d90532bcac5e0339b3161cec38abf49567cffb/core/tool/ant/source_gen/jetbrains/mps/build/ant/junit/LaunchTestTask.java#L44-L57
[mps-load-task]:
  https://github.com/JetBrains/MPS/blob/f4d90532bcac5e0339b3161cec38abf49567cffb/core/tool/ant/source_gen/jetbrains/mps/build/ant/MpsLoadTask.java#L477-L490
[mps-classpath-util]:
  https://github.com/JetBrains/MPS/blob/f4d90532bcac5e0339b3161cec38abf49567cffb/core/tool/ant/source_gen/jetbrains/mps/build/ant/MPSClasspathUtil.java#L81-L116
[launch-test-worker]:
  https://github.com/JetBrains/MPS/blob/f4d90532bcac5e0339b3161cec38abf49567cffb/plugins/mps-testing/solutions/launcher/source_gen/jetbrains/mps/lang/test/launcher/LaunchTestWorker.java#L35-L42
[platform-modules]:
  https://github.com/JetBrains/MPS/blob/f4d90532bcac5e0339b3161cec38abf49567cffb/workbench/mps-workbench/source/META-INF/module-descriptors.xml
[script-junit-launcher]:
  https://github.com/JetBrains/MPS/blob/f4d90532bcac5e0339b3161cec38abf49567cffb/plugins/mps-testing/languages/junit5/source_gen/jetbrains/mps/lang/test/junit5/ScriptJUnit5Launcher.java#L32-L44
[module-files-strategy]:
  https://github.com/JetBrains/MPS/blob/f4d90532bcac5e0339b3161cec38abf49567cffb/plugins/mps-testing/languages/junit5/source_gen/jetbrains/mps/lang/test/junit5/ModuleFilesListProjectStrategy.java#L30-L44
[script-junit-launcher-2024]:
  https://github.com/JetBrains/MPS/blob/6236c4073eac3cde78506add6b0fa90601d76009/plugins/mps-testing/languages/junit5/source_gen/jetbrains/mps/lang/test/junit5/ScriptJUnit5Launcher.java#L29-L36
[launch-test-worker-2026]:
  https://github.com/JetBrains/MPS/blob/499c4a0fe6bfe7f95462c33966c72192158446ca/plugins/mps-testing/solutions/launcher/source_gen/jetbrains/mps/lang/test/launcher/LaunchTestWorker.java#L57-L86
[script-junit-launcher-2026]:
  https://github.com/JetBrains/MPS/blob/499c4a0fe6bfe7f95462c33966c72192158446ca/plugins/mps-testing/languages/junit5/source_gen/jetbrains/mps/lang/test/junit5/ScriptJUnit5Launcher.java#L70-L85
[launch-test-worker-master]:
  https://github.com/JetBrains/MPS-development/blob/5d6bed88c917c0df00b985e34fcfb3adb31c6d0d/plugins/mps-testing/solutions/launcher/source_gen/jetbrains/mps/lang/test/launcher/LaunchTestWorker.java#L57-L86
