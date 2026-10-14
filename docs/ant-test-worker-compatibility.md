# Reusing Ant test execution with model selection

Can a headless client retain the shipped Ant worker's IDEA test mode and execution machinery while selecting a model,
module, or set of project modules? This note covers MPS 2024.1.6, 2025.1.4, 2026.1.1, and a pinned 2026.2 development
build, with a separate source check of master.

**Observed:** A small launcher subclass can filter a model's test classes while retaining the Ant launcher's MPS
session, JUnit engines, XML reporting, and failure detector on all four tested distributions. Module selection already
exists in the worker's `Script` input. Project selection requires the caller to supply the intended set of module files;
the worker does not derive the build's test-module inventory from a project path.

## Versions and evidence

Probes ran on 2026-10-01 using generic distributions on macOS/aarch64:

| MPS runtime                              | Distribution source revision               | JBR                |
| ---------------------------------------- | ------------------------------------------ | ------------------ |
| 2024.1.6, build `241.19416.1452`         | `384c6dcb02fdddafcc413a91b310624a3eb765b3` | `17.0.12-b1207.37` |
| 2025.1.4, build `251.28774.766`          | `f4d90532bcac5e0339b3161cec38abf49567cffb` | `21.0.8-b895.146`  |
| 2026.1.1, build `261.25134.877`          | `499c4a0fe6bfe7f95462c33966c72192158446ca` | `25.0.3-b329.124`  |
| 2026.2 development, build `262.9437.476` | `f1e6b1ed275d0bd5fa9ee9db176b2b21095215d9` | `25.0.3-b508.16`   |

**Verified:** Public master resolved to `49d37b63488a0a8e42eb0130cb867fd508f398ac` on 2026-10-01. Its
`LaunchTestWorker`, `ScriptJUnit5Launcher`, `AbstractJUnit5Launcher`, `TestDiscoveryContributor`, `LaunchTestTask`, and
`MpsLoadTask` source files were byte-for-byte identical to the 2026.1.1 versions. This is API/source evidence for that
master snapshot, not runtime validation of a binary built from that exact revision.

The 2024.1.6 source revision above is distinct from the 2024.1.7 snapshot used in parts of
[test-execution.md](test-execution.md). In 2024.1.6, `ScriptJUnit5Launcher.launchTests()` calls
`WorkerCallback.failBuild()` before returning; a failed suite can throw at that point. Callers must handle both
exceptions and returned failure counts.

## Reuse boundary

```text
Ant LaunchTestTask
  constructs classpath, JVM options, and Script
    LaunchTestWorker / small subclass
      inherited createEnvironment(): IDEA test mode
      stock module loading and discovery
        ScriptJUnit5Launcher / small subclass
          filter selected model's classes
          stock MPS session + JUnit execution + failure detector + reports
```

**Verified:** `LaunchTestWorker.createEnvironment()` uses `withTestModeOn()` across the inspected releases. Its
`WorkerBase` superclass handles script-configured libraries, plugins, macros, and process lifecycle. The worker's
`work()` method instantiates a fixed launcher class; there is no public launcher-factory override. Replacing that method
is one way to insert a launcher subclass while inheriting the environment setup.

**Observed:** The probe loaded its adapter classes with the stock launcher's MPS module classloader as parent. Merely
adding another JUnit copy to the boot classpath would not establish compatible class identities.

### MPS 2024.1.6 and 2025.1.4

**Verified and observed:** A subclass of `ScriptJUnit5Launcher(Script, Environment, WorkerCallback)` can override:

```java
public void launchTestsWithSession(Collection<Class<?>> classes, TestExecutionListener listener);
```

The stock `launchTests()` creates its module-only project and discovers test classes before calling this method. The
subclass can retain classes belonging to the selected model and call `super.launchTestsWithSession(filtered, listener)`.
The stock implementation then owns session creation, engine configuration, execution, reporting, and normal cleanup.
`WorkerCallback` on these releases is in `jetbrains.mps.lang.test.launcher`.

