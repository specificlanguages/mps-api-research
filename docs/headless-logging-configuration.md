# Logging configuration in headless MPS environments

**Question this answers:** when and how do MPS and the IntelliJ Platform configure logging in a headless
`MpsEnvironment` or `IdeaEnvironment`, how does test mode differ, and where can a host reliably suppress platform noise
while retaining an explicitly selected set of messages on stdout and stderr?

**Short answer:** all versions covered here use `java.util.logging` (JUL). Through MPS 2025.1, constructing either
environment automatically runs MPS's JUL initialization. `MpsEnvironment` stops there, while `IdeaEnvironment`
subsequently installs an IntelliJ logger factory that replaces the root handlers. In MPS 2026.1.1 and build 262, neither
environment invokes the MPS initializer: callers may explicitly call `LogInitializer.init()`, after which
`IdeaEnvironment` still performs the IntelliJ configuration phase. In test mode, IntelliJ installs `TestLoggerFactory`;
in non-test mode, normal platform startup installs `com.intellij.idea.LoggerFactory`. Both clear the root handlers.
Configure quiet-mode system properties and preserve the original standard streams before initialization, then install or
adjust the final JUL handlers after `IdeaEnvironment.init()` returns. Do not rely on a pre-start root handler surviving
IDEA startup.

Verified against these supported MPS releases:

- **2023.2.2**
- **2024.1.2**
- **2025.1.1**
- **2026.1.1**, with IntelliJ Platform build `261.25134.711`
- **build 262.9437.325**, which identifies itself as **MPS 2026.2**, built from MPS revision
  `3ad9e9c200f65b78aa76f71e9a41083fe57e6e86` and IntelliJ Platform build `262.9437.731`

The MPS classes are in `mps-boot-util.jar` (through 2025.1), `mps-environment.jar`, and `mps-platform.jar`. The platform
classes are in `app.jar` and `testFramework.jar` in 2023.2--2026.1; the 262 platform splits the application classes
across the newer `intellij.platform.*` jars while retaining `testFramework.jar`.

## The logging layers

MPS's `jetbrains.mps.logging.Logger` is a thin JUL adapter in every version examined:

```java
public static Logger getLogger(Class<?> requestor) {
  return new JULogger(java.util.logging.Logger.getLogger(requestor.getName()));
}
```

Consequently, MPS category names are ordinary fully qualified Java class names. MPS and IntelliJ log records ultimately
share the global JVM `java.util.logging.LogManager`, even though IntelliJ wraps JUL with
`com.intellij.openapi.diagnostic.Logger` and uses different logger factories in test and non-test startup.

JUL filtering has two independent gates: the logger's effective level and every handler's level. Setting only the root
logger or only a handler is insufficient. A JUL `ConsoleHandler` always writes to stderr; splitting informational output
to stdout and warnings/errors to stderr requires custom handlers or direct writes to preserved streams.

## MPS initialization through 2025.1

In 2023.2.2, 2024.1.2, and 2025.1.1, both concrete environment classes contain a static initializer:

```java
static {
  EnvironmentBase.initializeLog();
}
```

Class initialization occurs before the constructor. The class's static `LOG` field and the superclass's static `LOG`
field are created before this block, so some named JUL loggers already exist when configuration is read.
`EnvironmentBase.initializeLog()` calls `jetbrains.mps.util.LogInitializer.init()` and prints an error and stack trace
to stderr if initialization fails.

`LogInitializer.init()` does one of two things:

1. If it finds an explicit configuration file, it reads it and returns.
2. Otherwise it creates or reuses the log directory, adds an append-mode `FileHandler` for `idea.log` at `ALL`, and adds
   a root `ConsoleHandler` at `WARNING`. It sets the root level to `INFO` only when that level is currently null.

The method is not idempotent in the default branch: repeated calls add more handlers and open more file handles.

### Configuration-file property change

| MPS version  | Configuration selected explicitly by `LogInitializer`                        |
| ------------ | ---------------------------------------------------------------------------- |
| 2023.2.2     | `idea.log.config.properties.file` only                                       |
| 2024.1.2     | `idea.log.config.properties.file` only                                       |
| 2025.1.1     | `idea.log.config.properties.file`, otherwise `java.util.logging.config.file` |
| 2026.1.1     | Same precedence as 2025.1.1                                                  |
| 262.9437.325 | Same precedence as 2026.1.1                                                  |

