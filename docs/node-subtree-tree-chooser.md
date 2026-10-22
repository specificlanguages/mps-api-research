# Tree of a node's subtree for choosing a node

Which MPS UI classes can show a tree rooted at an arbitrary node, limited to selected concepts, and let the user pick
one node? Verified against MPS sources for 2025.1.4 and the `master` branch after 2026.2 (platform 263.6259); the
relevant classes are identical in 2024.1 except for one unrelated popup-place change in `MPSTree`.

## Short answer

MPS has no ready-made, configurable "choose a node from this subtree" dialog. It has a generic Swing tree framework that
makes such a dialog short to write:

- `jetbrains.mps.ide.ui.tree.MPSTree` (abstract tree, you implement `rebuild()`);
- `jetbrains.mps.ide.ui.tree.smodel.SNodeTreeNode` (lazy tree node for one `SNode`);
- `SNodeTreeNode.NodeChildrenProvider` (hook that decides the children of every `SNodeTreeNode` in the tree).

The MPS Node Explorer (`jetbrains.mps.ide.devkit.components.NodeExplorerComponent`) is a complete in-tree example of a
tree rooted at one node.

The `com.mbeddr.mpsutil.targetchooser` chooser from mbeddr (`TargetChooserDialog`) cannot start at a node. Its tree
always has the project, modules, and models above root nodes.

## mbeddr TargetChooserDialog

Verified in generated sources of mbeddr.core branch for MPS 2025.1
(`code/platform/com.mbeddr.mpsutil/solutions/com.mbeddr.mpsutil.targetchooser/source_gen/...`):

- `TargetChooser_ProjectTree extends ProjectTree`. Its `rebuild()` always creates a `ProjectTreeNode` and adds module
  nodes from `TargetChooserScope.getModules(project)` under it.
- `TargetChooserScope` has four methods: `getModules(Project)`, `showModel(SModel)`, `getRootNodes(SModel)`, and
  `getChildNodes(SNode)`. A custom scope can reduce the tree to one module, one model, and one root, but the project,
  module, and model levels remain visible.
- `TargetChooserOptions.setScope(Iterable<SNode>, SNodeFilter)` creates an `SNodesScope`. It shows only the given nodes
  (plus ancestors accepted by the ancestor filter), still under their project, module, and model.
- `TargetChooserOptions.setInitial(SNode)` only preselects a node; it does not change the tree root.
- `TargetChooserDialog.chooseTarget(project, options)` returns a `SelectedTarget`; `SelectionValidator` decides which
  selection enables OK.

Use it when "project → model → node" context is acceptable. Do not use it to show a tree that begins at a node.

## MPS tree framework contract

All in `workbench/mps-ui/source/jetbrains/mps/ide/ui/tree/` (MPS UI module, `mps-workbench` plugin).

`MPSTree` (`MPSTree.java`):

- `protected abstract MPSTreeNode rebuild()` (line 456) returns the root tree node.
- `rebuildNow()` (line 605) must run on the EDT; it throws otherwise.
- `runRebuildAction(Runnable, boolean)` (line 565) does nothing when `RuntimeFlags.isTestMode()`, in a headless
  application, or after disposal (line 566). A tree therefore stays empty in headless tests.
- The tree does not take a model read lock. To read nodes during `rebuild()` and lazy expansion, override both
  `runRebuildAction` and `doInit(MPSTreeNode, Runnable)` (line 201) and wrap the runnable in
  `jetbrains.mps.smodel.ModelReadRunnable`. The Javadoc at line 551 recommends this.
- `MPSTree` implements `Disposable`; call `dispose()` when the owner closes.
- Double click and F4 call `doubleClick(MPSTreeNode)`; override it to confirm a choice.

`SNodeTreeNode` (`smodel/SNodeTreeNode.java`):

- Constructors `SNodeTreeNode(SNode)`, `SNodeTreeNode(SNode, String role)`, and
  `SNodeTreeNode(SNode, String role, Condition<SNode>)` (line 53). The text is `role : presentation`; the icon comes
  from `GlobalIconManager`. The constructor reads the node, so call it under model read.
- Children are created lazily in `doInit()`. If an ancestor implements `NodeChildrenProvider`, `doInit()` calls its
  `populate(this)` (line 167). Otherwise it adds the direct children that pass the `Condition` (line 169). The
  `Condition` prunes a non-matching child together with its whole subtree, so it cannot show matching grandchildren
  below a non-matching child.
