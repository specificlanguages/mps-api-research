# Executing tests from MPS model nodes

How can an embedded MPS host discover and execute a selected test case or test method, with results and the correct MPS
environment?

MPS supplies model-node discovery through `jetbrains.mps.baseLanguage.unitTest.platform` and execution through JUnit.
Discovery produces generated Java class/method names and source references. Execution needs compiled test classes, the
owning modules' classloaders, the appropriate JUnit engines, and an MPS test session for MPS-aware tests.

## Versions and evidence scope

Source inspection performed on 2026-09-17:

| Line   | Exact source revision                      | Version declared by the testing plugin | Bundled Platform / Jupiter / Vintage |
| ------ | ------------------------------------------ | -------------------------------------- | ------------------------------------ |
| 2024.1 | `6236c4073eac3cde78506add6b0fa90601d76009` | 2024.1.7                               | 1.9.3 / 5.9.3 / 5.9.3                |
| 2025.1 | `f4d90532bcac5e0339b3161cec38abf49567cffb` | 2025.1.4                               | 1.9.3 / 5.9.3 / 5.9.3                |
| 2026.1 | `499c4a0fe6bfe7f95462c33966c72192158446ca` | 2026.1.1                               | 1.13.4 / 5.13.4 / 5.13.4             |
| master | `651a60b1cf3b890a2ae23f417734d94137962742` | 2026.2, platform build 262.9437.732    | 1.13.4 / 5.13.4 / 5.13.4             |

Master was checked against `JetBrains/MPS`'s remote `refs/heads/master`. Version labels above are source declarations,
not a claim that a corresponding binary release has shipped. The findings apply to these snapshots; every patch release
was not individually inspected. The test-platform and JUnit-launcher generated source directories in the 2026.1 and
master snapshots are identical.

This is source-verified research, not an end-to-end runtime probe. In particular, arbitrary editor tests, project reuse
in a custom host, cancellation, and execution under different outer locks have not been runtime-tested.

Version evidence: `plugins/mps-testing/META-INF/plugin.xml` and `build/version.properties` at each revision.

## API status

These APIs belong to bundled MPS modules and change between versions; they are not a version-independent execution
contract. The discovery and session classes inspected here have no stable-API annotation. `ProjectTestHelper` is
explicitly annotated `@ApiStatus.Experimental`. The version table and recipes below identify the compatibility
boundaries that an embedding host must account for.

## Discovery API

Classes in this section are in `jetbrains.mps.baseLanguage.unitTest.platform`.

| Operation                | 2024.1 and 2025.1                                                                | 2026.1 and master                                                                     |
| ------------------------ | -------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| Obtain platform          | `TestPlatform.getInstance()`                                                     | `environment.getPlatform().findComponent(TestPlatform.class)`                         |
| Discover one node        | Aggregate participant's `discover(node, request)`; also `request.discover(node)` | Aggregate participant's `discover(node, request)`; request convenience method removed |
| Participant registration | Singleton registration methods, called by runtime module activators              | `ModuleRuntime` extensions; runtime activator publishes the platform component        |
| Open/close session       | `openSession(TestSessionConfig)` / `closeSession(TestSession)`                   | Same signatures                                                                       |

The shared discovery surface is:

```java
TestDiscoveryRequest request = new TestDiscoveryRequest(new TestDescriptor());
Optional<TestDescriptor> result = testPlatform
    .getAggregateDiscoveryParticipant()
    .discover(selectedNode, request);
```

Run this while holding **MPS model read access**. Participants inspect properties, containment, references, and language
behavior; the shipped collectors call discovery inside MPS read actions. Pass an attached node: constructing
`SNodeTestSource` throws `IllegalArgumentException` if its model or module is absent.

Relevant signatures, common to the snapshots:

```java
TestDiscoveryParticipant TestPlatform.getAggregateDiscoveryParticipant();
Optional<TestDescriptor> TestDiscoveryParticipant.discover(SNode node, TestDiscoveryRequest request);
List<SAbstractConcept> TestDiscoveryParticipant.sourceConcepts();
TestDiscoveryRequest(TestDescriptor rootContainer);
TestDescriptor TestDiscoveryRequest.peekContainer();
String TestDescriptor.getFullName();
String TestDescriptor.getShortName();
boolean TestDescriptor.isContainer();
TestDescriptor TestDescriptor.getContainer();
List<TestDescriptor> TestDescriptor.getTests();
TestSource TestDescriptor.getSource();
<T> T TestDescriptor.getProperty(TestProperties.Key<T> key);
```

