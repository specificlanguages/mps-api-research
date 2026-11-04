# Parallel generation for one make

How to make MPS generate in parallel, with a chosen thread count, for one programmatic make only, without changing the
process-wide generation settings. Also: what MPS runs in parallel.

Verified against MPS 2024.1, 2025.1 and 2026.1 sources, and MPS `master` at build 263.6259 (for 263.x pre-releases).
Source paths are relative to the [JetBrains/MPS](https://github.com/JetBrains/MPS) repository root.

## Answer

Pass generation options to the make session through the script controller:

```kotlin
val settings = project.getComponent(GenerationSettingsProvider::class.java).generationSettings
val options = GenerationOptions.fromSettings(settings).generateInParallel(true, threads)
val controller = IScriptController.Stub2(session, GenerateFacetInitializer().setGenerationOptions(options))
```

The `Generate` facet uses these options for that make and reads the settings only when no options were set. The settings
object is not changed, so other makes in the same process keep their own behavior.

| Declaration                                                                                                 | Location                                                                                          |
| ----------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| `jetbrains.mps.generator.GenerationOptions.fromSettings(IGenerationSettings): OptionsBuilder`               | `core/generator/source/jetbrains/mps/generator/GenerationOptions.java`                            |
| `GenerationOptions.OptionsBuilder.generateInParallel(boolean generateInParallel, int numberOfThreads)`      | same file                                                                                         |
| `jetbrains.mps.internal.make.cfg.GenerateFacetInitializer.setGenerationOptions(OptionsBuilder)`             | `core/make-runtime/solutions/jetbrains.mps.make/source_gen/.../cfg/GenerateFacetInitializer.java` |
| `jetbrains.mps.make.script.IScriptController.Stub2(MakeSession, PropertyPoolInitializer...)`                | `core/make-runtime/solutions/jetbrains.mps.make.runtime/source_gen/.../IScriptController.java`    |
| `jetbrains.mps.generator.GenerationSettingsProvider.getGenerationSettings(): IModifiableGenerationSettings` | `core/generator/source/jetbrains/mps/generator/GenerationSettingsProvider.java`                   |

All of these are in `mps-core.jar` / `mps-generator.jar`; none needs `lib/mpsant/mps-tool.jar`. The declarations are the
same in all researched versions.

## How the Generate facet picks its options

**Verified.** `Generate_Facet.Target_configure` (in
`languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/Generate_Facet.java`)
does:

```java
if (vars(pa.global()).generationOptions() == null) {
  vars(pa.global()).generationOptions(GenerationOptions.fromSettings(settings));
}
```

then sets tracing, save-transient-models and the parameters provider on that builder, and later creates the
`GenerationFacade` from it. `GenerateFacetInitializer.populate` writes the builder into the
`jetbrains.mps.make.facets.Generate.configure` properties of the script's property pool, which `IScriptController.Stub2`
calls for each script it sets up. MPS's own `ReducedMakeFacetConfiguration`
(`core/tool/builder/source_gen/jetbrains/mps/tool/builder/make/ReducedMakeFacetConfiguration.java`) uses the same
pattern.

The facet mutates the builder it is given (tracing, transient models, parameters, custom plans), so use a fresh builder
for each make.

## Strict mode is required

**Verified.** `GenerationOptions.isGenerateInParallel()` returns `myGenerateInParallel && myStrictMode`. With strict
mode off, a parallel request silently generates sequentially. `fromSettings` copies strict mode from the settings;
`DefaultModifiableGenerationSettings` defaults it to `true` (parallel generation defaults to `false`, threads to `2`).

## What runs in parallel

**Verified.** Parallelism is inside the generation of one model, not across models:

- `GenerationController.generate` (`core/generator/source/jetbrains/mps/generator/impl/GenerationController.java`) loops
  over its generator tasks one model at a time.
- `GenerationSession.prepareToApplyRules` uses `ParallelTemplateGenerator` when `isGenerateInParallel()` is true, with
  the controller's `GenerationTaskPool(numberOfThreads, ...)`; otherwise the sequential `TemplateGenerator`.

So `generateInParallel(true, 1)` still takes the parallel code path with one worker thread. Pass `false` for a plain
sequential make.

## Changing the settings instead

`IModifiableGenerationSettings.setParallelGenerator` / `setNumberOfParallelThreads` change the settings object held by
the `GenerationSettingsProvider` core component, which every later make reads. In the IDE environment that object is
persisted by `jetbrains.mps.ide.generator.GenerationSettings` to `generationSettings.xml`
(`workbench/mps-workbench/source/jetbrains/mps/ide/generator/GenerationSettings.java`). Restoring them afterwards still
leaves a window in which another make in the same process sees the changed values.

## Unresolved

- No runtime probe measured the effect; the claims above come from source.
- The 263.x pre-release build used by a project may differ from the `master` build inspected here (inferred unchanged).
