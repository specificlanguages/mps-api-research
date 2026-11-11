# Devkits and used languages of a new model (auto-imports)

**Question this answers:** when MPS creates a new model, how does it give the model its devkits and used languages, for
example `jetbrains.mps.devkit.aspect.structure` on a language's `structure` aspect model, and what must a client do to
get the same result when it creates a model programmatically?

Verified against MPS **2024.1** (source checkout at `6236c4073eac`, branch `2024.1`) and MPS **2026.1** (source checkout
at `50652adb7523`, branch `2026.1`). Observed in a headless MPS 2024.1.6 project. Source paths are relative to the
[JetBrains/MPS](https://github.com/JetBrains/MPS) repository root; line numbers are for 2024.1 / 2026.1.

## Answer

Creating a model through a `ModelRoot` (`ModelRoot.createModel(...)`, `DefaultModelRoot.createModel(...)`,
`LanguageAspect.createNew(Language)`) makes an empty model: no devkits, no used languages, no imports. The devkits and
languages come from a separate step, `ModelsAutoImportsManager.performImports(SModule, SModel)`, which the MPS UI calls
right after creating the model. A client that wants what the MPS UI gives must call it too:

```java
SModel model = root.createModel(name /* , ... */);
ModelsAutoImportsManager autoImports = platform.findComponent(ModelsAutoImportsManager.class);
autoImports.performImports(model.getModule(), model);
((EditableSModel) model).save();
```

Call it after the model is registered with its module: the language contributor finds the aspect by looking the model up
among the module's models.

## API

`jetbrains.mps.project.ModelsAutoImportsManager` (`mps-project.jar`; source
`core/project/source/jetbrains/mps/project/ModelsAutoImportsManager.java`) is a `CoreComponent`. `MPSCore` creates it
and returns it from `findComponent(ModelsAutoImportsManager.class)`, so it is available in headless MPS as well as in
the IDE.

```java
public void performImports(SModule module, SModel model)                          // :85 / :75
public Set<SModuleReference> getDevkitsToImport(SModule contextModule, SModel forModel) // :66 / :65
public Set<SLanguage> getLanguagesToImport(SModule contextModule, SModel model)
public Set<SModel> getModelsToImport(SModule contextModule, SModel model)
```

`performImports` asks every registered `AutoImportsContributor` whose `isApplicable(module)` is true, and adds the
results to the model through `ModelImports` (`addModelImport`, `addUsedLanguage`, `addUsedDevKit`). It does not save the
model and does not change module dependencies.

## Contributors registered by MPS core

`core/kernel/source/jetbrains/mps/core/platform/MPSCore.java` (:161 / :170) registers two contributors:

| Module kind | Contributor                                                           | Devkit                                                                                                                                         | Used languages                                  | Model imports                                                             |
| ----------- | --------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------- | ------------------------------------------------------------------------- |
| `Language`  | `Language.LanguageModelsAutoImports` (`Language.java` :485 / :484)    | the aspect's default devkit, or `jetbrains.mps.devkit.general-purpose` if the model is not an aspect model or its aspect has no default devkit | `LanguageAspectSupport.getMainLanguages(model)` | none                                                                      |
| `Generator` | `Generator.GeneratorModelsAutoImports` (`Generator.java` :282 / :284) | `jetbrains.mps.devkit.templates` (`BootstrapLanguages.getGeneratorTemplatesDevKit()`)                                                          | none                                            | the source language's `structure` model, for a generator (template) model |

No core contributor applies to a solution, so `performImports` does nothing for a solution model. The workbench adds
`TestsModelAutoImports` (`workbench/mps-workbench/source/jetbrains/mps/ide/project/listener/TestsModelAutoImports.java`)
for test models; it is not registered in headless MPS core.

### How a language model finds its aspect

`LanguageAspectSupport.getDefaultDevkit(SModel)` (:100 / :85) returns `getNewAspect(model).getDefaultDevkit()`.
`getNewAspect` walks the `LanguageAspectDescriptor`s of the `jetbrains.mps.lang.aspect.LanguageAspectsEP` extension
point and returns the one whose `getAspectModels(module)` contains the model. For `LanguageAspectDescriptorBase` this
means the model is in a `Language` and its full name is `<language name>.<aspect id>` with no stereotype.

Aspect descriptors come from the `plugin` aspect of language modules (generated `LanguageAspectsEP_extension` classes),
for example
`languages/languageDesign/structure/source_gen/jetbrains/mps/lang/structure/plugin/LanguageAspectsEP_extension.java`,
whose `getDefaultDevkit()` returns `78434eb8-b0e5-444b-850d-e7c4ad2da9ab(jetbrains.mps.devkit.aspect.structure)`.

Observed in headless MPS 2024.1.6: the default devkits of the bundled aspects are

| Aspect                                                                                                   | Default devkit                                                                                                                                                            |
| -------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `structure`                                                                                              | `jetbrains.mps.devkit.aspect.structure`                                                                                                                                   |
| `constraints`                                                                                            | `jetbrains.mps.devkit.aspect.constraints`                                                                                                                                 |
| `typesystem`                                                                                             | `jetbrains.mps.devkit.aspect.typesystem`                                                                                                                                  |
| `dataFlow`                                                                                               | `jetbrains.mps.devkit.aspect.dataflow`                                                                                                                                    |
| `textGen`                                                                                                | `jetbrains.mps.devkit.aspect.textgen`                                                                                                                                     |
| `migration`                                                                                              | `jetbrains.mps.devkit.aspect.migration`                                                                                                                                   |
| `feedback`                                                                                               | `jetbrains.mps.devkit.aspect.feedback`                                                                                                                                    |
| `vcs`                                                                                                    | `jetbrains.mps.devkit.aspect.vcs`                                                                                                                                         |
| `editor`, `actions`, `behavior`, `refactorings`, `scripts`, `intentions`, `findUsages`, `plugin`, `test` | none, so `jetbrains.mps.devkit.general-purpose`, plus the aspect's main languages (for example `jetbrains.mps.lang.editor` and `jetbrains.mps.lang.actions` for `editor`) |

The table is a runtime observation of one version; read `getDefaultDevkit()` of the descriptors at runtime rather than
hard-coding it.

## Who calls `performImports`

- The **New Model** dialog: `ModelCreateHelper.createModel()`
  (`workbench/mps-workbench/jetbrains.mps.ide/source_gen/jetbrains/mps/ide/dialogs/project/creation/ModelCreateHelper.java`
  :123 in both versions), for any module. It then calls `new MissingDependenciesFixer(result).fixModuleDependencies()`
  (:125), which adds a module dependency for each imported model that is not yet in the module's scope.
- **New aspect** actions: `LanguageAspectDescriptorBase.create(CreateAspectContext)`
  (`core/kernel/source/jetbrains/mps/smodel/language/LanguageAspectDescriptorBase.java` :97–101 / :93–97), then saves
  the model again.
- New language and generator wizards: `LanguageProducer`
  (`workbench/mps-platform/jetbrains.mps.ide.platform/source_gen/jetbrains/mps/project/modules/LanguageProducer.java`).

`LanguageAspect.createNew(Language)` (`core/kernel/source/jetbrains/mps/smodel/LanguageAspect.java`,
`createNew(Language, boolean)`) does **not** call it; its source says it is a deprecated way to create aspect models.
Neither does `LanguageAspectDescriptorBase.create(SModule)`, the deprecated overload without a context.

## Version differences

None found between 2024.1 and 2026.1 for `ModelsAutoImportsManager`, the two core contributors, their registration,
`LanguageAspectSupport.getDefaultDevkit`, or the callers above. In 2026.1 `MissingDependenciesFixer` is deprecated and
delegates to `ModelDependencyUpdate.updateModuleDependencies(SRepository)`.

## Evidence

- **Verified** (source, both versions): the API, the contributors and their results, registration in `MPSCore`, and the
  callers listed above.
- **Observed** (MPS 2024.1.6, headless): `findComponent(ModelsAutoImportsManager.class)` returns the component;
  `getDevkitsToImport` for an empty `<language>.structure` model created through `DefaultModelRoot.createModel` returns
  `jetbrains.mps.devkit.aspect.structure`; the per-aspect devkit table above.
- **Observed** (MPS 2024.1.6): a language whose `structure` model uses `jetbrains.mps.lang.structure` and has a concept,
  but has no `jetbrains.mps.devkit.aspect.structure` devkit, fails to make: the generated `Language.java` refers to
  aspect descriptor classes that were not generated. With the devkit added, the language makes.