The signatures above are declarations summarized without modifiers. The `Optional` result represents discovery's
ordinary miss. Some implementations inconsistently annotate that `Optional` as nullable; inspected implementations
return an `Optional`. A root descriptor has no parent container. `getTests()` exposes the descriptor's mutable list.

For a test-case descriptor, `getFullName()` supplies the generated class name. For a method, its containing test case
supplies the class name and `getShortName()` supplies the method name. `SNodeTestSource.getNodeReference()` and
`getModuleReference()` retain identities for loading and reporting. Preserve these identities alongside names. Load the
class from the containing case's module. JUnit discovery includes inherited members, whose method source can belong to a
different module; the method's source reference remains useful for reporting, not class ownership.

Do not derive generated names from a node's displayed name. `LanguageTestDiscoveryParticipants` obtains names from
`ITestCase`/`ITestMethod` behavior; generator-test assertions use names such as `testTransformAndMatch0`.

The registered participants cover:

- BaseLanguage classes recognized as JUnit 3, 4, or 5 tests, including method discovery.
- `ITestCase` / `ITestMethod` implementations, including BaseLanguage unit tests and language tests. Abstract
  `BTestCase` nodes and cases without uncommented test methods are excluded.
- Generator-test roots, with child assertion descriptors.

For a module/model selection, enumerate non-stub model roots under read access and discover each root. For arbitrary
selected descendants, discover the enclosing test case and match child descriptors by source reference when the
participant does not support discovering that descendant directly.

The helper `jetbrains.mps.lang.test.junit5.TestDiscovery` is an unstable shortcut: its constructor takes only a visitor
in 2024.1, adds `ClassLoaderManager` in 2025.1, and adds `TestPlatform` while becoming package-private in 2026.1/master.
The public participant API is a better integration boundary.

Evidence: [2024.1 platform][platform24], [2026.1 platform][platform26], [discovery participants][participants], and
`LanguageTestDiscoveryParticipants`, `GeneratorTestDiscoveryParticipants`, `TestDescriptor`, and `SNodeTestSource` in
the source locations listed below.

## Load generated code before execution

Discovery is not generation or compilation. Generate and compile the selected test modules and their required
dependencies first, and ensure the resulting code is visible to MPS module classloading.

The current collector resolves the module in `MPSModuleRepository`, checks
`SModuleOperations.classesAvailableToMPS(module)`, and uses
`ClassLoaderManager.getClassLoader(module).loadOwnClass(className)` for explicit test selections. Its automatic
collector loads discovered classes through the module classloader. Missing classes are discovery/build errors, not
successful runs with no tests.

The 2026.1/master automatic collector warns about missing `TestsFacet` but still has compatibility fallbacks for modules
with test models or tests outside `@tests` models. A facet alone is therefore not a sufficient inventory of test nodes,
and the inspected implementation does not strictly require it for every explicit selection.

JUnit classes, launchers, extensions, and engines must share compatible class identities. MPS loads its launcher through
its **MPS solution classloader**, then temporarily sets the thread context classloader to that launcher's classloader so
JUnit engine discovery sees its dependencies. Loading a second copy of the platform classes directly onto a host
application's classpath can disconnect discovery, sessions, or JUnit interfaces.

The shipped bridge is `jetbrains.mps.tool.run.ModuleClassCode`:

```java
new ModuleClassCode("c234a56a-502f-4751-aded-6f9846fff7ce(jetbrains.mps.lang.test.junit5)");
```

`load(Platform, String)` locates the solution and class; `cons(Class<?>...)` and `instanceMethod(String, Class<?>...)`
return optional reflective members. `load` itself takes an MPS write action; do not invoke it from an outer MPS read
action. This observed write action is not proof that all classloading requires a write lock: the test collectors load
their test classes under read access.

Evidence: [TestDiscoveryContributor][contributor], [LaunchTestWorker][worker], and
`core/tool/builder/source_gen/jetbrains/mps/tool/run/ModuleClassCode.java`.

## Execute on 2024.1 and 2025.1

