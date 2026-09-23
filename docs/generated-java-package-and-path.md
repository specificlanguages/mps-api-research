# What determines the package and file path of MPS-generated Java

**Question this answers:** a BaseLanguage `ClassConcept` (or `Interface`, `EnumClass`, `Annotation`) is generated to a
`.java` file. What decides the `package` statement inside that file, and what decides the directory it is written to?
Which property must a programmatic caller set to place a classifier in a chosen Java package — and is `virtualPackage`
that property?

**Answer:** a single string property, `Classifier.packageName`, drives both, through two code paths that never consult
each other. When it is empty — its default, and the state of every freshly created or freshly parsed classifier —
neither path errors and neither falls back to the Java default package: both independently fall back to the **long name
of the containing model** (its qualified name without the stereotype). The failure mode of forgetting the property is
therefore a silently _different_ package, not an absent one. `virtualPackage` is a different property with a different
job and no textgen code reads it.

Verified against three MPS development builds:

| Build         | Line   | `JetBrains/MPS` commit           |
| ------------- | ------ | -------------------------------- |
| **252.23892** | 2025.2 | `0f12866f47e7` (branch `2025.2`) |
| **261.25134** | 2026.1 | `499c4a0fe6bf` (branch `2026.1`) |
| **262.9437**  | 2026.2 | `060a56244647` (branch `master`) |

All statements below were read from source; the two decisive methods (`BaseLanguageTextGen.getPackageName` and
`TextGenAspectDescriptor.getPath_ClassConcept`) are **character-identical** across all three builds. Line numbers are
from 262.9437; other builds can be shifted by a few lines. Source paths are relative to the
[JetBrains/MPS](https://github.com/JetBrains/MPS) repository root. For MPS's own languages the generated Java under
`source_gen/` is cited, since that is the readable form of the aspect models. Code excerpts are **condensed**: generated
`SPropertyOperations`/`SNodeOperations` qualifiers and property constants are shortened, and elisions are marked `...`.

## Where the API lives

Artifact locations were checked against an MPS 2025.3 distribution.

| Element                                                                            | Distribution artifact                                   |
| ---------------------------------------------------------------------------------- | ------------------------------------------------------- |
| `Classifier.packageName` (structure, textgen and behavior descriptors)             | `languages/baseLanguage/jetbrains.mps.baseLanguage.jar` |
| `jetbrains.mps.text.*` (`TextUnit`, `impl.ModelOutline`, `rt.TextGenModelOutline`) | `lib/mps-textgen.jar`                                   |
| `jetbrains.mps.project.facets.*`, `jetbrains.mps.smodel.*`                         | `lib/mps-core.jar`                                      |
| `org.jetbrains.mps.openapi.*`                                                      | `lib/mps-openapi.jar`                                   |
| `jetbrains.mps.lang.core.plugin.TextGen_Facet` (the make-time writer)              | `languages/make/jetbrains.mps.make.facets.jar`          |
| `jetbrains.mps.java.core.newparser.JavaParser`, `JavaToMpsConverter`               | `plugins/mps-java/lib/java-core.jar`                    |
| `org.eclipse.jdt.internal.compiler.ast.ImportReference` (ECJ)                      | `lib/eclipse.jar`                                       |

## The property

`packageName` is declared on `Classifier`, so `ClassConcept`, `Interface`, `EnumClass` and `Annotation` all inherit it
(**Verified**):

```java
// languages/baseLanguage/baseLanguage/source_gen/jetbrains/mps/baseLanguage/structure/StructureAspectDescriptor.java
// inside createDescriptorForClassifier(), origin node 1107461130800
b.property("packageName", 0x26be0cf68be19d69L).type(PrimitiveTypeId.STRING).origin("2791683072064593257").done();
```

It is a plain string property with no property constraint — no validator, getter or setter override for it anywhere in
`jetbrains.mps.baseLanguage`'s constraints aspect (**Verified**, by absence: `Classifier_Constraints.java` exists but
does not mention it, and no other `*_Constraints.java` in that language references the property).

