# Editor test fixtures in MPS 2025.1.4

This note answers how a native EditorTestCase obtains its editor and what its automatic assertions check. The contract
below is verified from the MPS 2025.1.4 source tag; execution through an external headless launcher is not established
by this source inspection.

## Fixture and editor lifecycle

`jetbrains.mps.lang.test.structure.EditorTestCase` has `testNodeBefore` and `testNodeResult` children of concept
`TestNode`, and a `code` child of BaseLanguage `StatementList`. `TestNode.nodeToCheck` contains the tested language
node. The structure declaration is in `plugins/mps-testing/languages/lang.test/languageModels/structure.mps`.

The generated runtime class `jetbrains.mps.lang.test.runtime.BaseEditorTestBody` owns the editor lifecycle:

```text
initEditorComponent(before, after)
  runInUIThreadAndWait
    doInitEditor
      project model write action
        prepareTestNodes
        find before AnonymousCellAnnotation -> initial CellReference
        openEditor, showNode, size/validate component, setupSelection
  flushEDTEvents

testMethod
  install log appender
  testMethodImpl
  checkAssertion
    UI thread + project model write action
      compare nodes if result exists
      compare selection if result has a cell reference
  dispose
  fail on captured unclean log events
  finally uninstall appender and clear undo queue
```

`doInitEditor` throws when no cell reference is found in the before fixture. `findCellReference` searches for
`AnonymousCellAnnotation` and uses its parent as the referenced node. The annotation's cell and selection properties
must correspond to the actual editor.

The protected method `getEditorComponent()` returns `jetbrains.mps.nodeEditor.EditorComponent`. Use the native
`EditorComponentExpression` when authoring native test code. Do not infer that arbitrary Java parsing contexts expose
generated inherited methods or native test expressions.

## Automatic assertions and limits

`checkAssertion()` uses `NodesMatcher` when an expected result exists, then checks selection if a finish reference
exists. It does not compare rendered pixels. A layout or painting test needs its own observable assertion.

The before fixture can therefore support explicit editor-state assertions without an expected result tree; the runtime
accepts an empty after identifier. This does not remove the initial cell-reference requirement.

Initialization and automatic assertions explicitly dispatch to the UI thread with model write access. This note does not
establish the execution context of every user-authored statement: inspect generator output and relevant helpers before
calling editor APIs with additional threading requirements.

## Evidence

All paths are relative to JetBrains/MPS at tag `2025.1.4`:

- `plugins/mps-testing/languages/lang.test/languageModels/structure.mps`: EditorTestCase, TestNode,
  AnonymousCellAnnotation and EditorComponentExpression declarations.
- `plugins/mps-testing/languages/lang.test/solutions/jetbrains.mps.lang.test.runtime/source_gen/jetbrains/mps/lang/test/runtime/BaseEditorTestBody.java`:
  `initEditorComponent`, `doInitEditor`, `findCellReference`, `checkAssertion`, `testMethod`, `getEditorComponent`.
- `testbench/testsolutions/quotation.test/models/editorTest@tests.mps`: native editor-test examples.

Unresolved: headless-launcher compatibility for arbitrary editor tests, APIs beyond this fixture contract, and
differences in other MPS releases. See [test execution](test-execution.md) for test discovery and session lifecycle.
