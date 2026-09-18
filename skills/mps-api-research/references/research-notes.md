# Research note standards

A research note is standalone, project-agnostic MPS documentation. Its title and introduction must state the API
question it answers and the MPS versions to which the answer applies.

## Useful content

Choose headings that fit the subject rather than forcing every note into one template. A useful note normally makes the
following easy to find:

- the concise answer;
- applicable release versions or pre-release build numbers;
- exact API declarations and artifact or plugin locations;
- behavior, side effects, lifecycle, threading, and locking constraints relevant to the question;
- differences between researched versions;
- evidence and enough coordinates for another reader to verify it;
- inferences and unresolved questions, clearly distinguished from verified or observed facts.

Use a small diagram, table, call tree, or file tree only when it communicates control flow, lifecycle, ownership, or
version differences more clearly than prose.

## Portable evidence

Cite MPS release versions or build numbers and give repository-root-relative source paths or `JetBrains/MPS` links. Name
relevant JARs, distributions, or plugins. Do not include absolute local paths or use Maven artifact coordinates as a
substitute for identifying the MPS distribution component that contains the API.

Keep examples and terminology independent of the project that motivated the research. Do not name its components,
describe its architecture, or assume that readers can access it.

Substantial third-party text, code, images, or other material must have a compatible license and its origin must be
identified. Source references and short excerpts used as technical evidence do not require a separate license notice in
each note. Contributions under `docs/` are dedicated under CC0 1.0 Universal as described in `docs/LICENSE`.

## Review checklist

Verify that the note:

- answers the question stated by its title and introduction;
- identifies applicable MPS versions;
- distinguishes verified, observed, inferred, and unresolved claims;
- records sufficient evidence to reproduce or verify its conclusions;
- covers relevant API signatures and runtime constraints;
- records meaningful version differences when multiple versions were researched;
- contains no local machine paths or project-specific terminology; and
- passes the repository's Markdown checks.