`jetbrains.mps.lang.test.junit5.AbstractJUnit5Launcher` provides:

```java
public AbstractJUnit5Launcher(Environment environment);
public void launchTestsWithSession(Collection<Class<?>> classes, TestExecutionListener listener);
public TestSessionConfig configureSession(TestSessionConfig config);
public abstract int launchTests();
```

Here `Environment` is `jetbrains.mps.tool.environment.Environment` and `TestExecutionListener` is
`org.junit.platform.launcher.TestExecutionListener`.

For an entire compiled test case, a small subclass can use `launchTestsWithSession`, override `configureSession` to set
`SystemProperties.PROJECT_PATH`, and supply a listener. That path does not dispose the supplied environment.
`SimpleJUnit5Launcher(Environment, Collection<Class<?>>)` also exists, but its default reporting writes XML into
`user.dir`; a dedicated subclass gives the host clearer control over results and reporting.

The launcher opens an MPS `TestSession` containing the environment, sets/restores the context classloader, executes
JUnit, and closes the MPS session in `finally`. Runtime module activation registers `EnvironmentAccessoryHandler`, which
injects the environment and project-path supplier into `EnvironmentAwareExtension` through static state. The launcher's
class-only entry point does not expose method selectors. For method selection, use a small JUnit adapter with
`DiscoverySelectors.selectMethod(testClass, generatedMethodName)` and the same MPS session setup.

Do **not** use the old `ScriptJUnit5Launcher(Script, Environment, jetbrains.mps.lang.test.launcher.WorkerCallback)` for
a persistent host. Its `launchTests()` creates a project, closes that project, and calls `environment.dispose()`. Its
cleanup is also not structured as a `finally` around the whole run.

The old `AbstractJUnit5Launcher` obtains a JUnit launcher from `LauncherFactory.openSession(...)` without closing that
JUnit session. If implementing a host adapter directly, close the JUnit `LauncherSession` as well as the MPS session and
restore the context classloader.

Evidence: [2024.1 abstract launcher][abstract24], [2025.1 abstract launcher][abstract25], [2024.1 script
launcher][script24], and [2025.1 script launcher][script25].

## Execute on 2026.1 and master

`jetbrains.mps.lang.test.junit5.ScriptJUnit5Launcher` is now usable with a supplied environment without disposing it:

```java
public ScriptJUnit5Launcher(Environment env, TestData tests, WorkerCallback callback, File projectDir);
public ScriptJUnit5Launcher(Environment env, JUnit5TestContributor tests, WorkerCallback callback, File projectDir);
public int launchTests();
public void launchTests(TestExecutionListener listener);
public void legacyXmlReport(File outputDir);
public void opentestReport(File outputDir);
public void teamcityReport();
```

`TestData` and `WorkerCallback` are now in `jetbrains.mps.tool.common`. `projectDir` is documented as optional; the
environment, test data, and callback are documented as non-null. The contributor interface is in
`jetbrains.mps.lang.test.junit5` and declares `List<DiscoverySelector> collectSelectors() throws Exception`.

For one selected case or method, the explicit test plan is:

```java
TestData plan = new TestData();
TestData.ModuleRecord module = new TestData.ModuleRecord(serializedModuleReference, false);
TestData.TestContainerRecord testCase = new TestData.TestContainerRecord(generatedClassName);
module.testCases.add(testCase);
plan.testModules.add(module);
// Leave tests empty for the whole class; otherwise use the discovered generated method name.
testCase.tests.add(new TestData.TestRecord(generatedMethodName));

new ScriptJUnit5Launcher(environment, plan, callback, projectDirectory).launchTests(listener);
```

This illustrates the call sequence; it is not a compiled integration example. Obtain names and the module reference
through discovery, and execute using the correct MPS/JUnit classloaders.

The launcher opens an MPS session, passes the environment as an accessory, and records the project path as a
session-local property. It also puts that session into the JUnit `LauncherSession` store under
`Namespace.create("MPS")`, key `"TestSession"`. `EnvironmentAwareExtension` and `TestParametersCacheExtension` retrieve
it from JUnit's launcher-session scope. Opening only the MPS session is **insufficient** on these versions. Both
sessions are closed and the context classloader restored by the shipped execution path.

