# MPS API research

This repository contains documentation produced by coding agents while researching MPS APIs and internals using the
included [mps-api-research](.agents/skills/mps-api-research/SKILL.md) agent skill.

The skill describes how to find information about MPS APIs and encourages agents to contribute to this repository.

Each API or topic has its own Markdown file. Each note records the exact class and signature, which artifact and jar
provide it, the source file and MPS version used to verify its behavior, the resulting contract, and the gotchas that
make the API easy to misuse.

## Usage

Clone the repository, then link (or copy) its skill to your agent's skills directory:

```bash
mkdir -p ~/.agents/skills
ln -s "$PWD/.agents/skills/mps-api-research" ~/.agents/skills/mps-api-research
```

or:

```bash
mkdir -p ~/.claude/skills
ln -s "$PWD/.agents/skills/mps-api-research" ~/.claude/skills/mps-api-research
```

## Markdown checks

Install [prek](https://prek.j178.dev/) and run `prek install` once after cloning. The pre-commit hook formats Markdown
with Prettier at 120 characters and checks it with markdownlint-cli2. Run the checks manually with
`prek run --all-files`.

## License

The research documents under [`docs/`](docs/) are dedicated to the public domain under
[CC0 1.0 Universal](docs/LICENSE). The rest of the repository, including the
[mps-api-research agent skill](.agents/skills/mps-api-research/SKILL.md), is licensed under the [MIT License](LICENSE).

Third-party material remains subject to its original license. Contributions are licensed according to their location;
see [CONTRIBUTING.md](CONTRIBUTING.md).
