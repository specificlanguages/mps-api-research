# Reading serialized MPS models

MPS `.mps` model files are XML. Inspect the model rather than guessing concrete node IDs, roles, or the node that
triggers a rule: these identifiers are opaque and may vary between fixtures.

Read the `<registry>` block first. It maps the short concept, property, and link indices used in node bodies, such as
`1TJDcQ`, to their human-readable names.

MPS supports several node-ID forms. A common regular node ID is numeric and unique within a model. It is persisted using
MPS's Java-friendly Base64 encoding, while user-visible values such as node references may contain the decoded decimal
form. Confirm the implementation in the source version being researched; in MPS 2026-era sources it is located under
`core/smodel/source/jetbrains/mps/smodel/JavaFriendlyBase64.java`.

Foreign IDs are another form and are commonly, but not exclusively, used for Java stubs. Determine the ID kind from the
serialized data and matching MPS implementation rather than inferring it from where the node appears.