`SimpleJUnit5Launcher` changed to `SimpleJUnit5Launcher(Collection<Class<?>>)`: it no longer provides an MPS session or
environment. It is not the replacement for the old environment-aware constructor.

The no-argument `launchTests()` returns a count of failed execution events and reports their text through the callback.
The listener overload delegates result tracking to the caller. Discovery can report `callback.fatal(...)` and proceed
with no selectors; record callback failures separately. A zero failure count alone does not prove that any test ran. For
structured output, record test/container failures, skipped and aborted tests, throwables, and discovered/executed counts
through the listener. Keep JUnit runtime identifiers alongside MPS source references; parameterized and dynamic tests
need more than a class/method pair for individual invocation identity.

Evidence: [2026.1 script launcher][script26], [master script launcher][scriptMaster], [current abstract
launcher][abstractMaster], [test plan][testData], and [fixture session integration][fixtureExtension].

## Existing-process execution, project lifecycle, and locks

An MPS test platform is not an interpreter for arbitrary nodes. Tests execute generated code and can need MPS project,
model, editor, typesystem, and generator services.

The discovery descriptors carry three inherited Boolean properties:

- `CAN_RUN_IN_PROCESS`, default `true`: eligibility for running inside an existing MPS process.
- `REQUIRES_MPS_PLATFORM`, default `false`: the test needs MPS services.
- `USE_COMPATIBILITY_MODE`, default `false`: legacy execution, notably JUnit 3.

These are independent properties. The JUnit class participants explicitly disable in-process execution. Language-test
participants derive the flags from concept behavior. Read the descriptor instead of hardcoding a list of safe test
concepts. The raw script launcher does not enforce the IDE's in-process eligibility checks.

The IDE's `InProcessExecutionFilter` additionally rejects a test model whose `TestInfo` requests reopening its project.
Its package-private `InProcessEnvironment` adapts an existing `Platform`: `openProject(File)` returns an already-open
matching project, `closeProject(Project)` is a no-op, and event flushing synchronizes with the UI thread. This is a
source pattern to reproduce in an embedding host, not a public instantiable class. Merely supplying a normal environment
does not promise reuse of the host's open project.

`TestParametersCache` calls `environment.openProject`, optionally closes/reopens it, and initializes a transient model
on the UI thread under an MPS write action. `ProjectTestHelper.asCommand` uses the UI thread plus an MPS command;
`asRead` takes MPS read access itself. Consequently:

- Use a worker thread for the overall test run; this matches `JUnitInProcessRunStarter`'s pooled-thread execution.
- Release discovery's MPS read action before execution. Holding it while a fixture waits for an EDT write action creates
  a deadlock risk. Do not wrap the full run in an MPS command or IDEA read/write action either.
- Treat this as an orchestration rule derived from the source paths, not a universal runtime-tested thread annotation.
  Tests and individual platform services may add their own constraints.
- Keep the environment, project, plugins, and module classloaders alive through session cleanup. Prevent module reload
  during a run through host scheduling, rather than holding a repository lock for the whole execution.
- Serialize host test sessions. The MPS platform tracks them as a LIFO stack; `closeSession` throws
  `IllegalStateException` when the supplied session is not the stack's current session. The older environment bridge
  also uses static state. Concurrent host sessions are not established as safe by the concurrent collection types.

The IDE temporarily sets `RuntimeFlags` to `TestMode.IN_PROCESS` and restores it after execution. A host aiming to match
that path should account for this global state as well. It supplies a separate JUnit 4 compatibility path for legacy
tests using `PushEnvironmentRunnerBuilder`. Do not assume that using Vintage alone reproduces that MPS-specific
initialization. Hosts that do not implement the compatibility path should report these tests as unsupported.

The JUnit 5 execution path has no general reliable cancellation mechanism here: the IDE's stop code explicitly notes
that its JUnit 5 `stopRun()` is a no-op. A separate process provides an enforceable timeout boundary and permits tests
that must reopen projects. `<launchtests>` / `LaunchTestWorker` demonstrates the isolated environment setup, although
its automatic module-level selection is not itself a single-node CLI API.

Evidence: [IDE runner][inProcess], [eligibility filter][filter], [existing-project adapter][environment], [fixture
lifecycle][fixture], and [command/read helpers][helper].

## Packaging and initialization

These are bundled plugin/module APIs, not all classes in `lib/mps-core.jar`:

| Capability                                    | Module/JAR                                         | Plugin                                    |
| --------------------------------------------- | -------------------------------------------------- | ----------------------------------------- |
| Discovery descriptors and sessions            | `jetbrains.mps.baseLanguage.unitTest.platform.jar` | `mps-testing`, ID `jetbrains.mps.testing` |
| JUnit discovery and environment extension     | `jetbrains.mps.baseLanguage.unitTest.runtime.jar`  | `mps-testing`                             |
| Language-test discovery and fixtures          | `jetbrains.mps.lang.test.runtime.jar`              | `mps-testing`                             |
| Script/abstract launcher                      | `jetbrains.mps.lang.test.junit5.jar`               | `mps-testing`                             |
| JUnit Platform, Jupiter, Vintage              | JUnit libraries under `plugins/mps-junit5/lib`     | ID `jetbrains.mps.junit5`                 |
| IDE run configurations and in-process wrapper | Execution configuration modules                    | `execution-configurations`                |

Initialize the relevant plugins and runtime modules before requesting discovery or sessions. On 2024.1/2025.1 an
existing singleton with no registered participants can yield empty discovery; on 2026.1/master the component is
published by `unitTest.runtime`'s activator. Existence of a JAR on disk does not establish either condition.

The shipped command-line worker initializes an `IdeaEnvironment` with test mode enabled. This establishes a supported
source path for headless launch; it does not prove that every editor test works in every headless configuration, or that
a core-only `MpsEnvironment` supports the same test population. Test fixtures and their required plugins determine the
actual service requirements.

## Source coordinates

All paths below are relative to `JetBrains/MPS`; use the revisions in the version table for comparisons. Generated Java
is committed source and exposes the exact Java signatures used above.

- Platform API:
  `plugins/mps-testing/languages/baseLanguage/unitTest/solutions/platform/source_gen/jetbrains/mps/baseLanguage/unitTest/platform/`.
- JUnit participants, activator, environment bridge:
  `plugins/mps-testing/languages/baseLanguage/unitTest/runtime/source_gen/jetbrains/mps/baseLanguage/unitTest/runtime/`.
- Launchers and collectors: `plugins/mps-testing/languages/junit5/source_gen/jetbrains/mps/lang/test/junit5/`.
- Language-test participants and fixtures:
  `plugins/mps-testing/languages/lang.test/solutions/jetbrains.mps.lang.test.runtime/source_gen/jetbrains/mps/lang/test/runtime/`.
- Generator participants:
  `plugins/mps-testing/languages/gentest.rt/source_gen/jetbrains/mps/lang/test/generator/rt/GeneratorTestDiscoveryParticipants.java`.
- Ant worker:
  `plugins/mps-testing/solutions/launcher/source_gen/jetbrains/mps/lang/test/launcher/LaunchTestWorker.java`.
- IDE eligibility and contributors:
  `plugins/execution-configurations/junit/source_gen/jetbrains/mps/baseLanguage/unitTest/execution/server/`.
- IDE runner/environment:
  `plugins/execution-configurations/plugin/source_gen/jetbrains/mps/execution/configurations/implementation/plugin/plugin/`.

[platform24]:
  https://github.com/JetBrains/MPS/blob/6236c4073eac3cde78506add6b0fa90601d76009/plugins/mps-testing/languages/baseLanguage/unitTest/solutions/platform/source_gen/jetbrains/mps/baseLanguage/unitTest/platform/TestPlatform.java
[platform26]:
  https://github.com/JetBrains/MPS/blob/499c4a0fe6bfe7f95462c33966c72192158446ca/plugins/mps-testing/languages/baseLanguage/unitTest/solutions/platform/source_gen/jetbrains/mps/baseLanguage/unitTest/platform/TestPlatform.java
[participants]:
  https://github.com/JetBrains/MPS/blob/651a60b1cf3b890a2ae23f417734d94137962742/plugins/mps-testing/languages/baseLanguage/unitTest/runtime/source_gen/jetbrains/mps/baseLanguage/unitTest/runtime/JUnitTestDiscoveryParticipants.java
[contributor]:
  https://github.com/JetBrains/MPS/blob/651a60b1cf3b890a2ae23f417734d94137962742/plugins/mps-testing/languages/junit5/source_gen/jetbrains/mps/lang/test/junit5/TestDiscoveryContributor.java