The probe obtained the selected model's generated test classes through `TestPlatform` discovery under model read access
and the owning module's classloader. It intersected these class identities with the stock-discovered collection. This
avoids guessing that a model name is a Java package name. A production selector should use a resolved model reference;
the focused probe used a unique model name.

**Limitation:** Filtering occurs after the stock launcher discovers and loads classes from the whole supplied module. An
error loading an unselected class can therefore still be reported. Build the owning module before execution and make
this limitation explicit for no-build runs. This hook does not provide discovery isolation at model granularity.

### MPS 2026.1.1 and the development build

**Verified and observed:** The subclass instead receives `Environment`, `TestData`,
`jetbrains.mps.tool.common.WorkerCallback`, and the optional project directory. Its interception point is:

```java
protected void launchTestsWithSession(List<DiscoverySelector> tests, TestExecutionListener listener);
```

The probe used the stock worker's library-registration sequence and automatic `TestData.ModuleRecord` discovery, then
retained `ClassSelector` entries for classes discovered in the selected model. It called the superclass method to
preserve MPS/JUnit session setup, including the JUnit session-store integration. The default no-argument `launchTests()`
continued to install the stock failure detector. These releases also support an explicit `TestData` plan, but the probe
kept automatic discovery to compare directly with the Ant baseline.

**Inferred:** Two small version-specific launcher adapters cover the observed API boundary. Supporting model-level
selection does not require method selectors, the IDE run-configuration machinery, or a separately implemented JUnit
execution loop.

## Runtime comparison

The fixture contained native BaseLanguage tests, an ordinary JUnit 3 test, Jupiter passing/failing/disabled/aborted
cases, a failing `BeforeAll` container, a parameterized method with two invocations, a language `NodesTestCase`, and a
generator assertion over an empty model. Interrupting and deliberately slow fixtures were excluded. The language fixture
used `TestParametersCache` to open its project.

Each distribution's `mps.make` task generated and compiled a separate fixture copy. The stock module run and filtered
model runs reused that same compiled output. The development fixture used that distribution's Jupiter stub-module
reference.

**Observed on each of the four distributions:**

| Run                                 | XML test cases | Outcome                                                                         |
| ----------------------------------- | -------------- | ------------------------------------------------------------------------------- |
| Stock Ant worker, whole test module | 13             | 8 passing, 2 skipped/aborted, 2 assertion failures, 1 setup error; nonzero exit |
| Adapter, language-test model        | 1              | Passing; zero exit                                                              |
| Adapter, ordinary-test model        | 9              | 5 passing, 2 skipped/aborted, 1 assertion failure, 1 setup error; nonzero exit  |

The `(class, test name, XML status)` tuples from each model run exactly matched the corresponding subset of its stock
Ant run. Each adapter asserted `RuntimeFlags.isTestMode()` and printed `RuntimeFlags.getTestMode() == USUAL`. The
fixture had one test module; mapping a multi-module project into a script was not runtime-tested.

The probe wrapped the supplied failure listener to observe execution events and forwarded events to it. This preserved
`FailureDetector`, including the 2026.1 test-logger completion hook. This demonstrates a place to add structured
results; it does not establish durable partial-report storage or cancellation behavior. The prototype's exception
wrapping changed some numeric failure exit codes; only zero/nonzero outcome equivalence was checked.

## Startup and plugin configuration

**Observed:** Ant's own `LaunchTestTask` supplied the worker classpath and module opens; no `product-info.json` launch
record was used. No images-plugin JAR was appended and no SVG module JAR was removed. The harness supplied assertions,
headless mode, a 2 GiB heap, isolated config/system/log directories, app-info loading, and
`-Djna.boot.library.path=<mps>/lib/jna/aarch64`. The make task needed the javac API module open. The 2024.1.6 test
worker also used `-Djava.system.class.loader=com.intellij.util.lang.PathClassLoader`.

A compact Ant test invocation has this shape, with the distribution's Ant tasks already registered:

