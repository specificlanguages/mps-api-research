# Headless orchestration of IDE out-of-process tests

Which IDE test-execution APIs can a non-UI host reuse on MPS 2024.1, 2025.1, 2026.1, and development builds, and what
startup and reporting adjustments are required?

## Source-verified contracts

The source snapshots and revisions in [test-execution.md](test-execution.md) define the compatibility scope. Runtime
observations below use 2024.1.6, 2025.1.4, 2026.1.1, and development build 262.9437.476, on macOS/aarch64 with the
corresponding JetBrains runtimes.

`JUnitRunTypes.doCollect(JUnitSettings_Configuration, MPSProject, ProgressMonitor)` is public in the inspected release
lines. It invokes the same PROJECT, MODULE, MODEL, NODE, and METHOD collectors used by `collect`, without displaying
`collect`'s modal progress task. The caller must supply MPS model read access, matching the wrapper in `collect`.
Configuration references and node parent/identity reads also require model read access.

```text
JUnitRunTypes.doCollect
  ITestNodeWrapper + TestParameters
  TestsWithParameters.createFromTest2RunList
  TestsWithParametersAndConfiguration(repository, tests)
    CPCalculator.calculate → native execution classpath
    ProgramParametersCalculator.calculate → selectors or ExecutorScript
  native external executor
```

`CPCalculator` and `ProgramParametersCalculator` are package-private classes with constructors taking
`TestsWithParametersAndConfiguration`; `calculate()` is public. A host outside that package needs reflective access or a
package bridge. The calculators obtain the native dependency closure, macro values, project URL, classpath, and
selection serialization. Call them outside an enclosing model read action; they acquire model access internally. These
are internal APIs, not a stable embedding interface.

`TestsWithParameters` computes a maximum compatible parameter set and skips incompatible wrappers. Partitioning by
executor class, needs-MPS flag, compatibility mode, classpath, and additional JVM arguments avoids this omission, at the
cost of separate execution processes and changed cross-kind shared state.

The native parameterized-method route serializes only a generated method name. `JUnit5TestContributorConverter` uses
`DiscoverySelectors.selectMethod(Class, String)`, without parameter types. Methods requiring arguments therefore need a
whole-class selection; a host must not assume the METHOD route supplies their signatures.

## Plugin loading and test mode

**Verified:** `WithPlatformTestExecutor` constructs an `IdeaEnvironment` with `withTestModeOn()` in all inspected
release lines. `EnvironmentBase.init` sets `RuntimeFlags` to `TestMode.USUAL`; `RuntimeFlags.isTestMode()` is true for
that mode. Ordinary `DefaultTestExecutor` and `LegacyTestLauncher` do not create an MPS application.

**Verified:** `IdeaEnvironment.createIdeaApplication` defaults `idea.force.use.core.classloader` to true when
unspecified. A host requiring plugin classloaders should pass false before application initialization. In test mode it
initializes `GraphicsEnvironment` with headless false, then changes the system property to true before obtaining
`TestApplicationManager`. A startup headless property alone does not describe its effective AWT behavior.

**Observed on 2026.1.1:** An `ExecutorScript` using automatic plugin discovery, launched with the IDE-calculated
classpath, loaded only IDEA CORE and failed to locate the `jetbrains.mps.lang.test.junit5` solution. Adding the host's
loaded plugin paths and IDs as `PluginData` to the script's startup arguments allowed the native executor to initialize
its test launcher. This is proper IntelliJ plugin loading, not flattening all plugin JARs onto the boot classpath.

The 2025.1 Images exclusion and SVG failure mechanism remain scoped as described in
[mps-boot-classpath-and-svg.md](mps-boot-classpath-and-svg.md). Do not apply that workaround to other release lines by
inference.

## Development bootstrap requirement

