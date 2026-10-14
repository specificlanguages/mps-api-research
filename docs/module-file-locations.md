# Locating a loaded module's descriptor, deployed jars, class output, and model sources

Given a loaded `SModule`, which MPS APIs give its descriptor file, the jar(s) it was deployed in, its compiled-class
locations, and where the files of its models live? Does MPS record the `-src.jar` relationship of a deployed module, or
must a consumer guess a sibling file?

Verified against the MPS source at tags `2024.1.6`, `2025.1.4`, and `2026.1.1`, and against the deployment descriptors
shipped in the 2024.1.6 and 2026.1.1 distribution archives. Every class named below is in `lib/mps-core.jar` of the
distribution. Source paths are relative to an MPS source checkout; `:N` is a line number at the named tag, given in the
order 2024.1.6 / 2025.1.4 / 2026.1.1 when the three differ.

## Answer

| Fact                     | Source module (descriptor on disk)                               | Deployed module (jar)                                                                        |
| ------------------------ | ---------------------------------------------------------------- | -------------------------------------------------------------------------------------------- |
| Descriptor file          | `AbstractModule.getDescriptorFile()` → `dir/name.msd`            | `AbstractModule.getDescriptorFile()` → `path/name.jar!/META-INF/module.xml`                  |
| Deployment info          | `ModuleDescriptor.getDeploymentDescriptor()` → `null`            | non-null `DeploymentDescriptor` (`getClasspath()`, `getLibraries()`, `getSourcesJar()`)      |
| Jar / module home        | n/a (`getModuleSourceDir()` is the descriptor's directory)       | the `getPath()` prefix before `!/`, or `IFile.getBundleHome()` / `stepUpToArchive()`         |
| Design-time class output | `JavaModuleFacet.getClassesGen()` → `…/classes_gen` (or null)    | `getClassesGen()` → `null`                                                                   |
| Runtime classpath        | `JavaModuleFacet.getClassPath()` → libraries + `classes_gen`     | `getClassPath()` → libraries + deployment-descriptor entries, resolved to absolute paths     |
| Source descriptor        | same as descriptor file                                          | `ModulesMiner.getSourceDescriptorFile(descriptorFile, dd)` → `name-src.jar!/module/name.msd` |
| Model files              | `FileBasedModelRoot.getContentDirectory()`, `SModel.getSource()` | same APIs; paths point into `name-src.jar!/module/…`                                         |

MPS **does** record the `-src.jar` relationship: the deployment descriptor `META-INF/module.xml` carries
`<sources jar="name-src.jar" descriptor="name.msd"/>`, read into `DeploymentDescriptor.getSourcesJar()` and
`getDescriptorFile()`, and `ModulesMiner.getSourceDescriptorFile` resolves it relative to the directory containing the
module jar. No sibling-file guess is needed. Moreover, in all three versions a deployed module whose source descriptor
cannot be resolved (no `<sources>` element, or the `-src.jar` absent on disk) is **not loaded at all**:
`ModulesMiner.loadDeploymentDescriptor` returns `null` and no module handle is created (see
[Deployed jars and classpath](#deployed-jars-and-classpath)). Every jar-backed module present in a repository therefore
has a resolvable `-src.jar`.

## Deployed layout and how the pieces are tied together

Observed in the 2024.1.6 and 2026.1.1 distribution archives (identical layout):

```text
languages/languageDesign/
├── jetbrains.mps.lang.core.jar            # deployed module; getDescriptorFile() points inside it
│   ├── META-INF/module.xml                # DeploymentDescriptor: <classpath><entry path="."/></classpath>
│   │                                      #   <sources descriptor="jetbrains.mps.lang.core.mpl" jar="jetbrains.mps.lang.core-src.jar"/>
│   ├── icons/
│   └── jetbrains/mps/lang/core/**/*.class # classpath entry "." resolves to the jar itself
├── jetbrains.mps.lang.core-src.jar        # getSourceDescriptorFile() → …-src.jar!/module/jetbrains.mps.lang.core.mpl
│   ├── module/
│   │   ├── jetbrains.mps.lang.core.mpl    # source ModuleDescriptor; ${module} expands to …-src.jar!/module
│   │   ├── languageModels/*.mpb           # FileBasedModelRoot content; SModel.getSource() → FileDataSource of these
│   │   └── generator/template/*.mps
│   └── jetbrains/mps/lang/core/**/*.java  # generated Java sources, beside module/
└── jetbrains.mps.lang.core-generator.jar  # generator module; its module.xml <sources> names the SAME -src.jar and .mpl
```

Plugin-bundled solutions follow the same pattern: `plugins/mps-devkit/languages/*.pluginSolution.jar` each carry
`<classpath><entry path="."/></classpath>` and `<sources descriptor="….msd" jar="…-src.jar"/>`.

```text
ModulesMiner.readModuleDescriptorsFromRegularFile(jar!/META-INF/module.xml)
  moduleHome = file.getBundleHome()            # 2026.1.1: stepUpToArchive(); directory layout: getParent().getParent()
  tryModuleFromDeploymentDescriptor(moduleHome, module.xml)
    loadDeploymentDescriptor(moduleHome, module.xml)
      DeploymentDescriptorPersistence.load(module.xml)        # <library>, <classpath>/<entry>, <sources>
      getSourceDescriptorFile(module.xml, dd)                 # -src.jar!/module/<descriptor>, or null
      loadSourceModuleDescriptor(sourceDescriptorFile)        # the real ModuleDescriptor (.msd/.mpl); skipped when null
      rewrite dd.getClasspath() entries to absolute paths     # "." → the jar path
      result.setDeploymentDescriptor(dd); fill dd.getLibrariesResolved()
      return result                                           # null when no source descriptor → module not registered
    fillOutcome(new ModuleHandle(module.xml, descriptor))     # the handle's file is module.xml
SLibrary.instantiateAndActivateModules
  ModuleRepositoryFacade.instantiate(descriptor, handle.getFile())   # → AbstractModule.myDescriptorFile = module.xml
```

## Descriptor file

`jetbrains.mps.project.AbstractModule` (`core/kernel/source/jetbrains/mps/project/AbstractModule.java`):

- `@Nullable public final IFile getDescriptorFile()` (:629 / :619 / :588). Returns the `IFile` the module was
  instantiated with (`AbstractModule(@Nullable IFile descriptorFile)`, :146 / :147 / :137); `null` for modules that do
  not come from a file. The Javadoc warns that the deployed descriptor name differs from the source one
  (`META-INF/module.xml` vs `mylang/my.lang.mpl`).
- `public IFile getModuleSourceDir()` (:613 / :603 / :572) is `getDescriptorFile().getParent()`. For a deployed module
  that is `name.jar!/META-INF`, despite the Javadoc's claim of `{module-name}-src.jar/module/`; do not use it to find
  sources of a deployed module.
- `public boolean isPackaged()` (:601 / :591 / :560) is
  `getModuleSourceDir() == null || getModuleSourceDir().isInZipArchive()`; `isReadOnly()` delegates to it.
- `public ModuleDescriptor getModuleDescriptor()` (:260 / :264 / :255); `ModuleDescriptor.getDeploymentDescriptor()`
  (`core/kernel/source/jetbrains/mps/project/structure/modules/ModuleDescriptor.java` :379 / :372 / :273) is non-null
  exactly for modules read through a deployment descriptor.

Why a deployed module's descriptor file is `module.xml`: `ModulesMiner`
(`core/project/source/jetbrains/mps/library/ModulesMiner.java`) creates `new ModuleHandle(moduleXml, moduleDescriptor)`
(:286 / :279 / :289), and `SLibrary.instantiateAndActivateModules` passes `moduleHandle.getFile()` to
`ModuleRepositoryFacade.instantiate(descriptor, file)` (`core/project/source/jetbrains/mps/library/SLibrary.java` :208 /
:210 / :210), which `GeneralModuleFactory.instantiate` forwards to the module constructor.

### Path format of a jar entry

`jetbrains.mps.vfs.iofs.jar.JarEntryFile.getPath()` (`core/vfs/source/jetbrains/mps/vfs/iofs/jar/JarEntryFile.java`
:148-149 / :153-154 / :159-160) returns `PathUtil.toSystemIndependent(jarFile.getAbsolutePath()) + "!/" + entryPath`,
e.g. `/opt/mps/languages/languageDesign/jetbrains.mps.lang.core.jar!/META-INF/module.xml`. The jar itself is therefore
the substring before the first `!/`. `IFile` (`core/vfs/source/jetbrains/mps/vfs/IFile.java`) also offers:

- `boolean isInZipArchive()` (:124 in all three tags): true for jar entries.
- `IFile getBundleHome()` (:191 / :199 / :216): for a `JarEntryFile`, the jar file as a local `IFile` (2024.1.6 and
  2025.1.4 resolve it through the `java.io.File` file system, `JarEntryFile.java` :247 / :252); for a `LocalFile`,
  merely `getParent()`. Marked `@Deprecated(since = "3.4", forRemoval = true)` in all three versions but present.
- `IFile stepUpToArchive()` (2026.1.1 :149, `@since 2025.2`): the closest containing archive, or `this` when the file is
  not an entry; the Javadoc says to guard the call with `isInZipArchive()`. **Absent in 2024.1.6 and 2025.1.4.**
  `IFile stepIntoArchive()` (`@since 2025.1`, :132 / :133) is present in 2025.1.4 and 2026.1.1 only.

In 2026.1.1 `ModulesMiner` (:398) and `SLibrary` (:214) switched from `getBundleHome()` to `stepUpToArchive()`, and
`JarEntryFile.getBundleHome()` delegates to `stepUpToArchive()` (:258-259).

## Deployed jars and classpath

`jetbrains.mps.project.facets.JavaModuleFacet`
(`core/project/source/jetbrains/mps/project/facets/JavaModuleFacet.java`), obtained with
`module.getFacet(JavaModuleFacet.class)`; `null` when the module has no Java facet:

- `@Nullable IFile getClassesGen()` (:71 / :77 / :71): the design-time `classes_gen` root; the Javadoc states it may be
  `null` for deployed modules.
- `Set<String> getClassPath()` (:178 / :184 / :178) and `Set<String> getLibraryClassPath()` (:175 / :181 / :175): path
  strings, not `IFile`s.

`JavaModuleFacetImpl` (`core/project/source/jetbrains/mps/project/facets/JavaModuleFacetImpl.java`):

- `load(...)` sets the generated-classes location to `null` when `isAtDeployedModule()` (:318 / :295 / :284), i.e. when
  `module.isPackaged()` or the module descriptor has a deployment descriptor (:510 / :463 / :399). So `getClassesGen()`
  is `null` for every deployed module, and non-null for a source module only if its facet memento has a
  `<classes generated="true" path="…"/>` entry (there is no `classes_gen` default on load).
- `getClassPath()` = library classpath + (`getClassesGen()` if non-null) + for a packaged module with a deployment
  descriptor, `dd.getClasspath()` (:166 / :165 / :158). The library classpath comes from the facet's library bundle,
  which for deployed modules `load()` fills from `dd.getLibrariesResolved()` (:322 / :299 / :288).

`DeploymentDescriptor` (`core/kernel/source/jetbrains/mps/project/structure/modules/DeploymentDescriptor.java`,
identical in all three tags):

- `List<String> getClasspath()` (:189): the `<classpath><entry path="…"/></classpath>` values, **rewritten in place by
  `ModulesMiner.loadDeploymentDescriptor`** (loop at :456 / :449 / :457): `"."` becomes the module home's path (the jar
  file itself, e.g. `/opt/mps/languages/languageDesign/jetbrains.mps.lang.core.jar`), any other entry becomes
  `jar!/entry` for a jar home or `home/entry` for a directory home. After loading, the entries are absolute paths usable
  as the "deployed jar(s)" of the module.
- `List<String> getLibraries()` (:163): `<library jar="…"/>` values as written; `List<IFile> getLibrariesResolved()`
  (:173, documented as a provisional hack): the same resolved by `ModulesMiner` relative to the jar's parent directory
  (or to the MPS home for entries starting with `/`).
- `String getSourcesJar()` (:128), `String getDescriptorFile()` (:136): from `<sources jar="…" descriptor="…"/>`.

`DeploymentDescriptorPersistence`
(`core/project/source_gen/jetbrains/mps/project/persistence/DeploymentDescriptorPersistence.java` :74-89, identical
across tags) reads the `library`, `classpath/entry`, and `sources` elements.

`ModulesMiner.loadDeploymentDescriptor` (:428 / :421 / :429) only builds a `ModuleDescriptor` from the source descriptor
(`loadSourceModuleDescriptor(sourceDescriptorFile)`); when `getSourceDescriptorFile` returns `null` the result stays
`null` (the code carries a `TODO create module without sources`, :479 / :472 / :480) and
`tryModuleFromDeploymentDescriptor` (:272 / :265 / :275) records no module handle.

## Source location of models

### Source descriptor and `${module}` of a deployed module

```java
public static @Nullable IFile ModulesMiner.getSourceDescriptorFile(
    @NotNull IFile deploymentFile, @NotNull DeploymentDescriptor dd)
```

Declared at :710 / :711 / :702 with an identical body:

- returns `null` if `dd.getSourcesJar()` or `dd.getDescriptorFile()` is `null`;
- if the sources jar is `""` or `"."` (single-jar packaging), returns
  `deploymentFile.getParent().getParent()/module/<descriptor>`, i.e. `name.jar!/module/name.msd`;
- otherwise strips `!/META-INF/module.xml` from `deploymentFile.getPath()`, resolves `getSourcesJar()` against the
  module jar's parent directory, and returns `sourcesJar.getPath() + "!/module/" + descriptor` **only if the sources jar
  exists**; otherwise `null`.

Pass `((AbstractModule) module).getDescriptorFile()` and `module.getModuleDescriptor().getDeploymentDescriptor()`; this
is exactly what `MacrosFactory.forModule(SModule)` (`core/kernel/source/jetbrains/mps/util/MacrosFactory.java` :82-88,
all tags) does to anchor `${module}` for a deployed module, and what
`workbench/mps-workbench/source/jetbrains/mps/ide/projectPane/logicalview/ProjectTreeFindHelper.java` (2026.1.1 :107)
does to match a module to a file. `${module}` expands to the source descriptor's parent, `name-src.jar!/module`.

### Model roots and model files

- `SModule.getModelRoots(): Iterable<ModelRoot>` (openapi). File-based roots are
  `jetbrains.mps.extapi.persistence.FileBasedModelRoot`
  (`core/kernel/source/jetbrains/mps/extapi/persistence/FileBasedModelRoot.java`):
  `@Nullable IFile getContentDirectory()` (:113 / :110 / :108) and `List<SourceRoot> getSourceRoots(SourceRootKind)`
  (:155 / :140 / :138) with `SourceRoot.getAbsolutePath(): IFile`. Content paths are macro-expanded with
  `MacrosFactory.forModule(module)` (2024.1.6 and 2025.1.4 in `setModule`, :309 / :316; 2026.1.1 via
  `PersistenceContextImpl.macroHelper`, `core/kernel/source/jetbrains/mps/module/PersistenceContextImpl.java` :34-43),
  so a deployed module's `${module}/models` resolves into the `-src.jar`.
- `SModel.getSource(): DataSource` (openapi, :112); for file-persisted models it is a
  `jetbrains.mps.extapi.persistence.FileDataSource` with `@NotNull IFile getFile()`
  (`core/kernel/source/jetbrains/mps/extapi/persistence/FileDataSource.java` :65, all tags) and `getLocation()` (:99) =
  `file.getPath()`. For a deployed module the path looks like
  `…/jetbrains.mps.lang.core-src.jar!/module/languageModels/behavior.mpb` (observed layout of the `-src.jar` in both
  archives: `module/jetbrains.mps.lang.core.mpl`, `module/languageModels/*.mpb`, `module/generator/template/*`).

A source module's models are in `getModuleSourceDir()` (the descriptor's directory) under its model roots; a deployed
module's are in the `-src.jar` named by `getSourceDescriptorFile`.

## Recipe

```text
locations(module: AbstractModule)
  descriptor = module.getDescriptorFile()                     # may be null
  dd = module.getModuleDescriptor()?.getDeploymentDescriptor()
  if dd == null                                               # source module
    sourceDir = descriptor.getParent()
    classesGen = module.getFacet(JavaModuleFacet)?.getClassesGen()
  else                                                        # deployed module
    jar = descriptor.getPath() before first "!/"              # or stepUpToArchive() on 2025.2+
    deployedJars = dd.getClasspath()                          # already absolute; "." → jar
    libraries = dd.getLibrariesResolved()
    sourceDescriptor = ModulesMiner.getSourceDescriptorFile(descriptor, dd)   # non-null for a loaded module
    sourcesJar = sourceDescriptor.getPath() before first "!/"
  classPath = module.getFacet(JavaModuleFacet)?.getClassPath()
  modelRoots = module.getModelRoots() as FileBasedModelRoot → getContentDirectory()
```

## Version differences (2024.1.6 → 2026.1.1)

- No change in `AbstractModule.getDescriptorFile/getModuleSourceDir/isPackaged`,
  `JavaModuleFacet.getClassesGen/getClassPath`, `DeploymentDescriptor`, `DeploymentDescriptorPersistence`,
  `ModulesMiner.getSourceDescriptorFile`, `MacrosFactory.forModule`, `FileBasedModelRoot.getContentDirectory`,
  `FileDataSource.getFile`, or the `!/` path format.
- `IFile.stepIntoArchive()` added in 2025.1; `IFile.stepUpToArchive()` added in 2025.2 (present in 2026.1.1, absent in
  2025.1.4). `getBundleHome()` is deprecated for removal in all three; splitting `getPath()` at `!/` is the only
  approach that is neither deprecated nor version-gated.
- `JavaModuleFacetImpl.load/save` and `FileBasedModelRoot.load/save` gained a `ModulePersistenceContext` parameter in
  2025.1 (`JavaModuleFacetImpl.load`) and 2026.1 (`FileBasedModelRoot`); the read accessors are unaffected.
- `ModulesMiner.ModuleHandle` gained a `userData` field in 2026.1.1; `saveHandle/loadHandle` (2024.1.6 :740 / :760) were
  removed.

## Evidence strength

- **Verified** (source at the tags): all signatures and line anchors above, the `module.xml` handle, the `null`
  `getClassesGen()` for deployed modules, the classpath rewriting, the `getSourceDescriptorFile` algorithm, the `null`
  result and skipped registration without a source descriptor, the `!/` path format, the deprecations.
- **Observed** (reading the 2024.1.6 and 2026.1.1 distribution archives): the shipped `META-INF/module.xml` files of
  `jetbrains.mps.lang.core.jar`, `jetbrains.mps.lang.core-generator.jar`, and the
  `plugins/mps-devkit/languages/*.pluginSolution.jar` solutions contain `<classpath><entry path="."/></classpath>` and
  `<sources jar="…-src.jar" descriptor="…"/>`; the `-src.jar` keeps the descriptor and models under `module/` and
  generated Java sources beside it.
- **Inferred**: none of the accessors above asserts model read access (`AbstractModule` has the `assertCanRead()` calls
  commented out at `getDescriptorFile`, `isPackaged`, `getModuleDescriptor`), but `getModels()`/`getModelRoots()`
  traversal is normally done inside a read action.
- **Unresolved**: not exercised in a running MPS; the exact `getPath()` strings were derived from source, not captured
  at runtime. A deployed module in a directory layout (`home/META-INF/module.xml`, not a jar) has a plain-file
  descriptor whose `getBundleHome()` is `META-INF`, not the module home; `ModulesMiner` (:395-400 / :388-393 / :398-401)
  special-cases this with `getParent().getParent()`, and consumers must do the same.