```xml
<launchtests mpshome="${mps.home}" haltonfailure="true" reports="${reports}">
  <jvmargs>
    <arg value="-ea" />
    <arg value="-Xmx2048m" />
    <arg value="-Djava.awt.headless=true" />
    <arg value="-Didea.config.path=${test.config}" />
    <arg value="-Didea.system.path=${test.system}" />
  </jvmargs>
  <project path="${project.dir}" />
  <testmodules>
    <fileset dir="${project.dir}/solutions/tests" includes="tests.msd" />
  </testmodules>
</launchtests>
```

Add the version/host-specific JVM options described above. The test-module descriptor and its dependencies must already
be generated and compiled. Project and library inputs must reflect the intended build configuration.

**Observed:** With the default plugin configuration, all four runtime probes avoided SVG parser errors. In 2025.1.4,
setting `autoPluginDiscovery="true"` on the stock Ant task reproduced `SvgParserDefinition` failing to resolve
`SvgLanguage`, including `TestLoggerAssertionError` during indexing. The 13 XML outcomes nevertheless matched the
default-plugin run. An all-passing two-test fixture also returned Ant exit 0 in six Images-enabled runs despite actual
`TestLoggerAssertionError` exceptions. Test mode does not guarantee that every asynchronous platform error is reflected
in test results or fails the Ant invocation. Concurrent stock-worker runs of the same all-passing fixture also
reproduced an SVG-caused abort before test discovery: the queued indexing exception escaped the explicit event drain
during temporary-project creation. Successful runs with logged errors do not establish safety. See the
[all-passing comparison](mps-boot-classpath-and-svg.md#effect-on-an-otherwise-passing-ant-build).

**Inferred:** Plugin selection is part of the worker startup contract. Avoiding disk-project opening during discovery
alone does not explain or guarantee avoidance of the SVG failure. Preserve the default and explicitly required plugin
configuration when reproducing an Ant build; automatically enabling every installed plugin changes that environment. A
controlled 2025.1.4 probe identified Images (`com.intellij.platform.images`) as sufficient to activate the failure:
adding only Images to the defaults reproduced it, and disabling Images under automatic discovery avoided it. Explicit
SVG parser lookup also reproduced it before test-project opening. See
[the SVG classpath note](mps-boot-classpath-and-svg.md#the-activating-plugin-is-images).

## Automatic discovery on development build 262.9437.476

**Observed on build `262.9437.476` / JBR `25.0.3-b508.16`, macOS/aarch64:** an Ant-compatible worker with
`autoPluginDiscovery="true"`, an isolated configuration, and no `plugin.path` loaded only IDEA CORE. Its repository
lacked `jetbrains.mps.lang.test.junit5`, so the launcher module lookup failed before discovery. Supplying the installed
plugin directories as a platform-separated `-Dplugin.path` list let normal plugin loading register the launcher module;
the selected language fixture then passed with `RuntimeFlags.getTestMode() == USUAL` and one XML test case. The generic
Linux distribution's JCEF plugin was rejected as incompatible with the macOS host; that rejection did not fail this
stock test-mode startup. These observations do not establish automatic-discovery behavior on other builds or hosts.

**Verified in the master source snapshot identified above:** `EnvironmentConfig.withAutomaticPluginDiscovery()` sets the
automatic-mode flag. `IdeaEnvironment.addRequiredPlugins()` returns without assigning `plugin.path` or restricting
plugin IDs in automatic mode. Explicit directory paths can therefore be supplied by the caller; they identify plugin
locations, rather than adding plugin JARs to the application classpath. The default-plugin probes above exercised
explicit plugin configuration instead of this automatic mode.

```text
Ant launchtests: automatic discovery + plugin.path=<installed plugin directories>
  LaunchTestWorker.createEnvironment(): withTestModeOn()
    IdeaEnvironment.addRequiredPlugins(): retain caller's plugin.path
      TestApplicationManager: load plugin descriptors and register language libraries
        launcher module classloader: load ScriptJUnit5Launcher
```

The matching source entry points are
`core/tool/environment/source_gen/jetbrains/mps/tool/environment/EnvironmentConfig.java` and
`workbench/mps-platform/jetbrains.mps.ide.platform/source_gen/jetbrains/mps/tool/environment/IdeaEnvironment.java`. The
runtime comparison held the already compiled fixture, worker classes, classpath, and model selector fixed; only
`plugin.path` changed. Its Ant file used the same test-module descriptor as both the repository library and test-module
input, in both runs.

## Limits and integration guidance

- Retain test mode and the selected distribution's Ant startup machinery; the normal application boot classpath is a
  separate contract.
- Prefer the stock failure detector and reporting hooks. Differences between release implementations are observed
  behavior, not evidence that those differences are intentional or correct.
- Module selection can pass descriptor files directly. Project selection needs the caller's intended module inventory;
  it cannot be inferred from the mere presence of a project-directory argument.
- A plain JUnit 3 success does not validate environment injection into arbitrary legacy `EnvironmentAware` classes.
  These probes preserve Ant's behavior; they do not establish parity with the IDE's separate compatibility runner.
- The results establish these fixtures on macOS/aarch64. Windows/Linux, arbitrary plugins, multiple-module dependency
  closure, project reopening, cancellation, and timeout behavior remain unverified here.
- Supporting master requires a pinned runtime test in addition to checking source/API compatibility. A rolling branch
  name alone does not identify a tested distribution.

## Source coordinates

Use the revisions above with these paths in `JetBrains/MPS`:

- `core/tool/ant/source_gen/jetbrains/mps/build/ant/junit/LaunchTestTask.java`
- `core/tool/ant/source_gen/jetbrains/mps/build/ant/MpsLoadTask.java`
- `core/tool/builder/source_gen/jetbrains/mps/tool/builder/WorkerBase.java`
- `core/tool/builder/source_gen/jetbrains/mps/tool/run/ModuleClassCode.java`
- `plugins/mps-testing/solutions/launcher/source_gen/jetbrains/mps/lang/test/launcher/LaunchTestWorker.java`
- `plugins/mps-testing/languages/junit5/source_gen/jetbrains/mps/lang/test/junit5/ScriptJUnit5Launcher.java`
- `plugins/mps-testing/languages/junit5/source_gen/jetbrains/mps/lang/test/junit5/AbstractJUnit5Launcher.java`
- `plugins/mps-testing/languages/junit5/source_gen/jetbrains/mps/lang/test/junit5/TestDiscoveryContributor.java`
- `plugins/mps-testing/languages/junit5/source_gen/jetbrains/mps/lang/test/junit5/FailureDetector.java`
- `plugins/mps-testing/languages/lang.test/solutions/jetbrains.mps.lang.test.runtime/source_gen/jetbrains/mps/lang/test/runtime/TestParametersCache.java`

Runtime source identities come from each distribution's `build.properties`. The model-filter interception points were
compiled against each tested distribution and exercised, including the development build.

## Development distribution bootstrap classpath

**Verified:** Build `262.9437.476` declares `-Xbootclasspath/a:$IDE_HOME/lib/nio-fs.jar` in its product-info launch
entry's `additionalJvmArguments`. Its `bin/mps.sh` also supplies that argument. The jar is separate from
`bootClassPathJarNames`, which describes the application classpath. A launcher that uses the latter list must also
preserve the declared bootstrap classpath argument and resolve `$IDE_HOME` to the distribution home.

**Observed:** A normal IDEA-mode daemon using only the declared application jars failed during platform startup with
`NoClassDefFoundError: com/intellij/platform/core/nio/fs/MultiRoutingFileSystem`. This occurs before test selection or
Ant worker startup. Applying the declared bootstrap argument restored daemon startup; all four targeted development
integration checks passed (Ant test mode, language/generator tests, two-module project inventory, and ordinary JUnit
behavior). The independently constructed Ant worker did not reproduce that failure in the direct probes.

```text
Normal daemon JVM
  product-info bootClassPathJarNames → application classpath
  additionalJvmArguments -Xbootclasspath/a → bootstrap classpath
    nio-fs.jar → MultiRoutingFileSystem
      IntelliJ platform filesystem startup
```
