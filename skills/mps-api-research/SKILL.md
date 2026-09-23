---
name: mps-api-research
description:
  Research a JetBrains MPS internal API instead of guessing at it. Confirm exact signatures, behavior, runtime
  constraints, and version differences against MPS sources, binaries, or focused probes. Use when writing or debugging
  code that calls MPS internals and the API contract is uncertain.
---

# Researching MPS APIs

MPS internals are easy to guess wrong. Answer the immediate API question from verifiable evidence and preserve reusable
findings when appropriate.

## Start with existing research

Search the notes in [`specificlanguages/mps-api-research`](https://github.com/specificlanguages/mps-api-research) for
the class, method, concept, or behavior in question.

- If a note fully answers the question for the relevant MPS version, report the answer and link the note.
- If it is incomplete or covers different versions, investigate only the missing or potentially outdated parts.

## Scope the investigation

Identify the MPS version used by the requesting project or artifact. Research additional versions only when the request
involves compatibility, migration, deprecation, or determining when behavior changed. Use a build number for a
pre-release version when no release version identifies it precisely.

State the concrete question before exploring. Prefer questions that can be answered from evidence, such as an exact
signature, caller responsibility, lifecycle constraint, or behavior under a particular condition.

## Choose the strongest practical evidence

Use the cheapest authoritative source that can answer the question. A typical order is:

1. A matching local MPS source checkout or worktree.
2. Patched IntelliJ Platform sources obtained by that MPS checkout's build.
3. Compiled MPS or platform binaries from a distribution or dependency cache.
4. The matching revision of the upstream MPS or IntelliJ Platform repository.
5. Published documentation for orientation and terminology.

Before downloading sources or decompiling binaries, search available local project directories and sibling worktrees for
a matching checkout. Prefer source inspection over decompilation when both represent the same build.

A clean MPS source checkout may not contain `lib/`. If dependencies are necessary, consult the checkout's `README.md`
and build configuration. Do not run a full build merely to answer a question already settled by available sources.

## Establish the contract

Gather only the facts relevant to the question. These commonly include:

- fully qualified declarations, parameter and return types, nullability, exceptions, and artifact or plugin location;
- behavior and side effects, including meaningful callers and implementations;
- EDT or background-thread requirements;
- MPS model-access or IntelliJ lock requirements;
- initialization, disposal, ownership, and validity lifetime;
- stability, deprecation, and differences between relevant versions.

Record evidence as you work: version or build number, repository-relative source paths, identifiers, and line numbers or
other stable anchors.

Label conclusions by evidence strength:

- **Verified:** directly established by matching source or bytecode.
- **Observed:** reproduced by a focused runtime probe.
- **Inferred:** supported by evidence but not directly guaranteed.
- **Unresolved:** evidence is insufficient or conflicting.

Absence of an assertion, annotation, or observed failure does not prove the absence of a runtime constraint.

## Probe only unresolved behavior

Use a focused runtime probe when static evidence cannot establish an important property. Define the claim being tested,
hold the MPS version and relevant environment constant, and record the invocation, fixture, and observation needed to
reproduce the result. Do not generalize beyond what the probe demonstrates.

When the investigation requires reading serialized MPS models, read
[references/mps-model-files.md](references/mps-model-files.md).

## Preserve reusable findings

When the investigation establishes a reusable fact not already documented for the relevant version, create or update a
focused note under `docs/` in a writable checkout of the specificlanguages/mps-api-research repository (where this skill
is likely linked from). Read [references/research-notes.md](references/research-notes.md) before writing or reviewing a
note.

Do not create a note when:

- the user requested read-only investigation;
- the checkout is outside the permitted task or filesystem scope;
- the finding is specific to the originating project; or
- the evidence is too incomplete to support a useful verified conclusion.

After establishing reusable findings, you MUST prepare a research note. The destination repository is
specificlanguages/mps-api-research, not the requesting project. If that repository is not writable, ask once for
permission to prepare a writable checkout. Filesystem restrictions change where you prepare the note; they do not waive
the documentation requirement. If permission is unavailable, include the complete ready-to-save note in your final
response.

## Finish

Report the answer, applicable MPS versions, evidence strength, unresolved questions, and any note created or updated. Do
not commit, push, fork, open a pull request, or otherwise change remote state without authorization.
