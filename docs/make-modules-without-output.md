# Make of modules with no output: no generation target facet, or no used language

What the MPS make does with two kinds of input that produce no files: models whose module has no
`GenerationTargetFacet`, and modules whose models use no language. Covers how the make reports each case through
`CoreMakeTask` and its `IMessageHandler`.

Verified against the MPS sources of the `2024.1`, `2025.1` and `2026.1` branches
([JetBrains/MPS](https://github.com/JetBrains/MPS)). Observed with a headless make on MPS 2024.1.6. Source paths are
relative to the MPS repository root. See [make-generation.md](make-generation.md) for the make flow itself.

## Answer

| Input                                                                 | MPS 2024.1, 2025.1                                                                      | MPS 2025.3, 2026.1                                                  |
| --------------------------------------------------------------------- | --------------------------------------------------------------------------------------- | ------------------------------------------------------------------- |
| Generatable model with content, module has no `GenerationTargetFacet` | Generated, then TextGen reports errors and the make fails                               | Generated, TextGen skips the model with warnings, the make succeeds |
| Module whose models use no language and import no language or devkit  | Left out of the make sequence; with no other module, the make fails and reports nothing | Same                                                                |

## No generation target facet

`ModelsToResources` keeps every generatable model (`GenerationFacade.canGenerate`), whatever facets its module has. The
Generate facet transforms the model; the TextGen facet then looks for an output location.

- **2024.1 and 2025.1 (Verified, Observed on 2024.1.6):** in
  `languages/languageDesign/make/solutions/jetbrains.mps.make.facets/source_gen/jetbrains/mps/lang/core/plugin/TextGen_Facet.java`,
  a model without an output location reports `IFeedback.ERROR("no output location for <model>")`. The text generation
  for that model then fails in `ModuleStaleFileManager.getPrimaryStreamHandler()` with
  `IllegalStateException("No module facet to specify textgen destination for <model>")`. The make reports three errors:
  the two above and `Error executing target jetbrains.mps.make.facets.TextGen.textGen`.
- **2025.3 and later (Verified):** since `MPS-38672 TextGen respects multiple output models and few GTFs` (commit
  `f2932f6b232c`, first in 2025.3.0), TextGen reports a warning `no output location for <model>, skipped` and a warning
  `Make could not save files generated from <module>. Does this module define generation facets?`, and skips the model.

`GenerationTargetFacet.find(SModel)` (`core/project/source/jetbrains/mps/project/facets/GenerationTargetFacet.java`,
static, present in all three branches) returns the facet TextGen would write through, or `null`. It skips a `TestsFacet`
for a model without the `tests` stereotype, as TextGen does. A caller that wants to leave such models out of the make
can filter on `GenerationTargetFacet.find(model) != null` before `ModelsToResources` (Inferred: TextGen's own facet
lookup in `ModuleStaleFileManager.getGenerationTargetFacet` differs in detail, for example it honors an explicit
generation target, but for a module with no `GenerationTargetFacet` both return none).

There is no module-level "do not generate" setting. The only per-model switch is `GeneratableSModel.isDoNotGenerate()`,
which `DefaultSModelDescriptor.isGeneratable()` negates.

## No used language

`ModulesClusterizer.clusterize`
(`core/make-runtime/solutions/jetbrains.mps.make/source_gen/jetbrains/mps/make/dependencies/ModulesClusterizer.java`)
drops every `MResource` whose models all have no used languages (`ModelContentUtil.getUsedLanguages`) and no imported
languages or devkits. An empty, newly created model is such a model. The MPS source refers to this as the MPS-26995
case. (Verified, same code in all three branches.)

When nothing else is in the input, `MakeSequence.steps()` is `0`. `CoreMakeTask.run` sets its result to
`IResult.FAILURE("not started", null)` before `doRun`, and `doRun` returns at once when there are no steps, without a
message to the handler. So `getResult()` is a failure and the handler saw nothing (Verified; Observed on 2024.1.6).
`IResult.message()` (since 2023.2) returns `not started`.

A caller can build the `MakeSequence` first and treat `steps() == 0` as "nothing to make" instead of running the task.
`MakeSequence` clusterizes in its constructor, inside its own read action.