The standard JUL manager can independently read `java.util.logging.config.file` during its own initialization in every
JDK. In 2023.2.2 and 2024.1.2, however, MPS does not treat that property as its explicit-config branch and still adds
its default file and console handlers. Use `idea.log.config.properties.file` when the MPS initializer itself must return
after loading the file.

For MPS's initializer, a relative configuration path is resolved against `user.home`. IntelliJ's `TestLoggerFactory`
uses `Path.of(value)` directly, so its relative path is resolved against the process working directory. Always supply an
absolute path if the same property is used in both phases.

## The 2026.1 change

MPS 2026.1 moved ownership of initial logging setup out of environment class initialization. The change is present in
2026.1.1 and continues unchanged in build 262.9437.325:

- `LogInitializer` moved to `jetbrains.mps.core.tool.environment.util.LogInitializer` in `mps-environment.jar`.
- The static initialization blocks were removed from `MpsEnvironment` and `IdeaEnvironment`.
- `EnvironmentBase.initializeLog()` is deprecated for removal, emits a warning to stderr, and delegates to the moved
  class.
- Callers that own logging must invoke `LogInitializer.init()` explicitly before initializing an environment. MPS's own
  `WorkerBase.workFromMain()` does so, but merely constructing `MpsEnvironment` or `IdeaEnvironment` no longer does.
- Loading an explicit configuration uses `LogManager.updateConfiguration(in, null)` rather than `readConfiguration(in)`.
  This matters when named loggers already exist: their configured `handlers` can now be updated. The shipped
  `bin/log.properties` illustrates handlers on `jetbrains.mps` and `org.jetbrains.mps` so those handlers survive IDEA's
  later removal of root handlers.

The launcher for the graphical IDE deliberately does not call this initializer and leaves logging to IntelliJ.

## Non-test `IdeaEnvironment`

`IdeaEnvironment.init()` calls the MPS headless platform starter. During IntelliJ bootstrap, the platform executes the
equivalent of:

```java
Logger.setFactory(new com.intellij.idea.LoggerFactory());
```

The factory constructor:

1. removes every root JUL handler;
2. sets the root logger level to `INFO`;
3. installs the platform `idea.log` file handler;
4. configures the platform console handler (version-dependent defaults below);
5. installs a severe-level dialog/error handler, which is ineffective as UI in headless operation but remains part of
   the root configuration.

Therefore MPS's provisional root handlers and a host's pre-start root handlers do not survive. Named logger handlers are
not removed by `JulLogger.clearHandlers()`, which targets only the root logger.

`-Didea.log.console=false` suppresses the platform console handler in non-test mode in every researched version. In
2023.2 and 2024.1, `JulLogger.configureLogFileAndConsole` reads the property. In 2025.1 and 2026.1, normal
`LoggerFactory` reads it and passes the resulting boolean to `JulLogger`.

Build 262 replaces the boolean with a console level. `intellij.console.log.level` accepts an IntelliJ `LogLevel` name
and takes precedence. Without it, a development build gets `WARNING`; a packaged build gets `WARNING` only when
`idea.log.console=true`, otherwise `OFF`. Thus packaged 262 has no platform console handler by default, and
`idea.log.console=false` remains a valid cross-version way to request silence. This is a change from the default-on
console in 2023.2--2026.1.

After configuring the logger, normal bootstrap defaults `intellij.log.stdout` to `true` and replaces `System.out` and
`System.err` with `PrintStreamLogger`s. These streams still write to the original streams and also copy complete lines
to the `STDOUT` and `STDERR` IDEA log categories. Set `-Dintellij.log.stdout=false` before startup to prevent this
wrapping. This property does not itself silence the original streams.

The unrelated `intellij.log.to.json.stdout=true` mode adds a JSON handler to stdout and disables the normal console
path; it is unsuitable for a quiet console.

## Test-mode `IdeaEnvironment`