In the IDE it is edited from the **inspector**, not the main editor
(`.../source_gen/jetbrains/mps/baseLanguage/editor/ClassConcept_InspectorBuilder_a.java:98`, and the equivalent builders
for `Interface`, `EnumClass`, `Annotation`). A reader who has only ever used the main editor will not have seen it,
which is part of why the fallback below is usually encountered rather than chosen.

## Path A — the `package` statement and cross-file imports

`ClassConcept_TextGen.generateText` calls `BaseLanguageTextGen.fileHeader`, which emits the package declaration into the
`HEADER` text area, and only for a top-level classifier (**Verified**):

```java
// languages/baseLanguage/baseLanguage/source_gen/jetbrains/mps/baseLanguage/textGen/BaseLanguageTextGen.java:131-147
public static void fileHeader(SNode cls, final TextGenContext ctx) {
  boolean topClassifier = !((boolean) Classifier__BehaviorDescriptor.isInner_idsWroEc0xXl.invoke(cls));
  if (topClassifier) {
    tgs.pushTextArea("HEADER");
    tgs.append("package " + BaseLanguageTextGen.getPackageName(cls, ctx) + ";");
    ...
```

```java
// same file, :270-276
protected static String getPackageName(SNode cls, final TextGenContext ctx) {
  if (isNotEmptyString(getString(as(getContainingRoot(cls), Classifier), packageName))) {
    return getString(as(getContainingRoot(cls), Classifier), packageName);
  }
  return SModelOperations.getModelName(SNodeOperations.getModel(cls));
}
```

Two things follow, both **Verified** from this method:

- The fallback is `SModelOperations.getModelName(model)`, which is `model.getName().getLongName()` — the model's
  qualified name without its stereotype
  (`core/kernel/smodelRuntime/source_gen/jetbrains/mps/lang/smodel/generator/smodelAdapter/SModelOperations.java:146`) —
  not the empty string.
- It reads `getContainingRoot(cls)`, so a **nested** classifier never contributes its own `packageName`; the root
  classifier's value wins for the whole file.

The same method is also consulted when _other_ files reference this classifier: `getPackageAndShortName` (`:209`, calls
at `:247` and `:252`) and `internalClassifierName` (`:104`, call at `:111`) take the referenced classifier's package
from it. The result is passed through `appendClassName` (`:295`) to `getClassName` (`:277`), which asks
`ClassifierUnitContext.getClassifierRefText` and, through it, `ImportsContext` whether the reference needs an import or
can be written short. A classifier's `packageName` therefore affects the text of files that merely mention it, not only
its own.

`ImportsContext` computes the _current_ file's package by a third route — `JavaNameUtil.packageName(getFqName(root))`
(`.../textGen/ImportsContext.java:34`) — and `Classifier.getFqName()` has the same precedence, preferring `packageName`
and otherwise deferring to `INamedConcept.getFqName()` (**Verified**):

```java
// .../behavior/Classifier__BehaviorDescriptor.java:243-252
if (parentClassifier != null) { return getFqName(parentClassifier) + '.' + name; }
if (isNotEmptyString(getString(__thisNode__, packageName))) { return getString(__thisNode__, packageName) + '.' + name; }
return INamedConcept__BehaviorDescriptor.getFqName_idhEwIO9y.invokeSuper(__thisNode__, Classifier);
```

```java
// languages/core/core/source_gen/jetbrains/mps/lang/core/behavior/INamedConcept__BehaviorDescriptor.java:31-42
String longName = SModelOperations.getModelName(model);
if (longName == null || longName.equals("")) { return name; }
return longName + '.' + name;
```

## Path B — the output directory

The output path is decided separately, in the textgen aspect descriptor, and does **not** call `getPackageName`
(**Verified**):

```java
// .../textGen/TextGenAspectDescriptor.java:402-407 (and the identical _Interface / _Annotation / _EnumClass twins)
private static String getPath_ClassConcept(SNode node) {
  if (isNotEmptyString(SPropertyOperations.getString(node, PROPS.packageName$n3Xr))) {
    return SPropertyOperations.getString(node, PROPS.packageName$n3Xr).replace('.', '/');
  }
  return null;                       // <- not "", and not the model name
}
```

