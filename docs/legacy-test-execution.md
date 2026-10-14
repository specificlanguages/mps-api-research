# Preparing legacy MPS tests alongside JUnit Platform tests

How can a host execute MPS tests whose discovery descriptor requests `USE_COMPATIBILITY_MODE`, while preserving the MPS
environment and selecting individual methods?

## Verified source contract

The contract below was inspected in MPS 2024.1 sources at `6236c4073eac3cde78506add6b0fa90601d76009`, MPS 2025.1.4 at
`f4d90532bcac5e0339b3161cec38abf49567cffb`, MPS 2026.1.1 at `499c4a0fe6bfe7f95462c33966c72192158446ca`, and master at
`ad595c29bd407dd8f9b3f527e194a701bba88cee`.

The legacy bridge is `jetbrains.mps.testbench.junit.runners.PushEnvironmentRunnerBuilder`, exposed through the
`Testbench` module (`920eaa0e-ecca-46bc-bee7-4e5c59213dd6`). It takes `jetbrains.mps.tool.environment.Environment` in
its constructor and implements JUnit 4's `RunnerBuilder`. `runnerForClass(Class<?>)` can throw `Throwable`; inherited
`safeRunnerForClass(Class<?>)` converts initialization failures to JUnit error-reporting runners. A host that promises
preparation before execution must inspect preparation failures rather than assuming that receiving a runner establishes
successful preparation.

For a discovered test class, the IDE's `NodeWrappersTestsContributor` constructs:

```text
load class through owning MPS module
  PushEnvironmentRunnerBuilder(environment).safeRunnerForClass(class)
    Request.runner(runner)
      filterWith(Description.createTestDescription(className, methodName))  [method selection only]
```

The builder injects the supplied environment into `EnvironmentAware` JUnit 3 test instances, JUnit 4 instances, and
annotated runners that implement that interface. Its JUnit 3 suite adapter implements filtering without replacing the
environment-aware suite. Using a plain Vintage launcher does not establish these injection semantics.

For an individual method, use the descriptor’s `getContainer()` to obtain its owning test case and `getShortName()` for
the method name. `TestDescriptor` also exposes the boolean `isContainer()`. In Groovy, call `getContainer()` explicitly:
the `container` property can resolve to the boolean accessor instead of the parent descriptor.

`TestNodeUtil.isJUnit4TestMethod` and `isJUnit5TestMethod` exclude methods annotated with `@Ignore` or `@Disabled` from
MPS discovery. Executing a discovered enclosing class still lets JUnit report those methods as skipped. Individual
selection through the MPS descriptors therefore differs from class execution for these disabled methods.

Load the builder and JUnit classes with compatible class identities. The module's display name is `Testbench`, not the
Java package name. Keep execution outside MPS model-access actions: environment-aware test fixtures may schedule EDT
commands or open projects themselves.

## Runtime observation

On MPS 2024.1.6, 2025.1.4, 2026.1.1, and 2026.2 EAP1 (`MPS-262.9437.166`, source
`46065cdc79a9467a53197d1a274b4e97d2a0f603`, JBR `25.0.3-b508.16`), isolated IDEA environments with the bundled plugins
successfully executed:

- native BaseLanguage test cases with individual pass/failure and source-node results;
- an ordinary JUnit 3 class through `PushEnvironmentRunnerBuilder`;
- ordinary Jupiter tests with disabled and assumption-aborted results and a failing `BeforeAll` container;
- an individually selected parameterized method with two invocations and distinct result IDs;
- a `NodesTestCase` using MPS fixture initialization;
- a generator `TransformationMatchAssertion` comparing an empty model with its generated output.

These observations establish representative paths, not every possible editor action, plugin combination, or custom
runner. The generator fixture uses an empty model and does not establish correctness of an arbitrary transformation. See
[test-execution.md](test-execution.md) for the MPS/JUnit session differences between release lines and the
classloading/discovery requirements.

## Source coordinates

Paths are relative to `JetBrains/MPS` at the revisions above:

- `plugins/mps-testing/languages/baseLanguage/unitTest/solutions/platform/source_gen/jetbrains/mps/baseLanguage/unitTest/platform/TestDescriptor.java`:
  `getContainer`, `isContainer`, and `getShortName`.
- `testbench/jetbrains.mps.testbench/source_gen/jetbrains/mps/testbench/junit/runners/PushEnvironmentRunnerBuilder.java`:
  `runnerForClass`, `PushEnvJUnit4Runner.createTest`, and `JUnit38SuiteAdapter.runTest`/`filter`.
- `plugins/execution-configurations/junit/source_gen/jetbrains/mps/baseLanguage/unitTest/execution/server/NodeWrappersTestsContributor.java`:
  `requestForTestClass` and `processTestMethod`.
- `plugins/mps-testing/languages/baseLanguage/unitTest/runtime/source_gen/jetbrains/mps/baseLanguage/unitTest/runtime/JUnitTestDiscoveryParticipants.java`:
  JUnit 3 discovery sets `USE_COMPATIBILITY_MODE` and retains method source identities.
- `plugins/mps-testing/languages/baseLanguage/unitTest/runtime/source_gen/jetbrains/mps/baseLanguage/unitTest/runtime/TestNodeUtil.java`:
  `isJUnit4TestMethod` and `isJUnit5TestMethod`.
- `plugins/mps-testing/languages/gentest.rt/source_gen/jetbrains/mps/lang/test/generator/rt/TransformHelper.java`:
  absent custom generation plans use the normal generation plan; `transform()` records generation status and output.