When `EnvironmentConfig.isTestMode()` is true, `IdeaEnvironment` uses
`com.intellij.testFramework.TestApplicationManager` instead of the headless platform starter. Loading that class calls
`initializeTestEnvironment()`, which installs `TestLoggerFactory` with `Logger.setFactory(...)`.

The first IntelliJ logger requested from that factory calls `TestLoggerFactory.reconfigure()`:

1. read `idea.log.config.properties.file`, or `${idea.home.path}/test-log.properties` when the property is absent;
2. print a missing-file diagnostic to stderr if that file does not exist;
3. create the test log directory (`idea.log.path`, otherwise a `testlog` directory under the system path);
4. clear the root handlers;
5. install a fresh, truncating `idea.log` file handler and a warning-level console handler.

In 2025.1 it prints the test-log path to stdout; 2026.1 and 262 print its URI. Test mode does not perform normal
platform standard-stream redirection.

There is an important version break for `idea.log.console=false`:

- In 2023.2 and 2024.1, `JulLogger` itself checks the property, so it suppresses the test console handler.
- In 2025.1, 2026.1, and 262, that check moved into normal `LoggerFactory`; `TestLoggerFactory` requests a console
  handler unconditionally. The property does **not** silence test-mode console logging in these versions. Disable or
  remove the resulting root console handler after environment initialization.

Test logging also has behavioral semantics beyond output routing. IntelliJ `Logger.error(...)` goes through
`TestLogger`, and the test error processor can rethrow it as `TestLoggerAssertionError`. Turning handlers off does not
disable that behavior. MPS's `jetbrains.mps.logging.Logger` calls JUL directly and therefore does not itself acquire
these IntelliJ test-wrapper semantics.

## A robust quiet-mode sequence

For a host that wants only an explicitly chosen set of build messages on stdout/stderr:

1. **Before touching either environment class**, save the original `System.out` and `System.err` references.
2. Before non-test IDEA startup, set `idea.log.console=false` and `intellij.log.stdout=false`. On 262, also leave
   `intellij.console.log.level` unset or set it to `OFF`, because it takes precedence over `idea.log.console`.
3. For 2023.2--2025.1, expect the environment class initializer to configure JUL. For 2026.1 and 262, decide explicitly
   whether to call the moved `LogInitializer`; it is unnecessary when the host will own all logging and does not need
   MPS's log file setup.
4. Initialize the environment.
5. After `IdeaEnvironment.init()` returns, inspect the actual root handlers. Remove them or set their levels to `OFF`
   for quiet mode. This post-start step is required in 2025.1+ test mode and is the common cross-version safety net.
6. Set the root logger level to `OFF` if no inherited categories should pass. For each allowed category, set an explicit
   level and attach a dedicated handler, normally with `useParentHandlers=false`.
7. Write selected application/build messages to handlers backed by the saved original streams. Use separate handlers if
   informational records belong on stdout and warnings/errors on stderr.
8. If arbitrary code can print directly, temporarily replace `System.out` and `System.err` with a sink and restore the
   saved streams only around explicitly allowed work. Synchronize such temporary restoration because standard streams
   are JVM-global.
9. On teardown, flush and close host-owned handlers and restore streams and system properties if the JVM will be reused.

Do not identify platform console handlers solely with `instanceof ConsoleHandler`: IntelliJ's optimized console handler
is a subclass today, but it is internal. Handler type plus stream behavior and installation phase should be treated as
version-sensitive implementation details.

## What a JUL properties file can and cannot do

A useful allow-list shape is:

```properties
handlers=
.level=OFF

example.useful.level=INFO
example.useful.useParentHandlers=false
example.useful.handlers=example.logging.StdoutHandler
```

The custom handler class must be visible to the system/application class loader when JUL instantiates it. A stock
`ConsoleHandler` cannot target stdout. A configuration that attaches handlers only to named categories is more likely to
survive IDEA startup than one that relies on root handlers, but applying or repairing the final configuration after
`IdeaEnvironment.init()` remains the most predictable approach.

In 2023.2--2025.1, `LogManager.readConfiguration` may fail to attach a configured `handlers` property to a named logger
that was already instantiated. MPS 2026.1 changed its explicit loader to `updateConfiguration` specifically to address
this; build 262 retains that implementation. Programmatic post-start handler installation avoids that difference.