[worker]:
  https://github.com/JetBrains/MPS/blob/651a60b1cf3b890a2ae23f417734d94137962742/plugins/mps-testing/solutions/launcher/source_gen/jetbrains/mps/lang/test/launcher/LaunchTestWorker.java
[abstract24]:
  https://github.com/JetBrains/MPS/blob/6236c4073eac3cde78506add6b0fa90601d76009/plugins/mps-testing/languages/junit5/source_gen/jetbrains/mps/lang/test/junit5/AbstractJUnit5Launcher.java
[abstract25]:
  https://github.com/JetBrains/MPS/blob/f4d90532bcac5e0339b3161cec38abf49567cffb/plugins/mps-testing/languages/junit5/source_gen/jetbrains/mps/lang/test/junit5/AbstractJUnit5Launcher.java
[script24]:
  https://github.com/JetBrains/MPS/blob/6236c4073eac3cde78506add6b0fa90601d76009/plugins/mps-testing/languages/junit5/source_gen/jetbrains/mps/lang/test/junit5/ScriptJUnit5Launcher.java
[script25]:
  https://github.com/JetBrains/MPS/blob/f4d90532bcac5e0339b3161cec38abf49567cffb/plugins/mps-testing/languages/junit5/source_gen/jetbrains/mps/lang/test/junit5/ScriptJUnit5Launcher.java
[script26]:
  https://github.com/JetBrains/MPS/blob/499c4a0fe6bfe7f95462c33966c72192158446ca/plugins/mps-testing/languages/junit5/source_gen/jetbrains/mps/lang/test/junit5/ScriptJUnit5Launcher.java
[scriptMaster]:
  https://github.com/JetBrains/MPS/blob/651a60b1cf3b890a2ae23f417734d94137962742/plugins/mps-testing/languages/junit5/source_gen/jetbrains/mps/lang/test/junit5/ScriptJUnit5Launcher.java
[abstractMaster]:
  https://github.com/JetBrains/MPS/blob/651a60b1cf3b890a2ae23f417734d94137962742/plugins/mps-testing/languages/junit5/source_gen/jetbrains/mps/lang/test/junit5/AbstractJUnit5Launcher.java
[testData]:
  https://github.com/JetBrains/MPS/blob/651a60b1cf3b890a2ae23f417734d94137962742/core/tool/common/source_gen/jetbrains/mps/tool/common/TestData.java
[fixtureExtension]:
  https://github.com/JetBrains/MPS/blob/651a60b1cf3b890a2ae23f417734d94137962742/plugins/mps-testing/languages/lang.test/solutions/jetbrains.mps.lang.test.runtime/source_gen/jetbrains/mps/lang/test/runtime/TestParametersCacheExtension.java
[inProcess]:
  https://github.com/JetBrains/MPS/blob/651a60b1cf3b890a2ae23f417734d94137962742/plugins/execution-configurations/plugin/source_gen/jetbrains/mps/execution/configurations/implementation/plugin/plugin/JUnitInProcessRunStarter.java
[filter]:
  https://github.com/JetBrains/MPS/blob/651a60b1cf3b890a2ae23f417734d94137962742/plugins/execution-configurations/junit/source_gen/jetbrains/mps/baseLanguage/unitTest/execution/server/InProcessExecutionFilter.java
[environment]:
  https://github.com/JetBrains/MPS/blob/651a60b1cf3b890a2ae23f417734d94137962742/plugins/execution-configurations/plugin/source_gen/jetbrains/mps/execution/configurations/implementation/plugin/plugin/InProcessEnvironment.java
[fixture]:
  https://github.com/JetBrains/MPS/blob/651a60b1cf3b890a2ae23f417734d94137962742/plugins/mps-testing/languages/lang.test/solutions/jetbrains.mps.lang.test.runtime/source_gen/jetbrains/mps/lang/test/runtime/TestParametersCache.java
[helper]:
  https://github.com/JetBrains/MPS/blob/651a60b1cf3b890a2ae23f417734d94137962742/plugins/mps-testing/languages/lang.test/solutions/jetbrains.mps.lang.test.runtime/source_gen/jetbrains/mps/lang/test/runtime/ProjectTestHelper.java
