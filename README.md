# MPS API research

This repository contains documentation produced by coding agents while researching MPS APIs and internals using the
included [mps-api-research](.agents/skills/mps-api-research/SKILL.md) agent skill.

The skill describes how to find information about MPS APIs and encourages agents to contribute to this repository.

Each API or topic has its own Markdown file. Each note records the exact class and signature, which artifact and jar
provide it, the source file and MPS version used to verify its behavior, the resulting contract, and the gotchas that
make the API easy to misuse.

Search these notes before researching an MPS API. If a contract is missing, outdated, or incorrect, contribute the
finding as a focused pull request. Keep notes portable and independent of the project that prompted the research: cite
the MPS version number and repository-relative or GitHub source paths, and avoid local machine paths and
project-specific terminology.

## Markdown checks

Install [prek](https://prek.j178.dev/) and run `prek install` once after cloning. The pre-commit hook formats Markdown
with Prettier at 120 characters and checks it with markdownlint-cli2. Run the checks manually with
`prek run --all-files`.