That value is handed to `UnitBuilder.path(...)` (same file, `:436`) and ends up as the unit's `getFilePath()`. The
contract for `null` is stated on the interface (**Verified**):

```java
// core/textgen/source/jetbrains/mps/text/TextUnit.java:48-58
/**
 * Tell desired location of the text outcome
 * @return {@code null} to use default value derived from qualified model name.
 */
@Nullable default String getFilePath() { return null; }
```

`ModelOutline` passes the value through unchanged in the ordinary case; its `myDefaultModelPath` is applied only on the
branch where the _unit name itself_ contains a separator
(`core/textgen/source/jetbrains/mps/text/impl/ModelOutline.java:115-142`). The real resolution of `null` happens at
write time, in the make facet (**Verified**):

```java
// languages/languageDesign/make/solutions/jetbrains.mps.make.facets/
//   source_gen/jetbrains/mps/lang/core/plugin/TextGen_Facet.java:334-343
if (tu.getFilePath() == null) {
  if (!(seenFileNames.add(tu.getFileName()))) { /* "Duplicate unit name ..." warning */ }
  javaSourcesLoc.saveStream(tu.getFileName(), tu.getBytes());
} else {
  FileDeltaCollector fdc = msfm.newPrimaryStreamHandler(tu.getFilePath());
  fdc.saveStream(tu.getFileName(), tu.getBytes());
  rdm.addDelta(fdc.getDelta());
}
```

The two destinations resolve differently:

```text
packageName empty  ->  javaSourcesLoc = msfm.getPrimaryStreamHandler()
                       = GenerationTargetFacet.getOutputLocation(model)
                       = overriddenOutputDir(model), else <outputRoot>/<model long name as path>/

packageName set    ->  msfm.newPrimaryStreamHandler(path)
                       = GenerationTargetFacet.getOutputRoot(model).getDescendant(<packageName as path>)
                       = ( overriddenOutputDir(model), else <outputRoot> ) + /<packageName as path>/
```

Supporting declarations (**Verified**): `ModuleStaleFileManager.ModelStaleFileManager.getPrimaryStreamHandler` /
`newPrimaryStreamHandler` in the same solution (`:144-178`); `JavaModuleFacet.getOutputLocation(SModel)` and
`getOutputRoot(SModel)` in `core/project/source/jetbrains/mps/project/facets/JavaModuleFacet.java:123-155`;
`FileGenerationUtil.getDefaultOutputDir(SModelReference, IFile)`, which takes `reference.getName().getLongName()`,
replaces `.` with the file-system separator and appends the result to the output root
(`core/kernel/source/jetbrains/mps/generator/fileGenerator/FileGenerationUtil.java:50-54`). Models of a test module
resolve the same way through `TestsFacet` (`core/project/source/jetbrains/mps/project/facets/TestsFacet.java:50-92`),
with the test output root in place of `<outputRoot>`. Both branches honour the per-model _generate into model folder_
override, which `JavaModuleOperations.getOverriddenOutputDir` derives from
`GeneratableSModel.isGenerateIntoModelFolder()` (`.../facets/JavaModuleOperations.java:71-81`); it returns the directory
that holds the model file, and only when the model's data source is a single file (`FileDataSource`). For file-per-root
persistence (`FilePerRootDataSource` extends `FolderDataSource`,
`core/kernel/source/jetbrains/mps/persistence/FilePerRootDataSource.java:31`) the option has no effect.

Consequences worth knowing:

- **Duplicate-name detection only guards the fallback branch** (**Verified** from the snippet above). `seenFileNames` is
  consulted only when `getFilePath() == null`. Two root classifiers in one model that share a name _and_ a non-empty
  `packageName` are both saved to the same file with no duplicate warning; that the second write silently replaces the
  first is **Inferred** — the stream handler's overwrite behavior was not traced.
- **Auxiliary artifacts do not follow the class** (**Verified**). Trace/debug info and the cross-model cache are
  registered against `javaSourcesLoc` — the model's standard location — regardless of any unit's own path
  (`TextGen_Facet.java:351-353`).
