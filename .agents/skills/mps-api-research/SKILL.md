---
name: mps-api-research
description:
  Research a JetBrains MPS internal API instead of guessing at it — confirm a class or method's exact signature or its
  runtime behavior against the local MPS jars and source. Use when writing or debugging code that calls MPS internals
  and you're unsure how an API is shaped or behaves.
---

# Researching MPS APIs

MPS internals are undocumented and easy to guess wrong. Confirm any non-trivial MPS API against a source of truth before
you call it.

Before you start researching, search the existing notes in
[`specificlanguages/mps-api-research`](https://github.com/specificlanguages/mps-api-research) for the class or topic.
Contribute new findings and corrections as pull requests, see the
[corresponding section](#step-6-contribute-the-finding) below.

## Step 1: Determine versions

Determine the MPS versions you are researching. If a project supports a particular version of MPS, it may still be
worthwhile researching subsequent versions for any API changes and deprecations.

## Step 2: Obtain source code

Prefer a local checkout/worktree of MPS when one is available and matches the version(s) supported by the project. If
not available, ask the user for permission to check it out. As a last resort, inspect the JetBrains/MPS repository on
GitHub.

Using a source checkout is preferable because running its Ant build will download the exact patched sources of IntelliJ
Platform that are used by a particular version of MPS.

Another possibility for research is the compiled binaries that may be present in the local Maven/Gradle cache.

[MPS documentation](https://www.jetbrains.com/help/mps/) may prove useful for an overview and to understand where to
look, but it does not describe the API in the necessary detail.

## Step 3: Build the sources

Some research may not need to build the project and it may be enough to just examine the checked out sources. Often,
however, you may need to access the dependencies such as the IDEA platform or jars in the `lib` directory. In this case
you may need to ensure that the project build was run.

A clean checkout of MPS sources will lack the `lib` directory. This directory is downloaded as part of the build. MPS
uses Ant for building, consult README.md in the checkout directory. In general, a full build of MPS is triggered by
running `ant -f build/build.xml` in the checkout root.

## Step 4: Examine

The exact exploration outcome is left to your judgement. Useful information to gather includes, but is not limited to:

- which JAR (in lib/) or plugin the API is part of,
- the exact method declaration, including fully qualified class name, method name, parameters, their types and
  nullability, return values, exceptions,
- whether the API must, must NOT, or should NOT, be called from an event dispatch thread (EDT),
- whether the API requires a read lock, write lock, no lock present, or is locking-agnostic. There is also a difference
  between MPS lock and IDEA lock, specify which is required,
- maturity status (new, stable, deprecated) and changes or differences between MPS versions,
- relevant lifecycle requirements: what needs to be initialized before the API can be called, how long do the returned
  objects remain valid, ownership, etc. - use your judgment.

Remember to record the evidence: source paths, line numbers, identifiers, etc.

**Distinguish confirmed requirements from properties merely not observed; absence of an assertion or annotation is not
evidence that no runtime constraint exists.** Construct and run a probe to confirm runtime properties if necessary.

## Reading `.mps` files

`.mps` models are XML — both the aspect definitions above and the fixtures you probe for concrete test data.

- The `<registry>` block maps each concept/property/link **index** (short codes like `1TJDcQ`) to its human name. Read
  it first; it decodes every node body below.
- There are several kinds of node identifiers. A common case is a numeric ("regular") ID, unique within a model. It is
  persisted in the .mps file as "Java-friendly Base64", and an encoder/decoder is present in the MPS sources
  (`core/smodel/source/jetbrains/mps/smodel/JavaFriendlyBase64.java` as of September 2026), but user-visible strings
  such as node references may contain the ID in the decoded, decimal form. There are other kinds of IDs, such as
  "foreign IDs" commonly (but not exclusively) used for Java stubs.
- For concrete data (node ids, roles, which node trips a rule), grep the model rather than assuming — the ids are opaque
  and unguessable.

## Step 5: Review

If available, optionally ask another agent (with a fresh context) to review the document you produced.

Verify that the note:

- answers the API question stated by its title and introduction,
- distinguishes verified facts from assumptions or unresolved questions,
- identifies the applicable MPS version(s),
- records enough evidence for another reader to verify the conclusions,
- describes relevant signatures, behavior, and runtime constraints,
- notes meaningful differences between researched versions (if multiple versions were researched),
- contains no local machine paths or project-specific terminology.

## Step 6: Contribute the finding

Contribute corrections and new findings to
[`specificlanguages/mps-api-research`](https://github.com/specificlanguages/mps-api-research) by forking the repository
if necessary. Add or update the focused topic note and match the repository's existing structure.

Forking, pushing a branch, opening a pull request, or otherwise changing the remote repository still requires the user's
authorization. Without that authorization, prepare the documentation change locally and tell the user it is ready to
contribute.

Remember to make the note reusable:

- **Portable coordinates:** Cite the MPS version (e.g. 2025.1.2) or the build number for pre-release versions (e.g.
  251.1234.56), name the JARs, and give source locations as repo-root-relative paths or `JetBrains/MPS` GitHub links. Do
  not use Maven artifact coordinates.

- **Project-agnostic:** Write the note as standalone MPS documentation. The research was likely motivated by the needs
  of some project but the note must make sense to a reader who has never heard of the project and/or has no access to
  it. Do not include the name of the project or the name of its components or refer to its architecture, and the like.