## Other output channels and lifecycle effects

- Direct writes to `System.out`/`System.err`, subprocess output, native launcher output, JVM crash output, and uncaught
  exception reporting are not JUL records. Logger levels cannot suppress them.
- IntelliJ's standard-stream wrappers duplicate lines into `idea.log`; they do not replace the original console write.
- `idea.log.path` controls the platform/MPS log directory. `idea.paths.selector` indirectly selects system and log
  directories and is cached early by `PathManager`; set path properties before touching that class.
- `idea.log.perf.stats` defaults to `false` in `IdeaEnvironment` unless the caller set it, suppressing internal startup
  performance statistics.
- In 2026.1 and 262, `idea.log.append` controls whether normal platform startup appends to `idea.log`. Test logging
  still configures a non-append test log.
- In 2026.1, `intellij.console.use.severe.log.level=true` changes the installed console handler's threshold from
  `WARNING` to `SEVERE`; it does not suppress the handler.
- In 262, `intellij.console.log.level` supersedes `idea.log.console` when set and can select a level other than the old
  fixed `WARNING` threshold.
- `idea.test.logs.echo.debug.to.stdout`, `idea.split.test.logs`, and test-log dumping can deliberately put test logs on
  stdout independently of ordinary console-handler filtering.
- In platform versions with asynchronous logging, `-Dintellij.platform.log.sync=true` avoids late shutdown logging
  races. It must be set before the logging classes initialize; see `async-log-shutdown-noise.md`.
- JUL and standard streams are process-global. Running multiple environment instances concurrently, or reconfiguring
  logging while unrelated code logs, is unsupported territory and can cause lost, duplicated, or misrouted records.

## Evidence

MPS source locations:

- 2023.2.2 and 2024.1.2: `startup/boot-util/source/jetbrains/mps/util/LogInitializer.java`
- 2025.1.1: same source location, with fallback to `java.util.logging.config.file`
- 2026.1.1 and 262.9437.325:
  `core/tool/environment/source_gen/jetbrains/mps/core/tool/environment/util/LogInitializer.java`
- all versions: `core/tool/environment/source_gen/jetbrains/mps/tool/environment/EnvironmentBase.java`,
  `MpsEnvironment.java`, and
  `workbench/mps-platform/jetbrains.mps.ide.platform/source_gen/jetbrains/mps/tool/environment/IdeaEnvironment.java`
- all versions: `core/logging/source/jetbrains/mps/logging/Logger.java` and `JULogger.java`
- 2026.1.1 and 262.9437.325: shipped `bin/log.properties`; build coordinates from `lib/build.txt` or `build.properties`

IntelliJ Platform sources shipped in the 2023.2--2026.1 MPS checkouts' `lib/src/platform-sources.zip`:

- `com/intellij/idea/LoggerFactory.java`
- `com/intellij/openapi/diagnostic/JulLogger.java`
- `com/intellij/testFramework/TestLoggerFactory.java`
- `com/intellij/testFramework/TestApplicationManager.kt`
- `com/intellij/testFramework/common/testEnvironment.kt`
- 2023.2: `com/intellij/idea/StartupUtil.kt` and `com/intellij/idea/PrintStreamLogger.java`
- 2024.1+: `com/intellij/platform/ide/bootstrap/main.kt` or `startup.kt`, and `PrintStreamLogger.java`

For build 262.9437.325, the corresponding behavior was also checked in the exact cached distribution's bytecode:

- `com.intellij.idea.LoggerFactory` in `lib/intellij.platform.ide.impl.jar`
- `com.intellij.openapi.diagnostic.JulLogger` in `lib/util-8.jar`
- `com.intellij.testFramework.TestLoggerFactory` in `lib/testFramework.jar`

## Re-verification triggers

Re-check this behavior when any of these change:

- `IdeaEnvironment` changes its application startup mechanism;
- `LoggerFactory`, `TestLoggerFactory`, or `JulLogger.configureLogFileAndConsole` changes signature;
- root clearing changes from `JulLogger.clearHandlers()`;
- the `idea.log.console` or `intellij.log.stdout` property moves again;
- MPS changes whether environment classes invoke `LogInitializer` automatically;
- the host begins embedding multiple environments in one JVM.
