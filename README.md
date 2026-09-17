# MPS API research

This repository contains documentation produced by coding agents while researching MPS APIs and internals using the
included [mps-api-research](skills/mps-api-research/SKILL.md) agent skill.

The skill describes how to find information about MPS APIs and instructs agents to preserve reusable findings in this
repository.

Each API or topic has its own Markdown file. Each note records the exact class and signature, which artifact and jar
provide it, the source file and MPS version used to verify its behavior, the resulting contract, and the gotchas that
make the API easy to misuse.

## Usage

Clone the repository, then link (or copy) its skill to your agent's skills directory:

```bash
mkdir -p ~/.agents/skills
ln -s "$PWD/skills/mps-api-research" ~/.agents/skills/mps-api-research
```

or:

```bash
mkdir -p ~/.claude/skills
ln -s "$PWD/skills/mps-api-research" ~/.claude/skills/mps-api-research
```

## License

The research documents under [`docs/`](docs/) are dedicated to the public domain under
[CC0 1.0 Universal](docs/LICENSE). The rest of the repository, including the
[mps-api-research agent skill](skills/mps-api-research/SKILL.md), is licensed under the [MIT License](LICENSE).

Third-party material remains subject to its original license. Contributions are licensed according to their location;
see [CONTRIBUTING.md](CONTRIBUTING.md).

JetBrains-authored MPS and IntelliJ Platform source referenced by the research documents is licensed under the
[Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0), unless the referenced source states otherwise.
