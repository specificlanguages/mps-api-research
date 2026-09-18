# Repairing `-1` used-language versions in models

Verified against MPS 2026.1.1 sources. The relevant behavior is in `mps-kernel.jar`, with persistence in
`mps-model-persistence.jar`.

## Meaning and origin

A model stores an integer version for every explicitly used language. In XML persistence v9, the value is the `version`
attribute of a `<use>` element. A value of `-1` means that MPS could not deduce the language version; it does not denote
a real language version.

The ordinary `SModelInternal.addLanguage(SLanguage)` path asks `SLanguage.getLanguageVersion()` for the value to record.
For the standard `SLanguageAdapter`, that method looks up the language's active `LanguageRuntime` and returns
`LanguageRuntime.getVersion()`. It returns `-1` when the runtime is unavailable. Therefore a model can acquire `-1` when
code adds a used language before its runtime has been loaded, after it has been unloaded, or when the runtime cannot be
created. Code using the lower-level `jetbrains.mps.smodel.SModel.addLanguage(SLanguage)` also records `-1`
unconditionally in MPS 2026.1; that overload is deprecated for removal.

Persistence does not normalize the value. `ModelWriter9.saveUsedLanguages` writes the model's recorded integer as-is.

Calling `SModelInternal.addLanguage` again is not a repair operation. If the import already exists with `-1` and the
runtime now reports a non-negative version, the method leaves the existing value unchanged.

## Repair

Obtain the version from an authoritative loaded representation of the language, then replace the existing model import
version explicitly:

```java
LanguageRuntime runtime = languageRegistry.getLanguage(language);
if (runtime == null) {
  // The proper deployed-language version is not available in this environment.
  return;
}

repository.getModelAccess().executeCommand(() -> {
  SModelInternal modelInternal = (SModelInternal) model;
  if (modelInternal.importedLanguageIds().contains(language)
      && modelInternal.getLanguageImportVersion(language) == -1) {
    modelInternal.setLanguageImportVersion(language, runtime.getVersion());
  }
});
```

`LanguageRegistry.getLanguage(SLanguage)` returns `null` when no runtime for the language is active. For a deployed
language, `LanguageRuntime.getVersion()` is the authoritative current version. When working specifically with a source
language module instead, `jetbrains.mps.smodel.Language.getLanguageVersion()` reads its language descriptor version. Do
not guess the version or blindly substitute `0`.

`SModelInternal.setLanguageImportVersion(SLanguage, int)` requires an existing explicit language import. Its standard
implementation checks model change access, so perform the update under the repository's MPS write/command access. The
setter marks the model changed; save the owning model or project through the normal MPS persistence workflow.

Replacing `-1` with the currently installed version asserts that the model is compatible with that version. If the model
was created against an older language version, run the language's migrations instead of relabeling it. The migration
executor advances model import versions as migration scripts are applied.

`ModuleDependencyVersions.update(SModule)` is not a substitute for this repair. It updates the language-version map in
the module descriptor and only checks model versions for inconsistencies; it does not replace `-1` values stored in
models.

## Evidence

- `core/openapi/source/org/jetbrains/mps/openapi/language/SLanguage.java`: `getLanguageVersion()` documents `-1` as a
  version that could not be deduced.
- `core/kernel/source/jetbrains/mps/smodel/adapter/structure/language/SLanguageAdapter.java`: `getLanguageVersion()`
  returns `-1` when `getLanguageDescriptor()` returns `null`, otherwise the runtime version.
- `core/kernel/source/jetbrains/mps/extapi/model/SModelDescriptorStub.java`: `addLanguage(SLanguage)` obtains the
  version from `SLanguage` and does not replace an existing `-1` with a later non-negative value.
- `core/kernel/source/jetbrains/mps/smodel/SModel.java`: the deprecated one-argument `addLanguage` records `-1`;
  `setLanguageImportVersion` requires an existing import and records the supplied value.
- `core/persistence/source/jetbrains/mps/smodel/persistence/def/v9/ModelWriter9.java`: `saveUsedLanguages` serializes
  `getLanguageImportVersion` directly.
- `core/kernel/source/jetbrains/mps/smodel/language/LanguageRegistry.java`: `getLanguage(SLanguage)` returns the active
  runtime or `null`.
- `core/kernel/kernelSolution/source_gen/jetbrains/mps/smodel/ModuleDependencyVersions.java`: module dependency-version
  updates do not modify model import versions.
- `plugins/mps-migration/migration-platform/solutions/component/source_gen/jetbrains/mps/ide/migration/MigrationExecutorImpl.java`:
  successful migrations explicitly call `setLanguageImportVersion` for affected models.
