# Java library classpaths and explicit reload in MPS 2025.3–2026.1

This note answers which classloading changes affect Java library declarations and explicit compiled-runtime replacement
when moving from MPS 2025.1 to MPS 2025.3 or 2026.1. Source comparisons use **2025.1.4**, **2025.3.0**, and
**2026.1.1**. Runtime observations use **2026.1.1 with JBR 25.0.3-b329.124**. APIs below are in `lib/mps-core.jar`.

## Java libraries must be declared in the Java facet

**Verified:** MPS 2025.1.4 reads solution-level `<stubModelEntries><stubModelEntry path="…"/></stubModelEntries>` into
`ModuleDescriptor.getJavaLibPersistedValues()`. `JavaModuleFacetImpl.load()` merges those values into its library
classpath. MPS 2025.3.0 and 2026.1.1 no longer perform that compatibility import.

The removal is commit
[`e5ba487447d9`](https://github.com/JetBrains/MPS/commit/e5ba487447d965d3f466265d652837618e0eca32), which is included in
2025.3.0. Thus this incompatibility is already present in 2025.3, not introduced in 2026.1. Use the Java facet's library
entries:

```xml
<facets>
  <facet type="java" compile="ext" classes="mps" ext="yes">
    <classes generated="true" path="${module}/classes_gen" />
    <library location="${module}/lib/example.jar" />
  </facet>
</facets>
```

The library location can name a JAR or a class directory. The significant difference is `library` / `location` inside
the facet, rather than `stubModelEntry` / `path` at solution level. MPS 2025.1.4 also supports the facet form.

Evidence:

- [2025.1.4 solution persistence](https://github.com/JetBrains/MPS/blob/2025.1.4/core/project/source_gen/jetbrains/mps/project/persistence/SolutionDescriptorPersistence.java#L77-L78)
  reads the legacy entries.
- [2025.1.4 Java facet](https://github.com/JetBrains/MPS/blob/2025.1.4/core/project/source/jetbrains/mps/project/facets/JavaModuleFacetImpl.java#L329-L340)
  reads facet entries and imports legacy values.
- [2026.1.1 Java facet](https://github.com/JetBrains/MPS/blob/2026.1.1/core/project/source/jetbrains/mps/project/facets/JavaModuleFacetImpl.java#L308-L323)
  reads the explicit facet entries without that import. This implementation is unchanged from tag 2025.3.0.

**Observed:** A 2026.1.1 source solution using only the legacy declaration had no library in
`JavaModuleFacet.getClassPath()`. An unchanged consumer's reflective method call failed with
`InvocationTargetException → NoClassDefFoundError → ClassNotFoundException` for the library class, before any file
replacement. Changing only the library declaration to the facet form made the class available.

## Explicit reload remains available

**Verified:** `jetbrains.mps.classloading.ClassLoaderManager` still exposes:

```java
public void reloadModules(Iterable<? extends SModule> modules);
public void reloadModules(Iterable<? extends SModule> modules, ProgressMonitor monitor);
```

The monitored overload calls `checkWriteAccess()`, flushes pending module events with `refresh()`, checks repository
membership, and reports `SModuleChangedEvent` for reloadable modules. Ordinary modules must remain registered;
unregistered modules other than `TempModule` cause `IllegalStateException`. The unmonitored overload supplies an
`EmptyProgressMonitor`.

```text
MPS write action
  ClassLoaderManager.reloadModules(changedModules)
    refresh pending repository events
    processModuleChanges(SModuleChangedEvent...)
      update dependency graph, including incoming dependents
      unload affected modules
      preload valid affected modules
```

See
[reloadModules](https://github.com/JetBrains/MPS/blob/2026.1.1/core/kernel/source/jetbrains/mps/classloading/ClassLoaderManager.java#L452-L474)
and
[ModuleUpdater](https://github.com/JetBrains/MPS/blob/2026.1.1/core/kernel/source/jetbrains/mps/classloading/ModuleUpdater.java#L231-L253).
These implementations are unchanged between tags 2025.3.0 and 2026.1.1. File-content change detection remains a caller
concern; see [external file and runtime refresh](external-file-runtime-refresh.md).

**Observed:** With modern facet declarations, external library JAR, directory, and directory-symlink replacements became
visible to an unchanged compiled consumer in the same daemon. The replaced artifact's size and modification time were
preserved. The host compared classpath contents and explicitly reloaded changed modules under a write action on the EDT.
This establishes that the tested reload path works; it does not establish that classloading generally requires the EDT.

A compiled-language replacement scenario also passed on 2026.1.1: an independent builder renamed a concept, then changed
its compiled alias; copying its language output into the running target made the new name resolve, the old name stop
resolving, and the alias change, without changing the target process ID. Alias replacement preserved compiled-file
paths, sizes, and modification times.

## Other version differences

**Verified:** The substantial classloader registry refactoring between 2025.1.4 and 2026.1.1 was already present in
2025.3.0. In particular:

- `getClassLoader(SModule)` no longer implicitly flushes pending module events merely because the caller holds write
  access. It checks graph validity and consults the registry. Do not rely on a classloader lookup as an event flush. See
  [commit 77d23d0a080d](https://github.com/JetBrains/MPS/commit/77d23d0a080d) and
  [2026.1.1 implementation](https://github.com/JetBrains/MPS/blob/2026.1.1/core/kernel/source/jetbrains/mps/classloading/ClassLoaderManager.java#L287-L305).
- Registry entries retain module references and classloader support objects; disposal uses `DisposeSession` and
  separates forgetting registry entries from disposing classloader objects. See
  [MPSClassLoadersRegistry](https://github.com/JetBrains/MPS/blob/2026.1.1/core/kernel/source/jetbrains/mps/classloading/MPSClassLoadersRegistry.java).

Between 2025.3.0 and 2026.1.1, `CLDependencies` removes the `mps.clm.deps.legacy` escape hatch: when a deployment
descriptor exists, dependency calculation uses it. This preserves the previous default but removes the legacy override.
See [commit cf8037e8e0af](https://github.com/JetBrains/MPS/commit/cf8037e8e0af). `ModuleClassLoader` also preserves the
disposed classloader exception as the cause of its class-not-found exception. Neither change explains a missing library
entry in a source solution's facet classpath.

## Distinguish search failures from runtime failures

An empty instance-search result alone does not establish a classloading failure. Check separately whether the concept
resolves, its compiled alias is current, and direct node traversal finds an equal concept. Instance search has
additional scope, indexing, and model-local lookup stages.

**Unresolved:** An initial empty instance-search result was not reproduced in an isolated execution of the complete
compiled-language replacement scenario. No specific indexing or timing cause was established. The passing observation
does not prove that the original failure cannot recur.
