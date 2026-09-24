# Disabling feature usage statistics and redirecting regional URL mapping

Can a custom MPS RCP disable IntelliJ feature usage statistics (FUS), or redirect the requests that produce
`RegionUrlMapper` SSL diagnostics? This note covers MPS **2025.1.4**, **2026.1.1**, and the **master snapshot
262.9437.SNAPSHOT**, commit `5d6bed88c917c0df00b985e34fcfb3adb31c6d0d` (2026-09-22).

## Disable the standard statistics pipeline

**Verified from source:** supply these JVM system properties when starting the RCP:

```text
-Didea.disable.collect.statistics=true
-Didea.suppress.statistics.report=true
-Didea.headless.enable.statistics=false
```

Put them in the RCP launcher's VM options, or pass them to the actual application/test JVM in CI. Setting them only on
the Gradle or Ant process does not establish that a forked application receives them. Set them before application
startup and restart existing processes.

The same property names and relevant checks exist in all three inspected versions:

| Property                                | Verified effect                                                                                                                              |
| --------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `idea.disable.collect.statistics=true`  | Makes `StatisticsUploadAssistant.isCollectAllowed()` return false in a non-headless application.                                             |
| `idea.suppress.statistics.report=true`  | Makes `StatisticsUploadAssistant.isSendAllowed()` return false, including in headless mode.                                                  |
| `idea.headless.enable.statistics=false` | Disables the separate headless collection branch, which is checked **before** the collection-disable property. False is already the default. |

`com.intellij.internal.statistic.utils.StatisticsUploadAssistant` exposes these as public static boolean methods. The
standard `FeatureUsageEventLoggerProvider` requires collection permission to record, and both recording and sending
permission to upload. The shutdown uploader also filters providers using `isSendEnabled()`.

Suppressing reporting alone is insufficient to prevent statistics-related downloads. Validation metadata updates are
gated by `StatisticsEventLoggerProvider.isLoggingEnabled()`, independently of upload permission:

```text
StatisticsJobsScheduler
  upload job
    provider.isSendEnabled() -> upload service
  validation metadata update
    provider.isLoggingEnabled() -> metadata/configuration downloads
```

In 2025.1.4 the scheduled validation update starts after three minutes and repeats every 180 minutes, unless the
internal initial-delay flag changes startup timing. In 2026.1.1 and the inspected master snapshot, the scheduler starts
per-provider validation storage updates; the logging-enabled gate remains.

These controls cover the standard consent-aware statistics providers. They are not an application-wide networking switch
or a guarantee that no statistics-related code executes. `isLoggingEnabled()` also accepts `isLoggingAlwaysActive()`,
and platform extensions can force logging independently of normal collection permission. Custom providers can implement
their own policies. Inspect those extensions if a tailored RCP still downloads FUS metadata after applying the
properties.

Do not use `idea.local.statistics.without.report=true` as an offline switch: it permits local collection and therefore
can leave metadata downloads enabled. Likewise, the test-endpoint properties select test infrastructure, not an offline
mode. In 2026.1.1 and master, TeamCity detection normally suppresses sending unless explicitly enabled; this is not a
collection or metadata-download disable switch.

## Exact meaning of headless

**Verified in all three versions:** the statistics checks call
`ApplicationManager.getApplication().isHeadlessEnvironment()`. This is an IntelliJ application flag, not a direct check
of CI environment variables, `DISPLAY`, or `GraphicsEnvironment.isHeadless()`.

`ApplicationImpl.isHeadlessEnvironment()` returns its final `myHeadlessMode` field. The normal application constructor
copies `AppMode.isHeadless()` into that field. During normal bootstrap, `AppMode.setFlags(args)` computes the mode:

| Version         | Condition for headless mode                                                                                                                                                                                                                 |
| --------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 2025.1.4        | `Boolean.getBoolean("java.awt.headless")`, **or** the first argument is in `AppMode.isHeadless(List<String>)`'s headless command list, **or** that argument has fewer than 20 characters and ends with the case-sensitive suffix `inspect`. |
| 2026.1.1        | `Boolean.getBoolean("java.awt.headless")`, **or** `WellKnownCommands.getCommandFor(args)` returns a command whose `isHeadless` is true. Its lookup also recognizes the same short `*inspect` suffix.                                        |
| Master snapshot | Same condition, with lookup moved to `WellKnownCommand.getCommandFor(args)` and its `isHeadless` field.                                                                                                                                     |

An empty argument list supplies no command-based headless condition. Recognized headless commands include `ant`,
`inspect`, `format`, and `buildEventsScheme`; the complete lists are in the source anchors below. An arbitrary custom
command is not automatically headless. When `AppMode` detects headless mode, it also sets `java.awt.headless=true`.
Consequently, an explicit `java.awt.headless=false` does not override a recognized headless command.

MPS's own `jetbrains.mps.ide.util.PlatformStarter` explicitly calls `AppMode.setFlags(listOf("mps-inspect"))` in all
three revisions. That name satisfies the suffix rule, so this bootstrap path sets the platform headless flag even
without an explicit JVM property.

The platform test bootstrap has a separate route. Its `ApplicationImpl(CoroutineContext, boolean isHeadless)`
constructor uses the supplied boolean and sets unit-test mode independently. In `testApplication.kt`,
`UITestUtil.getAndSetHeadlessProperty()` supplies that boolean: it returns false only when `java.awt.headless` is
exactly the string `false`; otherwise it sets the property to `true` and returns true. A custom test harness can
construct or install an application differently. Unit-test mode additionally disables the standard FUS recorder and the
statistics scheduler; running a build on CI does not itself establish unit-test mode.

For an already initialized application, these are the relevant diagnostic values:

```java
Application app = ApplicationManager.getApplication();
boolean platformHeadless = app.isHeadlessEnvironment();
boolean unitTestMode = app.isUnitTestMode();
boolean headlessStatisticsEnabled = Boolean.getBoolean("idea.headless.enable.statistics");
```

Here `Application` and `ApplicationManager` are from `com.intellij.openapi.application`. Read the application flag in
the actual JVM producing the message. Running under Xvfb, running without visible windows, or setting a CI environment
variable does not by itself satisfy the platform bootstrap condition. AWT can infer headlessness from the display
environment, but that inference is not the predicate used by `StatisticsUploadAssistant`. Changing `java.awt.headless`
after application construction does not update `myHeadlessMode`.

**Diagnostic implication:** if `platformHeadless` is true and `idea.headless.enable.statistics` is absent or false, the
ordinary statistics collection predicate already returns false. Persistent mapper messages then need a caller trace:
forced logging, direct service use, or another mapper consumer can explain them. The mapper itself has no headless
guard, and its messages alone do not prove that headless statistics was enabled.

## What a RegionUrlMapper diagnostic means

**Verified:** `com.intellij.ide.RegionUrlMapper` downloads a shared URL-rewriting configuration. Its default endpoint is
`https://www.jetbrains.com/config/JetBrainsResourceMapping.json`; China has a separate default host. A failure here does
not by itself establish that usage events were uploaded.

| Version         | Relationship between statistics and RegionUrlMapper                                                                                                                                                                                     |
| --------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 2025.1.4        | `EventLogInternalApplicationInfo.getTemplateUrl()` obtains the statistics region-mapping service. Instantiating `StatisticsRegionUrlMapperServiceImpl` starts a coroutine that calls the mapper immediately and then every ten minutes. |
| 2026.1.1        | Statistics instead reads a region code through `StatisticsRegionSettingsServiceImpl`; this implementation does not call the mapper.                                                                                                     |
| Master snapshot | Same separation as 2026.1.1.                                                                                                                                                                                                            |

The 2025.1.4 statistics mapper coroutine does not recheck collection permission on each iteration. Changing properties
after that service has started does not stop it. Apply the disable properties at process launch.

The mapper is shared infrastructure: `GeneralFeedbackSubmit.kt`, for example, also calls it. Thus, disabling FUS does
not guarantee removal of every `RegionUrlMapper` diagnostic. In 2026.1.1 and master, investigate the actual caller
instead of attributing a mapper message to the standard FUS region service.