- **Directory and package diverge under _generate into model folder_** (**Verified** from the resolution above). This
  applies only to single-file models; see above. Without the override, both paths yield the same dotted name —
  `packageName` if set, else the model long name — so the file lands in the directory that matches its `package`
  statement. With the override, an empty `packageName` writes directly into the model file's directory, and a set
  `packageName` writes into `<model file directory>/<packageName as path>/`. The directory is then derived from where
  the model file is stored, not from the package, and matches the package only if the model file happens to sit in a
  matching directory.

## `virtualPackage` is a different property with a different job

`virtualPackage` is declared on `BaseConcept`, not on `Classifier`
(`core/kernel/kernelSolution/source_gen/jetbrains/mps/smodel/SNodeUtil.java:69`, property id `0x115eca8579f` in
`jetbrains.mps.lang.core`). It is a **logical-view organisation** property: it groups roots into folders in the project
pane and in editor tabs, and drag-and-drop in that pane is implemented by rewriting it (**Verified**):

- `workbench/mps-ui/source/jetbrains/mps/ide/ui/tree/smodel/SModelTreeNode.java:117` and
  `.../tree/smodel/PackageNode.java:59` — tree grouping.
- `workbench/mps-workbench/source/jetbrains/mps/ide/projectPane/ProjectPaneDnDListener.java:116-137` — drag-and-drop
  assigns it.
- `workbench/mps-workbench/source/jetbrains/mps/ide/editorTabs/tabfactory/tabs/CreateGroupsBuilder.java:178-180` —
  propagated to aspect roots so concept tabs stay grouped together.

