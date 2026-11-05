# Generation options for one make

Which generation options a programmatic make can set for itself only, without changing the process-wide generation
settings. For each option: the API, where the make facets read it or overwrite it, and what it changes. Also: how
generator messages are filtered by level, and what "clean make" does.

Verified against the sources of MPS 2024.1.6, 2025.1.5, 2026.1.1, and `master` at
[`ef47222`](https://github.com/JetBrains/MPS/tree/ef4722267c8ea92f19c7de3c14728533b55bd3d4) (263.x). Source paths are
relative to the [JetBrains/MPS](https://github.com/JetBrains/MPS) repository root. No runtime probe was run for this
note. Claims are marked **Verified** (read in source), **Inferred** (follows from source but not checked at run time) or
**Unresolved**.

See [parallel-generation-per-make.md](parallel-generation-per-make.md) for how to pass options to one make, and
[make-generation.md](make-generation.md) for running a make headlessly.

## Answer

Pass a fresh `GenerationOptions.OptionsBuilder` through `GenerateFacetInitializer`, as for parallel generation:

```kotlin
val settings = project.getComponent(GenerationSettingsProvider::class.java).generationSettings
val options = GenerationOptions.fromSettings(settings)
    .strictMode(true)
    .reporting(/* showInfo = */ true, /* showWarnings = */ true, settings.isKeepModelsWithWarnings, settings.numberOfModelsToKeep)
val controller = IScriptController.Stub2(
    session,
    GenerateFacetInitializer().setGenerationOptions(options),
    TextGenFacetInitializer().generateDebugInfo(false),
)
```

**Verified.** The `Generate` facet takes this builder, but it then overwrites two of its fields from the global
settings. So, for one make:

| Option                            | Settable for one make?                                              |
| --------------------------------- | ------------------------------------------------------------------- |
| Strict mode                       | Yes, `strictMode(boolean)`                                          |
| Parallel generation, thread count | Yes, `generateInParallel(boolean, int)`                             |
| Show info, show warnings          | Yes, `reporting(...)`                                               |
| Keep models with warnings         | Yes, `reporting(...)`                                               |
| Number of transient models kept   | Yes, `reporting(...)`                                               |
| In-place transformations          | Yes, `useInplaceTransformations(boolean)`                           |
| Bad-child warnings                | Yes, `showBadChildWarning(boolean)`                                 |
| Static references, warn dyn→stat  | Yes, `useDynamicReferences(boolean)`, `warnDynamicToStatic(...)`    |
| Debug info (`trace.info`)         | Yes, but through `TextGenFacetInitializer`, not `GenerationOptions` |
| Clean make                        | Yes, through `ModelsToResources(models, cleanMake)`                 |
| **Save transient models**         | **No.** The facet overwrites it from the settings.                  |
| **Performance tracing level**     | **No.** The facet overwrites it from the settings.                  |
| Check models before generation    | No. Only IDE make actions read it.                                  |
| Incremental generation            | Does not exist in these versions.                                   |
| Filter duplicate messages         | Does not exist in these versions.                                   |

The API is the same in all four versions. The only behavior difference found is a `NullPointerException` in the 2024.1
TextGen facet; see [Debug info](#debug-info-traceinfo).

## The builder and its defaults

**Verified.** `jetbrains.mps.generator.GenerationOptions` and its nested `OptionsBuilder`
(`core/generator/source/jetbrains/mps/generator/GenerationOptions.java`, in `mps-generator.jar`) are the same in all
four versions. The builder has a private constructor and no getters. Get one from `GenerationOptions.getDefaults()` or
`GenerationOptions.fromSettings(IGenerationSettings)`.

The builder defaults differ from the settings defaults (`DefaultModifiableGenerationSettings`). Start from
`fromSettings` unless you want the builder defaults:

| Field                     | Builder default | Settings default |
| ------------------------- | --------------- | ---------------- |
| strict mode               | `false`         | `true`           |
| parallel / threads        | `false` / 4     | `false` / 2      |
| show info                 | `false`         | `false`          |
| show warnings             | `true`          | `true`           |
| keep models with warnings | `true`          | `true`           |
| number of models to keep  | 16              | -1 (no limit)    |
| in-place transformations  | `false`         | `true`           |
| static references         | dynamic refs    | static refs      |
| warn dynamic to static    | `false`         | `false`          |

`fromSettings` also turns bad-child warnings and dynamic-to-static warnings off when the settings turn warnings off. A
caller that later calls `reporting(_, false, ...)` must also call `showBadChildWarning(false)` and
`warnDynamicToStatic(false)` to get the same result.

The `Generate` facet changes the builder it gets (tracing, save-transient, parameters, custom plans). Build a new one
for each make.

## How the Generate facet treats the builder

**Verified.** `Generate_Facet.Target_configure` has a job and an `IConfig`
(`languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/Generate_Facet.java`,
same lines in all four versions):

```text
Script.execute
  ctl.setup(pool)                        # Stub2 runs each PropertyPoolInitializer: our builder goes into the pool
  configureTargets                       # inside ctl.runConfigWithMonitor
    Generate.configure IConfig           # saveTransient := settings.isSaveTransientModels()      (L211)
  executeTargets
    Generate.configure job
      if pool builder == null: builder := fromSettings(settings)                              (L166-167)
      builder.tracing(settings.getPerformanceTracingLevel())                                  (L171)
      builder.saveTransientModels(saveTransient)                                              (L172)
      builder.parameters(new DefaultGenerationParametersProvider())                           (L173-174)
      transientModelsProvider.removeAllTransient()                                            (L184)
    Generate.generate job
      new GenerationFacade(repo, builder.create())                                            (L459)
```

`setup` runs before the configs (`Script.java` L211, L215). So even a `saveTransient` value written into the property
pool by an initializer is overwritten by the `IConfig`. Tracing is overwritten inside the job itself.

**Inferred, not tested.** `IScriptController.Stub2` is a public, non-final class. Its `runConfigWithMonitor` wraps the
whole `configureTargets` loop (`Script.java` L322-328). A subclass that keeps the pool from `setup` and writes
`Generate.configure` slot `_0` (`Variables.saveTransient`, `Generate_Facet.java` L272-274) after
`super.runConfigWithMonitor(...)` returns would set save-transient for that make only. This relies on generated facet
internals (the tuple slot order). No such hook exists for tracing: the job reads the settings and calls `create()` in
the same `runJobWithMonitor` call.

## Options

### Strict mode

**Verified.** `OptionsBuilder.strictMode(boolean)`. Read by:

- `GenerationOptions.isGenerateInParallel()` returns `myGenerateInParallel && myStrictMode` (L82). Without strict mode,
  a parallel request generates sequentially without a message.
- `GenerationSession`: a generation plan with conflicting priority rules cancels generation only in strict mode
  (`checkGenerationPlan`, 2024.1/2025.1 L161, 2026.1/master L152). Otherwise generation goes on.
- `TemplateGenerator.areMappingsAvailable()` returns `!strict || mappingsReady` (master L841). In strict mode, mapping
  labels can be read only once the mappings are ready.
- `WeaveRuleBase` (`core/generator/source/jetbrains/mps/generator/runtime/WeaveRuleBase.java` master L102): a bad
  weaving context is an error in strict mode and a warning otherwise.
- `ReferenceInfo_Template.checkCrossRootTemplateReference` (master L79): the cross-root reference check runs only in
  strict mode.

So turning strict mode off can turn generation errors into warnings or into nothing.

### Parallel generation

See [parallel-generation-per-make.md](parallel-generation-per-make.md). It needs strict mode.

### Message levels: show info, show warnings

**Verified.** `OptionsBuilder.reporting(showInfo, showWarnings, keepModelsWithWarnings, numberOfModelsToKeep)`.
`fromSettings` copies all four from the settings (L143).

`GenerationFacade.process0` wraps the session's `IMessageHandler` in
`new GeneratorLoggerAdapter(handler, isShowInfo(), isShowWarnings())` (`GenerationFacade.java` L167).
`GeneratorLoggerAdapter` (`core/generator/source/jetbrains/mps/generator/impl/GeneratorLoggerAdapter.java`):

- `info(...)` returns early unless show-info is on (L73-86).
- `warning(...)` returns early unless show-warnings is on (L95-111).
- `error(...)` always reports (L112-122).
- `needsInfo()` / `needsWarnings()` return the flags. `GenerationController` and `GenerationSession` check `needsInfo()`
  before they build the long info texts: "[model ...]", used languages, mapping configurations, and "generation
  completed successfully in N ms".
- `trace(String)` reports INFORMATION without the check. No caller was found in `core/generator`.

So the filter applies to messages of the model-to-model generator only. These messages reach the session's message
handler without a level filter:

- The performance report of the `Generate` facet: `mh.handle(new Message(MessageKind.INFORMATION, s))`
  (`Generate_Facet.java` L493).
- Generation plan messages: `GenPlanExtractor` gets the session handler directly (`Generate_Facet.java` L404).
- Facet feedback. `IScriptController.Stub2` maps every `IFeedback` to a `Message` through `MessageFeedbackStrategy`
  (ERROR, WARNING, INFO → INFORMATION, anything else → ERROR) with no filter. TextGen uses it for "No models to
  generate", "no output location", timeouts and failures. JavaCompile uses it for the compile result (ERROR, WARNING or
  INFORMATION, `JavaCompile_Facet.java` master L114-121).
- Compiler messages. `JavaCompile_Facet` gives `ModuleMaker` the session handler (master L96).

A caller that wants one strict level for all messages has to filter by `MessageKind` in its own handler as well.

### Keep models with warnings, number of models to keep

**Verified.** Same `reporting(...)` call. Used in `GenerationSession.recordAccessedTransientModels` (2024.1/2025.1
L810-835, 2026.1/master L807-832):

- Only when `keepTransientForMessageNavigation()` is true, which is `!RuntimeFlags.isTestMode()`.
- Transient models that ERROR messages point to are always candidates. Models that WARNING messages point to are
  candidates only when show-warnings and keep-models-with-warnings are both on.
- Candidates are kept in the transient module (`addModelToKeep`) so a user can navigate from the message to the
  transient model. `TransientModelsProvider.canKeepOneMore` (L176-186) limits how many are kept. A value of 0 or less
  means no limit.
- Other transient models are removed unless save-transient is on.

**Inferred, not tested.** When save-transient is off, the `generate` job adds a "Drop transient models" cleanup
(`Generate_Facet.java` L500-509), which `Make.cleanup` runs (`Make_Facet.java` L189-194). So after a successful make the
kept models are gone. When a model fails, the job returns FAILURE (L475) before it adds the cleanup. The kept models
then stay in the project repository until the next make's `configure` job clears them (L184).

### Save transient models

**Verified.** `OptionsBuilder.saveTransientModels(boolean)` exists, but the facet overwrites it from the settings (see
above). With it on, the generator keeps every intermediate model and writes a generation trace (`GenerationSession`
reads `isSaveTransientModels()` at 2026.1 L463, L601, L651, L822, L903), and the `generate` job adds no "Drop transient
models" cleanup.

Transient models live in modules named after the input module, created by `TransientModelsProvider.createModule`. The
`generate` job publishes them into the project repository (`Generate_Facet.java` L466). With an IDE environment the
provider is `TransientModelsComponent` (`workbench/mps-workbench/source/META-INF/MPSComponents.xml`: a project component
in 2024.1, a project service from 2025.1). Without a provider the facet creates its own and disposes it in a cleanup
step (L180-193).

**Inferred.** The models stay in the repository until the next make's `configure` job calls `removeAllTransient()`
(L184), or until the project closes.

The IDE changes this option only through the global settings (`SaveTransientModelsAction`, `TransientModelsWidget` in
`plugins/mps-make`).

### Performance tracing

**Verified.** `OptionsBuilder.tracing(int)` with `TRACE_OFF`, `TRACE_STEPS`, `TRACE_LANGS`, `TRACE_TYPES` (0-3). The
`configure` job overwrites it from `settings.getPerformanceTracingLevel()` (L171). `GenerationController` creates a
`PerformanceTracer` per model when it is not off. The `generate` job reports the result as INFORMATION messages, one per
line, with no level filter (L493).

### In-place transformations, bad-child warnings, static references

**Verified.** Per make through the builder. In-place transformations let a step change its input model directly when the
step has no weaving rules (`TemplateGenerator`). Bad-child warnings check that a child fits its containment role.
`useDynamicReferences(false)` replaces dynamic references with static ones after a step; `warnDynamicToStatic(true)`
warns when that fails. These are generator internals; they do not change what a correct generator produces.

### Debug info (`trace.info`)

**Verified.** Not in `GenerationOptions`. `IGenerationSettings.isGenerateDebugInfo()` is deprecated; the value lives in
`jetbrains.mps.text.TextGenSettings.isGenerateDebugInfo()` (default `true`). The TextGen facet has no `IConfig`, so an
initializer value wins:

```kotlin
IScriptController.Stub2(session, TextGenFacetInitializer().generateDebugInfo(false))
```

`TextGenFacetInitializer.generateDebugInfo(Boolean)` writes slot `_0` of the `jetbrains.mps.make.facets.TextGen.textGen`
properties
(`core/make-runtime/solutions/jetbrains.mps.make/source_gen/jetbrains/mps/internal/make/cfg/TextGenFacetInitializer.java`
L18-30). TextGen then writes `trace.info` only when the flag is on (master L350).

**Version difference (Verified).** The TextGen facet decides the flag like this
(`languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/TextGen_Facet.java`):

```java
// 2024.1, L228
final boolean _generateDebugInfo = (vars(pa.global()).generateDebugInfo() == null && tgs.isGenerateDebugInfo()) || vars(pa.global()).generateDebugInfo();
// 2025.1 L228, 2026.1 and master L235
final boolean _generateDebugInfo = (vars(pa.global()).generateDebugInfo() == null ? tgs.isGenerateDebugInfo() : vars(pa.global()).generateDebugInfo());
```

In 2024.1, when the property is `null` and the setting is `false`, the second operand unboxes `null` and throws a
`NullPointerException`. With the default setting (`true`) the first operand short-circuits. Setting the property to a
non-null value avoids the problem in every version.

### Clean make

**Verified.** The flag that matters is `MakeKeys.CLEAN_MAKE` on each resource, not `MakeSession.isCleanMake()`.

- `ModelsToResources(models, cleanMake)` puts it on each `MResource`
  (`core/resources/source_gen/jetbrains/mps/smodel/resources/ModelsToResources.java` L68). The one-argument constructor
  passes `false` (L30-41).
- The `generate` job passes it on to its output (`Generate_Facet.java` L437, L481). The generator itself does not use
  it. In 2024.1, L437 reads the key without `hasKey`; resources from `ModelsToResources` always have it.
- TextGen calls `collectGeneratedFilesForceClean()` for a clean resource and `collectGeneratedFiles()` otherwise
  (2024.1/2025.1 L205-209, 2026.1/master L211-215). Force-clean treats every file under the model's output root on disk
  as stale. The normal path treats only files recorded as generated before as stale. Stale files that this make does not
  write again are deleted.
- `MakeSession.isCleanMake()` is read only by `WorkbenchMakeService`, for the "Rebuild"/"Make" script name.

Which models are made is up to the caller. The IDE filters out models that do not need generation
(`ModelGenerationStatusManager`); `ModelsToResources` does not.

### Check models before generation

**Verified.** Settings only (`isCheckModelsBeforeGeneration`). Read by IDE make actions through
`workbench/mps-workbench/source/jetbrains/mps/ide/generator/GenerationCheckHelper.java` and by the model checker's
`ModelValidatorAdapter`. The make facets do not read it.

### Not present

**Verified** by search of `core`, `languages`, `plugins` and `workbench`: there is no incremental-generation option
(`GenerationCacheContainer` is looked up in the `configure` job and not used), and no duplicate-message filter.

## The global settings

**Verified.** `GenerationSettingsProvider` (a core component) holds one `IModifiableGenerationSettings` for the process.
Its `init` installs a `DefaultModifiableGenerationSettings`. In an IDE environment,
`jetbrains.mps.ide.generator.GenerationSettings` replaces it with its own state. That class is an application-level
`PersistentStateComponent` stored in `generationSettings.xml` (an application component in 2024.1, an application
service from 2025.1). Its state includes debug info and the TextGen timeout, which it copies into `TextGenSettings`.

Readers of the settings during a make: the `Generate` facet (`configure` job and `IConfig`), the `Documentation` facet
of `jetbrains.mps.lang.core.doc` (it builds its own options with `fromSettings`), and IDE-only code (pre-generation
model check, transient-model widget and its settings listener).

**Inferred.** Changing the settings object for one make changes it for everything else in the process during that time.
Whether the change is written to `generationSettings.xml` depends on application saving; in a headless application
saving is off by default (see [project-settings-save.md](project-settings-save.md)).

## Source locations

Lines are the same in all four versions unless listed per version.

| What                                                                                            | Where                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| ----------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `configure` job uses caller's builder if set (`Generate_Facet.java`)                            | L166-167: [2024.1](https://github.com/JetBrains/MPS/blob/2024.1.6/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/Generate_Facet.java#L166-L167), [2025.1](https://github.com/JetBrains/MPS/blob/2025.1.5/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/Generate_Facet.java#L166-L167), [2026.1](https://github.com/JetBrains/MPS/blob/2026.1.1/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/Generate_Facet.java#L166-L167), [master](https://github.com/JetBrains/MPS/blob/ef4722267c8ea92f19c7de3c14728533b55bd3d4/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/Generate_Facet.java#L166-L167)                                                       |
| `configure` job overwrites tracing from settings (`Generate_Facet.java`)                        | L171: [2024.1](https://github.com/JetBrains/MPS/blob/2024.1.6/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/Generate_Facet.java#L171), [2025.1](https://github.com/JetBrains/MPS/blob/2025.1.5/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/Generate_Facet.java#L171), [2026.1](https://github.com/JetBrains/MPS/blob/2026.1.1/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/Generate_Facet.java#L171), [master](https://github.com/JetBrains/MPS/blob/ef4722267c8ea92f19c7de3c14728533b55bd3d4/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/Generate_Facet.java#L171)                                                                               |
| `configure` job overwrites save-transient from `saveTransient` property (`Generate_Facet.java`) | L172: [2024.1](https://github.com/JetBrains/MPS/blob/2024.1.6/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/Generate_Facet.java#L172), [2025.1](https://github.com/JetBrains/MPS/blob/2025.1.5/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/Generate_Facet.java#L172), [2026.1](https://github.com/JetBrains/MPS/blob/2026.1.1/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/Generate_Facet.java#L172), [master](https://github.com/JetBrains/MPS/blob/ef4722267c8ea92f19c7de3c14728533b55bd3d4/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/Generate_Facet.java#L172)                                                                               |
| `configure` job replaces parameters provider (`Generate_Facet.java`)                            | L173-174: [2024.1](https://github.com/JetBrains/MPS/blob/2024.1.6/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/Generate_Facet.java#L173-L174), [2025.1](https://github.com/JetBrains/MPS/blob/2025.1.5/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/Generate_Facet.java#L173-L174), [2026.1](https://github.com/JetBrains/MPS/blob/2026.1.1/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/Generate_Facet.java#L173-L174), [master](https://github.com/JetBrains/MPS/blob/ef4722267c8ea92f19c7de3c14728533b55bd3d4/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/Generate_Facet.java#L173-L174)                                                       |
| `configure` job clears transients of the previous make (`Generate_Facet.java`)                  | L184: [2024.1](https://github.com/JetBrains/MPS/blob/2024.1.6/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/Generate_Facet.java#L184), [2025.1](https://github.com/JetBrains/MPS/blob/2025.1.5/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/Generate_Facet.java#L184), [2026.1](https://github.com/JetBrains/MPS/blob/2026.1.1/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/Generate_Facet.java#L184), [master](https://github.com/JetBrains/MPS/blob/ef4722267c8ea92f19c7de3c14728533b55bd3d4/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/Generate_Facet.java#L184)                                                                               |
| `configure` IConfig sets `saveTransient` from settings (`Generate_Facet.java`)                  | L211: [2024.1](https://github.com/JetBrains/MPS/blob/2024.1.6/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/Generate_Facet.java#L211), [2025.1](https://github.com/JetBrains/MPS/blob/2025.1.5/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/Generate_Facet.java#L211), [2026.1](https://github.com/JetBrains/MPS/blob/2026.1.1/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/Generate_Facet.java#L211), [master](https://github.com/JetBrains/MPS/blob/ef4722267c8ea92f19c7de3c14728533b55bd3d4/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/Generate_Facet.java#L211)                                                                               |
| `Variables.saveTransient` is tuple slot `_0` (`Generate_Facet.java`)                            | L272-274: [2024.1](https://github.com/JetBrains/MPS/blob/2024.1.6/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/Generate_Facet.java#L272-L274), [2025.1](https://github.com/JetBrains/MPS/blob/2025.1.5/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/Generate_Facet.java#L272-L274), [2026.1](https://github.com/JetBrains/MPS/blob/2026.1.1/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/Generate_Facet.java#L272-L274), [master](https://github.com/JetBrains/MPS/blob/ef4722267c8ea92f19c7de3c14728533b55bd3d4/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/Generate_Facet.java#L272-L274)                                                       |
| `generate` job creates `GenerationFacade` from the builder (`Generate_Facet.java`)              | L459: [2024.1](https://github.com/JetBrains/MPS/blob/2024.1.6/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/Generate_Facet.java#L459), [2025.1](https://github.com/JetBrains/MPS/blob/2025.1.5/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/Generate_Facet.java#L459), [2026.1](https://github.com/JetBrains/MPS/blob/2026.1.1/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/Generate_Facet.java#L459), [master](https://github.com/JetBrains/MPS/blob/ef4722267c8ea92f19c7de3c14728533b55bd3d4/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/Generate_Facet.java#L459)                                                                               |
| `generate` job publishes transient modules (`Generate_Facet.java`)                              | L466: [2024.1](https://github.com/JetBrains/MPS/blob/2024.1.6/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/Generate_Facet.java#L466), [2025.1](https://github.com/JetBrains/MPS/blob/2025.1.5/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/Generate_Facet.java#L466), [2026.1](https://github.com/JetBrains/MPS/blob/2026.1.1/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/Generate_Facet.java#L466), [master](https://github.com/JetBrains/MPS/blob/ef4722267c8ea92f19c7de3c14728533b55bd3d4/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/Generate_Facet.java#L466)                                                                               |
| `generate` job returns FAILURE on a failed model (`Generate_Facet.java`)                        | L475: [2024.1](https://github.com/JetBrains/MPS/blob/2024.1.6/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/Generate_Facet.java#L475), [2025.1](https://github.com/JetBrains/MPS/blob/2025.1.5/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/Generate_Facet.java#L475), [2026.1](https://github.com/JetBrains/MPS/blob/2026.1.1/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/Generate_Facet.java#L475), [master](https://github.com/JetBrains/MPS/blob/ef4722267c8ea92f19c7de3c14728533b55bd3d4/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/Generate_Facet.java#L475)                                                                               |
| `generate` job passes `CLEAN_MAKE` on to TextGen (`Generate_Facet.java`)                        | L481: [2024.1](https://github.com/JetBrains/MPS/blob/2024.1.6/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/Generate_Facet.java#L481), [2025.1](https://github.com/JetBrains/MPS/blob/2025.1.5/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/Generate_Facet.java#L481), [2026.1](https://github.com/JetBrains/MPS/blob/2026.1.1/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/Generate_Facet.java#L481), [master](https://github.com/JetBrains/MPS/blob/ef4722267c8ea92f19c7de3c14728533b55bd3d4/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/Generate_Facet.java#L481)                                                                               |
| `generate` job sends the performance report as INFORMATION (`Generate_Facet.java`)              | L493: [2024.1](https://github.com/JetBrains/MPS/blob/2024.1.6/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/Generate_Facet.java#L493), [2025.1](https://github.com/JetBrains/MPS/blob/2025.1.5/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/Generate_Facet.java#L493), [2026.1](https://github.com/JetBrains/MPS/blob/2026.1.1/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/Generate_Facet.java#L493), [master](https://github.com/JetBrains/MPS/blob/ef4722267c8ea92f19c7de3c14728533b55bd3d4/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/Generate_Facet.java#L493)                                                                               |
| `generate` job adds the "Drop transient models" cleanup (`Generate_Facet.java`)                 | L500-509: [2024.1](https://github.com/JetBrains/MPS/blob/2024.1.6/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/Generate_Facet.java#L500-L509), [2025.1](https://github.com/JetBrains/MPS/blob/2025.1.5/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/Generate_Facet.java#L500-L509), [2026.1](https://github.com/JetBrains/MPS/blob/2026.1.1/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/Generate_Facet.java#L500-L509), [master](https://github.com/JetBrains/MPS/blob/ef4722267c8ea92f19c7de3c14728533b55bd3d4/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/Generate_Facet.java#L500-L509)                                                       |
| `generate` job reads `CLEAN_MAKE` (no `hasKey` in 2024.1) (`Generate_Facet.java`)               | L437: [2024.1](https://github.com/JetBrains/MPS/blob/2024.1.6/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/Generate_Facet.java#L437), [2025.1](https://github.com/JetBrains/MPS/blob/2025.1.5/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/Generate_Facet.java#L437), [2026.1](https://github.com/JetBrains/MPS/blob/2026.1.1/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/Generate_Facet.java#L437), [master](https://github.com/JetBrains/MPS/blob/ef4722267c8ea92f19c7de3c14728533b55bd3d4/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/Generate_Facet.java#L437)                                                                               |
| `isGenerateInParallel()` = parallel && strict (`GenerationOptions.java`)                        | L82: [2024.1](https://github.com/JetBrains/MPS/blob/2024.1.6/core/generator/source/jetbrains/mps/generator/GenerationOptions.java#L82), [2025.1](https://github.com/JetBrains/MPS/blob/2025.1.5/core/generator/source/jetbrains/mps/generator/GenerationOptions.java#L82), [2026.1](https://github.com/JetBrains/MPS/blob/2026.1.1/core/generator/source/jetbrains/mps/generator/GenerationOptions.java#L82), [master](https://github.com/JetBrains/MPS/blob/ef4722267c8ea92f19c7de3c14728533b55bd3d4/core/generator/source/jetbrains/mps/generator/GenerationOptions.java#L82)                                                                                                                                                                                                                                                                                                                                |
| `fromSettings` copies settings (`GenerationOptions.java`)                                       | L138-147: [2024.1](https://github.com/JetBrains/MPS/blob/2024.1.6/core/generator/source/jetbrains/mps/generator/GenerationOptions.java#L138-L147), [2025.1](https://github.com/JetBrains/MPS/blob/2025.1.5/core/generator/source/jetbrains/mps/generator/GenerationOptions.java#L138-L147), [2026.1](https://github.com/JetBrains/MPS/blob/2026.1.1/core/generator/source/jetbrains/mps/generator/GenerationOptions.java#L138-L147), [master](https://github.com/JetBrains/MPS/blob/ef4722267c8ea92f19c7de3c14728533b55bd3d4/core/generator/source/jetbrains/mps/generator/GenerationOptions.java#L138-L147)                                                                                                                                                                                                                                                                                                   |
| `OptionsBuilder` field defaults (`GenerationOptions.java`)                                      | L162-177: [2024.1](https://github.com/JetBrains/MPS/blob/2024.1.6/core/generator/source/jetbrains/mps/generator/GenerationOptions.java#L162-L177), [2025.1](https://github.com/JetBrains/MPS/blob/2025.1.5/core/generator/source/jetbrains/mps/generator/GenerationOptions.java#L162-L177), [2026.1](https://github.com/JetBrains/MPS/blob/2026.1.1/core/generator/source/jetbrains/mps/generator/GenerationOptions.java#L162-L177), [master](https://github.com/JetBrains/MPS/blob/ef4722267c8ea92f19c7de3c14728533b55bd3d4/core/generator/source/jetbrains/mps/generator/GenerationOptions.java#L162-L177)                                                                                                                                                                                                                                                                                                   |
| `OptionsBuilder` private constructor (`GenerationOptions.java`)                                 | L183: [2024.1](https://github.com/JetBrains/MPS/blob/2024.1.6/core/generator/source/jetbrains/mps/generator/GenerationOptions.java#L183), [2025.1](https://github.com/JetBrains/MPS/blob/2025.1.5/core/generator/source/jetbrains/mps/generator/GenerationOptions.java#L183), [2026.1](https://github.com/JetBrains/MPS/blob/2026.1.1/core/generator/source/jetbrains/mps/generator/GenerationOptions.java#L183), [master](https://github.com/JetBrains/MPS/blob/ef4722267c8ea92f19c7de3c14728533b55bd3d4/core/generator/source/jetbrains/mps/generator/GenerationOptions.java#L183)                                                                                                                                                                                                                                                                                                                           |
| `reporting(...)` (`GenerationOptions.java`)                                                     | L230: [2024.1](https://github.com/JetBrains/MPS/blob/2024.1.6/core/generator/source/jetbrains/mps/generator/GenerationOptions.java#L230), [2025.1](https://github.com/JetBrains/MPS/blob/2025.1.5/core/generator/source/jetbrains/mps/generator/GenerationOptions.java#L230), [2026.1](https://github.com/JetBrains/MPS/blob/2026.1.1/core/generator/source/jetbrains/mps/generator/GenerationOptions.java#L230), [master](https://github.com/JetBrains/MPS/blob/ef4722267c8ea92f19c7de3c14728533b55bd3d4/core/generator/source/jetbrains/mps/generator/GenerationOptions.java#L230)                                                                                                                                                                                                                                                                                                                           |
| `tracing(int)` (`GenerationOptions.java`)                                                       | L238: [2024.1](https://github.com/JetBrains/MPS/blob/2024.1.6/core/generator/source/jetbrains/mps/generator/GenerationOptions.java#L238), [2025.1](https://github.com/JetBrains/MPS/blob/2025.1.5/core/generator/source/jetbrains/mps/generator/GenerationOptions.java#L238), [2026.1](https://github.com/JetBrains/MPS/blob/2026.1.1/core/generator/source/jetbrains/mps/generator/GenerationOptions.java#L238), [master](https://github.com/JetBrains/MPS/blob/ef4722267c8ea92f19c7de3c14728533b55bd3d4/core/generator/source/jetbrains/mps/generator/GenerationOptions.java#L238)                                                                                                                                                                                                                                                                                                                           |
| Settings defaults (`DefaultModifiableGenerationSettings.java`)                                  | L24-40: [2024.1](https://github.com/JetBrains/MPS/blob/2024.1.6/core/generator/source/jetbrains/mps/generator/DefaultModifiableGenerationSettings.java#L24-L40), [2025.1](https://github.com/JetBrains/MPS/blob/2025.1.5/core/generator/source/jetbrains/mps/generator/DefaultModifiableGenerationSettings.java#L24-L40), [2026.1](https://github.com/JetBrains/MPS/blob/2026.1.1/core/generator/source/jetbrains/mps/generator/DefaultModifiableGenerationSettings.java#L24-L40), [master](https://github.com/JetBrains/MPS/blob/ef4722267c8ea92f19c7de3c14728533b55bd3d4/core/generator/source/jetbrains/mps/generator/DefaultModifiableGenerationSettings.java#L24-L40)                                                                                                                                                                                                                                     |
| Logger adapter built from `isShowInfo`/`isShowWarnings` (`GenerationFacade.java`)               | L167: [2024.1](https://github.com/JetBrains/MPS/blob/2024.1.6/core/generator/source/jetbrains/mps/generator/GenerationFacade.java#L167), [2025.1](https://github.com/JetBrains/MPS/blob/2025.1.5/core/generator/source/jetbrains/mps/generator/GenerationFacade.java#L167), [2026.1](https://github.com/JetBrains/MPS/blob/2026.1.1/core/generator/source/jetbrains/mps/generator/GenerationFacade.java#L167), [master](https://github.com/JetBrains/MPS/blob/ef4722267c8ea92f19c7de3c14728533b55bd3d4/core/generator/source/jetbrains/mps/generator/GenerationFacade.java#L167)                                                                                                                                                                                                                                                                                                                               |
| `info` returns early unless show-info (`GeneratorLoggerAdapter.java`)                           | L73-86: [2024.1](https://github.com/JetBrains/MPS/blob/2024.1.6/core/generator/source/jetbrains/mps/generator/impl/GeneratorLoggerAdapter.java#L73-L86), [2025.1](https://github.com/JetBrains/MPS/blob/2025.1.5/core/generator/source/jetbrains/mps/generator/impl/GeneratorLoggerAdapter.java#L73-L86), [2026.1](https://github.com/JetBrains/MPS/blob/2026.1.1/core/generator/source/jetbrains/mps/generator/impl/GeneratorLoggerAdapter.java#L73-L86), [master](https://github.com/JetBrains/MPS/blob/ef4722267c8ea92f19c7de3c14728533b55bd3d4/core/generator/source/jetbrains/mps/generator/impl/GeneratorLoggerAdapter.java#L73-L86)                                                                                                                                                                                                                                                                     |
| `warning` returns early unless show-warnings (`GeneratorLoggerAdapter.java`)                    | L95-111: [2024.1](https://github.com/JetBrains/MPS/blob/2024.1.6/core/generator/source/jetbrains/mps/generator/impl/GeneratorLoggerAdapter.java#L95-L111), [2025.1](https://github.com/JetBrains/MPS/blob/2025.1.5/core/generator/source/jetbrains/mps/generator/impl/GeneratorLoggerAdapter.java#L95-L111), [2026.1](https://github.com/JetBrains/MPS/blob/2026.1.1/core/generator/source/jetbrains/mps/generator/impl/GeneratorLoggerAdapter.java#L95-L111), [master](https://github.com/JetBrains/MPS/blob/ef4722267c8ea92f19c7de3c14728533b55bd3d4/core/generator/source/jetbrains/mps/generator/impl/GeneratorLoggerAdapter.java#L95-L111)                                                                                                                                                                                                                                                                |
| `error` always reports (`GeneratorLoggerAdapter.java`)                                          | L112-122: [2024.1](https://github.com/JetBrains/MPS/blob/2024.1.6/core/generator/source/jetbrains/mps/generator/impl/GeneratorLoggerAdapter.java#L112-L122), [2025.1](https://github.com/JetBrains/MPS/blob/2025.1.5/core/generator/source/jetbrains/mps/generator/impl/GeneratorLoggerAdapter.java#L112-L122), [2026.1](https://github.com/JetBrains/MPS/blob/2026.1.1/core/generator/source/jetbrains/mps/generator/impl/GeneratorLoggerAdapter.java#L112-L122), [master](https://github.com/JetBrains/MPS/blob/ef4722267c8ea92f19c7de3c14728533b55bd3d4/core/generator/source/jetbrains/mps/generator/impl/GeneratorLoggerAdapter.java#L112-L122)                                                                                                                                                                                                                                                           |
| `checkGenerationPlan` cancels only in strict mode (`GenerationSession.java`)                    | [2024.1 L161](https://github.com/JetBrains/MPS/blob/2024.1.6/core/generator/source/jetbrains/mps/generator/impl/GenerationSession.java#L161), [2025.1 L161](https://github.com/JetBrains/MPS/blob/2025.1.5/core/generator/source/jetbrains/mps/generator/impl/GenerationSession.java#L161), [2026.1 L152](https://github.com/JetBrains/MPS/blob/2026.1.1/core/generator/source/jetbrains/mps/generator/impl/GenerationSession.java#L152), [master L152](https://github.com/JetBrains/MPS/blob/ef4722267c8ea92f19c7de3c14728533b55bd3d4/core/generator/source/jetbrains/mps/generator/impl/GenerationSession.java#L152)                                                                                                                                                                                                                                                                                         |
| `recordAccessedTransientModels` (`GenerationSession.java`)                                      | [2024.1 L810-835](https://github.com/JetBrains/MPS/blob/2024.1.6/core/generator/source/jetbrains/mps/generator/impl/GenerationSession.java#L810-L835), [2025.1 L810-835](https://github.com/JetBrains/MPS/blob/2025.1.5/core/generator/source/jetbrains/mps/generator/impl/GenerationSession.java#L810-L835), [2026.1 L807-832](https://github.com/JetBrains/MPS/blob/2026.1.1/core/generator/source/jetbrains/mps/generator/impl/GenerationSession.java#L807-L832), [master L807-832](https://github.com/JetBrains/MPS/blob/ef4722267c8ea92f19c7de3c14728533b55bd3d4/core/generator/source/jetbrains/mps/generator/impl/GenerationSession.java#L807-L832)                                                                                                                                                                                                                                                     |
| `keepTransientForMessageNavigation` = not test mode (`GenerationSession.java`)                  | [2024.1 L837-841](https://github.com/JetBrains/MPS/blob/2024.1.6/core/generator/source/jetbrains/mps/generator/impl/GenerationSession.java#L837-L841), [2025.1 L837-841](https://github.com/JetBrains/MPS/blob/2025.1.5/core/generator/source/jetbrains/mps/generator/impl/GenerationSession.java#L837-L841), [2026.1 L834-838](https://github.com/JetBrains/MPS/blob/2026.1.1/core/generator/source/jetbrains/mps/generator/impl/GenerationSession.java#L834-L838), [master L834-838](https://github.com/JetBrains/MPS/blob/ef4722267c8ea92f19c7de3c14728533b55bd3d4/core/generator/source/jetbrains/mps/generator/impl/GenerationSession.java#L834-L838)                                                                                                                                                                                                                                                     |
| strict mode read by `TemplateGenerator` (`TemplateGenerator.java`)                              | [2024.1 L158](https://github.com/JetBrains/MPS/blob/2024.1.6/core/generator/source/jetbrains/mps/generator/impl/TemplateGenerator.java#L158), [2025.1 L158](https://github.com/JetBrains/MPS/blob/2025.1.5/core/generator/source/jetbrains/mps/generator/impl/TemplateGenerator.java#L158), [2026.1 L135](https://github.com/JetBrains/MPS/blob/2026.1.1/core/generator/source/jetbrains/mps/generator/impl/TemplateGenerator.java#L135), [master L135](https://github.com/JetBrains/MPS/blob/ef4722267c8ea92f19c7de3c14728533b55bd3d4/core/generator/source/jetbrains/mps/generator/impl/TemplateGenerator.java#L135)                                                                                                                                                                                                                                                                                         |
| `setup` runs before `configureTargets` (`Script.java`)                                          | L211-215: [2024.1](https://github.com/JetBrains/MPS/blob/2024.1.6/core/make-runtime/solutions/jetbrains.mps.make.runtime/source_gen/jetbrains/mps/internal/make/runtime/script/Script.java#L211-L215), [2025.1](https://github.com/JetBrains/MPS/blob/2025.1.5/core/make-runtime/solutions/jetbrains.mps.make.runtime/source_gen/jetbrains/mps/internal/make/runtime/script/Script.java#L211-L215), [2026.1](https://github.com/JetBrains/MPS/blob/2026.1.1/core/make-runtime/solutions/jetbrains.mps.make.runtime/source_gen/jetbrains/mps/internal/make/runtime/script/Script.java#L211-L215), [master](https://github.com/JetBrains/MPS/blob/ef4722267c8ea92f19c7de3c14728533b55bd3d4/core/make-runtime/solutions/jetbrains.mps.make.runtime/source_gen/jetbrains/mps/internal/make/runtime/script/Script.java#L211-L215)                                                                                   |
| `configureTargets` runs all IConfigs inside `runConfigWithMonitor` (`Script.java`)              | L322-328: [2024.1](https://github.com/JetBrains/MPS/blob/2024.1.6/core/make-runtime/solutions/jetbrains.mps.make.runtime/source_gen/jetbrains/mps/internal/make/runtime/script/Script.java#L322-L328), [2025.1](https://github.com/JetBrains/MPS/blob/2025.1.5/core/make-runtime/solutions/jetbrains.mps.make.runtime/source_gen/jetbrains/mps/internal/make/runtime/script/Script.java#L322-L328), [2026.1](https://github.com/JetBrains/MPS/blob/2026.1.1/core/make-runtime/solutions/jetbrains.mps.make.runtime/source_gen/jetbrains/mps/internal/make/runtime/script/Script.java#L322-L328), [master](https://github.com/JetBrains/MPS/blob/ef4722267c8ea92f19c7de3c14728533b55bd3d4/core/make-runtime/solutions/jetbrains.mps.make.runtime/source_gen/jetbrains/mps/internal/make/runtime/script/Script.java#L322-L328)                                                                                   |
| `Stub2` (not final) and its `setup` (`IScriptController.java`)                                  | L57-95: [2024.1](https://github.com/JetBrains/MPS/blob/2024.1.6/core/make-runtime/solutions/jetbrains.mps.make.runtime/source_gen/jetbrains/mps/make/script/IScriptController.java#L57-L95), [2025.1](https://github.com/JetBrains/MPS/blob/2025.1.5/core/make-runtime/solutions/jetbrains.mps.make.runtime/source_gen/jetbrains/mps/make/script/IScriptController.java#L57-L95), [2026.1](https://github.com/JetBrains/MPS/blob/2026.1.1/core/make-runtime/solutions/jetbrains.mps.make.runtime/source_gen/jetbrains/mps/make/script/IScriptController.java#L57-L95), [master](https://github.com/JetBrains/MPS/blob/ef4722267c8ea92f19c7de3c14728533b55bd3d4/core/make-runtime/solutions/jetbrains.mps.make.runtime/source_gen/jetbrains/mps/make/script/IScriptController.java#L57-L95)                                                                                                                     |
| Feedback severity to `MessageKind`, no filter (`MessageFeedbackStrategy.java`)                  | L17-37: [2024.1](https://github.com/JetBrains/MPS/blob/2024.1.6/core/make-runtime/solutions/jetbrains.mps.make.runtime/source_gen/jetbrains/mps/internal/make/runtime/script/MessageFeedbackStrategy.java#L17-L37), [2025.1](https://github.com/JetBrains/MPS/blob/2025.1.5/core/make-runtime/solutions/jetbrains.mps.make.runtime/source_gen/jetbrains/mps/internal/make/runtime/script/MessageFeedbackStrategy.java#L17-L37), [2026.1](https://github.com/JetBrains/MPS/blob/2026.1.1/core/make-runtime/solutions/jetbrains.mps.make.runtime/source_gen/jetbrains/mps/internal/make/runtime/script/MessageFeedbackStrategy.java#L17-L37), [master](https://github.com/JetBrains/MPS/blob/ef4722267c8ea92f19c7de3c14728533b55bd3d4/core/make-runtime/solutions/jetbrains.mps.make.runtime/source_gen/jetbrains/mps/internal/make/runtime/script/MessageFeedbackStrategy.java#L17-L37)                         |
| JavaCompile result as ERROR/WARNING/INFORMATION feedback (`JavaCompile_Facet.java`)             | L114-121: [2024.1](https://github.com/JetBrains/MPS/blob/2024.1.6/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/make/facets/JavaCompile_Facet.java#L114-L121), [2025.1](https://github.com/JetBrains/MPS/blob/2025.1.5/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/make/facets/JavaCompile_Facet.java#L114-L121), [2026.1](https://github.com/JetBrains/MPS/blob/2026.1.1/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/make/facets/JavaCompile_Facet.java#L114-L121), [master](https://github.com/JetBrains/MPS/blob/ef4722267c8ea92f19c7de3c14728533b55bd3d4/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/make/facets/JavaCompile_Facet.java#L114-L121)                                                               |
| `TextGenFacetInitializer.generateDebugInfo(Boolean)` (`TextGenFacetInitializer.java`)           | L18-30: [2024.1](https://github.com/JetBrains/MPS/blob/2024.1.6/core/make-runtime/solutions/jetbrains.mps.make/source_gen/jetbrains/mps/internal/make/cfg/TextGenFacetInitializer.java#L18-L30), [2025.1](https://github.com/JetBrains/MPS/blob/2025.1.5/core/make-runtime/solutions/jetbrains.mps.make/source_gen/jetbrains/mps/internal/make/cfg/TextGenFacetInitializer.java#L18-L30), [2026.1](https://github.com/JetBrains/MPS/blob/2026.1.1/core/make-runtime/solutions/jetbrains.mps.make/source_gen/jetbrains/mps/internal/make/cfg/TextGenFacetInitializer.java#L18-L30), [master](https://github.com/JetBrains/MPS/blob/ef4722267c8ea92f19c7de3c14728533b55bd3d4/core/make-runtime/solutions/jetbrains.mps.make/source_gen/jetbrains/mps/internal/make/cfg/TextGenFacetInitializer.java#L18-L30)                                                                                                     |
| TextGen debug-info decision (`TextGen_Facet.java`)                                              | [2024.1 L228](https://github.com/JetBrains/MPS/blob/2024.1.6/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/TextGen_Facet.java#L228), [2025.1 L228](https://github.com/JetBrains/MPS/blob/2025.1.5/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/TextGen_Facet.java#L228), [2026.1 L235](https://github.com/JetBrains/MPS/blob/2026.1.1/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/TextGen_Facet.java#L235), [master L235](https://github.com/JetBrains/MPS/blob/ef4722267c8ea92f19c7de3c14728533b55bd3d4/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/TextGen_Facet.java#L235)                                                                     |
| TextGen clean make: force-clean stale files (`TextGen_Facet.java`)                              | [2024.1 L205-209](https://github.com/JetBrains/MPS/blob/2024.1.6/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/TextGen_Facet.java#L205-L209), [2025.1 L205-209](https://github.com/JetBrains/MPS/blob/2025.1.5/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/TextGen_Facet.java#L205-L209), [2026.1 L211-215](https://github.com/JetBrains/MPS/blob/2026.1.1/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/TextGen_Facet.java#L211-L215), [master L211-215](https://github.com/JetBrains/MPS/blob/ef4722267c8ea92f19c7de3c14728533b55bd3d4/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/TextGen_Facet.java#L211-L215)                                 |
| `collectGeneratedFilesForceClean` / `collectGeneratedFiles` (`ModuleStaleFileManager.java`)     | [2024.1 L78-109](https://github.com/JetBrains/MPS/blob/2024.1.6/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/ModuleStaleFileManager.java#L78-L109), [2025.1 L78-109](https://github.com/JetBrains/MPS/blob/2025.1.5/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/ModuleStaleFileManager.java#L78-L109), [2026.1 L100-131](https://github.com/JetBrains/MPS/blob/2026.1.1/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/ModuleStaleFileManager.java#L100-L131), [master L100-131](https://github.com/JetBrains/MPS/blob/ef4722267c8ea92f19c7de3c14728533b55bd3d4/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/ModuleStaleFileManager.java#L100-L131) |
| `ModelsToResources(models)` defaults `cleanMake` to false (`ModelsToResources.java`)            | L30-41: [2024.1](https://github.com/JetBrains/MPS/blob/2024.1.6/core/resources/source_gen/jetbrains/mps/smodel/resources/ModelsToResources.java#L30-L41), [2025.1](https://github.com/JetBrains/MPS/blob/2025.1.5/core/resources/source_gen/jetbrains/mps/smodel/resources/ModelsToResources.java#L30-L41), [2026.1](https://github.com/JetBrains/MPS/blob/2026.1.1/core/resources/source_gen/jetbrains/mps/smodel/resources/ModelsToResources.java#L30-L41), [master](https://github.com/JetBrains/MPS/blob/ef4722267c8ea92f19c7de3c14728533b55bd3d4/core/resources/source_gen/jetbrains/mps/smodel/resources/ModelsToResources.java#L30-L41)                                                                                                                                                                                                                                                                 |
| `ModelsToResources` puts `CLEAN_MAKE` on each resource (`ModelsToResources.java`)               | L68: [2024.1](https://github.com/JetBrains/MPS/blob/2024.1.6/core/resources/source_gen/jetbrains/mps/smodel/resources/ModelsToResources.java#L68), [2025.1](https://github.com/JetBrains/MPS/blob/2025.1.5/core/resources/source_gen/jetbrains/mps/smodel/resources/ModelsToResources.java#L68), [2026.1](https://github.com/JetBrains/MPS/blob/2026.1.1/core/resources/source_gen/jetbrains/mps/smodel/resources/ModelsToResources.java#L68), [master](https://github.com/JetBrains/MPS/blob/ef4722267c8ea92f19c7de3c14728533b55bd3d4/core/resources/source_gen/jetbrains/mps/smodel/resources/ModelsToResources.java#L68)                                                                                                                                                                                                                                                                                    |
| `MakeSession.isCleanMake()` (`MakeSession.java`)                                                | L29-48: [2024.1](https://github.com/JetBrains/MPS/blob/2024.1.6/core/make-runtime/solutions/jetbrains.mps.make.runtime/source_gen/jetbrains/mps/make/MakeSession.java#L29-L48), [2025.1](https://github.com/JetBrains/MPS/blob/2025.1.5/core/make-runtime/solutions/jetbrains.mps.make.runtime/source_gen/jetbrains/mps/make/MakeSession.java#L29-L48), [2026.1](https://github.com/JetBrains/MPS/blob/2026.1.1/core/make-runtime/solutions/jetbrains.mps.make.runtime/source_gen/jetbrains/mps/make/MakeSession.java#L29-L48), [master](https://github.com/JetBrains/MPS/blob/ef4722267c8ea92f19c7de3c14728533b55bd3d4/core/make-runtime/solutions/jetbrains.mps.make.runtime/source_gen/jetbrains/mps/make/MakeSession.java#L29-L48)                                                                                                                                                                         |
| `Make.cleanup` runs cleanup activities (`Make_Facet.java`)                                      | L189-194: [2024.1](https://github.com/JetBrains/MPS/blob/2024.1.6/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/make/facets/Make_Facet.java#L189-L194), [2025.1](https://github.com/JetBrains/MPS/blob/2025.1.5/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/make/facets/Make_Facet.java#L189-L194), [2026.1](https://github.com/JetBrains/MPS/blob/2026.1.1/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/make/facets/Make_Facet.java#L189-L194), [master](https://github.com/JetBrains/MPS/blob/ef4722267c8ea92f19c7de3c14728533b55bd3d4/languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/make/facets/Make_Facet.java#L189-L194)                                                                                           |
| `canKeepOneMore` (`TransientModelsProvider.java`)                                               | L176-186: [2024.1](https://github.com/JetBrains/MPS/blob/2024.1.6/core/generator/source/jetbrains/mps/generator/TransientModelsProvider.java#L176-L186), [2025.1](https://github.com/JetBrains/MPS/blob/2025.1.5/core/generator/source/jetbrains/mps/generator/TransientModelsProvider.java#L176-L186), [2026.1](https://github.com/JetBrains/MPS/blob/2026.1.1/core/generator/source/jetbrains/mps/generator/TransientModelsProvider.java#L176-L186), [master](https://github.com/JetBrains/MPS/blob/ef4722267c8ea92f19c7de3c14728533b55bd3d4/core/generator/source/jetbrains/mps/generator/TransientModelsProvider.java#L176-L186)                                                                                                                                                                                                                                                                           |
| `GenerationSettingsProvider` default settings (`GenerationSettingsProvider.java`)               | L56-61: [2024.1](https://github.com/JetBrains/MPS/blob/2024.1.6/core/generator/source/jetbrains/mps/generator/GenerationSettingsProvider.java#L56-L61), [2025.1](https://github.com/JetBrains/MPS/blob/2025.1.5/core/generator/source/jetbrains/mps/generator/GenerationSettingsProvider.java#L56-L61), [2026.1](https://github.com/JetBrains/MPS/blob/2026.1.1/core/generator/source/jetbrains/mps/generator/GenerationSettingsProvider.java#L56-L61), [master](https://github.com/JetBrains/MPS/blob/ef4722267c8ea92f19c7de3c14728533b55bd3d4/core/generator/source/jetbrains/mps/generator/GenerationSettingsProvider.java#L56-L61)                                                                                                                                                                                                                                                                         |
| Persistent `GenerationSettings` (`generationSettings.xml`) (`GenerationSettings.java`)          | [2024.1 L34-39](https://github.com/JetBrains/MPS/blob/2024.1.6/workbench/mps-workbench/source/jetbrains/mps/ide/generator/GenerationSettings.java#L34-L39), [2025.1 L39-44](https://github.com/JetBrains/MPS/blob/2025.1.5/workbench/mps-workbench/source/jetbrains/mps/ide/generator/GenerationSettings.java#L39-L44), [2026.1 L39-44](https://github.com/JetBrains/MPS/blob/2026.1.1/workbench/mps-workbench/source/jetbrains/mps/ide/generator/GenerationSettings.java#L39-L44), [master L39-44](https://github.com/JetBrains/MPS/blob/ef4722267c8ea92f19c7de3c14728533b55bd3d4/workbench/mps-workbench/source/jetbrains/mps/ide/generator/GenerationSettings.java#L39-L44)                                                                                                                                                                                                                                 |

## Unresolved

- No runtime probe. Everything above comes from reading source.
- Whether the controller-subclass hook for save-transient works at run time.
- Whether transient models kept for messages survive a failed make in practice, and what happens to them in a make with
  several module clusters (each cluster runs its own script, so its own `configure` job).
- Whether the IDE-only settings listener (`TransientModelsNotification` in `plugins/mps-make`) does anything visible in
  a headless application when a caller changes the settings.
