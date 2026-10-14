# Containment traversal with SNodeOperations

Verified against the MPS 2026.1.1 source tag: how can callers obtain descendants and ancestors as lists?

## API and semantics

`jetbrains.mps.lang.smodel.generator.smodelAdapter.SNodeOperations` provides these static methods in `mps-core.jar`:

```java
List<SNode> getNodeDescendants(SNode node, SAbstractConcept childConcept, boolean inclusion)
List<SNode> getNodeAncestors(SNode node, SAbstractConcept ancestorConcept, boolean inclusion)
```

Pass `null` for the concept to accept all concepts and `false` for inclusion to exclude the receiver. Descendants follow
containment in depth-first pre-order and native sibling order. Ancestors run from the parent to the root. Neither
operation follows references. A null receiver returns an empty list.

```text
getNodeDescendants(node, null, false)
  SNodeUtil.getDescendants(node, null, false)
    DescendantsTreeIterator
  IterableUtil.asList

getNodeAncestors(node, null, false)
  node.getParent()
  repeat getParent() until null
```

The results contain native nodes and do not extend their validity. Ancestors are collected into a new ArrayList;
descendants are materialized with IterableUtil.asList. Callers requiring immutability should wrap the result. Traversal
reads native containment; callers must hold the model access required by their nodes. These methods do not establish a
read action.

## Evidence

Verified source at tag `2026.1.1`:

- `core/kernel/smodelRuntime/source_gen/jetbrains/mps/lang/smodel/generator/smodelAdapter/SNodeOperations.java`:
  `getNodeAncestors` (line 158), `getNodeDescendants` (line 190).
- `core/openapi/source/org/jetbrains/mps/openapi/model/SNodeUtil.java`: the three-argument `getDescendants` (line 129).
- `core/openapi/source/org/jetbrains/mps/util/DescendantsTreeIterator.java`: pre-order contract and `next`
  implementation.

Observed in an MPS 2026.1.1 Groovy execution: sibling and grandchild ordering, exclusion of the receiver and reference
targets, empty leaf/root results, and list membership retained across a subsequent containment edit.

Other MPS versions were not investigated.