Normal mapping failure falls back to the original URL. In 2025.1.4 the loader logs the failure message at INFO and
rethrows into the asynchronous cache; the mapping entry point recovers with an empty mapping. In 2026.1.1 and master,
the mapper handles I/O failures at INFO, JSON parse failures at WARN, and other failures at ERROR. The
`force.region.mappings.load=true` test flag changes failure handling and is not a disable switch.

## Redirect the mapper's configuration download

**Verified:** all three versions read these system properties during `RegionUrlMapper` class initialization:

```text
-Djb.mapper.configuration.url=http://127.0.0.1:8765/JetBrainsResourceMapping.json
-Djb.mapper.configuration.url.europe=http://127.0.0.1:8765/JetBrainsResourceMapping.json
```

The unsuffixed property applies **only to `Region.NOT_SET`**. It is not a fallback for named regions. Set the property
for the selected region; to cover every possible selection, provide all of these suffixes:

```text
africa
americas
apac
china
europe
middle_east
oceania
```

The selected region comes from the `JetBrains.region.code` preference through `RegionSettings.getRegion()`. Blank
override values are ignored. The override table is initialized once, so set properties before class loading.

Serve HTTP 200 with this JSON body from the configured endpoint to provide an empty mapping:

```json
[]
```

**Verified from parsing and application code:** an empty array leaves all original URLs unchanged. This redirects the
mapping download itself away from JetBrains; it does **not** disable a subsequent request by the caller to its original
URL. The endpoint must actually be reachable. A nonexistent endpoint merely substitutes a different connection failure.
The cache's default expiration is two minutes; requests reload on cache misses.

No mapper-disable property appears in the inspected `RegionUrlMapper` implementations. In particular, leaving the region
unset still causes a mapping lookup. If absolutely no mapping request is acceptable, eliminate the calling subsystem or
patch the platform's mapper to return the original URL without loading a configuration.

The HTTP override is the common implementation-supported approach across the inspected versions. This note does not
establish a portable `file:` URL workaround: 2025.1.4 uses `HttpRequests`, while 2026.1.1 and master use
`PlatformHttpClient`.

## Redirecting statistics itself

The mapper override is not an upload endpoint setting. Its JSON contains an array of single-entry objects mapping URL
substrings to replacement strings; the first matching pattern is replaced, ignoring case when locating it.

In 2025.1.4, the statistics configuration template is
`https://resources.jetbrains.com/storage/fus/config/v4/%s/%s.json`, populated with recorder ID and product code.
Configuration then supplies separate `send`, `metadata`, and `dictionary` endpoints. A mapping rule can rewrite the
configuration template, but this is not a reliable way to prevent the first JetBrains request: the statistics service
initially returns null while its asynchronous mapping lookup runs, and `getTemplateUrl()` falls back to the JetBrains
template. Redirecting configuration also requires controlling the endpoints inside that configuration.

In 2026.1.1 and master, `EventLogUploadSettingsClient` uses the FUS reporting library's
`ConfigurationClientFactory.create(...)`, passing the recorder, product, version, and region code. It no longer obtains
the statistics configuration template through `RegionUrlMapper`. Consequently, the mapper override cannot be used to
redirect this statistics path.

**Unresolved:** a complete supported custom FUS backend configuration has not been established. The inspected
application wiring does not expose a simple common upload-URL property across these versions. Disabling the standard
pipeline with the startup properties is the verified configuration route for an RCP that does not need FUS.

## Evidence and limits

All conclusions marked verified come from static source inspection; no full RCP or CI reproduction was run. The
properties' effect on the requesting application's exact exception remains unobserved without its caller stack and
runtime configuration.

The matching patched platform source archives are `lib/src/platform-sources.zip`. Platform versions were checked against
the MPS checkout's `build/version.properties` and the dependency distribution's `lib/build.txt`:

| MPS revision                                             | Patched platform build |
| -------------------------------------------------------- | ---------------------- |
| Tag `2025.1.4`                                           | `MPS-251.28774.587`    |
| Tag `2026.1.1`                                           | `MPS-261.25134.711`    |
| Master commit `5d6bed88c917c0df00b985e34fcfb3adb31c6d0d` | `MPS-262.9437.732`     |

The statistics classes are in `lib/stats.jar` in the two releases and `lib/intellij.platform.statistics.jar` in the
master platform. `RegionUrlMapper` is in `lib/app.jar` in 2025.1.4 and `lib/intellij.platform.ide.impl.jar` in 2026.1.1
and master.

Relevant source-archive entries and stable anchors:

- `com/intellij/openapi/application/impl/ApplicationImpl.java`: normal and test constructors, `isHeadlessEnvironment`.
- `com/intellij/idea/AppMode.java`: `setFlags`; also `isHeadless(List<String>)` in 2025.1.4.
- `com/intellij/idea/WellKnownCommands.kt` in 2026.1.1 and `com/intellij/idea/WellKnownCommand.java` in master: command
  tables and `getCommandFor`.
- `com/intellij/testFramework/common/testApplication.kt` and `com/intellij/testFramework/UITestUtil.java`: test
  bootstrap and `getAndSetHeadlessProperty`.
- MPS repository file `workbench/mps-platform/source/jetbrains/mps/ide/util/PlatformStarter.kt` in all three revisions:
  `CMD_NAME` and `doStartApplication`.
- `com/intellij/internal/statistic/utils/StatisticsUploadAssistant.java`: property constants, `isCollectAllowed`,
  `isSendAllowed`, and the headless and TeamCity branches.
- `com/intellij/internal/statistic/eventLog/fus/FeatureUsageEventLoggerProvider.kt`: `isRecordEnabled`, `isSendEnabled`.
- `com/intellij/internal/statistic/eventLog/StatisticsEventLogger.kt`: `isLoggingEnabled`, `isLoggingAlwaysActive`,
  `StatisticsEventLoggerProviderExt`.
- `com/intellij/internal/statistic/eventLog/StatisticsEventLogProviderUtil.kt`: `forceLoggingAlwaysEnabled`.
- `com/intellij/internal/statistic/updater/StatisticsJobsScheduler.kt`: send-job and metadata-update gates.
- `com/intellij/internal/statistic/eventLog/uploader/EventLogExternalUploader.kt`: `isSendEnabled` provider filters.
- `com/intellij/internal/statistic/eventLog/EventLogInternalApplicationInfo.java`: `getTemplateUrl` in 2025.1.4;
  `getRegionalCode` in 2026.1.1 and master.
- `com/intellij/internal/statistic/StatisticsRegionUrlMapperServiceImpl.kt`: initialization and `updateUrl`, 2025.1.4.
- `com/intellij/internal/statistic/StatisticsRegionSettingsServiceImpl.kt`: initialization and `getRegionCode`, 2026.1.1
  and master.
- `com/intellij/internal/statistic/eventLog/connection/EventLogUploadSettingsService.java`: `getConfigUrl` and endpoint
  accessors, 2025.1.4.
- `com/intellij/internal/statistic/eventLog/connection/EventLogUploadSettingsClient.kt`: configuration-client creation,
  2026.1.1 and master.
- `com/intellij/ide/RegionUrlMapper.java`: static override table, `tryMapUrl`, `doLoadMappingOrThrow`, `getConfigUrl`,
  `RegionMapping.fromJson`, and mapping application.
- `com/intellij/ide/Region.java` and `RegionSettings.java`: region names and preference lookup.
- `com/intellij/platform/feedback/impl/GeneralFeedbackSubmit.kt`: independent mapper caller.

These are IntelliJ internal APIs. The blocking mapper overloads are annotated `@RequiresBackgroundThread` and
`@RequiresReadLockAbsence`; the asynchronous overloads return `CompletableFuture<String>`. Startup JVM properties avoid
invoking those APIs from an RCP plugin and require no MPS model access.