- `MPSTreeNode.getAncestor(Class)` checks parent tree nodes first, then the owning `MPSTree` (`MPSTreeNode.java` line
  117). An `MPSTree` that implements `NodeChildrenProvider` therefore controls the children of every `SNodeTreeNode` in
  it.
- With a provider, an unexpanded node reports `isLeaf() == !provider.isShowMembers()` (line 147). With the default
  `isShowMembers() == true`, every node shows an expand handle until it is expanded, even when it has no children.
- `getSNode()` returns the node for the selected tree node.

## Recipe

`NodeExplorerComponent` (`plugins/mps-devkit/source/jetbrains/mps/ide/devkit/components/NodeExplorerComponent.java`,
lines 94–145) uses this pattern. Adapted for a chooser that shows only nodes of certain concepts and skips the nodes in
between:

```java
class SubtreeChooserTree extends MPSTree implements SNodeTreeNode.NodeChildrenProvider {
  private final ModelAccess myModelAccess;
  private final SNode myRoot;
  private final Predicate<SNode> myShown;

  SubtreeChooserTree(ModelAccess modelAccess, SNode root, Predicate<SNode> shown) {
    myModelAccess = modelAccess;
    myRoot = root;
    myShown = shown;
  }

  @Override
  protected void runRebuildAction(Runnable rebuildAction, boolean saveExpansion) {
    super.runRebuildAction(new ModelReadRunnable(myModelAccess, rebuildAction), saveExpansion);
  }

  @Override
  protected void doInit(MPSTreeNode node, Runnable nodeInitRunnable) {
    super.doInit(node, new ModelReadRunnable(myModelAccess, nodeInitRunnable));
  }

  @Override
  protected MPSTreeNode rebuild() {
    setRootVisible(false);
    TextTreeNode invisibleRoot = new TextTreeNode("root");
    invisibleRoot.add(new SNodeTreeNode(myRoot));
    return invisibleRoot;
  }

  @Override
  public void populate(SNodeTreeNode treeNode) {
    addNearestShown(treeNode, treeNode.getSNode());
  }

  // Adds the closest descendants accepted by myShown; nodes in between are not shown.
  private void addNearestShown(SNodeTreeNode treeNode, SNode parent) {
    for (SNode child : parent.getChildren()) {
      if (myShown.test(child)) {
        treeNode.add(new SNodeTreeNode(child, child.getContainmentLink().getName()));
      } else {
        addNearestShown(treeNode, child);
      }
    }
  }

  @Nullable
  SNode getSelectedNode() {
    TreePath path = getSelectionPath();
    Object last = path == null ? null : path.getLastPathComponent();
    return last instanceof SNodeTreeNode ? ((SNodeTreeNode) last).getSNode() : null;
  }
}
```

Put the tree in a `com.intellij.openapi.ui.DialogWrapper`: return `ScrollPaneFactory.createScrollPane(tree)` from
`createCenterPanel()`, call `tree.rebuildNow()` on the EDT, check the selected concept in `doValidate()`, and dispose
the tree in `dispose()`. Read `getSelectedNode()` under model read if you use more than the reference.

## Other candidates checked

- `jetbrains.mps.ide.refactoring.NodeHierarchyChooser` (`plugins/mps-rcp/modules/ide.refactoring/...`) shows the concept
  inheritance hierarchy only.
- `jetbrains.mps.ide.platform.dialogs.choosers.NodeChooserDialog` and `CommonChoosers` are list choosers, not trees.
  _Inferred_ from class names and location; not read in detail.
- MPS-extensions `com.mbeddr.mpsutil.projectview` defines custom project views in a DSL (`ProjectView`,
  `TreeNodeElement`). `CustomProjectTree` builds its root from a registered `ViewId`, so it is a tool-window view, not a
  chooser rooted at a given node. _Inferred_; the language was not examined in detail.

## Evidence strength

- Verified: all statements about `MPSTree`, `MPSTreeNode`, `SNodeTreeNode`, `NodeExplorerComponent`, and the mbeddr
  target chooser classes, from the sources named above.
- Inferred: the recipe works as written. It follows `NodeExplorerComponent` but was not compiled or run.
- Unresolved: whether `SNodeTreeNode` presentation updates after the initial build always happen under model read when
  the tree is used outside a project view.