**Verified in master source revision `49d37b63488a0a8e42eb0130cb867fd508f398ac`:** `Mps_Command` adds `lib/nio-fs.jar`
through `-Xbootclasspath/a`. The build 262.9437.476 distribution declares the same argument in `product-info.json`.
Native test parameters alone do not supply it. **Observed:** Without this bootstrap argument, the normal host failed
application initialization with `NoClassDefFoundError` for `com.intellij.platform.core.nio.fs.MultiRoutingFileSystem`.
Hosts must retain their normal distribution-classpath handling and supply the bootstrap requirement separately,
including for MPS-aware external executors.

## Reporting boundaries

**Verified:** `DefaultTestExecutionListener` receives native JUnit unique IDs, statuses, sources, skip reasons, and
exception details. Its output tokens omit much of that information: aborted tests use failure tokens, container
exceptions are omitted, and IDs are mostly generated class/method names. It also omits engine containers.

A callback observation hook can preserve those native results while retaining the listener's original callbacks and
executor behavior. A host that reports failed containers as failures will return a stronger result than the native
listener's process exit count. A hook that prints through redirected `System.out` must avoid its flush path:
`JUnit5TestExecutor.createTestExecutionListener` and `WithPlatformTestExecutor` explicitly suppress flushing because of
MPS-37852. Writing a separate observation channel avoids that coupling. Legacy executors can continue using
`TestEventMessage.parse`.

**Observed on 2026.1.1:** Whole-class parameterized selection retained two distinct invocation IDs through observation
of the native listener. METHOD selection of the same one-argument method failed native JUnit discovery. Native
BaseLanguage passing/failing tests, language and generator fixtures, whole-project mixed selection, and no-argument
method selection ran through the IDE external path. **Observed on 2024.1.6 and 2025.1.4:** The generator fixture's model
argument failed in native `BaseGeneratorTest.findModel` with `IllegalModelAccessError`. Its implementation resolves a
model reference without acquiring model read access; the inspected 2025.1 and 2026.1 source snapshots have the same
unresolved-access comment and implementation. A preceding successful run on a newer runtime does not establish that the
API is safe outside model access. **Inferred:** Repository cache population may affect the resolution path; this has not
been established by a controlled paired cache probe. This host deliberately preserved the native failure instead of
adding a lock around the test run.

## Evidence coordinates

Paths are relative to the MPS repository, using the snapshots in [test-execution.md](test-execution.md):

- `plugins/execution-configurations/junit/source_gen/jetbrains/mps/baseLanguage/unitTest/execution/settings/`:
  `JUnitRunTypes.java`, `JUnitSettings_Configuration.java`, and the scope collectors.
- `plugins/execution-configurations/junit/source_gen/jetbrains/mps/baseLanguage/unitTest/execution/client/`:
  `TestsWithParameters.java`, `TestsWithParametersAndConfiguration.java`, `CPCalculator.java`,
  `ProgramParametersCalculator.java`, `ClientExecutorScript.java`, and `TestDescriptorWrapper.java`.
- `plugins/execution-configurations/junit/launcher/source_gen/jetbrains/mps/baselanguage/unitTest/execution/launcher/`:
  `WithPlatformTestExecutor.java`, `DefaultTestExecutor.java`, `LegacyTestLauncher.java`,
  `DefaultTestExecutionListener.java`, `JUnit5TestExecutor.java`, and `JUnit5TestContributorConverter.java`.
- `workbench/mps-platform/jetbrains.mps.ide.platform/source_gen/jetbrains/mps/tool/environment/IdeaEnvironment.java`.
- `core/tool/environment/source_gen/jetbrains/mps/tool/environment/EnvironmentBase.java`.
- `core/smodel/source/jetbrains/mps/RuntimeFlags.java`.

The development bootstrap evidence is in
`plugins/execution-configurations/plugin/source_gen/jetbrains/mps/execution/configurations/implementation/plugin/plugin/Mps_Command.java`
and the distribution's `product-info.json` launch arguments.