The only generation-side use is `TemplateGenerator` copying it from an input root to the corresponding output root, with
a comment confirming the root-only scope
(`core/generator/source/jetbrains/mps/generator/impl/TemplateGenerator.java:447-452`). **No textgen code reads it**
(**Verified**, by absence: no occurrence in `core/textgen/`, in the generated textgen code of
`jetbrains.mps.baseLanguage`, or in the `TextGen_Facet` write path — the only hit under those trees is the property
being used to organise the aspect model's own roots, which is the logical-view role described above). Setting it has no
effect on generated Java.

## Recovering the package from parsed Java

`JavaParser` recovers the package declaration of parsed source but does not write it into any node; neither `JavaParser`
nor the `newparser` converters set `Classifier.packageName` (**Verified**, by absence: nothing under `.../newparser/`
references the property; the string `packageName` occurs there only as local variable names):

```java
// plugins/mps-java/core/modules/jetbrains.mps.java.core/source_gen/jetbrains/mps/java/core/newparser/JavaParser.java
// :104-110, inside case CLASS / CLASS_STUB
if (compRes.currentPackage != null) {
  StringBuffer sb = new StringBuffer();
  compRes.currentPackage.print(0, sb, false);
  resultPackageName = sb.toString();
}
// exposed as JavaParseResult.getPackage() (:565-567)
```

The string is the **bare dotted name** — `com.acme`, with no `package` keyword and no semicolon — which is exactly the
form `packageName` expects. **Verified** by disassembling ECJ's
`org.eclipse.jdt.internal.compiler.ast.ImportReference.print(int, StringBuffer, boolean)` as bundled in
`lib/eclipse.jar` of the MPS 2025.3 distribution (not in the three source builds above): it joins `tokens` with `'.'`
and appends `".*"` only when the third argument is `true` _and_ the on-demand bit is set; MPS passes `false`.

Limits, all **Verified** from the same switch:

- `resultPackageName` is populated only for `FeatureKind.CLASS` and `CLASS_STUB`. The `CLASS_CONTENT` and `STATEMENTS`
  branches never assign it, so `getPackage()` is `null` there.
- It is also `null` when the source declares no package, since `compRes.currentPackage` is then `null`.

MPS's own importer, `JavaToMpsConverter` (same directory), carries the package across by **model name** instead of by
property, and only in one of its two modes (**Verified**). In both modes `convertToMps` first runs `parseFile` on every
file (`:165`): it reads `getPackage()` (`:359`), rejects files in the default package (`:364`) or whose package does not
match their directory (`:372`), and groups roots by package. It then branches on the target (`:188`):

- **Module target** (constructors taking an `SModule`, used for example by `MigrateSourcesToMPS_Action`,
  `NewModelFromSource_Action` and the IntelliJ plugin's `ConvertPackageToModel`). `getModel` (`:950-958`) places each
  group in the module's model whose full name equals the package, creating it if needed. The comparison uses
  `SModel.getModelName()`, which includes any stereotype, so an existing `com.acme@tests` model is not reused for
  package `com.acme`. The empty `packageName` falls back to that model's name, so the generated package matches the
  source.
- **Model target** (constructor taking an `SModel`, `:137`, used by `GetModelContentsFromSource_Action`). Every root
  goes into the given model regardless of its declared package (`:203-212`).

`JavaPaster` (paste Java as MPS nodes) uses neither mode: it calls `JavaParser.parse` directly
(`plugins/mps-java/platform/modules/jetbrains.mps.java.platform/source_gen/jetbrains/mps/java/platform/util/JavaPaster.java:128`)
and builds a model-target converter only to resolve references (`:237-238`).

So any flow that inserts parsed classifiers into a model whose long name is not their declared package — the converter's
model-target mode, `JavaPaster`, or any other direct caller of `JavaParser` — produces classes whose generated package
is the model's long name, silently disagreeing with the source they were parsed from, unless the caller copies
`getPackage()` into `Classifier.packageName`. Putting the classifier in a model named after the package, or setting
`packageName` on it, are the two ways to keep them in agreement.

## Writing the property

`packageName` is an ordinary string property, so the usual model-write rules apply: the write needs MPS write access (a
command or write action), and the choice of setter follows [`property-value-setters.md`](property-value-setters.md) —
for a string property both `setProperty` and `setPropertyValue` happen to round-trip, but
`setProperty(node, property, String)` is the one that mirrors the read path for every datatype.

Set it on the **root** classifier: a value on a nested classifier is read by neither path. Path A takes
`getContainingRoot` (**Verified**); Path B runs only for text units, and `TextGenAspectDescriptor.breakdownToUnits`
(`:340-352`) creates those only for the model's root nodes whose concept is exactly `ClassConcept`, `Interface`,
`EnumClass` or `Annotation` (**Verified**). Because the match uses `equals`, a root of a _subconcept_ gets no unit from
this descriptor; it depends on its own language's textgen (see Unresolved).

Clearing it (setting `null` or `""`) returns both paths to the model-name fallback — `isNotEmptyString` guards both
(**Verified**).

## Version differences

None found across the three builds checked. `BaseLanguageTextGen.getPackageName` is at `:270-276` in all three, and
`TextGenAspectDescriptor.getPath_ClassConcept` at `:398-403` (252) / `:402-407` (261, 262), with identical bodies; the
line shift is unrelated surrounding generated code. The `packageName` property id `0x26be0cf68be19d69` is stable.

Not checked: releases before the 2025.2 line. `FileGenerationUtil` is `@Deprecated(since = "3.4", forRemoval = true)`
and `TextUnit.getFilePath` is marked `PROVISIONAL API`, so the _path_ half of this note is the half more likely to move
in a future release; the `packageName` property itself is load-bearing for the emitted `package` statement and unlikely
to change shape.

## Unresolved

- Only `jetbrains.mps.baseLanguage` was checked, where the four `getPath_*` methods are identical. Other languages that
  generate `.java` were not surveyed. That includes languages whose root concepts extend `ClassConcept` and so get no
  text unit from BaseLanguage's `breakdownToUnits`. Whether their own textgen derives the path and package with the same
  precedence is unknown.
- The behavior described is read from source only. No runtime probe was run to observe a generated file landing in a
  `packageName`-derived directory, nor to confirm the silent-overwrite case for two same-named roots sharing a
  `packageName`. Both are straightforward to demonstrate with a two-root model and one make.
