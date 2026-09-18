# Remote debugging of headless MPS running under Ant

**Question this answers:** can a Java remote debugger attach to MPS when a build runs headless through the Ant MPS tasks
(`ant generate`), and where must the JDWP agent be placed?

**Short answer:** yes. The Ant MPS tasks (`<generate>`, `<make>`, `<gentest>`, `<runMPS>`, `launchtests`) are subclasses
of `jetbrains.mps.build.ant.MpsLoadTask`, which by default **forks a dedicated worker JVM** per task and writes the
task's nested `<jvmargs>` at the head of that forked JVM's command line. A standard JDWP `-agentlib` placed in
`<jvmargs>` therefore opens a listening socket on the worker JVM; with `suspend=y` the worker blocks before its `main`
starts, so a debugger attaching from an IDE catches startup and every later step. When a script sets `fork="false"`
instead, the worker runs inside the Ant JVM, `<jvmargs>` is rejected, and the JDWP agent must be placed on the Ant
process itself (e.g. `ANT_OPTS`).

Verified against these sources:

- **MPS 2024.1.2** (checkout): `core/tool/ant`, `core/tool/builder`,
  `plugins/mps-build/languages/build.mps/generator/template/.../main@generator.mps`
- **JetBrains/MPS `master`** (as of September 2026): same `core/tool/ant/source_gen` paths; the fork semantics and the
  placement of `<jvmargs>` are unchanged.

The tasks ship in the `ant-mps` library, `lib/ant/lib/ant-mps.jar` in the MPS distribution. The distribution's own
`build.xml` loads them as `taskdef resource="jetbrains/mps/build/ant/antlib.xml"`
(`core/tool/ant/resources/jetbrains/mps/build/ant/antlib.xml`).

## How the Ant tasks run MPS

`antlib.xml` maps these tasks to their classes:

| Ant element   | Class (`jetbrains.mps.build.ant...`)                                               |
| ------------- | ---------------------------------------------------------------------------------- |
| `<generate>`  | `.generation.GenerateTask` (worker `...make.GeneratorWorker`)                      |
| `<mps.make>`  | `.MakeTask`                                                                        |
| `<gentest>`   | `.generation.GenTestTask`                                                          |
| `<runMPS>`    | `.generation.MpsRunnerTask`                                                        |
| `launchtests` | `.junit.LaunchTestTask`                                                            |
| `<jvmargs>`   | `.JvmArgs` (nestable `<arg value="..."/>`; deprecated single `<jvmarg>` is `.Arg`) |

All extend `org.apache.tools.ant.Task` through `MpsLoadTask`, whose execution model is:

```java
// MpsLoadTask, 2024.1.2 (core/tool/ant/source_gen/jetbrains/mps/build/ant/MpsLoadTask.java)
private boolean myFork = true;          // default: fork
setFork(boolean)                        // <generate fork="false"> opt-out
addConfiguredJvmArgs(JvmArgs)            // nested <jvmargs>; throws BuildException when !myFork

if (myFork) {
  // command line, in order:
  //   java
  //   <myJvmArgs.getArgs()>                       <- the nested <jvmargs>
  //   -classpath <ant cp + MPS lib + deps>        (@argfile on newer master)
  //   [--add-opens=... when open-packages]        [-Djna.boot.library.path=...]
  //   <worker class>  <script tmp file> [extra args]
  // executed with org.apache.tools.ant.taskdefs.Execute
} else {
  // worker instantiated (Script|ProjectComponent constructor) and "work()" invoked
  // in-process in the Ant JVM via an URLClassLoader with the computed classpath
}
```

For generation the worker is `jetbrains.mps.tool.builder.make.GeneratorWorker`, an `MpsEnvironment`-based runner with
`public static void main(String[] args)`
(`core/tool/builder/source_gen/jetbrains/mps/tool/builder/make/GeneratorWorker.java`), so the forked JVM is a fully
standalone MPS process.

`JvmArgs` (`core/tool/ant/source_gen/jetbrains/mps/build/ant/JvmArgs.java`) seeds the command line with
`-Xmx512m -XX:+HeapDumpOnOutOfMemoryError` unless a user-supplied argument overrides that pattern; user arguments are
appended after the defaults. The deprecated single-argument `<jvmarg>` form logs a warning.

## Attaching to the forked worker

Add a JDWP agent to the task in the build script:

```xml
<generate fork="true" ...>
  <chunk>...</chunk>
  <jvmargs>
    <arg value="-agentlib:jdwp=transport=dt_socket,server=y,suspend=y,address=*:5005" />
  </jvmargs>
</generate>
```

Then, from the IDE, either select the worker in _Attach to Process_ or use a Remote JVM run configuration pointing at
`localhost:5005`. `suspend=y` makes the worker wait for the attachment before `GeneratorWorker.main` executes, so
breakpoints in the headless MPS bootstrap and in generator/make code are reachable from the very start. The debugger is
allowed in every non-forking mode of execution; parallel generation (`parallelThreads`/`parallelMode`) runs on threads
inside the same worker JVM, so one attachment covers it.

Because the worker is started through the `java` executable of the JVM running Ant, an environment-only alternative
works even for generated scripts that are not convenient to edit:

```bash
JAVA_TOOL_OPTIONS="-agentlib:jdwp=transport=dt_socket,server=y,suspend=y,address=*:5005" ant generate
```

The JVM launcher prints `Picked up JAVA_TOOL_OPTIONS: ...` to stderr, and the setting applies to every `java` process
spawned by the build, not only the MPS worker — either harmless or relevant depending on the build.

## The `fork="false"` case

Setting `fork="false"` runs the worker inside the Ant JVM (in-process via an `URLClassLoader`). Then nested `<jvmargs>`
is rejected: `MpsLoadTask.addConfiguredJvmArgs` throws `BuildException("Nested jvmargs is only allowed in fork mode.")`.
Debug in this mode by placing the JDWP agent on the Ant JVM itself, for example with `ANT_OPTS` (or
`JAVA_TOOL_OPTIONS`), and attach to the `ant` process. A debugger attached to the Ant JVM can then suspend threads
inside the in-process MPS worker.

## Caveats and lifecycles

- **One worker JVM per task.** A script with several `<generate>`/`<make>` blocks forks and exhausts one worker per
  task. With `suspend=y`, each worker waits for its own attachment; attach repeatedly or give each task a distinct port
  (and drop `suspend=y` or use `suspend=n` with an auto-attaching agent).
- **Startup blocking is designed for this.** `suspend=y` is the reliable mode for catching MPS bootstrap failures; it is
  otherwise easy to miss exceptions thrown before the generator reports anything in the Ant log.
- **Compiled code only.** Breakpoint placement requires the debugger to load the classes of interest, so languages,
  generator modules, and solution code must be present on the worker classpath as compiled artifacts (built language
  jars or classes directories on the MPS module dependencies / `lib`). The Ant task logs a warning if `-ea` is absent;
  assertions are recommended but not required for debugging.
- **Locking does not interfere.** MPS/IDEA read and write locks and EDT dispatch are internal to the worker JVM; remote
  debugging suspends arbitrary threads without violating them (suspension simply stops threads mid-lock, like any Java
  debugger).

## Evidence

MPS sources, 2024.1.2 checkout:

- `core/tool/ant/source_gen/jetbrains/mps/build/ant/MpsLoadTask.java` — fork branch, `execute()`,
  `addConfiguredJvmArgs`, `addConfiguredJvmArg`, `myFork` default, `getJreExecutable`, `Execute` invocation.
- `core/tool/ant/source_gen/jetbrains/mps/build/ant/JvmArgs.java` — default `-Xmx512m -XX:+HeapDumpOnOutOfMemoryError`.
- `core/tool/ant/source_gen/jetbrains/mps/build/ant/generation/GenerateTask.java` and `MpsRunnerTask.java` — worker
  class names, super calls.
- `core/tool/ant/resources/jetbrains/mps/build/ant/antlib.xml` — task/typedef mapping.
- `core/tool/builder/source_gen/jetbrains/mps/tool/builder/make/GeneratorWorker.java` — worker `main`.
- `build.xml` ("MPS Bootstrapper"): `declare-mps-tasks` and the `generate` target using `<generate fork="true">` with
  nested `<jvmargs>`.

JetBrains/MPS `master` (via the same repo-root-relative paths): the fork branch persists; the classpath is passed
through a temp `@argfile` instead of inline `-classpath` tokens, `myJvmArgs` is a `JvmArgs` field, and a default
`-Dintellij.platform.load.app.info.from.resources=true` is no longer emitted. None of these affects the `<jvmargs>`
placement or the fork model.

## Re-verification triggers

- `MpsLoadTask.execute()` stops forking or reorders the command line so `<jvmargs>` no longer reaches the worker JVM;
- `<jvmargs>` becomes allowed in non-fork mode;
- the Ant tasks stop using a separate worker `main` (e.g. move to Gradle or an in-process API);
- the `antlib.xml` task names or worker class names change.
